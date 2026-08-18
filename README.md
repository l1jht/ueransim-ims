# UERANSIM-IMS

在 [UERANSIM](https://github.com/aligungr/UERANSIM) 的 `nr-ue` 进程内嵌入轻量 IMS SIP 客户端——UE 建立 ims PDU 会话后**自动完成 SIP 注册与呼叫信令**（REGISTER / INVITE / 应答 / BYE），替代外部 pjsua。

SIP 包在 UERANSIM 进程内直推 NAS/GTP 数据面（`NmUeAppToNas::UPLINK_DATA_DELIVERY`），源 IP 即 PDU 会话 IP，绕开 pjsua 的 TUN 绑定、DNS/传输选择与策略路由问题。

## 特性

- **自动 SIP 注册**：ims 会话建立即触发；**Digest-MD5 与 IMS AKA（AKAv1-MD5）双鉴权**（challenge 算法自动跟随）；退避重试 5/15/45/60s；跟随 200 OK 的 expires 并按 90% 提前重注册；**带凭据 REGISTER 收到 401 时自动重挑战一次（每注册流程 1 次，nonce 过期/竞争兜底）**
- **P-CSCF 双来源**：**EPCO 下发优先**（PDU 会话建立请求带 0x000C，SMF 回写 P-CSCF），`ims.pcscf` 配置兜底
- **自动应答呼叫**：`autoAnswer`（默认 true）自动回 180/200；INVITE 带两条 Route（P-CSCF + Service-Route）与真实媒体 SDP；in-dialog 路由遵循 Record-Route；**2xx 按 RFC 3261 Timer G 每 500ms 重传直至 ACK（弱网下 200 丢失可救回）**
- **RTP 媒体流（P3）**：PCMU 静音流 20ms 双向收发（rtpengine 中转），收包计数暴露于 `ims-status`；P-CSCF N5 AUDIO 组件授权
- **CLI 控制**：`ims-register`（手动强制重注册）/ `ims-call <uri>` / `ims-answer` / `ims-hangup` / `ims-sms <uri> <text>` / `ims-status`
- **SMS over IMS**：`ims-sms` 发送 SIP MESSAGE（标准封装：RP-DATA+SMS-SUBMIT，TS 23.040/24.011/24.341，UCS2 编码，202 应答 + RP-ACK/RP-ERROR 处理）；接收自动应答并打印，最近 10 条经 `ims-status` 的 `smsHistory` 可见（配合短信中心 smsc 存储+异步投递）
- **零外部依赖**：自实现 SIP 协议栈、MD5（RFC 1321）、Digest（RFC 2617）、AKA（Milenage）、IPv4/UDP 封包（RFC 768）——无 libcurl/libxml/pjsip
- **与既有行为兼容**：`ims.enable` 默认 false；非 SIP/RTP 流量走原 TUN 路径不变；IMS 失败不影响数据面
- **单元测试**：`ctest` 覆盖 MD5 向量、IP/UDP 校验和（含奇数长度）、SIP 编解码回环、Digest 已知样本、AKA 挑战（hex/Base64 nonce）、RTP/SDP

## 架构

```
src/ue/ims/
├── ims.cpp/hpp        ImsClientTask（NtsTask）：会话绑定、注册/呼叫状态机、CLI、定时器、媒体生命周期
├── sip_stack.cpp/hpp  轻量 SIP 协议栈：消息构造/解析（纯文本编解码）
├── packet.cpp/hpp     IPv4/UDP 传输层：封包、校验和、下行判定、RTP 包构造
├── auth.cpp/hpp       DigestAuth：有状态挑战对象（MD5 与 AKAv1-MD5，Milenage 复用）
├── digest.cpp/hpp     MD5 + DigestResponse 纯函数库
├── media.cpp/hpp      RtpSender：PCMU 静音流发送器（独立于呼叫状态机）
└── config.hpp         ImsConfig（ue.yaml `ims:` 段）
tests/                 ims-tests：纯函数单元测试
```

数据面（不经 TUN/Linux 路由）：

```
上行：ImsClientTask ──UPLINK_DATA_DELIVERY──► NAS ──GTP──► gNB ──► UPF ──► P-CSCF
下行：P-CSCF ──► UPF ──► gNB ──► NAS ──DOWNLINK_DATA_DELIVERY──► appTask ──按 ims.sipPort 分流──► ImsClientTask
```

## 快速开始

依赖：Ubuntu 22.04、cmake ≥ 3.17、g++ ≥ 11、libsctp-dev、yaml-cpp（构建时会连同 UERANSIM 一起编译）。

```bash
# 1. 拉取 UERANSIM 源码并应用本项目的 IMS 补丁
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM
git apply <path-to>/patch/ims.patch

# 2. 构建
mkdir build && cd build
cmake ..
make -j$(nproc)

# 3. 运行（需要 root，TUN 接口）
sudo ./nr-gnb -c <path-to>/config/gnb.yaml
sudo ./nr-ue -c <path-to>/config/ue.yaml

# 4. 验证
./nr-cli imsi-001011234567895 -e ims-status
./nr-cli imsi-001011234567895 -e "ims-call sip:001011234567896@ims.mnc001.mcc001.3gppnetwork.org"
```

详见 [docs/BUILD.md](docs/BUILD.md) 与 [docs/RUN.md](docs/RUN.md)。

## 文档

| 文档 | 内容 |
|------|------|
| [docs/BUILD.md](docs/BUILD.md) | 依赖、从零编译、补丁应用、单元测试 |
| [docs/RUN.md](docs/RUN.md) | 宿主运行拓扑、配置说明、CLI、验证观测点、已知坑 |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 模块设计、状态机、信令流程、测试策略 |
| [docs/CONTAINER-COMPARISON.md](docs/CONTAINER-COMPARISON.md) | 与容器版 pjsua 验证环境的对照 |

## 验证状态

- ✅ P1 注册：S-CSCF usrloc（3 IMPU）、P-CSCF N5 201、`ims-status` REGISTERED、重注册稳定
- ✅ P2 呼叫：INVITE → 200 → ACK → CONFIRMED → BYE，主叫与被叫自动应答均验证（in-dialog 路由遵循 Record-Route）
- ✅ 双向互呼：正/反向全通（注册/CLI 防护/SMS/双向呼叫/媒体/EPCO 共 23 项回归断言全绿）；双 UERANSIM IMS 客户端互呼 10/10 稳定性（早前"外部客户端经 I-CSCF 偶发 500"经查为被叫注册过期表象，注册稳定后不复现）
- ✅ P3 媒体：RTP 静音流双向收发（rtpengine 中转）+ P-CSCF N5 AUDIO 组件 201
- ✅ P4 增强：EPCO 下发 P-CSCF（0x000C，EPCO 优先/配置兜底）+ IMS AKA（AKAv1-MD5，S-CSCF `ALGORITHM IS [AKAv1-MD5]` + `Auth succeeded`）
- ✅ P5 SMS over IMS：`ims-sms` 双向互发（**单模式 TPDU**：标准 RP-DATA/UCS2，TS 24.341），特殊字符与中文完整、202 应答、RP-ACK/RP-ERROR、弱网 25% 无重复投递、23 断言套件全绿（2026-08-18/19）

## 许可

本项目是 [UERANSIM](https://github.com/aligungr/UERANSIM)（**AGPL-3.0**）的衍生修改，随附 [LICENSE](LICENSE)。本项目的 IMS 模块代码同样以 AGPL-3.0 发布。
