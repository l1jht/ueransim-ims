# ARCHITECTURE.md — 模块设计与信令流程

## 模块

```
src/ue/ims/
├── ims.cpp/hpp        ImsClientTask（NtsTask 子类）
│                       ├── 会话绑定（apn=="ims" 的 PduSession，pduAddress 作源 IP/Contact）
│                       ├── 注册状态机（IDLE/REGISTERING/REGISTERED/FAILED）
│                       ├── 呼叫状态机（IDLE/CALLING/RINGING/ANSWERING/CONFIRMED/HANGING_UP）
│                       ├── CLI 分发（ims-* 命令）
│                       └── 定时器（注册超时/退避/重注册/事务重传/呼叫超时）
├── sip_stack.cpp/hpp  轻量 SIP 协议栈（纯文本编解码，不碰字节层）
│                       ├── 构造：BuildRegister / BuildInvite / BuildInDialogRequest / BuildSipReply
│                       ├── 解析：ParseSipResponse / ParseSipRequest / ExtractRoutes
│                       └── PlaceholderSdp
├── packet.cpp/hpp     IPv4/UDP 传输层
│                       ├── BuildUdpPacket（IP 头校验和 + UDP 校验和含伪头，RFC 768）
│                       ├── IsUdpPacket（下行 SIP 判定：IPv4+UDP+目的端口）
│                       └── ExtractUdpPayload
├── auth.cpp/hpp       DigestAuth（有状态挑战对象）
│                       ├── beginChallenge（校验 nonce 与 qop 列表含 auth，重置 nc/生成 cnonce）
│                       └── authorization(method, uri) → 完整 Authorization 头值
├── transaction.cpp/hpp SipTransactionTable（纯逻辑事务表，键 = Call-ID+CSeq）
│                       ├── arm/cancel/cancelAll/tick（tick 返回本轮应重发的 body 列表）
│                       └── 容量 8 满淘汰最旧；每事务独立 4 次重传计数
├── digest.cpp/hpp     MD5（RFC 1321）+ DigestResponse（RFC 2617）纯函数库
└── config.hpp         ImsConfig（ue.yaml `ims:` 段，含 enable 开关）
```

设计取舍：

