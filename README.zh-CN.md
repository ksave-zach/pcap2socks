# pcap2socks

[English](README.md) | **简体中文**

**pcap2socks** 是一个用 Rust 编写的透明代理：通过 pcap 把 TCP 与 UDP 流量重定向到 SOCKS5 代理。

可以使用 [pcap2socks GUI](https://github.com/zhxie/pcap2socks-gui) 作为图形前端。

pcap2socks 最初用于加速游戏主机的网络，同时也适合作为 Linux 路由器、单板计算机 (SBC) 上的轻量透明网关。

本分支基于 [chaos-zhu/pcap2socks](https://github.com/chaos-zhu/pcap2socks)（原项目 [zhxie/pcap2socks](https://github.com/zhxie/pcap2socks)），修复了长时间运行的稳定性问题，详见下方「本分支的修复」。

## 本分支的修复

- **连接回收**：空闲超过 10 分钟、或连续 8 次重传仍无进展的 TCP 连接，会被定期巡检发现并清理，不再永久驻留。
- **RST 风暴修复**：陌生连接的复位包改为「纯 RST + 精确序列号」，并对每个连接限速（每秒最多 1 个）。此前发送的 ACK/RST 携带的确认号为 0，对端会拒绝（[RFC 5961](https://datatracker.ietf.org/doc/html/rfc5961)）并回发 challenge ACK，造成无限对拍、传输停滞。
- **连接表有上限**：并发连接上限为 2048 个，超出后新的 SYN 会被构造正确的 RST 拒绝；静默超过 60 秒的死连接也可被新 SYN 替换。
- **内网流量直连**：发往私网、链路本地地址的流量不再被接管，局域网内的互访不绕行代理。
- **分片表清理**：过期的 IPv4 分片组会被回收，不再累积。
- **工具链更新**：依赖升级（pnet 0.34、ipnetwork 0.20）。

以上修复已在 Linux 路由器 (aarch64) 上长时间运行验证。

## 特性

- **流量重定向**：把 TCP 与 UDP 流量重定向到 SOCKS 代理。
- **Proxy ARP**：可代替指定地址应答 ARP 请求。
- **跨平台**。
- **Full Cone NAT**。

## 依赖

1. Windows：需要 [Npcap](http://www.npcap.org/) 或 WinPcap（若用 Npcap，安装时勾选 "Install Npcap in WinPcap API-compatible Mode"）。
2. Linux：无额外依赖（使用 AF_PACKET 处理数据包）。需以 root 运行，或一次性授予能力：
   ```
   sudo setcap cap_net_raw+ep path_to_pcap2socks
   ```
3. macOS：以 root 运行。

## 编译

需要较新的稳定版 [Rust](https://www.rust-lang.org/) 工具链。

```
cargo build --release
```

### 交叉编译

在 macOS / Linux 上交叉编译 Linux 目标，推荐 [cargo-zigbuild](https://github.com/rust-cross/cargo-zigbuild) 搭配 [Zig](https://ziglang.org/)，可精确对齐目标系统的 glibc 版本：

```
# aarch64，glibc 2.36+（如 Debian 12）
rustup target add aarch64-unknown-linux-gnu
cargo zigbuild --release --target aarch64-unknown-linux-gnu.2.36

# armv7 硬浮点，glibc 2.35+（如 RK3506 开发板上的 Ubuntu 22.04）
rustup target add armv7-unknown-linux-gnueabihf
cargo zigbuild --release --target armv7-unknown-linux-gnueabihf.2.35
```

预编译的 Linux 二进制见本仓库 [Releases](../../releases) 页面。

## 使用方法

```
pcap2socks -s <地址>

# 更通用的写法，使用 Proxy ARP（推荐）
pcap2socks -s <地址> -p <地址> -d <地址>
```

### 示例：Linux 路由器 / 单板计算机做透明网关

先在本机运行一个 SOCKS5 服务（如 v2ray、Xray、sing-box），监听 `127.0.0.1:1080`，然后在下游网络的接口上启动 pcap2socks：

```
sudo ./pcap2socks -s 192.168.1.1/24 -i eth0 -d 127.0.0.1:1080
```

把需要代理的设备的「网关」与「DNS」指向本机，其 TCP/UDP 流量就会被重定向到 SOCKS5 代理。

### 标志（Flags）

`-h, --help`：打印帮助信息。

`-v, --verbose`：打印详细日志（`-vv` 更详细）。

`-V, --version`：打印版本信息。

`--force-associate-destination`、`--force-associate-bind-address`：强制 UDP ASSOCIATE 关联到目标地址或对端回传的 bind 地址。默认情况下，若对端回传的 bind 地址属于私网，pcap2socks 会关联到目标地址；加上此标志则强制关联到目标地址 / 回传的 bind 地址。两者同时设置时，以 `--force-associate-destination` 为准。

### 选项（Options）

`-i, --interface <INTERFACE>`：监听接口。

`--mtu <VALUE>`：MTU。通常 pcap2socks 会自动获取 MTU，也可以用此选项覆盖。MTU 用于本机发往来源方向的流量。

`-P, --preset <PRESET>`：预设。可使用市面上游戏加速器的预设来源与发布地址。可选值：`t`、`tencent` 为[腾讯网游加速器](https://jiasu.qq.com/)；`n`、`netease`、`u`、`uu` 为[网易 UU 加速器](https://uu.163.com/)。

`-s, --source <ADDRESS>`：来源。可以是单个 IPv4 地址（如 `192.168.1.2`），也可以是 IPv4 CIDR 网段（如 `10.10.0.1/24`）。

`-p, --publish <ADDRESS>`：ARP 发布地址。设置后，pcap2socks 会以该地址（不在本网段内）的拥有者身份应答 ARP 请求，即 Proxy ARP。

`-d, --destination <ADDRESS>`：目标地址，默认 `127.0.0.1:1080`。

`--username <VALUE>`：用户名。仅当 SOCKS5 服务端要求用户名/密码认证时设置。

`--password <VALUE>`：密码。仅当 SOCKS5 服务端要求用户名/密码认证时设置。

## 故障排查

1. 来源发出的数据包应当只由 pcap2socks 处理，因此需要关闭 IP forward，或按以下命令配置防火墙。更多说明见 [IkaGo](https://github.com/zhxie/ikago#troubleshoot) 的故障排查段落。

   ```
   // Linux
   sysctl -w net.ipv4.ip_forward=0

   // macOS
   sysctl -w net.inet.ip.forwarding=0
   ```

2. 部分系统默认要求 root 权限运行 pcap2socks。也可以在启动前先执行以下命令，以普通用户运行。
   ```
   // Linux
   setcap cap_net_raw+ep path_to_pcap2socks
   ```

3. 日志中出现的 `Cleaning up N expired TCP connection(s)`（定期回收）与 `Too many connections, rejecting SYN`（达到连接上限）属于本分支的预期行为。

## 限制

1. 暂不支持 IPv6。

2. 由于只有 SOCKS5 能转发 UDP 流量，目前 pcap2socks 仅支持 SOCKS5；未来会发布一个支持 SOCKS4、不转发 UDP 流量的版本。

## 已知问题

1. Windows 上的 VMWare Workstation 等应用会自行实现 IP 转发，把本应由 pcap2socks 处理的数据包也转发出去，导致 pcap2socks 行为异常。

2. 目前尚未实现流量控制（flow control）。某些场景下单条连接的一侧可能占满带宽，导致另一侧及其他连接无法正常传输。

## 许可证

pcap2socks 使用 [MIT 许可证](LICENSE)。本分支基于 [zhxie/pcap2socks](https://github.com/zhxie/pcap2socks) 与 [chaos-zhu/pcap2socks](https://github.com/chaos-zhu/pcap2socks)。
