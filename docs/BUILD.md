# BUILD.md — 构建指南

## 环境要求

- Ubuntu 22.04（或兼容发行版）
- CMake ≥ 3.17
- g++ ≥ 11（C++17）
- libsctp-dev（SCTP，gNB↔AMF N2 需要）
- yaml-cpp（随 UERANSIM 一起编译，无需单独安装）
- 可选：clang-tidy / valgrind

## 从零编译

```bash
# 拉取 UERANSIM（v3.3.0 基线）
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM

# 应用 IMS 补丁（本交付包中生成）
git apply <path-to-deliver>/patch/ims.patch

# 可选：确认补丁只触及预期文件
git status --short

# 构建（GLOB 文件，新增源文件需重新 cmake）
mkdir build && cd build
cmake ..
make -j$(nproc)
```

产物：`build/nr-ue`、`build/nr-gnb`、`build/nr-cli`、`build/nr-binder`。

## 单元测试

```bash
cd build
ctest --output-on-failure
```

`ims-tests` 覆盖：

- **MD5**：RFC 1321 测试向量（空串 / "abc" / "The quick brown fox…"）
- **IP/UDP 校验和**：RFC 768 参考实现对照，含偶数与**奇数长度** payload（历史上奇数长度末字节漏算导致 SIP 包被 UPF 静默丢弃）
- **SIP 编解码**：BuildInvite → ParseSipRequest 回环、BuildSipReply → ParseSipResponse
- **Digest**：RFC 2617 已知样本、DigestAuth 挑战采纳/qop 校验/nc 递增/reset
- **下行判定**：IsUdpPacket 端口匹配

## 常见问题

| 症状 | 处理 |
|------|------|
| `fatal error: sctp/sctp.h: No such file or directory` | `sudo apt install libsctp-dev` |
| 改了 `src/ue/ims/` 源码后构建没反映 | GLOB 收集，重新 `cmake ..` 后再 `make` |
| `nr-ue` 启动报 TUN 权限 | 需要 root 运行（`sudo`），并确认 `/dev/net/tun` 存在 |