- **纯函数与有状态分离**：digest（纯）→ auth（有状态）→ sip_stack（纯文本）→ packet（字节层）→ ims（编排）。越靠下越可单测、可复用
- **校验和独立成模块**：IPv4/UDP 封包在 `packet` 内自含，测试按 RFC 768 参考逐字节验证——历史上奇数长度 payload 校验和 bug 即藏于此
- **鉴权状态每路独立**：注册机与呼叫机各持一个 `DigestAuth`，互不串扰（旧实现曾共享 cnonce）
- **重传事务表化**（2026-08-17）：INVITE/BYE/MESSAGE 的重传生命周期由 `SipTransactionTable` 按 Call-ID+CSeq 键控，单一 500ms 扫描定时器驱动。**修复两个缺陷**：① 旧单槽位 `m_pendingTxRequest` 被后发事务覆盖（INVITE 重传窗口内发 SMS 会丢 INVITE 重传）；② SMS 响应只按 Call-ID 匹配（所有 SMS 复用同一 Call-ID），迟到的旧响应会 cancelTx 杀掉其他事务的重传。现：响应只取消自身事务；SMS 分支补 CSeq 校验，错配迟到响应 debug 日志后丢弃。注册不入表（保持 TIMER_REG_TIMEOUT 机制）
- **1xx 停重传（RFC 3261 §17.1.1，2026-08-17）**：事务条目携带 method（`isInvite()` 查询）；取消谓词 = `≥200 最终响应 || INVITE 事务的任意响应`——INVITE 收 1xx 进入 proceeding 态停 Timer A（与真 UE 一致），BYE/MESSAGE 仍等最终响应。200 丢失恢复由被叫侧 Timer G 端到端覆盖（旧"180 后继续重传 INVITE"是冗余恢复路径，已移除）
- **MESSAGE UAS 去重**（2026-08-17，弱网测试发现的既有缺陷）：被叫 200 丢失时短信中心会重传（同 Call-ID+CSeq 的 uac 事务重传）或重投递（SMS_WORKER 轮询的新事务、同 X-MSG-ID），旧实现两种都重复入 history。现：键缓存（16 条）= 有 X-MSG-ID 用 `id:<msgId>`、否则 `<callId>:<cseq>`；命中则重发 200（中心需要它删记录）但不重复处理（RFC 3261 §17.2.2 非 INVITE 服务端事务语义）。ParseSipRequest 增加 X-MSG-ID 提取
- **SMS TPDU 单模式**（2026-08-19）：SMS over IMS 仅用 TS 24.341 标准封装——上行 MESSAGE body = RP-DATA(MS→net) + SMS-SUBMIT TPDU，Content-Type `application/vnd.3gpp.sms`；下行解析 RP-DATA + SMS-DELIVER（TP-OA/TP-DCS/**TP-SCTS**/TP-UD）与 RP-ACK（仅回 200、不入 history），不可解析回 200 + RP-ERROR（cause 95），非 3gpp.sms 内容类型回 **415**。上行应答接受任意 2xx（smsc 3GPP 路径回 202）。**编码统一 UCS2**（DCS=0x08，容量 70 字符）：kamailio smsops 的 gsm7bit_codes 表在 septet 0x2D 错位（0xAD 软连字符而非 '-'），7-bit 路径对其不可靠——UCS2 是 TS 23.038 强制支持的编码、实测全链通过。接收侧保留 7-bit/UCS2 双解码（标准兼容）。**OTT text/plain 路径已移除**（此前为开发过渡形态）；Contact 注册 `+g.3gpp.smsip` 特性标签（TS 24.341）使短信中心走 3GPP TPDU 投递。TPDU 模块为纯函数（sms_tpdu.cpp，字节层编解码 + 7-bit/UCS2/半八位）

## 信令流程

### 注册

```
UE ──REGISTER（无鉴权）──► P-CSCF ──► I-CSCF ──UAR/UAA──► S-CSCF
S-CSCF ──401（WWW-Authenticate: Digest, nonce, qop="auth[,auth-int]"）──► UE
UE ──REGISTER（Authorization: Digest response=..., qop=auth, cnonce, nc）──► ... ──► S-CSCF
S-CSCF ──200 OK（Service-Route, Expires）──► UE
```

- 客户端固定回 `qop=auth`（kamailio ims_auth 实测 REGISTER 挑战 `qop="auth,auth-int"`、INVITE 挑战 `qop="auth"`）
- `Service-Route` 缓存，供呼叫的 INVITE 第二条 Route 使用
- expires 跟随 200 OK，按 90% 提前重注册；失败退避 5/15/45/60s 封顶
- 200 OK 重传（同 CSeq）去重，防止重注册定时器堆积产生 4s 注册循环；60s 内重复重注册触发忽略
- **P-CSCF 来源（P4）**：EPCO 下发优先（`PduSession.epcoPcscf`，建立请求 PCO 带 0x000C → SMF 回写），配置 `ims.pcscf` 兜底

### 鉴权（P1 Digest-MD5 / P4 AKAv1-MD5）

- **Digest-MD5**（P1）：qop=auth，password = Ki
- **AKAv1-MD5**（P4）：`setAkaCredentials(OPc, AMF)`（OPc 复用 NAS 层 op/key 派生）；nonce = **Base64(RAND‖AUTN)**（兼容 64hex）；AUTN 校验（SEP-bit + MAC-A）；**digest password = XRES 原始字节**（kamailio ims_auth 实测，非 RFC 3310 RK）
- 模式切换：scscf 配置 `REG_AUTH_DEFAULT_ALG`（testvonr 挂载源 `docker-open5gs/scscf/scscf.cfg:38`）——AKAv1-MD5 或 MD5，UE 自动跟随 challenge algorithm

### 主叫

