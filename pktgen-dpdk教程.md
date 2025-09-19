# DPDK Pktgen 网卡绑定与启动教程

本教程指导如何在 Ubuntu 系统中使用 DPDK 和 Pktgen 进行网络性能测试，包括绑定网卡到 DPDK 驱动、启动 Pktgen、设置参数以测试大包吞吐量（如 1500 字节包，目标 1 Gbps 线速，约 984 Mbps，82,106 packets/s）。适用于 Ubuntu 24.04，Pktgen 25.08.0，DPDK 24.11.3。

## 前提条件
- **系统**：Ubuntu 24.04（内核 6.8+）。
- **软件**：已安装 DPDK 24.11.3 和 Pktgen 25.08.0。
- **硬件**：支持 DPDK 的网卡（如 Intel I219-LM、Mellanox ConnectX），20 个 CPU 核心，单 NUMA 节点。
- **Hugepages**：已配置至少 512 个 2MB 页面（1024MB）。
  ```bash
  cat /proc/meminfo | grep Huge
  ```
  - 确保 `HugePages_Total` ≥ 512。
- **IOMMU**：已启用（GRUB 配置 `intel_iommu=on iommu=pt`）。

## 步骤 1：检查网卡状态
每次启动 DPDK（Pktgen）前，需确保网卡绑定到 DPDK 驱动（如 `vfio-pci`）。

1. **查看网卡状态**：
   ```bash
   sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status
   ```
   - 示例输出：
     ```
     Network devices using DPDK-compatible driver
     ============================================
     None

     Network devices using kernel driver
     ===================================
     0000:01:00.0 'Ethernet Controller' if=enp1s0 drv=e1000e unused=vfio-pci
     ```
     ![网卡状态](image.png)
   - 如果网卡（如 `0000:01:00.0`）在“kernel driver”下，需绑定到 DPDK 驱动。

2. **确认网卡型号**：
   ```bash
   lspci | grep Ethernet
   ```
   - 示例输出：
     ```
     01:00.0 Ethernet controller: Intel Corporation Ethernet Connection (7) I219-LM
     ```
   - 验证网卡支持 DPDK：https://doc.dpdk.org/guides/nics/.

## 步骤 2：绑定网卡到 DPDK 驱动
1. **加载 `vfio-pci` 模块**：
   ```bash
   sudo modprobe vfio-pci
   ```
   - 验证模块加载：
     ```bash
     lsmod | grep vfio
     ```
     - 应看到 `vfio_pci`, `vfio`, `vfio_iommu_type1`。

2. **绑定网卡**：
   ```bash
   sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --bind=vfio-pci 0000:01:00.0
   ```
   - 替换 `0000:01:00.0` 为你的网卡 PCI ID。
   - 如果绑定失败，检查 IOMMU：
     ```bash
     cat /proc/cmdline
     ```
     - 确保包含 `intel_iommu=on iommu=pt`。若无，编辑 GRUB：
       ```bash
       sudo nano /etc/default/grub
       ```
       修改：
       ```
       GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt"
       ```
       然后：
       ```bash
       sudo update-grub
       sudo reboot
       ```

3. **验证绑定**：
   ```bash
   sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status
   ```
   - 确保网卡在“DPDK-compatible driver”下：
     ```
     Network devices using DPDK-compatible driver
     ============================================
     0000:01:00.0 'Ethernet Controller' drv=vfio-pci unused=
     ```

