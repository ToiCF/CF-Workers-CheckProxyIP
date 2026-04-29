# 基于 TLSClient 的真实链路 ProxyIP 检测器 - Check_Proxy.js

`Check_Proxy.js` 是一个面向 ProxyIP 场景的 Cloudflare Worker 检测器。它不再依赖传统 `/cdn-cgi/trace` 这类**描述型判断**，而是直接在 Worker 内发起**真实 TLS 握手 + 真实 HTTP 请求**，验证候选入口是否真的具备可用的 Cloudflare CDN 反代能力。

核心思路是：

```text
候选入口 IP:PORT
  → Cloudflare Worker connect()
  → 内嵌 TLSClient
  → 真实 SNI / Host
  → TLS 握手
  → HTTP 请求目标站
  → 解析出口 IP / 地理 / 栈类型
```

也就是说，它不是去看“像不像 ProxyIP”，而是直接去测“能不能按真实使用方式跑通”。

## 为什么不用 `/cdn-cgi/trace`

很多检测只会看：

- `/cdn-cgi/trace` 能不能打开；
- 返回格式像不像 Cloudflare；
- 某个 header 是否存在。

这类方法只能说明“表面上像”，不能说明它**真的能作为 ProxyIP 使用**。

`Check_Proxy.js` 走的是另一条路：

- 直接把候选入口当成真实前置；
- 用真实 `SNI/Host` 建立 TLS；
- 对真实由 Cloudflare CDN 反代的域名发 HTTP 请求；
- 根据返回的出口 IP / ASN / 地理信息判断这条链路是否真正成立。

因此它更接近真实使用场景，能明显减少“有 trace 但不能用”的误判，把判断从**表面特征识别**提升到**实际功能验证**。

## 功能特性

- 基于 Workers `cloudflare:sockets` 发起 TCP 出站。
- 内嵌 `TLSClient`，在 Worker 内手动完成 TLS 握手。
- 使用真实 `SNI` / `Host` / HTTP 请求验证候选入口。
- 同时探测多个目标源，交叉判断出口结果。
- 自动判断候选入口是：
  - `dual_stack`
  - `ipv4_only`
  - `ipv6_only`
  - `unknown`
- 支持单个候选或批量候选输入。
- 支持 `IPv4:PORT` 与 `[IPv6]:PORT` 格式。
- 返回详细时延数据：`connect_ms`、`tls_ms`、`http_ms`。

## 当前探测目标

当前内置两组真实探测目标：

- `https://api.db-ip.com/v2/free/self`
- `https://api.ip.sb/geoip`

它们分别用于返回出口侧的：

- IP
- 地理位置
- ASN / 组织信息
- IPv4 / IPv6 栈特征

Worker 会把候选入口分别连到这些目标，再综合结果推断这条 ProxyIP 的实际能力。

## 文件结构

| 文件 | 说明 |
| --- | --- |
| [Check_Proxy.js](./Check_Proxy.js) | Worker 主实现，内嵌 TLSClient 与真实链路探测逻辑。 |

## API 用法

## 单个候选

### GET

```text
/probe?candidate=118.34.215.56:34042
```

### POST

```json
{
  "candidate": "118.34.215.56:34042"
}
```

## 批量候选

### GET

```text
/probe?candidates=1.1.1.1:443,2.2.2.2:443
```

### POST

```json
{
  "candidates": [
    "1.1.1.1:443",
    "[2606:4700::1111]:443"
  ]
}
```

## 可选参数

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `timeoutMs` | 单步超时，作用于 TCP / TLS / HTTP 读取。 | `5000` |
| `readLimit` | 读取响应的最大字节数。 | `65536` |

示例：

```text
/probe?candidate=118.34.215.56:34042&timeoutMs=8000&readLimit=131072
```

## 返回结果

单个候选时返回对象，多候选时返回数组。

核心字段：

| 字段 | 说明 |
| --- | --- |
| `candidate` | 输入的候选入口。 |
| `ok` | 至少一个真实目标验证成功。 |
| `inferred_stack` | 推断出的栈类型：`dual_stack` / `ipv4_only` / `ipv6_only` / `unknown`。 |
| `supports_ipv4` | 是否支持 IPv4 出口。 |
| `supports_ipv6` | 是否支持 IPv6 出口。 |
| `dual_stack` | 是否为双栈。 |
| `probe_results` | 每个真实目标的详细探测结果。 |

`probe_results` 内会附带：

- `probe_name`
- `target`
- `connect_ms`
- `tls_ms`
- `http_ms`
- `status_code`
- `exit_ip`
- `exit_family`
- `exit_country`
- `exit_region`
- `exit_city`
- `exit_colo`
- `exit_asn`
- `exit_org`
- `error`

## 栈判断逻辑

当前实现的判断规则比较直接：

- `dbip = IPv4` 且 `ipsb = IPv6` → `dual_stack`
- `dbip = IPv4` 且 `ipsb = IPv4` → `ipv4_only`
- 只有 `ipsb = IPv6` 成立 → `ipv6_only`
- 其余情况 → `unknown`

这不是理论推断，而是基于真实目标返回结果的实测归类。

## 适合场景

- 检测候选入口是否具备真实 ProxyIP 能力。
- 区分“能开 trace”与“能跑真实 CDN 反代”的差别。
- 快速筛选 IPv4 / IPv6 / 双栈前置。
- 作为纯 Workers 路线下的真实链路验证入口。

## 已知限制

- 这是**功能验证器**，不是通用扫描器。
- 结果依赖外部探测目标的可达性与返回格式。
- 候选入口是否可用，仍会受到目标站、网络质量、地区、时段等因素影响。
- 当前验证重点是 **TLS + HTTP 路径是否真实可用**，不是泛化到所有协议。

## 测试入口

已部署测试入口：

<https://pr-apis.ekt.me/probe?candidate=118.34.215.56:34042>

## 相关链接

- 开源协议：[GPL-3.0](./LICENSE)
- 频道 / 交流群组：<https://t.me/Enkelte_notif>
