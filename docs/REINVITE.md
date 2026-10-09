# REINVITE.md — 锚点迁移后的媒体重锚（re-INVITE，同一通话恢复）

**适用场景**：PDU 会话重激活（SSC mode 2 锚点迁移 / 切换触发重锚 / 锚点失效重选）后本地 IP 变化，**已建立通话**的媒体用新地址自动恢复——不挂断、不重拨。

> 本增强已于 2026-10-09 并入 `patch/ims.patch`（v3.3.0 基线），无需额外补丁；本文档记录其设计、三个关键修正与已知局限。

## 1. 背景与问题

1. UE 的 ims PDU 会话被释放并以 `REACTIVATION_REQUESTED` 触发重激活（切换后锚点不服务新 TAC / 锚点 UPF 失效重选）；
2. UE 自动重建会话，落到**新锚、新 IP**，随后重发 REGISTER；
3. 客户端原行为仅此而已：SIP 层通话状态仍为 `CONFIRMED`（对话无感知），但网络侧（rtpengine）媒体端点仍指向**旧 IP**；
4. 实测后果：迁移侧媒体丢失，rtpengine 只剩对端单向 ~92 pkt/s（满流 ~184 pkt/s）；唯一恢复方式是挂断 + 重拨。

## 2. 方案

| 阶段 | 行为 |
|---|---|
| 会话释放（通话中） | `stopMedia()`；保留通话状态，等待重绑定 |
| 会话重绑定 | `CONFIRMED` 且已知媒体对端 → 用**新本地 IP**重建媒体；主叫置 `m_reinvitePending` |
| 本端注册成功 | 主叫发送带 SDP 的 in-dialog **re-INVITE**（新 IP；目标 = 对端 AOR） |
| 对端（UAS） | 同 Call-ID 的 in-dialog INVITE → 按 SDP 更新媒体对端 → 200 OK（带自身 SDP）+ Timer G + 重传判重 |
| 主叫收 200 | ACK；按 200 的 SDP 按需刷新媒体（不采信其 Contact） |

期望日志标记（实测时序）：

```
Media stopped
IMS session bound [local=新IP]
Media started [peer=...]
Re-INVITE pending until registration success
IMS registration succeeded
Re-INVITE sent [local=新IP]
（对端）Re-INVITE received, media re-anchored, 200 OK sent
（本端）Re-INVITE accepted, media re-anchored
```

## 3. 三个关键设计修正（r1 → r2，均为实测逼出）

1. **不采信 re-INVITE 200 OK 的 Contact**：核心对 200 的 Contact 做 NAT 别名改写（alias → S-CSCF 地址），若更新对话 Contact，后续 in-dialog 请求被 S-CSCF 404（`destination user not found`）。原对话 Contact 仍有效（本端迁移不改变对端位置）。
2. **re-INVITE 目标用对端 AOR（`m_remoteUri`）而非存量 Contact**：双端同时迁移时对端旧 Contact 已失效，交由核心按注册绑定解析到对端当前地址。
3. **仅主叫发送 + 推迟到注册成功后发送**：被叫方向发起的反向 in-dialog 请求在核心（Kamailio IMS）无法匹配既有对话（`dlg_onreq(): Failed to create dialog` → 500），由「仅主叫」规避（被叫媒体经其对主叫 re-INVITE 的 200 OK SDP 答案刷新）；迁移后 re-INVITE 与 REGISTER 竞争（404/408），由 `m_reinvitePending` 推迟到 `onRegSuccess` 规避。

## 4. 修改落点（4 文件 / 13 处）

| # | 文件 | 位置 | 改动 |
|---|---|---|---|
| 1 | `ims.cpp` | `handleSessionBound()` 末尾 | `CONFIRMED` 时 `startMedia(新IP)`；主叫置 `m_reinvitePending=true` |
| 2 | `ims.cpp` | `handleSessionReleased()` 末尾 | 通话中释放：`stopMedia()`，保留通话状态 |
| 3 | `ims.cpp` | 新增 `sendReInvite()`（`sendBye()` 后） | 带 SDP 的 in-dialog INVITE：CSeq+1、新 branch、`armTx`；目标 = 对端 AOR |
| 4 | `ims.cpp` | `handleCallResponse()` 新增 `CONFIRMED` 分支 | 仅处理 `cseq==m_reinviteCSeq` 的最终响应：200 → `sendAck()` + 按需重启媒体；≥300 → `sendErrorAck()` + 告警 |
| 5 | `ims.cpp` | `handleCallRequest()` INVITE 分支 | `CONFIRMED` 且同 Call-ID → 更新媒体 + 200 OK（带 SDP）+ Timer G + 判重 |
| 6 | `ims.cpp` | `handleCallRequest()` ACK 分支 | `CONFIRMED` 下 ACK → 清除 2xx 重传 |
| 7 | `ims.cpp` | `startMedia()` 开头 | 记录 `m_mediaPeerIp/Port` |
| 8 | `ims.cpp` | `onRegSuccess()` 末尾 | `m_reinvitePending && CONFIRMED` → `sendReInvite()` |
| 9 | `ims.cpp` | `startCall()` / 新呼叫 / `callFail()` | 维护 `m_isCaller` 与 `m_reinvitePending` |
| 10 | `ims.hpp` | 成员区 + 方法声明 | 6 个成员 + `sendReInvite()` |
| 11 | `sip_stack.hpp` | `BuildInDialogRequest` 声明 | 增加可选 `contentType/contentBody`（默认空） |
| 12 | `sip_stack.cpp` | `BuildInDialogRequest` 定义 | 体非空时带 `Content-Type` + 动态 `Content-Length`；ACK/BYE 行为不变 |
| 13 | — | 说明 | 三个关键修正见 §3 |

## 5. 兼容性与已知局限

- **对端也需打补丁**（UAS 分支，见 §4-5）；未打补丁的被呼端对 in-dialog INVITE 回 486 Busy Here；
- 仅处理 `CONFIRMED` 通话（空闲 / 呼叫建立阶段不触发）；仅覆盖**主叫侧迁移**，被叫单独迁移不覆盖（待核心侧修复反向 in-dialog 后可放开双端，注意 glare）；
- RFC 4028 会话定时器未实现；re-INVITE 非 2xx 仅告警（不自动重试/复位）；
- UAS 本地 CSeq 由初始 INVITE 递推，多次 re-INVITE 后可能小于远端 CSeq（后续可统一推进）；
- 媒体 SSRC 每次重绑定递增（下游测量按「新流」处理）；媒体端口固定（`ims.mediaPort`）；
- 同一通话的多次连续迁移受核心侧对话/contact 生命周期限制（第 6–7 次迁移出现 S-CSCF 对话匹配失败/陈旧 contact——核心侧状态问题，非本增强引入）。

## 6. 验证摘要（2026-10-05）

- **切换触发重锚（主叫迁移）**：多轮冒烟全过；媒体恢复 ≈ **2.1–2.3 s**；CIT（UE 感知）≈280–350 ms；同一通话无需重拨；正式批两档各 31/31 中继有效；
- **锚点失效重选（双 UE 同锚同时迁移）**：检测 ≈19.3–29.3 s，媒体恢复 ≈检测 + 0.5 s，CIT ≈8.2–18.3 s；被叫媒体由 200 OK 的 SDP 刷新；
- **回归**：无迁移通话行为不变（不产生 re-INVITE 日志）。
