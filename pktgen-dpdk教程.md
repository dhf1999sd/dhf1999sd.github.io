# DPDK Pktgen �������������̳�

���̳�ָ������� Ubuntu ϵͳ��ʹ�� DPDK �� Pktgen �����������ܲ��ԣ������������� DPDK ���������� Pktgen�����ò����Բ��Դ������������ 1500 �ֽڰ���Ŀ�� 1 Gbps ���٣�Լ 984 Mbps��82,106 packets/s���������� Ubuntu 24.04��Pktgen 25.08.0��DPDK 24.11.3��

## ǰ������
- **ϵͳ**��Ubuntu 24.04���ں� 6.8+����
- **���**���Ѱ�װ DPDK 24.11.3 �� Pktgen 25.08.0��
- **Ӳ��**��֧�� DPDK ���������� Intel I219-LM��Mellanox ConnectX����20 �� CPU ���ģ��� NUMA �ڵ㡣
- **Hugepages**������������ 512 �� 2MB ҳ�棨1024MB����
  ```bash
  cat /proc/meminfo | grep Huge
  ```
  - ȷ�� `HugePages_Total` �� 512��
- **IOMMU**�������ã�GRUB ���� `intel_iommu=on iommu=pt`����

## ���� 1���������״̬
ÿ������ DPDK��Pktgen��ǰ����ȷ�������󶨵� DPDK �������� `vfio-pci`����

1. **�鿴����״̬**��
   ```bash
   sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status
   ```
   - ʾ�������
     ```
     Network devices using DPDK-compatible driver
     ============================================
     None

     Network devices using kernel driver
     ===================================
     0000:01:00.0 'Ethernet Controller' if=enp1s0 drv=e1000e unused=vfio-pci
     ```
     ![����״̬](image.png)
   - ����������� `0000:01:00.0`���ڡ�kernel driver���£���󶨵� DPDK ������

2. **ȷ�������ͺ�**��
   ```bash
   lspci | grep Ethernet
   ```
   - ʾ�������
     ```
     01:00.0 Ethernet controller: Intel Corporation Ethernet Connection (7) I219-LM
     ```
   - ��֤����֧�� DPDK��https://doc.dpdk.org/guides/nics/.

## ���� 2���������� DPDK ����
1. **���� `vfio-pci` ģ��**��
   ```bash
   sudo modprobe vfio-pci
   ```
   - ��֤ģ����أ�
     ```bash
     lsmod | grep vfio
     ```
     - Ӧ���� `vfio_pci`, `vfio`, `vfio_iommu_type1`��

2. **������**��
   ```bash
   sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --bind=vfio-pci 0000:01:00.0
   ```
   - �滻 `0000:01:00.0` Ϊ������� PCI ID��
   - �����ʧ�ܣ���� IOMMU��
     ```bash
     cat /proc/cmdline
     ```
     - ȷ������ `intel_iommu=on iommu=pt`�����ޣ��༭ GRUB��
       ```bash
       sudo nano /etc/default/grub
       ```
       �޸ģ�
       ```
       GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt"
       ```
       Ȼ��
       ```bash
       sudo update-grub
       sudo reboot
       ```

3. **��֤��**��
   ```bash
   sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status
   ```
   - ȷ�������ڡ�DPDK-compatible driver���£�
     ```
     Network devices using DPDK-compatible driver
     ============================================
     0000:01:00.0 'Ethernet Controller' drv=vfio-pci unused=
     ```

## ���� 3������ Pktgen
1. **���� Pktgen**��
   ```bash
   sudo pktgen -l 2-4 -n 1 --socket-mem 1024 --file-prefix=pktgen0 -- -T -P -m "[3:3.0,4:4.1]"
   ```
   - **����˵��**��
     - `-l 2-4`��ʹ�ú��� 2-4������ 2 ���ڹ���3 �� 4 ���ڶ˿ڣ���
     - `-n 1`��1 ���ڴ�ͨ������ NUMA �ڵ㣩��
     - `--socket-mem 1024`������ 1GB �ڴ档
     - `--file-prefix=pktgen0`�������ʵ����ͻ��
     - `-T`�������ն���ɫ��
     - `-P`�����û���ģʽ��
     - `-m "[3:3.0,4:4.1]"`������ 3 ���ƶ˿� 0 TX/RX������ 4 ���ƶ˿� 1 TX/RX��
   - **���˿ڣ����˿� 0��**��
     ```bash
     sudo pktgen -l 2-4 -n 1 --socket-mem 1024 --file-prefix=pktgen0 -- -T -P -m "[3:4.0]"
     ```

