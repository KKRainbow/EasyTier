# EasyTier

[![GitHub](https://img.shields.io/github/license/KKRainbow/EasyTier)](https://github.com/KKRainbow/EasyTier/blob/main/LICENSE)
[![GitHub last commit](https://img.shields.io/github/last-commit/KKRainbow/EasyTier)](https://github.com/KKRainbow/EasyTier/commits/main)
[![GitHub issues](https://img.shields.io/github/issues/KKRainbow/EasyTier)](https://github.com/KKRainbow/EasyTier/issues)
[![GitHub actions](https://github.com/KKRainbow/EasyTier/actions/workflows/rust.yml/badge.svg)](https://github.com/KKRainbow/EasyTier/actions/)

[简体中文](/README_CN.md) | [English](/README.md)

一个简单、安全、去中心化的内网穿透 VPN 组网方案，使用 Rust 语言和 Tokio 框架实现。

## 特点

- **去中心化**：无需依赖中心化服务，节点平等且独立。
- **安全**：支持利用 WireGuard 加密通信。
- **跨平台**：支持 MacOS/Linux/Windows，未来将支持 IOS 和 Android。可执行文件静态链接，部署简单。
- **无公网 IP 组网**：支持利用共享的公网节点组网，可参考 [配置指南](#无公网IP组网)
- **NAT 穿透**：支持基于 UDP 的 NAT 穿透，即使在复杂的网络环境下也能建立稳定的连接。
- **子网代理（点对网）**：节点可以将可访问的网段作为代理暴露给 VPN 子网，允许其他节点通过该节点访问这些子网。
- **智能路由**：根据流量智能选择链路，减少延迟，提高吞吐量。
- **TCP 支持**：在 UDP 受限的情况下，通过并发 TCP 链接提供可靠的数据传输，优化性能。
- **高可用性**：支持多路径和在检测到高丢包率或网络错误时切换到健康路径。

## 安装

1. **下载预编译的二进制文件**

    访问 [GitHub Release 页面](https://github.com/KKRainbow/EasyTier/releases) 下载适用于您操作系统的二进制文件。

2. **通过 crates.io 安装**
   ```sh
   cargo install easytier
   ```


3. **通过源码安装**
   ```sh
   cargo install --git https://github.com/KKRainbow/EasyTier.git
   ```

## 快速开始

确保已按照 [安装指南](#安装) 安装 EasyTier，并且 easytier-core 和 easytier-cli 两个命令都已经可用。

### 双节点组网

假设双节点的网络拓扑如下

```mermaid
flowchart LR

subgraph 节点 A IP 22.1.1.1
nodea[EasyTier\n10.144.144.1]
end

subgraph 节点 B
nodeb[EasyTier\n10.144.144.2]
end

nodea <-----> nodeb

```

1. 在节点 A 上执行：
   ```sh
   sudo easytier-core --ipv4 10.144.144.1
   ```
   命令执行成功会有如下打印。

   ![alt text](assets/image-2.png)

2. 在节点 B 执行
   ```sh
   sudo easytier-core --ipv4 10.144.144.2 --peers udp://22.1.1.1:11010
   ```

3. 测试联通性

   两个节点应成功连接并能够在虚拟子网内通信
   ```sh
   ping 10.144.144.2
   ```

   使用 easytier-cli 查看子网中的节点信息
   ```sh
   easytier-cli peer
   ```
   ![alt text](assets/image.png)
   ```sh
   easytier-cli route
   ```
   ![alt text](assets/image-1.png)

---

### 多节点组网

基于刚才的双节点组网例子，如果有更多的节点需要加入虚拟网络，可以使用如下命令。

```
sudo easytier-core --ipv4 10.144.144.2 --peers udp://22.1.1.1:11010
```

其中 `--peers ` 参数可以填写任意一个已经在虚拟网络中的节点的监听地址。

---

### 子网代理（点对网）配置

假设网络拓扑如下，节点 B 想将其可访问的子网 10.1.1.0/24 共享给其他节点。

```mermaid
flowchart LR

subgraph 节点 A IP 22.1.1.1
nodea[EasyTier\n10.144.144.1]
end

subgraph 节点 B
nodeb[EasyTier\n10.144.144.2]
end

id1[[10.1.1.0/24]]

nodea <--> nodeb <-.-> id1

```

则节点 B 的 easytier 启动参数为（新增 -n 参数）

```sh
sudo easytier-core --ipv4 10.144.144.2 -n 10.1.1.0/24
```

子网代理信息会自动同步到虚拟网络的每个节点，各个节点会自动配置相应的路由，节点 A 可以通过如下命令检查子网代理是否生效。

1. 检查路由信息是否已经同步，proxy_cidrs 列展示了被代理的子网。

   ```sh
   easytier-cli route
   ```
   ![alt text](assets/image-3.png)

2. 测试节点 A 是否可访问被代理子网下的节点

   ```sh
   ping 10.1.1.2
   ```

---

### 无公网IP组网

EasyTier 支持共享公网节点进行组网。目前已部署共享的公网节点 ``tcp://easytier.public.kkrainbow.top:11010``。

使用共享节点时，需要每个入网节点提供相同的 ``--network-name`` 和 ``--network-secret`` 参数，作为网络的唯一标识。

以双节点为例，节点 A 执行：

```sh
sudo easytier-core -i 10.144.144.1 --network-name abc --network-secret abc -e 'tcp://easytier.public.kkrainbow.top:11010'
```

节点 B 执行

```sh
sudo easytier-core --ipv4 10.144.144.2 --network-name abc --network-secret abc -e 'tcp://easytier.public.kkrainbow.top:11010'
```

命令执行成功后，节点 A 即可通过虚拟 IP 10.144.144.2 访问节点 B。

---

### 其他配置

可使用 ``easytier-core --help`` 查看全部配置项


# 路线图

- [ ] 完善文档和用户指南。
- [ ] 支持 TCP 打洞等特性。
- [ ] 支持 Android、IOS 等移动平台。
- [ ] 支持 Web 配置管理。

# 社区和贡献

我们欢迎并鼓励社区贡献！如果你想参与进来，请提交 [GitHub PR](https://github.com/KKRainbow/EasyTier/pulls)。详细的贡献指南可以在 [CONTRIBUTING.md](https://github.com/KKRainbow/EasyTier/blob/main/CONTRIBUTING.md) 中找到。

# 相关项目和资源

- [ZeroTier](https://www.zerotier.com/): 一个全球虚拟网络，用于连接设备。
- [TailScale](https://tailscale.com/): 一个旨在简化网络配置的 VPN 解决方案。
- [vpncloud](https://github.com/dswd/vpncloud): 一个 P2P Mesh VPN

# 许可证

EasyTier 根据 [Apache License 2.0](https://github.com/KKRainbow/EasyTier/blob/main/LICENSE) 许可证发布。

# 联系方式

- 提问或报告问题：[GitHub Issues](https://github.com/KKRainbow/EasyTier/issues)
- 讨论和交流：[GitHub Discussions](https://github.com/KKRainbow/EasyTier/discussions)
- QQ 群： 949700262