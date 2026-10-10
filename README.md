# pcap2socks

**English** | [简体中文](README.zh-CN.md)

**pcap2socks** is a proxy which redirects TCP and UDP traffic to a SOCKS5 proxy with pcap, written in Rust.

You can use [pcap2socks GUI](https://github.com/zhxie/pcap2socks-gui) for a front-end interface.

Originally designed to accelerate games in game consoles, pcap2socks also works well as a lightweight transparent gateway on Linux routers and single-board computers.

This branch is based on [chaos-zhu/pcap2socks](https://github.com/chaos-zhu/pcap2socks), the continuation of the original [zhxie/pcap2socks](https://github.com/zhxie/pcap2socks), and adds long-running stability fixes (see [Changes in this fork](#changes-in-this-fork)).

## Changes in this fork

- **Connection reclamation.** Connections idle for over 10 minutes, or stalled in 8 consecutive unacknowledged retransmissions, are detected by a periodic sweep and cleaned up instead of living forever.
- **RST storm fix.** Resets for untracked connections are now sent as plain RST carrying the exact expected sequence number, throttled to at most one per second per connection. The previous ACK/RST used a zero acknowledgement number, which an established peer rejects per [RFC 5961](https://datatracker.ietf.org/doc/html/rfc5961) and answers with a challenge ACK, causing an endless RST/ACK storm that stalled the data path.
- **Bounded connection table.** Concurrent connections are capped at 2048 and further SYNs are rejected with a well-formed RST; a new SYN can replace a connection that has been dead for more than 60 seconds.
- **LAN traffic stays local.** Traffic to private or link-local destinations is no longer redirected, so LAN traffic never detours through the proxy.
- **Fragment table cleanup.** Stale IPv4 fragment groups are expired instead of accumulating.
- **Toolchain updates.** Dependency bumps (pnet 0.34, ipnetwork 0.20).

The fixes have been validated in a long-running deployment on a Linux router.

## Features

- **Redirect Traffic**: Redirect TCP and UDP traffic to a SOCKS proxy.
- **Proxy ARP**: Reply ARP request as it owns the specified address which is not on the network.
- **Cross Platform**
- **Full Cone NAT**

## Dependencies

1. On Windows, [Npcap](http://www.npcap.org/) or WinPcap (if using Npcap, make sure to install with the "Install Npcap in WinPcap API-compatible Mode" option).
2. On Linux, no external libraries are required (packets are handled with AF_PACKET). Run as root, or grant the binary the capability once:
   ```
   sudo setcap cap_net_raw+ep path_to_pcap2socks
   ```
3. On macOS, run as root.

## Build

A recent stable [Rust](https://www.rust-lang.org/) toolchain is required.

```
cargo build --release
```

### Cross-compilation

To build Linux binaries from macOS or Linux with exact glibc versioning, use [cargo-zigbuild](https://github.com/rust-cross/cargo-zigbuild) together with [Zig](https://ziglang.org/):

```
# aarch64, glibc 2.36+ (e.g. Debian 12)
rustup target add aarch64-unknown-linux-gnu
cargo zigbuild --release --target aarch64-unknown-linux-gnu.2.36

# armv7 hard-float, glibc 2.35+ (e.g. Ubuntu 22.04 on RK3506 boards)
rustup target add armv7-unknown-linux-gnueabihf
cargo zigbuild --release --target armv7-unknown-linux-gnueabihf.2.35
```

Prebuilt Linux binaries are attached to the [Releases](../../releases) page.

## Usage

```
pcap2socks -s <ADDRESS>

# Or a more general one using proxy ARP (recommended)
pcap2socks -s <ADDRESS> -p <ADDRESS> -d <ADDRESS>
```

### Example: Linux router / SBC as a transparent gateway

Run a local SOCKS5 server (for example v2ray, Xray or sing-box) listening on `127.0.0.1:1080`, then start pcap2socks on the interface of the downstream network:

```
sudo ./pcap2socks -s 192.168.1.1/24 -i eth0 -d 127.0.0.1:1080
```

Point the gateway and DNS of the devices to be proxied at this machine, and their TCP/UDP traffic will be redirected to the SOCKS5 proxy.

### Flags

`-h, --help`: Prints help information.

`-v, --verbose`: Prints verbose information (`-vv` for vverbose).

`-V, --version`: Prints version information.

`--force-associate-destination`, `--force-associate-bind-address`: Force to associate with the destination/replied bind address. pcap2socks will associate with the destination instead of the replied bind address in UDP ASSOCIATE if the replied bind address is in the private network by default. If this flag is set, pcap2socks will force to associate with the destination/replied bind address. If both flags are set, the `--force-associate-destination` will take effect.

### Options

`-i, --interface <INTERFACE>`: Interface for listening.

`--mtu <VALUE>`: MTU. Generally, pcap2socks will automatically obtain the MTU, but you can also override by setting this option. The MTU is set in the traffic from local to the source.

`-P, --preset <PRESET>`: Preset. You can use preset source and publish of game accelerators in the market. Available values are `t`, `tencent` for [Tencent Online Game Accelerator](https://jiasu.qq.com/) and `n`, `netease`, `u`, `uu` for [Netease UU Game Accelerator](https://uu.163.com/).

`-s, --source <ADDRESS>`: Source. The source can be a single IPv4 address like `192.168.1.2`, or an IPv4 CIDR network like `10.10.0.1/24`.

`-p, --publish <ADDRESS>`: ARP publishing address. If this option is set, pcap2socks will reply ARP request as it owns the specified address which is not on the network, also called proxy ARP.

`-d, --destination <ADDRESS>`: Destination, default as `127.0.0.1:1080`.

`--username <VALUE>`: Username. This value should be set only when the SOCKS5 server requires the username/password authentication.

`--password <VALUE>`: Password. This value should be set only when the SOCKS5 server requires the username/password authentication.

## Troubleshoot

1. Because the packet sent from sources should only be handled by pcap2socks, you have to disable IP forward or configure the firewall with the following command statement. For more information, please refer to the troubleshoot paragraph in [IkaGo](https://github.com/zhxie/ikago#troubleshoot).

   ```
   // Linux
   sysctl -w net.ipv4.ip_forward=0

   // macOS
   sysctl -w net.inet.ip.forwarding=0
   ```

2. pcap2socks requires root permission in some OS by default. But you can run pcap2socks in non-root by executing the following command before opening pcap2socks.
   ```
   // Linux
   setcap cap_net_raw+ep path_to_pcap2socks
   ```

3. Log lines like `Cleaning up N expired TCP connection(s)` (periodic reclamation) and `Too many connections, rejecting SYN` (connection cap reached) are expected behaviours of this fork.

## Limitations

1. IPv6 is not supported yet.

2. Because only SOCKS5 can forward UDP traffic, pcap2socks only support SOCKS5 at this point. A version with SOCKS4 support without redirecting UDP traffic will release in the future.

## Known Issues

1. Applications like VMWare Workstation on Windows may implement their own IP forwarding and forward packets which should be handled by pcap2socks, resulting in abnormal operations in pcap2socks.

2. The traffic flow control was not implemented in pcap2socks currently. On certain occasions, bandwidth may be heavily occupied by one side of a connection, causing the other side of the connection and other connections to be unable to transmit data normally.

## License

pcap2socks is licensed under [the MIT License](LICENSE). This branch is based on [zhxie/pcap2socks](https://github.com/zhxie/pcap2socks) and [chaos-zhu/pcap2socks](https://github.com/chaos-zhu/pcap2socks).
