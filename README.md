# 🚦 Linux TC 流量控制教程 / Linux Traffic Control (tc) Tutorial

> **掌握 Linux TC（Traffic Control）实现精确的网络流量控制——带宽限制、延迟模拟、优先级队列等**
> **Master Linux Traffic Control (tc) for precise network traffic management — bandwidth limiting, latency simulation, priority queuing, and more**

---

## 📋 目录 / Table of Contents

- [1. 概述 / Overview](#1-概述--overview)
- [2. 原理讲解 / Principles](#2-原理讲解--principles)
- [3. 环境准备 / Environment Setup](#3-环境准备--environment-setup)
- [4. TC 核心组件详解 / TC Core Components](#4-tc-核心组件详解--tc-core-components)
  - [4.1 QDisc 队列规则](#41-qdisc-队列规则)
  - [4.2 Class 流量分类](#42-class-流量分类)
  - [4.3 Filter 过滤器](#43-filter-过滤器)
- [5. 部署方式 / Deployment Methods](#5-部署方式--deployment-methods)
  - [5.1 原生方式 / Native Method](#51-原生方式--native-method)
  - [5.2 Docker 方式 / Docker Method](#52-docker-方式--docker-method)
- [6. 配置示例 / Configuration Examples](#6-配置示例--configuration-examples)
- [7. 常用命令 / Common Commands](#7-常用命令--common-commands)
- [8. 常见问题 / FAQ](#8-常见问题--faq)
- [9. 进阶场景 / Advanced Scenarios](#9-进阶场景--advanced-scenarios)
- [10. 参考资源 / References](#10-参考资源--references)
- [☕ 支持 / Support](#-支持--support)

---

## 1. 概述 / Overview

### 中文

**Linux TC（Traffic Control，流量控制）** 是 Linux 内核提供的一套强大的网络流量管理框架，用于控制数据包的发送和接收行为。通过 TC，你可以实现：

- 🚧 **带宽限制**：限制特定接口或流量的最大带宽
- ⏱ **延迟模拟**：模拟网络延迟，测试应用的高延迟表现
- 📦 **丢包模拟**：模拟网络丢包，验证应用的可靠性
- 📊 **优先级队列**：为关键业务流量分配更高的优先级
- 🔄 **流量整形**：平滑突发流量，使网络流量更符合预期规格
- 🧪 **网络仿真**：模拟各种网络条件（抖动、丢包、延迟）

TC 广泛应用于 QoS（服务质量）、网络测试、数据中心带宽管理、ISP 流量控制等场景。它是 Linux 网络管理员和 DevOps 工程师必备的核心技能。

**核心概念：**

| 概念 | 英文 | 说明 |
|------|------|------|
| 队列规则 | QDisc | 数据包排队和调度策略 |
| 分类 | Class | 将流量分组管理的层级结构 |
| 过滤器 | Filter | 根据规则将数据包分类到特定类 |
| 策略器 | Policing | 限制流量速率，超出则丢弃 |
| 整形器 | Shaping | 延迟流量以符合速率限制 |

### English

**Linux TC (Traffic Control)** is a powerful network traffic management framework integrated into the Linux kernel, used to control packet sending and receiving behavior. With TC, you can:

- 🚧 **Bandwidth Limiting**: Cap the maximum bandwidth for specific interfaces or traffic
- ⏱ **Latency Simulation**: Simulate network latency to test application behavior under high latency
- 📦 **Packet Loss Simulation**: Simulate packet loss to verify application reliability
- 📊 **Priority Queuing**: Assign higher priority to critical business traffic
- 🔄 **Traffic Shaping**: Smooth bursty traffic to conform to expected specifications
- 🧪 **Network Emulation**: Simulate various network conditions (jitter, loss, latency)

TC is widely used in QoS (Quality of Service), network testing, data center bandwidth management, ISP traffic control, and more. It's an essential skill for Linux network administrators and DevOps engineers.

**Core Concepts:**

| Concept | Description |
|---------|-------------|
| QDisc | Packet queuing and scheduling strategy |
| Class | Hierarchical structure grouping traffic |
| Filter | Classifies packets into specific classes based on rules |
| Policing | Limits traffic rate, drops excess packets |
| Shaping | Delays traffic to conform to rate limits |

---

## 2. 原理讲解 / Principles

### 中文

#### 2.1 TC 工作流程

Linux TC 在内核网络栈中工作，位于网络设备和协议栈之间。数据包在发送或接收时经过 TC 处理。

```
                   ┌──────────────────────────┐
                   │     应用程序 / App        │
                   └────────────┬─────────────┘
                                │
                   ┌────────────▼─────────────┐
                   │    协议栈 / Protocol Stack│
                   └────────────┬─────────────┘
                                │
             ┌──────────────────▼──────────────────┐
             │         TC 出口队列规则              │
             │   (Egress QDisc - 控制发送速率)       │
             │                                       │
             │   ┌─────┐ ┌─────┐ ┌─────┐            │
             │   │QDisc│ │QDisc│ │QDisc│ ← 过滤器    │
             │   └──┬──┘ └──┬──┘ └──┬──┘            │
             │      └───┬───┘        │               │
             │          ▼            ▼               │
             │      ┌────────┐  ┌────────┐          │
             │      │ Class  │  │ Class  │           │
             │      └────────┘  └────────┘          │
             └──────────────────┬──────────────────┘
                                │
             ┌──────────────────▼──────────────────┐
             │          网络设备 / NIC              │
             └─────────────────────────────────────┘
```

#### 2.2 数据包处理方向

| 方向 | 术语 | 说明 |
|------|------|------|
| **出口（发送）** | Egress / Outgoing | 数据包离开系统时被整形 |
| **入口（接收）** | Ingress / Incoming | 数据包进入系统时被策略化 |

TC 对 **出口流量** 做完整的队列管理（可以整形、分类），对 **入口流量** 只能做简单的策略（丢弃或标记）。

#### 2.3 常用 QDisc 类型

| QDisc 类型 | 全称 | 特点 | 适用场景 |
|------------|------|------|---------|
| **pfifo_fast** | 快速先进先出 | 默认 QDisc，3 个优先级波段 | 一般用途 |
| **HTB** | 层级令牌桶 | 支持带宽限制和层级分类 | **企业带宽管理** |
| **TBF** | 令牌桶过滤器 | 简单速率限制 | 固定速率限制 |
| **RED** | 随机早期检测 | 主动队列管理，避免拥塞 | 拥塞控制 |
| **SFQ** | 随机公平队列 | 公平分配带宽给各个会话 | 公平带宽分配 |
| **FQ_Codel** | 公平队列控制延迟 | 低延迟、低抖动 | 现代网络优化 |
| **netem** | 网络仿真器 | 模拟延迟、丢包、抖动 | **网络测试/仿真** |
| **prio** | 优先级队列 | 严格优先级调度 | QoS 优先级 |

#### 2.4 HTB 工作原理

HTB (Hierarchical Token Bucket) 是最常用的流量整形 QDisc，支持层级化的带宽分配：

```
HTB Root (总带宽 1000mbit)
│
├── class 1:1 (默认类)
│
├── class 1:10 (保证 200mbit, 最高 500mbit)
│   └── QDisc (sfq 或 pfifo)
│
├── class 1:20 (保证 300mbit, 最高 500mbit)
│   └── QDisc (sfq 或 pfifo)
│
└── class 1:30 (保证 100mbit, 最高 200mbit)
    └── QDisc (sfq 或 pfifo)
```

- **rate**: 保证带宽（承诺信息速率 CIR）
- **ceil**: 最大带宽（峰值信息速率 PIR），可借用父类的空闲带宽
- **burst**: 突发大小，允许短时间超过 rate 的传输量

### English

#### 2.1 TC Workflow

Linux TC operates in the kernel network stack, between the network device and the protocol stack. Packets pass through TC processing when being sent or received.

#### 2.2 Packet Processing Direction

| Direction | Term | Description |
|-----------|------|-------------|
| **Outgoing** | Egress | Packets are shaped when leaving the system |
| **Incoming** | Ingress | Packets are policed when entering the system |

TC performs full queue management (shaping, classification) for **egress traffic**, but only simple policing (drop or mark) for **ingress traffic**.

#### 2.3 Common QDisc Types

| QDisc Type | Full Name | Features | Use Case |
|------------|-----------|----------|----------|
| **pfifo_fast** | Fast FIFO | Default QDisc, 3 priority bands | General purpose |
| **HTB** | Hierarchical Token Bucket | Bandwidth limiting with hierarchy | **Enterprise bandwidth mgmt** |
| **TBF** | Token Bucket Filter | Simple rate limiting | Fixed rate limits |
| **RED** | Random Early Detection | Active queue management | Congestion control |
| **SFQ** | Stochastic Fair Queuing | Fair bandwidth distribution | Fair sharing |
| **FQ_Codel** | Fair Queuing Controlled Delay | Low latency, low jitter | Modern network optimization |
| **netem** | Network Emulator | Simulates delay, loss, jitter | **Network testing/emulation** |
| **prio** | Priority Queue | Strict priority scheduling | QoS priorities |

#### 2.4 HTB Working Principle

HTB (Hierarchical Token Bucket) is the most commonly used traffic shaping QDisc, supporting hierarchical bandwidth allocation:

- **rate**: Guaranteed bandwidth (Committed Information Rate, CIR)
- **ceil**: Maximum bandwidth (Peak Information Rate, PIR), can borrow idle bandwidth from parent class
- **burst**: Burst size, allows short-term transmission exceeding rate

---

## 3. 环境准备 / Environment Setup

### 中文

#### 系统要求

- **操作系统**: Linux (kernel ≥ 2.6，推荐 ≥ 4.19)
- **必要工具**: `tc` (iproute2 包)、`iptables`、`iperf3`、`ping`
- **权限**: root 或 sudo 权限（TC 操作需要管理员权限）

#### 安装 TC 工具

```bash
# Debian/Ubuntu
apt-get update && apt-get install -y iproute2 iperf3 net-tools

# CentOS/RHEL/Fedora
yum install -y iproute iperf3 net-tools

# Alpine
apk add iproute2 iperf3
```

#### 检查 TC 支持

```bash
# 检查 TC 是否可用
tc -V
# 输出示例: tc iproute2-6.1.0

# 查看当前网络接口
ip link show
# 注意接口名称（如 eth0, ens33, wlp2s0 等）

# 查看当前接口上的 QDisc
tc qdisc show dev eth0
# 输出示例: qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```

#### 安装 iperf3（用于带宽测试）

```bash
# 服务端（默认端口 5201）
iperf3 -s

# 客户端
iperf3 -c <server_ip> -t 30 -i 1
# -t 30: 测试 30 秒
# -i 1: 每秒报告一次
```

### English

#### System Requirements

- **OS**: Linux (kernel ≥ 2.6, recommended ≥ 4.19)
- **Required Tools**: `tc` (iproute2 package), `iptables`, `iperf3`, `ping`
- **Privileges**: root or sudo (TC operations require admin privileges)

#### Install TC

```bash
# Debian/Ubuntu
apt-get update && apt-get install -y iproute2 iperf3 net-tools

# CentOS/RHEL/Fedora
yum install -y iproute iperf3 net-tools

# Alpine
apk add iproute2 iperf3
```

#### Verify TC Support

```bash
# Check TC availability
tc -V

# View current network interfaces
ip link show

# View current QDisc on an interface
tc qdisc show dev eth0
```

---

## 4. TC 核心组件详解 / TC Core Components

### 4.1 QDisc 队列规则

#### 中文

QDisc（Queueing Discipline，队列规则）是 TC 的核心调度器。它决定数据包如何排队以及何时发送。

##### pfifo_fast（默认 QDisc）

```bash
# 查看默认 QDisc
tc qdisc show dev eth0

# pfifo_fast 的优先级映射
# priomap: 3 个 band
# band 0: 最高优先级（交互流量）
# band 1: 普通优先级
# band 2: 尽力而为（批量流量）
```

##### HTB（推荐用于带宽管理）

```bash
# 在 eth0 上设置 HTB QDisc
tc qdisc add dev eth0 root handle 1: htb default 30

# 说明：
# - root: 根 QDisc
# - handle 1:: 句柄（父:子）
# - default 30: 未分类的流量进入 class 1:30
```

##### TBF（简单速率限制）

```bash
# 限制 eth0 发送速率为 1mbit
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 50ms

# 参数说明：
# - rate: 速率限制
# - burst: 突发大小
# - latency: 延迟上限（超出的包被丢弃）
```

##### netem（网络仿真）

```bash
# 添加 100ms 延迟
tc qdisc add dev eth0 root netem delay 100ms

# 添加延迟 + 抖动
tc qdisc add dev eth0 root netem delay 100ms 20ms

# 添加丢包
tc qdisc add dev eth0 root netem loss 5%

# 添加重复包
tc qdisc add dev eth0 root netem duplicate 1%

# 添加包损坏
tc qdisc add dev eth0 root netem corrupt 0.1%

# 组合多个参数
tc qdisc add dev eth0 root netem delay 50ms 10ms loss 2% duplicate 0.5%
```

### English

#### 4.1 QDisc (Queueing Discipline)

QDisc is the core scheduler of TC. It determines how packets are queued and when they are sent.

##### pfifo_fast (Default QDisc)

```bash
# View default QDisc
tc qdisc show dev eth0

# pfifo_fast priority bands:
# band 0: Highest priority (interactive traffic)
# band 1: Normal priority
# band 2: Best effort (bulk traffic)
```

##### HTB (Recommended for Bandwidth Management)

```bash
# Set HTB QDisc on eth0
tc qdisc add dev eth0 root handle 1: htb default 30

# Explanation:
# - root: Root QDisc
# - handle 1:: Handle (parent:child)
# - default 30: Unclassified traffic goes to class 1:30
```

##### TBF (Simple Rate Limiting)

```bash
# Limit eth0 send rate to 1mbit
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 50ms
```

##### netem (Network Emulation)

```bash
# Add 100ms latency
tc qdisc add dev eth0 root netem delay 100ms

# Add latency + jitter
tc qdisc add dev eth0 root netem delay 100ms 20ms

# Add packet loss
tc qdisc add dev eth0 root netem loss 5%

# Add duplicate packets
tc qdisc add dev eth0 root netem duplicate 1%

# Add packet corruption
tc qdisc add dev eth0 root netem corrupt 0.1%

# Combine multiple parameters
tc qdisc add dev eth0 root netem delay 50ms 10ms loss 2% duplicate 0.5%
```

### 4.2 Class 流量分类 / Class (Traffic Classification)

#### 中文

Class 用于将流量分层管理，每个 Class 可以有自己的 QDisc。

```bash
# 创建 HTB 根 QDisc
tc qdisc add dev eth0 root handle 1: htb

# 创建子类
# classid 格式: <父句柄>:<子编号>

# 默认类（1:30）
tc class add dev eth0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit

# Web 流量 (20% 保证, 50% 最大)
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 200mbit ceil 500mbit

# 数据库流量 (30% 保证, 80% 最大)
tc class add dev eth0 parent 1:1 classid 1:20 htb rate 300mbit ceil 800mbit

# 其他流量 (10% 保证, 20% 最大)
tc class add dev eth0 parent 1:1 classid 1:30 htb rate 100mbit ceil 200mbit
```

**HTB 关键参数：**

| 参数 | 含义 | 说明 |
|------|------|------|
| `rate` | 保证速率 | 该类至少能获得的带宽 |
| `ceil` | 最大速率 | 该类最多能使用的带宽 |
| `burst` | 突发大小 | 允许瞬时超过 rate 的量 |
| `cburst` | 最大突发 | 允许瞬时超过 ceil 的量 |
| `quantum` | 量子值 | 每次轮询发送的字节数 |

### English

#### 4.2 Class (Traffic Classification)

Classes are used to manage traffic hierarchically. Each class can have its own QDisc.

```bash
# Create HTB root QDisc
tc qdisc add dev eth0 root handle 1: htb

# Create child classes
tc class add dev eth0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit

# Web traffic (20% guaranteed, 50% max)
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 200mbit ceil 500mbit

# Database traffic (30% guaranteed, 80% max)
tc class add dev eth0 parent 1:1 classid 1:20 htb rate 300mbit ceil 800mbit

# Other traffic (10% guaranteed, 20% max)
tc class add dev eth0 parent 1:1 classid 1:30 htb rate 100mbit ceil 200mbit
```

**HTB Key Parameters:**

| Parameter | Meaning | Description |
|-----------|---------|-------------|
| `rate` | Guaranteed rate | Minimum bandwidth this class will receive |
| `ceil` | Maximum rate | Maximum bandwidth this class can use |
| `burst` | Burst size | Amount of traffic allowed to exceed rate momentarily |
| `cburst` | Maximum burst | Amount of traffic allowed to exceed ceil momentarily |
| `quantum` | Quantum value | Bytes sent per round-robin poll |

### 4.3 Filter 过滤器 / Filter

#### 中文

Filter 将数据包分类到指定的 Class。TC 支持多种过滤器类型：

| 过滤器类型 | 匹配方式 | 适用场景 |
|-----------|---------|---------|
| **u32** | 按 IP/端口等包头信息 | 通用 IP 流量分类 |
| **fw** | 按 iptables 设置的 mark | 配合 iptables 使用 |
| **cgroup** | 按 cgroup 分类 | 容器流量控制 |
| **flower** | 按多维匹配 | OpenFlow 风格 |
| **bpf** | eBPF 程序 | 高性能自定义匹配 |

##### u32 过滤器示例

```bash
# 将目标端口 80 的流量路由到 class 1:10
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
    match ip dport 80 0xffff \
    flowid 1:10

# 将目标端口 443 的流量路由到 class 1:10
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
    match ip dport 443 0xffff \
    flowid 1:10

# 将源 IP 为 192.168.1.0/24 的流量路由到 class 1:20
tc filter add dev eth0 protocol ip parent 1:0 prio 2 u32 \
    match ip src 192.168.1.0/24 \
    flowid 1:20

# 将特定源端口 3306 的流量路由到 class 1:20
tc filter add dev eth0 protocol ip parent 1:0 prio 2 u32 \
    match ip protocol 6 0xff \
    match ip dport 3306 0xffff \
    flowid 1:20

# 多条件组合（源 IP + 目标端口）
tc filter add dev eth0 protocol ip parent 1:0 prio 3 u32 \
    match ip src 10.0.0.0/8 \
    match ip dport 22 0xffff \
    flowid 1:30
```

##### fw (firewall mark) 过滤器

```bash
# 1. iptables 设置 mark
iptables -t mangle -A POSTROUTING -p tcp --dport 80 -j MARK --set-mark 10
iptables -t mangle -A POSTROUTING -p tcp --dport 443 -j MARK --set-mark 10

# 2. TC 根据 mark 分类
tc filter add dev eth0 protocol ip parent 1:0 prio 1 handle 10 fw flowid 1:10

# 更复杂的 mark 示例
iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -j MARK --set-mark 20
tc filter add dev eth0 protocol ip parent 1:0 prio 2 handle 20 fw flowid 1:20
```

### English

#### 4.3 Filter

Filters classify packets into specific classes. TC supports multiple filter types:

| Filter Type | Matching Method | Use Case |
|------------|----------------|----------|
| **u32** | IP/port based header matching | General IP traffic classification |
| **fw** | iptables firewall mark | Integration with iptables |
| **cgroup** | cgroup classification | Container traffic control |
| **flower** | Multi-dimensional matching | OpenFlow-style |
| **bpf** | eBPF programs | High-performance custom matching |

##### u32 Filter Examples

```bash
# Route port 80 traffic to class 1:10
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
    match ip dport 80 0xffff \
    flowid 1:10

# Route port 443 traffic to class 1:10
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
    match ip dport 443 0xffff \
    flowid 1:10

# Route traffic from 192.168.1.0/24 to class 1:20
tc filter add dev eth0 protocol ip parent 1:0 prio 2 u32 \
    match ip src 192.168.1.0/24 \
    flowid 1:20
```

##### fw (Firewall Mark) Filter

```bash
# 1. Set mark in iptables
iptables -t mangle -A POSTROUTING -p tcp --dport 80 -j MARK --set-mark 10
iptables -t mangle -A POSTROUTING -p tcp --dport 443 -j MARK --set-mark 10

# 2. TC classifies based on mark
tc filter add dev eth0 protocol ip parent 1:0 prio 1 handle 10 fw flowid 1:10
```

---

## 5. 部署方式 / Deployment Methods

### 5.1 原生方式 / Native Method

#### 中文

原生方式直接在 Linux 系统上配置 TC，适合生产环境和服务器的流量管理。

##### 完整的 TC 配置脚本

```bash
#!/bin/bash
# 文件名: tc_setup.sh
# 功能：在 eth0 上配置多层 HTB 流量控制

INTERFACE="eth0"
# 如不确定接口名，使用: INTERFACE=$(ip route get 8.8.8.8 | awk '{print $5}')

# 清除现有配置
tc qdisc del dev $INTERFACE root 2>/dev/null || true

# 1. 创建根 QDisc (HTB)
tc qdisc add dev $INTERFACE root handle 1: htb

# 2. 创建根类（总带宽 1000mbit，根据实际情况调整）
tc class add dev $INTERFACE parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit

# 3. 创建子类

# --- 高优先级：交互/管理流量 (10% 保证, 20% 最大) ---
tc class add dev $INTERFACE parent 1:1 classid 1:10 htb rate 100mbit ceil 200mbit
tc qdisc add dev $INTERFACE parent 1:10 handle 10: sfq perturb 10

# --- 中优先级：Web 服务流量 (40% 保证, 60% 最大) ---
tc class add dev $INTERFACE parent 1:1 classid 1:20 htb rate 400mbit ceil 600mbit
tc qdisc add dev $INTERFACE parent 1:20 handle 20: sfq perturb 10

# --- 低优先级：批量下载/P2P (10% 保证, 30% 最大) ---
tc class add dev $INTERFACE parent 1:1 classid 1:30 htb rate 100mbit ceil 300mbit
tc qdisc add dev $INTERFACE parent 1:30 handle 30: sfq perturb 10

# --- 默认类：其他流量 (20% 保证, 40% 最大) ---
tc class add dev $INTERFACE parent 1:1 classid 1:40 htb rate 200mbit ceil 400mbit
tc qdisc add dev $INTERFACE parent 1:40 handle 40: sfq perturb 10

# 4. 配置过滤器

# SSH (端口 22) -> 高优先级
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 1 u32 \
    match ip dport 22 0xffff flowid 1:10

# DNS (端口 53) -> 高优先级
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 1 u32 \
    match ip dport 53 0xffff flowid 1:10

# HTTP/HTTPS (端口 80, 443) -> 中优先级
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 2 u32 \
    match ip dport 80 0xffff flowid 1:20
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 2 u32 \
    match ip dport 443 0xffff flowid 1:20

# ICMP -> 高优先级
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 1 u32 \
    match ip protocol 1 0xff flowid 1:10

echo "=== TC Configuration Applied ==="
tc -s qdisc show dev $INTERFACE
tc -s class show dev $INTERFACE
```

##### 清除 TC 配置

```bash
#!/bin/bash
# 文件名: tc_cleanup.sh

INTERFACE="eth0"

# 删除根 QDisc（会自动删除所有子类和过滤器）
tc qdisc del dev $INTERFACE root 2>/dev/null && echo "TC config cleared for $INTERFACE" || echo "No TC config found"
```

##### systemd 服务管理 TC

```ini
# /etc/systemd/system/tc-qos.service
[Unit]
Description=TC QoS Configuration
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/tc_setup.sh
ExecStop=/usr/local/bin/tc_cleanup.sh

[Install]
WantedBy=multi-user.target
```

```bash
# 启用服务
chmod +x /usr/local/bin/tc_setup.sh /usr/local/bin/tc_cleanup.sh
systemctl daemon-reload
systemctl enable --now tc-qos.service
```

### English

#### 5.1 Native Method

The native method configures TC directly on the Linux system, suitable for production environments and server traffic management.

**Complete TC Configuration Script:**

```bash
#!/bin/bash
# File: tc_setup.sh
# Purpose: Configure multi-layer HTB traffic control on eth0

INTERFACE="eth0"

# Clear existing configuration
tc qdisc del dev $INTERFACE root 2>/dev/null || true

# 1. Create root QDisc (HTB)
tc qdisc add dev $INTERFACE root handle 1: htb

# 2. Create root class (total bandwidth 1000mbit)
tc class add dev $INTERFACE parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit

# 3. Create sub-classes

# --- High priority: Interactive/admin traffic (10% guaranteed, 20% max) ---
tc class add dev $INTERFACE parent 1:1 classid 1:10 htb rate 100mbit ceil 200mbit
tc qdisc add dev $INTERFACE parent 1:10 handle 10: sfq perturb 10

# --- Medium priority: Web service traffic (40% guaranteed, 60% max) ---
tc class add dev $INTERFACE parent 1:1 classid 1:20 htb rate 400mbit ceil 600mbit
tc qdisc add dev $INTERFACE parent 1:20 handle 20: sfq perturb 10

# --- Low priority: Bulk download/P2P (10% guaranteed, 30% max) ---
tc class add dev $INTERFACE parent 1:1 classid 1:30 htb rate 100mbit ceil 300mbit
tc qdisc add dev $INTERFACE parent 1:30 handle 30: sfq perturb 10

# --- Default class: Other traffic (20% guaranteed, 40% max) ---
tc class add dev $INTERFACE parent 1:1 classid 1:40 htb rate 200mbit ceil 400mbit
tc qdisc add dev $INTERFACE parent 1:40 handle 40: sfq perturb 10

# 4. Configure filters
# SSH (port 22) -> High priority
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 1 u32 \
    match ip dport 22 0xffff flowid 1:10

# DNS (port 53) -> High priority
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 1 u32 \
    match ip dport 53 0xffff flowid 1:10

# HTTP/HTTPS (port 80, 443) -> Medium priority
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 2 u32 \
    match ip dport 80 0xffff flowid 1:20
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 2 u32 \
    match ip dport 443 0xffff flowid 1:20

# ICMP -> High priority
tc filter add dev $INTERFACE protocol ip parent 1:0 prio 1 u32 \
    match ip protocol 1 0xff flowid 1:10

echo "=== TC Configuration Applied ==="
```

### 5.2 Docker 方式 / Docker Method

#### 中文

Docker 容器也可以应用 TC 规则进行流量控制，适用于测试和微服务流量管理。

##### 在容器内使用 TC

```bash
# 运行一个容器并给予 NET_ADMIN 权限
docker run -d --name net-test --cap-add=NET_ADMIN alpine sleep 3600

# 进入容器
docker exec -it net-test sh

# 在容器内查看网络接口
ip addr
# 通常是 eth0@if<N>

# 给容器内的 eth0 添加延迟
tc qdisc add dev eth0 root netem delay 200ms

# 验证
ping google.com -c 5

# 限制带宽
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 50ms
```

##### 使用 Docker Compose 进行网络测试

```yaml
# docker-compose.yml
version: '3.8'

services:
  server:
    image: nginx:alpine
    networks:
      - test-net
    cap_add:
      - NET_ADMIN

  client:
    image: alpine
    command: sh -c "apk add curl iperf3 && sleep 3600"
    networks:
      - test-net
    cap_add:
      - NET_ADMIN
    depends_on:
      - server

  router:
    image: alpine
    command: sh -c "
      echo 1 > /proc/sys/net/ipv4/ip_forward &&
      apk add iptables iproute2 &&
      iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE &&
      sleep 3600
    "
    cap_add:
      - NET_ADMIN
      - NET_RAW
    networks:
      - test-net
    ports:
      - "8080:80"

networks:
  test-net:
    driver: bridge
```

##### 在宿主机上控制容器流量

```bash
# 找到容器的 veth 接口
CONTAINER_ID=$(docker ps -ql)
VETH_HOST=$(docker exec $CONTAINER_ID cat /sys/class/net/eth0/iflink)
VETH_IFACE=$(ip link | grep "^${VETH_HOST}:" | awk '{print $2}' | tr -d '@:')

# 在宿主机的 veth 接口上设置 TC
tc qdisc add dev $VETH_IFACE root netem delay 100ms loss 2%

# 查看
tc qdisc show dev $VETH_IFACE
```

### English

#### 5.2 Docker Method

Docker containers can also apply TC rules for traffic control, useful for testing and microservice traffic management.

```bash
# Run a container with NET_ADMIN capability
docker run -d --name net-test --cap-add=NET_ADMIN alpine sleep 3600

# Enter container
docker exec -it net-test sh

# Add latency inside container
tc qdisc add dev eth0 root netem delay 200ms

# Verify
ping google.com -c 5

# Limit bandwidth
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 50ms
```

**Controlling Container Traffic from Host:**

```bash
# Find container's veth interface
CONTAINER_ID=$(docker ps -ql)
VETH_HOST=$(docker exec $CONTAINER_ID cat /sys/class/net/eth0/iflink)
VETH_IFACE=$(ip link | grep "^${VETH_HOST}:" | awk '{print $2}' | tr -d '@:')

# Apply TC on host's veth interface
tc qdisc add dev $VETH_IFACE root netem delay 100ms loss 2%
```

---

## 6. 配置示例 / Configuration Examples

### 中文

#### 示例 1：办公室 QoS — 确保 VoIP 和视频会议质量

```bash
#!/bin/bash
# 办公室 QoS：VoIP (端口 5060, 10000-20000) + 视频会议 + 普通上网

IFACE="eth0"

tc qdisc del dev $IFACE root 2>/dev/null || true

# Root QDisc
tc qdisc add dev $IFACE root handle 1: htb
tc class add dev $IFACE parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit

# VoIP (最高优先级, 保证 100kbps)
tc class add dev $IFACE parent 1:1 classid 1:10 htb rate 100kbit ceil 500kbit
tc qdisc add dev $IFACE parent 1:10 handle 10: pfifo limit 50

# 视频会议 (高优先级, 保证 2mbps)
tc class add dev $IFACE parent 1:1 classid 1:20 htb rate 2mbit ceil 5mbit
tc qdisc add dev $IFACE parent 1:20 handle 20: pfifo limit 200

# Web 和邮件 (中优先级)
tc class add dev $IFACE parent 1:1 classid 1:30 htb rate 100mbit ceil 500mbit
tc qdisc add dev $IFACE parent 1:30 handle 30: sfq perturb 10

# 下载/P2P (低优先级)
tc class add dev $IFACE parent 1:1 classid 1:40 htb rate 10mbit ceil 100mbit
tc qdisc add dev $IFACE parent 1:40 handle 40: sfq perturb 10

# 过滤器
# VoIP SIP 信令
tc filter add dev $IFACE protocol ip parent 1:0 prio 1 u32 \
    match ip dport 5060 0xffff flowid 1:10
tc filter add dev $IFACE protocol ip parent 1:0 prio 1 u32 \
    match ip dport 5061 0xffff flowid 1:10

# VoIP RTP 媒体流
tc filter add dev $IFACE protocol ip parent 1:0 prio 1 u32 \
    match ip sport 10000 0xffc0 flowid 1:10

# 视频会议 (Zoom/Teams/Webex 端口范围)
tc filter add dev $IFACE protocol ip parent 1:0 prio 2 u32 \
    match ip sport 3478 0xffff flowid 1:20
tc filter add dev $IFACE protocol ip parent 1:0 prio 2 u32 \
    match ip dport 3478 0xffff flowid 1:20
tc filter add dev $IFACE protocol ip parent 1:0 prio 2 u32 \
    match ip sport 8801 0xffff flowid 1:20
tc filter add dev $IFACE protocol ip parent 1:0 prio 2 u32 \
    match ip dport 8801 0xffff flowid 1:20

# 小包（ACK、SYN 等控制包）-> 高优先级
tc filter add dev $IFACE protocol ip parent 1:0 prio 1 u32 \
    match ip protocol 6 0xff \
    match u8 0x05 0x0f at 0 \
    match u16 0x0000 0xffc0 at 2 \
    flowid 1:10
```

#### 示例 2：网络模拟测试

```bash
#!/bin/bash
# 模拟不同网络质量

# 3G 网络 (高延迟, 适度丢包)
setup_3g() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    tc qdisc add dev eth0 root netem delay 100ms 20ms loss 3% duplicate 0.5%
    echo "3G Network: delay 100ms±20ms, loss 3%"
}

# 卫星网络 (极高延迟)
setup_satellite() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    tc qdisc add dev eth0 root netem delay 600ms 100ms loss 1%
    echo "Satellite Network: delay 600ms±100ms, loss 1%"
}

# 拥塞网络
setup_congested() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    # 限制带宽 + 高延迟
    tc qdisc add dev eth0 root handle 1: htb default 30
    tc class add dev eth0 parent 1: classid 1:1 htb rate 5mbit ceil 10mbit
    tc qdisc add dev eth0 parent 1:1 handle 10: netem delay 50ms 10ms loss 5%
    echo "Congested Network: 5mbit, delay 50ms±10ms, loss 5%"
}

# 完美网络 (无损耗)
setup_perfect() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    echo "Perfect Network: no restrictions"
}

# 企业网络
setup_corporate() {
    # 清除
    tc qdisc del dev eth0 root 2>/dev/null || true

    # 限速 + 低延迟
    tc qdisc add dev eth0 root handle 1: htb default 30
    tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit ceil 100mbit

    # 正常流量
    tc class add dev eth0 parent 1:1 classid 1:10 htb rate 80mbit ceil 100mbit
    tc qdisc add dev eth0 parent 1:10 handle 10: netem delay 5ms 1ms

    # 批量传输 (限速)
    tc class add dev eth0 parent 1:1 classid 1:30 htb rate 20mbit ceil 30mbit
    tc qdisc add dev eth0 parent 1:30 handle 30: netem delay 10ms 2ms

    echo "Corporate Network: 100mbit, delay 5ms±1ms"
}

# 使用示例
# setup_3g
# setup_satellite
# setup_congested
# create_corporate
# setup_perfect
```

#### 示例 3：多重带宽限制

```bash
#!/bin/bash
# 为不同用户或 VLAN 分配带宽

# 创建 VLAN 接口（假设物理接口为 eth0）
ip link add link eth0 name eth0.100 type vlan id 100
ip link add link eth0 name eth0.200 type vlan id 200
ip link set eth0.100 up
ip link set eth0.200 up

# 为 VLAN 100 配置带宽限制
tc qdisc add dev eth0.100 root handle 1: htb default 30
tc class add dev eth0.100 parent 1: classid 1:1 htb rate 500mbit ceil 500mbit
tc class add dev eth0.100 parent 1:1 classid 1:10 htb rate 200mbit ceil 500mbit
tc class add dev eth0.100 parent 1:1 classid 1:20 htb rate 150mbit ceil 500mbit
tc class add dev eth0.100 parent 1:1 classid 1:30 htb rate 150mbit ceil 500mbit

# 为 VLAN 200 配置带宽限制（更严格）
tc qdisc add dev eth0.200 root handle 1: htb default 30
tc class add dev eth0.200 parent 1: classid 1:1 htb rate 200mbit ceil 200mbit
tc class add dev eth0.200 parent 1:1 classid 1:10 htb rate 100mbit ceil 200mbit
tc class add dev eth0.200 parent 1:1 classid 1:20 htb rate 50mbit ceil 200mbit
tc class add dev eth0.200 parent 1:1 classid 1:30 htb rate 50mbit ceil 200mbit

echo "VLAN bandwidth limits applied:"
echo "  VLAN 100: 500mbit total"
echo "  VLAN 200: 200mbit total"
```

#### 示例 4：IMQ 实现入口流量整形

```bash
# 注意：IMQ 需要内核模块支持
modprobe imq
ip link set imq0 up

# 将入口流量重定向到 IMQ
iptables -t mangle -A PREROUTING -j IMQ --todev 0

# 在 IMQ 上配置入口带宽限制
tc qdisc add dev imq0 root handle 1: htb default 30
tc class add dev imq0 parent 1: classid 1:1 htb rate 100mbit ceil 100mbit
tc class add dev imq0 parent 1:1 classid 1:10 htb rate 20mbit ceil 50mbit
tc qdisc add dev imq0 parent 1:10 handle 10: sfq perturb 10
```

### English

#### Example 1: Office QoS — Ensure VoIP and Video Conference Quality

```bash
#!/bin/bash
# Office QoS: VoIP (port 5060, 10000-20000) + Video Conference + General Web

IFACE="eth0"

tc qdisc del dev $IFACE root 2>/dev/null || true

# Root QDisc
tc qdisc add dev $IFACE root handle 1: htb
tc class add dev $IFACE parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit

# VoIP (highest priority, 100kbps guaranteed)
tc class add dev $IFACE parent 1:1 classid 1:10 htb rate 100kbit ceil 500kbit
tc qdisc add dev $IFACE parent 1:10 handle 10: pfifo limit 50

# Video Conference (high priority, 2mbps guaranteed)
tc class add dev $IFACE parent 1:1 classid 1:20 htb rate 2mbit ceil 5mbit
tc qdisc add dev $IFACE parent 1:20 handle 20: pfifo limit 200

# Web and Mail (medium priority)
tc class add dev $IFACE parent 1:1 classid 1:30 htb rate 100mbit ceil 500mbit
tc qdisc add dev $IFACE parent 1:30 handle 30: sfq perturb 10

# Download/P2P (low priority)
tc class add dev $IFACE parent 1:1 classid 1:40 htb rate 10mbit ceil 100mbit
tc qdisc add dev $IFACE parent 1:40 handle 40: sfq perturb 10
```

#### Example 2: Network Simulation Testing

```bash
#!/bin/bash
# Simulate different network qualities

# 3G Network (high latency, moderate loss)
setup_3g() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    tc qdisc add dev eth0 root netem delay 100ms 20ms loss 3% duplicate 0.5%
    echo "3G Network: delay 100ms±20ms, loss 3%"
}

# Satellite Network (very high latency)
setup_satellite() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    tc qdisc add dev eth0 root netem delay 600ms 100ms loss 1%
    echo "Satellite Network: delay 600ms±100ms, loss 1%"
}

# Congested Network
setup_congested() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    tc qdisc add dev eth0 root handle 1: htb default 30
    tc class add dev eth0 parent 1: classid 1:1 htb rate 5mbit ceil 10mbit
    tc qdisc add dev eth0 parent 1:1 handle 10: netem delay 50ms 10ms loss 5%
    echo "Congested Network: 5mbit, delay 50ms±10ms, loss 5%"
}

# Perfect Network (no restrictions)
setup_perfect() {
    tc qdisc del dev eth0 root 2>/dev/null || true
    echo "Perfect Network: no restrictions"
}
```

---

## 7. 常用命令 / Common Commands

### 中文

| 命令 | 说明 |
|------|------|
| `tc qdisc show` | 查看所有 QDisc |
| `tc qdisc show dev eth0` | 查看 eth0 的 QDisc |
| `tc qdisc add dev eth0 root <type>` | 添加 QDisc |
| `tc qdisc del dev eth0 root` | 删除 QDisc |
| `tc class show dev eth0` | 查看 eth0 的分类 |
| `tc class add dev eth0 parent ...` | 添加分类 |
| `tc filter show dev eth0` | 查看 eth0 的过滤器 |
| `tc filter add dev eth0 ...` | 添加过滤器 |
| `tc -s qdisc show dev eth0` | 查看 QDisc 统计信息 |
| `tc -s class show dev eth0` | 查看 Class 统计信息 |
| `tc qdisc add dev eth0 root netem delay 100ms` | 添加 100ms 延迟 |
| `tc qdisc add dev eth0 root netem loss 5%` | 添加 5% 丢包 |
| `tc qdisc add dev eth0 root tbf rate 1mbit ...` | 限制速率 1mbit |
| `tc qdisc add dev eth0 root handle 1: htb` | 创建 HTB 根 QDisc |
| `tc -p -d -s qdisc show` | 详细查看 QDisc 状态 |
| `tc monitor` | 监控 TC 事件 |

### English

| Command | Description |
|---------|-------------|
| `tc qdisc show` | Show all QDiscs |
| `tc qdisc show dev eth0` | Show QDisc on eth0 |
| `tc qdisc add dev eth0 root <type>` | Add a QDisc |
| `tc qdisc del dev eth0 root` | Delete a QDisc |
| `tc class show dev eth0` | Show classes on eth0 |
| `tc class add dev eth0 parent ...` | Add a class |
| `tc filter show dev eth0` | Show filters on eth0 |
| `tc filter add dev eth0 ...` | Add a filter |
| `tc -s qdisc show dev eth0` | Show QDisc statistics |
| `tc -s class show dev eth0` | Show class statistics |
| `tc qdisc add dev eth0 root netem delay 100ms` | Add 100ms latency |
| `tc qdisc add dev eth0 root netem loss 5%` | Add 5% packet loss |
| `tc qdisc add dev eth0 root tbf rate 1mbit ...` | Limit to 1mbit rate |
| `tc qdisc add dev eth0 root handle 1: htb` | Create HTB root QDisc |

---

## 8. 常见问题 / FAQ

### 中文

#### Q1: 为什么我的 TC 配置没有生效？

**常见原因：**

| 原因 | 解决方案 |
|------|---------|
| 接口名称错误 | 用 `ip link show` 确认正确的接口名 |
| 未使用 root 权限 | 所有 TC 命令需要 root 执行 |
| 配置顺序错误 | 必须先创建父类，再创建子类 |
| 过滤器优先级冲突 | 检查 prio 参数，确保正确排序 |
| 硬件 offload 干扰 | 禁用 TCP 校验和 offload: `ethtool -K eth0 tx off rx off` |

#### Q2: 如何验证 TC 配置的效果？

```bash
# 查看统计信息
tc -s qdisc show dev eth0
tc -s class show dev eth0

# 带宽测试
iperf3 -c <server> -t 30

# 延迟测试
ping -c 20 <target>

# MTR 测试
mtr <target>

# 实时监控
watch -n 1 'tc -s class show dev eth0'
```

#### Q3: TC 配置在重启后会丢失吗？

是的，TC 配置是临时性的，重启后丢失。要持久化配置，可以：
- 使用 systemd 服务（见第 5 节）
- 将命令写入 `/etc/rc.local`
- 使用网络管理工具如 NetworkManager 的 dispatcher 脚本

#### Q4: HTB 的 rate 和 ceil 有什么区别？

- `rate`：保证的最小带宽，类在任何时候都能获得的带宽
- `ceil`：最大带宽限制，当父类有空闲带宽时可以借用达到的最大值

当 `rate == ceil` 时，该类被严格限制在固定带宽。

#### Q5: 如何同时对多个接口应用相同的 TC 规则？

使用 shell 循环或脚本：

```bash
for iface in eth0 eth1 eth2; do
    tc qdisc add dev $iface root handle 1: htb default 30
    tc class add dev $iface parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
    # ... 更多配置
done
```

#### Q6: netem 和 tc 其他 QDisc 能一起使用吗？

可以。HTB 的叶子类可以附加 netem QDisc：

```bash
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 100mbit ceil 200mbit
# 在 class 1:10 上附加 netem
tc qdisc add dev eth0 parent 1:10 handle 10: netem delay 100ms loss 2%
```

#### Q7: 为什么 iperf3 测试结果和 TC 设置不一致？

可能原因：
- 测试时间太短（建议 ≥ 30 秒）
- 使用了 UDP 测试但设置的速率针对 TCP
- 硬件 offload 干扰测试结果
- 链路上有多个网络设备

#### Q8: 如何查看已使用 TC 的详细统计？

```bash
# 查看分包统计
tc -s qdisc show dev eth0

# 查看类统计
tc -s class show dev eth0

# 查看过滤器命中次数
tc -s filter show dev eth0
```

### English

#### Q1: Why isn't my TC configuration working?

**Common causes:**

| Cause | Solution |
|-------|----------|
| Wrong interface name | Use `ip link show` to confirm |
| No root privileges | All TC commands need root |
| Wrong configuration order | Create parent class before child classes |
| Filter priority conflict | Check prio values for correct ordering |
| Hardware offload interference | Disable TCP checksum offload: `ethtool -K eth0 tx off rx off` |

#### Q2: How to verify TC configuration effects?

```bash
# View statistics
tc -s qdisc show dev eth0
tc -s class show dev eth0

# Bandwidth test
iperf3 -c <server> -t 30

# Latency test
ping -c 20 <target>

# MTR test
mtr <target>

# Realtime monitoring
watch -n 1 'tc -s class show dev eth0'
```

#### Q3: Does TC configuration persist after reboot?

No, TC configuration is temporary and lost on reboot. To persist:
- Use systemd service (see Section 5)
- Add commands to `/etc/rc.local`
- Use network manager dispatcher scripts

#### Q4: What's the difference between HTB rate and ceil?

- `rate`: Guaranteed minimum bandwidth the class always receives
- `ceil`: Maximum bandwidth limit; can borrow up to this from parent's idle bandwidth

When `rate == ceil`, the class is strictly limited to that fixed bandwidth.

---

## 9. 进阶场景 / Advanced Scenarios

### 中文

#### 场景 1：统计特定流量的带宽使用

```bash
#!/bin/bash
# 监控特定应用的带宽使用

# 使用 iptables + tc 统计
iptables -t mangle -I POSTROUTING -p tcp --dport 443 -j CONNMARK --set-mark 100
iptables -t mangle -I POSTROUTING -m connmark --mark 100 -j RETURN

# 创建统计用的 class
tc qdisc add dev eth0 root handle 1: htb
tc class add dev eth0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
tc class add dev eth0 parent 1:1 classid 1:100 htb rate 1000mbit ceil 1000mbit
tc qdisc add dev eth0 parent 1:100 handle 100: sfq perturb 10

# 通过 fw 过滤统计
tc filter add dev eth0 parent 1:0 protocol ip prio 1 handle 100 fw flowid 1:100

# 查看 HTTPS 流量统计
watch -n 2 'tc -s class show dev eth0 | grep -A 5 "class htb 1:100"'
```

#### 场景 2：动态调整带宽（基于时间或条件）

```bash
#!/bin/bash
# 工作日/非工作日不同带宽策略

DAY=$(date +%u)  # 1=星期一, 7=星期日
HOUR=$(date +%H)

apply_workday() {
    echo "Applying workday QoS..."
    tc qdisc replace dev eth0 root handle 1: htb
    tc class replace dev eth0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
    # 工作时间：限制 P2P，保证业务带宽
    tc class replace dev eth0 parent 1:1 classid 1:10 htb rate 500mbit ceil 800mbit
    tc class replace dev eth0 parent 1:1 classid 1:20 htb rate 50mbit ceil 100mbit
}

apply_night() {
    echo "Applying night QoS..."
    tc qdisc replace dev eth0 root handle 1: htb
    tc class replace dev eth0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
    # 非工作时间：放开限制
    tc class replace dev eth0 parent 1:1 classid 1:10 htb rate 800mbit ceil 1000mbit
    tc class replace dev eth0 parent 1:1 classid 1:20 htb rate 200mbit ceil 1000mbit
}

if [ $DAY -le 5 ] && [ $HOUR -ge 9 ] && [ $HOUR -lt 18 ]; then
    apply_workday
else
    apply_night
fi
```

#### 场景 3：使用 eBPF 实现高效的流量分类

```bash
#!/bin/bash
# eBPF + TC 实现高性能流量分类

# 编译 eBPF 程序（需要 bpftool 和 clang）
cat > tc_bpf.c << 'EOF'
#include <linux/bpf.h>
#include <linux/pkt_cls.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <net/if.h>

SEC("classifier")
int tc_classify(struct __sk_buff *skb) {
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct iphdr *ip = data + sizeof(struct ethhdr);
    if ((void *)(ip + 1) > data_end) return TC_ACT_OK;

    if (ip->protocol == IPPROTO_TCP) {
        struct tcphdr *tcp = (void *)ip + sizeof(struct iphdr);
        if ((void *)(tcp + 1) > data_end) return TC_ACT_OK;

        // SSH: 高优先级
        if (tcp->dest == htons(22)) return 1;
        // HTTP: 中优先级
        if (tcp->dest == htons(80) || tcp->dest == htons(443)) return 2;
    }
    return 0;  // 默认
}
EOF

# 编译
clang -O2 -target bpf -c tc_bpf.c -o tc_bpf.o

# 加载到 TC
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj tc_bpf.o sec classifier
tc filter add dev eth0 egress bpf da obj tc_bpf.o sec classifier
```

#### 场景 4：结合网络命名空间做隔离测试

```bash
#!/bin/bash
# 在隔离的网络命名空间中测试 TC

# 创建命名空间
ip netns add test-ns

# 创建 veth pair 连接
ip link add veth-host type veth peer name veth-ns
ip link set veth-ns netns test-ns

# 配置 IP
ip addr add 10.200.0.1/24 dev veth-host
ip link set veth-host up
ip netns exec test-ns ip addr add 10.200.0.2/24 dev veth-ns
ip netns exec test-ns ip link set veth-ns up
ip netns exec test-ns ip link set lo up

# 在 veth-host 上应用 TC（模拟从 ns 到外部的网络条件）
tc qdisc add dev veth-host root netem delay 50ms loss 1%

# 从命名空间测试
ip netns exec test-ns ping 10.200.0.1 -c 10

# 在命名空间内部也设置 TC
ip netns exec test-ns tc qdisc add dev veth-ns root tbf rate 10mbit burst 32kbit latency 50ms

# 测试
ip netns exec test-ns iperf3 -c 10.200.0.1 -t 30

# 清理
ip netns del test-ns
ip link del veth-host 2>/dev/null || true
```

### English

#### Scenario 1: Bandwidth Usage Statistics for Specific Traffic

```bash
#!/bin/bash
# Monitor bandwidth usage for specific applications

iptables -t mangle -I POSTROUTING -p tcp --dport 443 -j CONNMARK --set-mark 100
tc qdisc add dev eth0 root handle 1: htb
tc class add dev eth0 parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
tc class add dev eth0 parent 1:1 classid 1:100 htb rate 1000mbit ceil 1000mbit
tc filter add dev eth0 parent 1:0 protocol ip prio 1 handle 100 fw flowid 1:100

# Monitor HTTPS traffic
watch -n 2 'tc -s class show dev eth0 | grep -A 5 "class htb 1:100"'
```

#### Scenario 4: TC with Network Namespaces for Isolated Testing

```bash
#!/bin/bash
# Test TC in an isolated network namespace

# Create namespace
ip netns add test-ns

# Create veth pair
ip link add veth-host type veth peer name veth-ns
ip link set veth-ns netns test-ns

# Configure IPs
ip addr add 10.200.0.1/24 dev veth-host
ip link set veth-host up
ip netns exec test-ns ip addr add 10.200.0.2/24 dev veth-ns
ip netns exec test-ns ip link set veth-ns up
ip netns exec test-ns ip link set lo up

# Apply TC on host side
tc qdisc add dev veth-host root netem delay 50ms loss 1%

# Test from namespace
ip netns exec test-ns ping 10.200.0.1 -c 10

# Apply TC inside namespace too
ip netns exec test-ns tc qdisc add dev veth-ns root tbf rate 10mbit burst 32kbit latency 50ms

# Cleanup
ip netns del test-ns
ip link del veth-host 2>/dev/null || true
```

---

## 10. 参考资源 / References

### 中文

- Linux TC 官方文档: [Linux Advanced Routing & Traffic Control HOWTO](https://tldp.org/HOWTO/Adv-Routing-HOWTO/)
- TC 命令手册: [tc(8)](https://man7.org/linux/man-pages/man8/tc.8.html)
- HTB 官方文档: [HTB Linux Queuing Discipline](https://www.sf.net/projects/htb/)
- netem 文档: [NetEm - Network Emulator](https://man7.org/linux/man-pages/man8/tc-netem.8.html)
- 内核文档: [Linux Kernel TC Documentation](https://www.kernel.org/doc/Documentation/networking/tc-actions-env-rules.txt)

### English

- Linux TC Official Doc: [Linux Advanced Routing & Traffic Control HOWTO](https://tldp.org/HOWTO/Adv-Routing-HOWTO/)
- TC Manual: [tc(8)](https://man7.org/linux/man-pages/man8/tc.8.html)
- HTB Official: [HTB Linux Queuing Discipline](https://www.sf.net/projects/htb/)
- netem Docs: [NetEm - Network Emulator](https://man7.org/linux/man-pages/man8/tc-netem.8.html)
- Kernel Docs: [Linux Kernel TC Documentation](https://www.kernel.org/doc/Documentation/networking/tc-actions-env-rules.txt)

---

## ☕ 支持 / Support

如果这个教程对你有帮助，欢迎请我喝杯咖啡：

**USDT (TRC20)**
```
TVbQerV1SF4MXB1JCcAzQxarewHwEPYTKm
```