2. **��֤�˿�**��
   �� `Pktgen:/>` ��ʾ���£�
   ```bash
   show port summary all
   ```
   - ȷ�϶˿ڣ�0 ��/�� 1��״̬Ϊ `Link Up`���ٶ�Ϊ `1000 Mbps`��1 Gbps����

## ���� 4������ Pktgen ����
���� Pktgen �������棨`Pktgen:/>`�������ò����Բ��Դ����1500 �ֽڣ���������

![Pktgen ����](image-1.png)

1. **�������**��
   ```bash
   show config
   ```
   - ȷ�� `size`������С���� `rate`���������ʣ���

2. **���ô���� 100% ����**��
   ```bash
   set all size 1500
   set all rate 100
   set all count 0
   ```
   - `size 1500`�����ð���СΪ 1500 �ֽڡ�
   - `rate 100`��100% ���٣�1 Gbps �� 984 Mbps��82,106 packets/s����
   - `count 0`�����޷��͡�

3. **��������**��
   ```bash
   start all
   ```

4. **�鿴ͳ��**��
   ```bash
   stats 0
   stats 1
   page stats
   ```
   - **Ԥ�������1 Gbps��1500 �ֽڰ���**��
     ```
     Port 0   RX        TX        Pkts/s    Bytes/s   Errors   Misses
     ------  --------  --------  --------  --------  --------  ------
     Total   X         Y         82106     984000000 0        0
     Rate    X         82106     82106     984000000 0        0
     ```

## ���� 5����֤����
��Ŀ������ϣ�
```bash
sudo ip link set <interface> promisc on
sudo tcpdump -i <interface>
```
- ȷ���յ� 1500 �ֽڰ������ʽӽ� 984 Mbps��

## �����Ų�
1. **�˿���Ч**���� `Invalid port ID 0`����
   - ��������󶨣�
     ```bash
     sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status
     ```
   - ���Զ˿ڣ�
     ```bash
     sudo ~/dpdk-stable-24.11.3/build/app/dpdk-testpmd -- -i
     show port summary all
     ```

2. **��������**����֮ǰ 82 Mbps��122,528 packets/s����
   - ȷ�ϰ���С��`show config` ��� `size`��
   - ȷ�������ٶȣ�`show port summary all` ��� `Link Speed`��
   - ���ö���У�
     ```bash
     sudo pktgen -l 2-4 -n 1 --socket-mem 1024 --file-prefix=pktgen0 -- -T -P -m "[3:4.0]" --nb-rxd 4096 --nb-txd 4096
     ```

3. **Hugepages ����**��
   ```bash
   cat /proc/meminfo | grep Huge
   ```
   - ȷ�� `HugePages_Total` �� 512��
   - ���ӣ�
     ```bash
     echo 'vm.nr_hugepages=1024' | sudo tee /etc/sysctl.d/99-dpdk.conf
     sudo sysctl -p /etc/sysctl.d/99-dpdk.conf
     ```

## �ܽ�
ͨ�����ϲ��裬����Գɹ��������� DPDK ���������� Pktgen������ 1500 �ֽڰ�����ʵ�ֽӽ� 1 Gbps ��������������984 Mbps��82,106 packets/s����������������ṩ������Ϣ�Ա��һ���Ų飺
- `sudo ~/dpdk-stable-24.11.3/build/app/dpdk-devbind.py --status` �����
- `lspci | grep Ethernet` �����
- `show port summary all`, `show config`, `stats 0` �����
- �����ٶȣ�`sudo ethtool <interface>`����

������������٣�?