```
ims-call <uri>
INVITE（Route: <P-CSCF;lr> + Route: <Service-Route;lr>，占位 SDP）
  → 100/401（带鉴权重发）→ 180 → 200 OK → ACK → CONFIRMED
ims-hangup → BYE → 200 → IDLE
```

- **in-dialog 请求（ACK/BYE）Route = 200 OK 的 Record-Route 原序**（S-CSCF 需 `did/rm` 参数匹配 dialog；Service-Route 无 did 会被丢弃）
- 非 2xx 最终响应（404/500）回 **Error ACK**（RFC 3261 §17.1.1.3，无 Route/Contact），止 kamailio 32s 重传

### 被叫（自动应答）

```
INVITE 到达 → 180 Ringing → 200 OK（占位 SDP，回带 Record-Route）→ 收 ACK → CONFIRMED
（autoAnswer=false 时等待 ims-answer；ANSWER 后无 ACK 超时 15s 复位）
```

- 应答 **Via 只取 INVITE 第一条**（P-CSCF 事务匹配）+ **回带完整 Via 链**（代理逐跳剥头）
- **2xx 应答回带 Record-Route**（RFC 3261 §12.1.1）——主叫据此构造 dialog Route 集
- 被叫发 in-dialog 请求用 Record-Route **逆序**；INVITE 重传（同 Call-ID+CSeq）重发上次应答

## 数据面

```
上行：ImsClientTask ──NmUeAppToNas::UPLINK_DATA_DELIVERY──► nasTask ──GTP──► gNB ──► UPF ──► P-CSCF
下行：P-CSCF ──► UPF ──► gNB ──► nasTask ──NmUeNasToApp::DOWNLINK_DATA_DELIVERY──► appTask ──► ImsClientTask
```

- SIP 包为**完整 IPv4/UDP 包**，由 `packet::BuildUdpPacket` 构造，源 IP = ims 会话的 `pduAddress`
- 下行分流在 `app/task.cpp`：`psi == imsPsi && (dport==ims.sipPort || dport==ims.mediaPort)` → IMS；其余走原 TUN 路径
- **IP 分片重组**（`app/task.cpp::reassembleIpv4Fragment`）：大 INVITE（>路径 MTU，被 UPF 分片）的分片2 无 UDP 头，会被 IsUdpPacket 误判走 TUN 丢失——按 srcIP+ID 缓存重组，MF=0 后重建 IP 头+校验和输出，30s 过期清理
- 会话缺失/释放 → IMS 静默/停止，不抢建会话；IMS 失败不影响 TUN 数据面

## 媒体（P3）

- **RTP 直推 NAS**（与 SIP 同路径）：`MediaSession` 构造 IP/UDP/RTP 包 → `UPLINK_DATA_DELIVERY`；下行按 `ims.mediaPort` 分流进 IMS
- **PCMU 静音流**（PT=0）：12B RTP 头 + 160B 全零帧（20ms），ts 160/包、seq+1，SSRC 派生自 `generateId`（已实现并验证：双端 mediaRxCount 持续增长）
- **触发/停止**：200 OK（双方）启动；BYE/callFail/会话释放停止；尽力而为（错误仅日志）
- **SDP**：`m=audio <ims.mediaPort> RTP/AVP 0` + `c=IN IP4 <pduAddress>`；对端目标解析 200 OK SDP answer 的 `c=`/`m=`（rtpengine 中转地址）
- **QoS**：P-CSCF N5 AUDIO 组件 201（medType AUDIO）+ 5QI=1 流（探测项）；UE 上行绑定 5QI=1 为 UERANSIM 平台限制（PDU 会话修改未实现）

## 测试策略

`tests/ims-tests` 直接编译纯函数源码（auth/digest/packet/sip_stack + 最小 utils），零外部框架：

- MD5 向量（RFC 1321）——回归 MD5 无限循环类 bug
- IP/UDP 校验和（RFC 768 参考 + 奇偶长度）——回归 SIP 包被 UPF 静默丢弃类 bug
- SIP 构造→解析回环
- DigestResponse（RFC 2617 样本）+ DigestAuth 行为（挑战采纳/nc 递增/reset）

运行：`cd build && ctest`
