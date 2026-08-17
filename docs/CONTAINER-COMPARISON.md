# CONTAINER-COMPARISON.md — 与容器版 pjsua 验证环境对照

## 背景

IMS 域用户侧最初依赖**外部 pjsua 进程**绑定 UE 的 ims TUN 接口（容器化部署）。本项目将 SIP 客户端内嵌进 UERANSIM `nr-ue`，替代 pjsua。下表对照两种形态。

## 对照

| 维度 | 容器版（pjsua） | 宿主版（UERANSIM-IMS） |
|------|----------------|------------------------|
| SIP 进程 | 容器内独立 pjsua 进程 | `nr-ue` 内嵌 `ImsClientTask`（NtsTask 线程） |
| 数据路径 | pjsua socket → ims TUN → 策略路由 → GTP | 进程内 `UPLINK_DATA_DELIVERY` 直推 NAS → GTP |
| 源 IP | 绑定 TUN IP（SMF 动态分配，脚本须动态取） | ims 会话 `pduAddress`（GTP 内层头部保证） |
| 注册/呼叫 | 手工/脚本驱动 pjsua 交互（socat PTY） | 自动注册 + 自动应答 + nr-cli 命令 |
| 鉴权 | pjsua 内置 Digest | 自实现 Digest-MD5（qop=auth） |
| 媒体 | pjsua 可发 RTP（C4 矩阵验证过媒体流） | 占位 SDP（m=audio 0 语义 / 后续 P3 媒体） |
| 自动化 | 脆弱（--null-audio 管道、非 tty 卡死、动态 IP） | 稳定（进程内、CLI 可脚本化） |
| 依赖 | pjsua 二进制、socat、动态 IP 解析脚本 | 无外部库；仅 UERANSIM + 标准工具链 |

## 为何内嵌而非保留 pjsua

容器版暴露的痛点（UERANSIM-IMS.md §1）：

1. UE IP 动态变化 → pjsua 每次需动态取 TUN IP
2. `--null-audio` 与管道 stdin 组合不稳定 → 呼叫发起不可靠
3. pjsua 交互模式在非 tty 下注册卡住 / URI 输入丢失 → 无法自动化
4. DNS NAPTR 顺序使 pjsua 选 TCP 发 INVITE，TCP 握手被 gtp5g 源改写破坏 → 477
5. 绑定 TUN IP 后流量走宿主策略路由强制入 GTP → 排障困难
6. 双 UE 测试需双容器 + 双 pjsua 手工编排 → 验证成本高

内嵌后全部消除：SIP 包不经 Linux 路由/TUN，源 IP 由数据面保证，注册/呼叫全自动。

## 验证环境对照

| 项 | 容器版 | 宿主版（当前） |
|----|--------|----------------|
| gNB | 容器内 nr-gnb | 宿主 nr-gnb（绑 docker 桥 172.21.0.1） |
| UE | ueransim / ueransim2 容器 | 宿主 nr-ue（imsi-895）；ueransim2 容器保留作对照 |
| 核心网/IMS | free5gc + kamailio 容器 | 同左（容器不动） |
| 观测 | docker logs + pjsua 日志 | docker logs + `nr-cli ims-status` JSON |
| 回归 | C4 矩阵（注册/主叫/被叫/挂断/稳定性 10 次） | 同口径（宿主形态） |

## 验证结果摘要

- **P1 注册**：S-CSCF usrloc 3 IMPU registered、P-CSCF N5 201（app-session）、`ims-status` REGISTERED、到期重注册稳定
- **P2 呼叫**：`ims-call` → INVITE（两条 Route + digest）→ 200 → ACK → CONFIRMED → `ims-hangup` → BYE → 200；被叫自动应答（180/200，回带 Record-Route）正常
- **双向互呼**：23 项回归断言全绿（正/反向 CONFIRMED + 挂断 IDLE + 媒体 + AKA/EPCO + 核心网断言 + 数据面回归）
- **双客户端互呼**：双 UERANSIM IMS 客户端 10 轮稳定性 10/10（对比：pjsua 对照组同测试因续注册写死 expires-30、忙时掉注册而失败——对照组固有局限）
- **P3 媒体**：RTP 静音流双向收发（rtpengine 中转）+ P-CSCF N5 AUDIO 组件 201
- **P4 增强**：EPCO 下发 P-CSCF（0x000C）+ IMS AKA（AKAv1-MD5）注册，S-CSCF `Auth succeeded`
- **数据面隔离**：ims.pcscf 配错时 TUN ping 不受影响；无上行环路（宿主 UE 需 `tunNetmask: 255.255.255.0`）
- **已知限制（已消除）**：早期"外部客户端经 I-CSCF 偶发 500"实为被叫注册过期表象（LIR 定位失败），双客户端/注册稳定后不复现；本客户端主叫走 Service-Route 直连 S-CSCF

## 回退与对照

容器镜像保留 pjsua（对照/回退），两种形态可并存：宿主 UE 跑内嵌 IMS 客户端，容器 UE2 跑 pjsua（或换装新二进制跑第二个 IMS 客户端），在同一核心网下互相对照注册与呼叫。