## 步骤 3：启动 Pktgen
1. **运行 Pktgen**：
   ```bash
   sudo pktgen -l 2-4 -n 1 --socket-mem 1024 --file-prefix=pktgen0 -- -T -P -m "[3:3.0,4:4.1]"
   ```
   - **参数说明**：
     - `-l 2-4`：使用核心 2-4（核心 2 用于管理，3 和 4 用于端口）。
     - `-n 1`：1 个内存通道（单 NUMA 节点）。
     - `--socket-mem 1024`：分配 1GB 内存。
     - `--file-prefix=pktgen0`：避免多实例冲突。
     - `-T`：启用终端颜色。
     - `-P`：启用混杂模式。
     - `-m "[3:3.0,4:4.1]"`：核心 3 控制端口 0 TX/RX，核心 4 控制端口 1 TX/RX。
   - **单端口（仅端口 0）**：
     ```bash
     sudo pktgen -l 2-4 -n 1 --socket-mem 1024 --file-prefix=pktgen0 -- -T -P -m "[3:4.0]"
     ```

2. **验证端口**：
   在 `Pktgen:/>` 提示符下：
   ```bash
   show port summary all
   ```
   - 确认端口（0 和/或 1）状态为 `Link Up`，速度为 `1000 Mbps`（1 Gbps）。

## 步骤 4：设置 Pktgen 参数
进入 Pktgen 交互界面（`Pktgen:/>`）后，配置参数以测试大包（1500 字节）吞吐量。

![Pktgen 界面](image-1.png)

1. **检查配置**：
   ```bash
   show config
   ```
   - 确认 `size`（包大小）和 `rate`（发送速率）。

2. **设置大包和 100% 线速**：
   ```bash
   set all size 1500
   set all rate 100
   set all count 0
   ```
   - `size 1500`：设置包大小为 1500 字节。
   - `rate 100`：100% 线速（1 Gbps ≈ 984 Mbps，82,106 packets/s）。
   - `count 0`：无限发送。

3. **启动测试**：
   ```bash
   start all
   ```

4. **查看统计**：
   ```bash
   stats 0
   stats 1
   page stats
   ```
   - **预期输出（1 Gbps，1500 字节包）**：
     ```
     Port 0   RX        TX        Pkts/s    Bytes/s   Errors   Misses
     ------  --------  --------  --------  --------  --------  ------
     Total   X         Y         82106     984000000 0        0
     Rate    X         82106     82106     984000000 0        0
     ```

## 步骤 5：验证接收
在目标机器上：
```bash
sudo ip link set <interface> promisc on
sudo tcpdump -i <interface>
```
- 确认收到 1500 字节包，速率接近 984 Mbps。

## 故障排查
1. **端口无效**（如 `Invalid port ID 0`）：
   - 检查网卡绑定：
     ```bash
     sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status
     ```
   - 测试端口：
     ```bash
     sudo ~/dpdk-stable-24.11.3/build/app/dpdk-testpmd -- -i
     show port summary all
     ```

2. **低吞吐量**（如之前 82 Mbps，122,528 packets/s）：
   - 确认包大小：`show config` 检查 `size`。
   - 确认网卡速度：`show port summary all` 检查 `Link Speed`。
   - 启用多队列：
     ```bash
     sudo pktgen -l 2-4 -n 1 --socket-mem 1024 --file-prefix=pktgen0 -- -T -P -m "[3:4.0]" --nb-rxd 4096 --nb-txd 4096
     ```

3. **Hugepages 不足**：
   ```bash
   cat /proc/meminfo | grep Huge
   ```
   - 确保 `HugePages_Total` ≥ 512。
   - 增加：
     ```bash
     echo 'vm.nr_hugepages=1024' | sudo tee /etc/sysctl.d/99-dpdk.conf
     sudo sysctl -p /etc/sysctl.d/99-dpdk.conf
     ```

## 总结
通过以上步骤，你可以成功绑定网卡到 DPDK 驱动，启动 Pktgen，配置 1500 字节包，并实现接近 1 Gbps 的线速吞吐量（984 Mbps，82,106 packets/s）。如果遇到错误，提供以下信息以便进一步排查：
- `sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status` 输出。
- `lspci | grep Ethernet` 输出。
- `show port summary all`, `show config`, `stats 0` 输出。
- 网卡速度（`sudo ethtool <interface>`）。

快跑满大包线速！?