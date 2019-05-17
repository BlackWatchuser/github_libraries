# Linux 上的增强联网

[原文](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/enhanced-networking.html)

增强联网使用单个根 I/O 虚拟化 (`SR-IOV`) 在支持的实例类型上提供高性能的联网功能。**`SR-IOV 是一种设备虚拟化方法，与传统虚拟化网络接口相比，它不仅能提高 I/O 性能，还能降低 CPU 使用率。增强联网可以提高带宽，提高每秒数据包数 (PPS) 性能，并不断降低实例间的延迟`**。使用增强联网不收取任何额外费用。

## 增强联网类型

根据您的实例类型，可以使用以下机制之一启用增强联网：

**`Elastic Network Adapter (ENA)`**

* 对于支持的实例类型，**`Elastic Network Adapter (ENA)`** 最多支持 **100 Gbps** 的网络速度。

* **`A1, C5, C5d, C5n, F1, G3, H1, I3, m4.16xlarge, M5, M5a, M5d, P2, P3, R4, R5, R5a, R5d, T3, u-6tb1.metal, u-9tb1.metal, u-12tb1.metal, X1, X1e`**, and **`z1d`** 实例使用 `Elastic Network Adapter` 实现增强联网。

**`Intel 82599 虚拟功能 (VF) 接口`**

* 对于受支持的实例类型，`Intel 82599 虚拟功能接口`最多支持 **10 Gbps** 的网络速度。

* **`C3、C4、D2、I2、M4 (m4.16xlarge 除外) 和 R3`** 实例使用 `Intel 82599 VF` 接口实现增强联网。

有关每个实例类型支持的网络速度的信息，请参阅 [Amazon EC2 实例类型](https://aws.amazon.com/cn/ec2/instance-types/)。

------

## 在 Linux 实例上启用 Elastic Network Adapter (ENA) 增强联网

### 要求

要使用 ENA 准备增强联网，请按如下方式设置您的实例：

* 从以下支持的实例类型中选择：**`A1, C5, C5d, C5n, F1, G3, H1, I3, m4.16xlarge, M5, M5a, M5d, P2, P3, R4, R5, R5a, R5d, T3, u-6tb1.metal, u-9tb1.metal, u-12tb1.metal, X1, X1e`**, and **`z1d`**。

* 使用支持的 Linux 内核版本及支持的发行版启动实例，以便为实例自动启用 ENA 增强联网。有关更多信息，请参阅 [ENA Linux 内核驱动程序发行说明](https://github.com/amzn/amzn-drivers/blob/ena_linux_1.6.0/kernel/linux/ena/RELEASENOTES.md)。

* 确保实例具有 Internet 连接。

* 将 [AWS CLI](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-install.html) 或 [适用于 Windows PowerShell 的 AWS 工具](https://docs.aws.amazon.com/zh_cn/powershell/latest/userguide/pstools-welcome.html) 安装到您选择的任意计算机上（最好是您的本地台式计算机或笔记本电脑）并进行配置。有关更多信息，请参阅访问 Amazon EC2。不能从 Amazon EC2 控制台管理增强联网。

* 如果您的实例上有重要的数据需要保留，则应立即从您的实例创建 AMI，来备份这些数据。**`更新内核和内核模块以及启用 enaSupport 属性可能会导致实例不兼容或操作系统无法访问`**；如果您有最新备份，则可在发生这种情况时保留数据。

### 测试是否启用了增强联网功能

**`若要测试是否已启用了增强联网，请确认实例上已安装 ena 模块且设置了 enaSupport 属性`**。如果实例满足这两个条件，则 `ethtool -i ethn` 命令应显示该模块已在网络接口上使用。

``` shell
# ethtool -i eth0
driver: vif
version:
firmware-version:
bus-info: vif-0
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: no
```

#### 内核模块 (ena)

要确认已安装 **`ena`** 模块，请使用 **`modinfo`** 命令，如下所示：

``` shell
[ec2-user ~]$ modinfo ena
filename:       /lib/modules/4.14.33-59.37.amzn2.x86_64/kernel/drivers/amazon/net/ena/ena.ko
version:        1.5.0g
license:        GPL
description:    Elastic Network Adapter (ENA)
author:         Amazon.com, Inc. or its affiliates
srcversion:     692C7C68B8A9001CB3F31D0
alias:          pci:v00001D0Fd0000EC21sv*sd*bc*sc*i*
alias:          pci:v00001D0Fd0000EC20sv*sd*bc*sc*i*
alias:          pci:v00001D0Fd00001EC2sv*sd*bc*sc*i*
alias:          pci:v00001D0Fd00000EC2sv*sd*bc*sc*i*
depends:
retpoline:      Y
intree:         Y
name:           ena
...
```

在上述 Amazon Linux 情况中，ena 模块已安装。

#### 实例属性 (enaSupport)

要检查实例是否设置了增强联网 enaSupport 属性，请使用以下任一命令。如果该属性已设置，则响应为 true。

* **`describe-instances (AWS CLI)`**

``` shell
aws ec2 describe-instances --instance-ids instance_id --query "Reservations[].Instances[].EnaSupport"
```

``` shell
aws ec2 describe-instances --instance-ids i-0c08b32dd2dc22026 --query "Reservations[].Instances[].EnaSupport"
```

------

**`与 R4 相比，R5 实例为每个 vCPU 提供额外 5% 的内存，且最高可以提供 768GiB 内存。此外，相对于 R4，R5 实例每 GiB 的价格降低了 10%，CPU 性能提高了约 20%`**。

特点：

* 最高 3.1GHz Intel Xeon® Platinum 8175 处理器，并配有全新的 Intel Advanced Vector Extension (AXV-512) 指令集

* 每个实例提供高达 768GiB 的内存

* 由 AWS Nitro 系统（专用硬件和轻量级管理程序的组合）提供支持

* 借助 R5d 实例，基于 NVMe 的本地 SSD 可以物理连接到主机服务器，并提供与 R5 实例的生命周期相耦合的块级存储

| 型号 | vCPU | 内存 (GiB) | 存储 (GB) | 专用 EBS 带宽 (Mbps) | 联网性能 (Gbps) |
| :- | :- | :- | :- | :- | :- |
| r4.2xlarge | 8 | 61 | 仅限 EBS | - | 最高 10 |
| r5.2xlarge | 8 | 64 | 仅限 EBS | 高达 3500 | 最高 10 |

``` shell
[ec2@r5.2xlarge ~]# free -g
             total       used       free     shared    buffers     cached
Mem:            62          0         61          0          0          0
-/+ buffers/cache:          0         61
Swap:            0          0          0
```

``` shell
[ec2@r4.2xlarge ~]# free -g
             total       used       free     shared    buffers     cached
Mem:            59         48         11          0          0         24
-/+ buffers/cache:         23         36
Swap:            0          0          0
```
------