# RUN.md — 运行指南

## 拓扑

本项目验证环境采用**宿主运行 + 容器核心网**的形态：

```
宿主（root）
├── nr-gnb    （RadioLink 172.21.0.1:4997 / N2 SCTP→AMF / N3 GTP-U→UPF）
├── nr-ue     （宿主 UE，imsi-895，内嵌 IMS 客户端）
└── ueransim2 容器（可选：第二个 UE + 外部 pjsua 对照）

容器网络 172.21.0.0/24（free5gc + kamailio IMS）
├── amf 172.21.0.5    smf 172.21.0.4     upf 172.21.0.2
├── pcscf 172.21.0.21  scscf 172.21.0.20  icscf 172.21.0.19
└── pcf 172.21.0.27    pyhss 172.21.0.18  dns 172.21.0.15
```

> 容器 IP 为 docker 动态分配，**重启后可能漂移**。部署前用 `docker inspect <c> --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'` 核对，并同步修正 `config/gnb.yaml` 的 `amfConfigs`、`config/ue.yaml` 的 `pcscf`、以及 P-CSCF 指向 UPF 的静态路由（`192.168.101.0/24 via <UPF-IP>`）。

## 前置条件

1. 核心网 + IMS（free5gc + kamailio）在线，P-CSCF 可路由到 UE 网段（`192.168.101.0/24 via <UPF>`）
2. UE 订阅（imsi / Ki / op）与 `config/ue.yaml` 一致
3. DNS 容器可解析 `ims.mnc001.mcc001.3gppnetwork.org`（NAPTPR/SRV 指向 P-CSCF）
4. 宿主 `/dev/net/tun` 存在，有 root

## 配置

### gNB（`config/gnb.yaml`）

| 项 | 说明 |
|----|------|
| `linkIp/ngapIp/gtpIp` | 宿主在 docker 桥上的地址（默认 172.21.0.1），容器可达 |
| `amfConfigs.address` | AMF 容器实际 IP（默认 172.21.0.5） |
| `slices` | 必须与 UE 请求及核心网一致（sst=1, sd=0x010203） |

### UE（`config/ue.yaml`）

| 项 | 说明 |
|----|------|
| `tunNetmask: 255.255.255.0` | **必须**。默认 /16 会在宿主产生 `192.168.0.0/16` 连接路由，捕获 UPF N6 回环的同源包 → 上行洪水环路 |
| `ims.enable` | true 启用内嵌客户端；默认 false 保持纯 UERANSIM 行为 |
| `ims.pcscf` | P-CSCF 地址（配置化兜底；EPCO 下发优先，`SESSION_BOUND` 日志 `(EPCO)` 标记） |
| `ims.password` | Ki（kamailio scscf MD5 模式） |
| `ims.sipPort` | 同一宿主跑多个 UE 时需错开 |

## 运行

```bash
sudo ./nr-gnb -c config/gnb.yaml
sudo ./nr-ue  -c config/ue.yaml
```

## CLI

```bash
./nr-cli imsi-001011234567895 -e ims-status        # JSON：state/registered/expires/callState
./nr-cli imsi-001011234567895 -e ims-register      # 手动强制重注册（正常流程自动注册，此命令用于测试/刷新；注册进行中再次触发返回 "registration already in progress"）
./nr-cli imsi-001011234567895 -e "ims-call sip:001011234567896@ims.mnc001.mcc001.3gppnetwork.org"
./nr-cli imsi-001011234567895 -e ims-answer        # autoAnswer=false 时手动应答
./nr-cli imsi-001011234567895 -e ims-hangup
```

## 验证观测点

| 链路 | 观测 |
|------|------|
| 注册 | S-CSCF usrloc 出现 3 个 IMPU（sip:+tel: registered）；P-CSCF N5 201（`N5 QoS Session successfully created`）；`ims-status` 显示 REGISTERED；鉴权算法 S-CSCF `ALGORITHM IS [AKAv1-MD5]` + `Auth succeeded`（AKA，P4）/ `[MD5]`（Digest） |
| EPCO（P4） | 会话绑定日志 `pcscf=... (EPCO)`（SMF 0x000C 下发优先）；`(config)` 表示配置兜底 |
| 主叫 | P-CSCF/S-CSCF 日志 INVITE 链路；`ims-status` callState=CONFIRMED |
| 被叫 | 收到 INVITE 自动 180→200（autoAnswer，回带 Record-Route）；主叫收到 ACK（ACK 需带 200 OK 的 Record-Route Route 集，否则被叫侧 ACK 超时） |
| 挂断 | BYE 双向；S-CSCF dialog 清理；N5 会话删除 |
| 媒体（P3） | 双端 `ims-status` mediaRxCount>0；P-CSCF N5 body 出现 `medType: "AUDIO"` + qosReference（201）；5QI=1 QoS 流为探测项 |
| 双向互呼 | 正/反向各一轮：双向 CONFIRMED + 挂断 IDLE（`scripts/ims-verify.sh` 23 项断言（含 AKA/EPCO/媒体）；反向触发依赖对端形态：pjsua 控制台或容器内 `nr-cli`） |
| 稳定性 | 到期 90% 提前重注册成功；连续呼叫无 477/403/504（注意 usrloc "slot 477" 日志误报） |
| 数据面隔离 | `ims.pcscf` 故意配错时，TUN ping 不受影响，IMS 仅日志退避重试 |

## 已知坑

- **容器动态 IP 漂移**：核心网容器重启会重排 free5gc 容器 IP，连锁破坏 gNB/AMF、NRF 发现、P-CSCF 路由——部署后核对并修正配置
- **ims TUN /16 环路**：宿主 UE 必须 `tunNetmask: 255.255.255.0`
- **UPF MASQUERADE**：若 ims 流量源端口被 NAT 改写（双 UE 端口冲突），需在 UPF 加例外：`iptables -t nat -I POSTROUTING 1 -o eth0 -d <P-CSCF-IP> -j ACCEPT`
- **gNB RLS 映射**：同 gNB 频繁 kill/重连 UE 会致 STI→ueId 残留映射，重启 gNB 恢复
- **大 SIP 包 IP 分片**：被叫 INVITE > 路径 MTU 会被 UPF 分片，客户端已实现重组；若改动下行分流需保留
- **pjsua 对照组掉注册**：pjsua 续注册间隔写死 expires-30（余量仅 30s），系统忙时延迟即掉注册（呼叫 404/LIR 500）——建议用双 UERANSIM IMS 客户端互呼验证稳定性
