# Rust VPS 流量 JSON 服务：公网鉴权版

## Summary

构建一个 Linux Rust 常驻服务，适配 CentOS 7+、Debian 11+、Ubuntu 20.04+。程序默认监听公网地址，所有流量数据接口必须使用 Bearer Token 鉴权；通过读取指定网卡的 Linux 字节计数器，按购买日月周期计算 VPS 本账期已用和剩余流量。

## Key Changes

- 构建与兼容：
  - 发布 `x86_64-unknown-linux-musl` 静态二进制，避免 CentOS 7 的旧 glibc / OpenSSL 依赖问题。
  - 不依赖 systemd 以外的发行版特性；统计只读 `/sys/class/net/<iface>/statistics/{rx_bytes,tx_bytes}`。
  - 提供 systemd unit，适用于 CentOS 7+、Debian 11+、Ubuntu 20.04+。
- 默认配置：
  - `listen_addr = "0.0.0.0:9733"`，默认允许公网访问。
  - `auth_token` 必填；缺失、为空或仍是示例值时服务拒绝启动。
  - `interfaces = ["eth0"]`，用户按实际网卡名修改。
  - `quota_bytes`、`billing_mode = "total"`、`cycle_anchor`、`cycle_months = 1`。
  - `state_path = "/var/lib/vps-trafficd/state.json"`。
- 鉴权与接口：
  - `GET /api/v1/traffic` 必须带 `Authorization: Bearer <token>`。
  - 鉴权失败返回 `401`，不泄露流量、网卡、账期等信息。
  - `GET /healthz` 可不鉴权，只返回最小健康信息，不包含敏感数据。
- 统计与账期：
  - 聚合配置中指定网卡的 rx/tx。
  - 接口同时返回 `rx_bytes`、`tx_bytes`、`used_bytes`、`remaining_bytes`、`usage_ratio`。
  - `used_bytes` 按 `billing_mode` 选择 rx、tx 或 total。
  - 根据购买日锚点推算当前账期；短月份没有对应日期时使用月末同一时间。
- 运维命令：
  - `vps-trafficd --config /etc/vps-trafficd/config.toml`
  - `vps-trafficd check --config ...` 检查配置、网卡、权限、token。
  - `vps-trafficd calibrate --rx <bytes> --tx <bytes>` 手动对齐服务商面板。

## Test Plan

- 在 CentOS 7、Debian 11、Ubuntu 20.04 的容器或虚拟机中验证二进制可运行、systemd unit 可启动。
- 测试公网监听默认值为 `0.0.0.0:9733`，但 `/api/v1/traffic` 无 token 或 token 错误时返回 `401`。
- 测试购买日账期计算：跨月、跨年、29/30/31 号、短月份。
- 测试网卡计数器正常增长、系统重启归零、计数器变小后的累计逻辑。
- 测试 `rx` / `tx` / `total` 三种计费口径和剩余流量不低于 0。
- 测试 `check` 能发现缺失网卡、不可写状态目录、空 token、示例 token。

## Assumptions

- 程序本身只提供 HTTP，不内置 HTTPS；公网部署时仍建议外层加 Nginx/Caddy/TLS。
- 默认公网监听是产品默认值，但 API 强制鉴权，避免裸露流量信息。
- 纯网卡计数器无法补回程序停止期间的流量；需要用 `calibrate` 手动校准。
- 默认提供 amd64 静态二进制；如 VPS 是 ARM，再额外发布 `aarch64-unknown-linux-musl`。
