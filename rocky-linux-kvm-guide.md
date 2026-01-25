# Rocky Linux KVM 完整配置指南

---

## 一、前提条件（嵌套虚拟化环境）

如果宿主机是 VMware 虚拟机，需在 VMware vSwitch 设置：

| 设置 | 值 |
|------|-----|
| 混杂模式 | 接受 |
| MAC 地址更改 | 接受 |
| 伪传输 | 接受 |

---

## 二、安装 KVM

```bash
# 检查 CPU 虚拟化支持
grep -E '(vmx|svm)' /proc/cpuinfo

# 安装 KVM 组件
sudo dnf install -y qemu-kvm libvirt virt-install virt-manager virt-viewer

# 启动服务
sudo systemctl enable --now libvirtd

# 验证安装
lsmod | grep kvm
virsh list --all

# 将当前用户加入 libvirt 组（可选，免 sudo）
sudo usermod -aG libvirt $(whoami)
```

---

## 三、配置桥接网络

```bash
# 1. 查看网卡名称
nmcli device status

# 2. 创建网桥
sudo nmcli connection add type bridge con-name br0 ifname br0

# 3. 配置 IPv4/IPv6 自动获取
sudo nmcli connection modify br0 ipv4.method auto
sudo nmcli connection modify br0 ipv6.method auto

# 4. 将物理网卡加入网桥（替换 eth1 为实际网卡名）
sudo nmcli connection add type ethernet slave-type bridge con-name br0-port1 ifname eth1 master br0

# 5. 启用网桥
sudo nmcli connection up br0

# 6. 删除原网卡连接（替换为实际连接名）
sudo nmcli connection delete "eth1"

# 7. 验证
ip addr show br0
bridge link
```

### 命令详解

#### 创建网桥

```bash
sudo nmcli connection add type bridge con-name br0 ifname br0
```

| 参数 | 含义 |
|------|------|
| `nmcli` | NetworkManager 命令行工具 |
| `connection add` | 添加一个新的网络连接配置 |
| `type bridge` | 连接类型为网桥（虚拟交换机） |
| `con-name br0` | 连接配置的名称，用于后续管理时引用（可自定义，如 `my-bridge`） |
| `ifname br0` | 实际创建的网络接口名称，会出现在 `ip addr` 输出中 |

**说明**：`con-name` 是 NetworkManager 中的配置名称，`ifname` 是系统中实际的网络接口名称，通常设置为相同值便于管理。

---

#### 将物理网卡加入网桥

```bash
sudo nmcli connection add type ethernet slave-type bridge con-name br0-port1 ifname eth1 master br0
```

| 参数 | 含义 |
|------|------|
| `type ethernet` | 这是一个以太网类型的连接 |
| `slave-type bridge` | 作为网桥的从属端口（类似把网线插入交换机） |
| `con-name br0-port1` | 这个从属连接的配置名称 |
| `ifname eth1` | 实际的物理网卡接口名（根据 `nmcli device status` 输出替换） |
| `master br0` | 指定主设备为 `br0` 网桥 |

**说明**：执行后，物理网卡 `eth1` 成为网桥 `br0` 的一个端口，不再直接持有 IP 地址，所有网络流量通过网桥转发。

---

#### 网络结构变化

```
配置前：
  路由器 ──── eth1 (IP: 192.168.31.33) ──── 宿主机

配置后：
  路由器 ──── eth1 (无IP，作为端口) ──── br0 (IP: 192.168.31.33)
                                            │
                                    ┌───────┴───────┐
                                  宿主机          虚拟机
                                              (IP: 192.168.31.x)
```

---

#### 配置文件位置

命令执行后会在以下目录生成配置文件：

```bash
ls /etc/NetworkManager/system-connections/
# 输出：br0.nmconnection  br0-port1.nmconnection
```

---

#### IPv4/IPv6 方法可选值

| 值 | 含义 |
|------|------|
| `auto` | 自动获取（DHCP/SLAAC） |
| `manual` | 手动配置静态 IP |
| `disabled` | 禁用该协议 |
| `ignore` | 忽略（仅 IPv6） |

---

## 四、创建虚拟机

### 方法一：命令行方式

```bash
virt-install \
  --name rocky-vm \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rocky-vm.qcow2,size=20 \
  --os-variant rocky9 \
  --network bridge=br0 \
  --graphics vnc,listen=0.0.0.0,password=yourpassword \
  --cdrom /path/to/Rocky-9.iso
```

### 方法二：图形界面方式（X11 转发）

#### 宿主机配置

```bash
# 安装 xauth
sudo dnf install -y xauth
```

编辑 `/etc/ssh/sshd_config`，确保以下配置：

```
X11Forwarding yes
X11UseLocalhost no
```

重启 SSH：

```bash
sudo systemctl restart sshd
```

#### macOS 客户端配置

1. 安装 XQuartz：

```bash
brew install --cask xquartz
```

2. 注销并重新登录 macOS

3. 连接并启动 virt-manager：

```bash
ssh -Y root@宿主机IP
virt-manager
```

#### virt-manager 创建虚拟机步骤

1. 点击"创建新虚拟机"
2. 选择"本地安装介质（ISO）"
3. 浏览选择 ISO 文件
4. 配置内存和 CPU
5. 配置存储大小
6. 勾选"在安装前自定义配置"
7. 网络选择"桥接设备"，填入 `br0`
8. 点击"开始安装"

---

## 五、配置 VNC 远程访问

### 方法一：开放 VNC 端口（内网使用）

编辑虚拟机配置：

```bash
virsh edit 虚拟机名称
```

修改 `<graphics>` 部分：

```xml
<graphics type='vnc' port='5900' autoport='no' listen='0.0.0.0' passwd='你的密码'>
  <listen type='address' address='0.0.0.0'/>
</graphics>
```

重启虚拟机并开放防火墙：

```bash
virsh destroy 虚拟机名称
virsh start 虚拟机名称

sudo firewall-cmd --add-port=5900/tcp --permanent
sudo firewall-cmd --reload
```

VNC 客户端连接：`宿主机IP:5900`

### 方法二：SSH 隧道（更安全）

客户端执行：

```bash
ssh -L 5900:127.0.0.1:5900 root@宿主机IP
```

VNC 客户端连接：`127.0.0.1:5900`

---

## 六、配置 virsh console

### 虚拟机内配置

```bash
# 启用串口控制台服务
sudo systemctl enable --now serial-getty@ttyS0.service

# 配置 GRUB 输出到串口（可选，显示启动信息）
sudo grubby --update-kernel=ALL --args="console=ttyS0,115200n8"
sudo reboot
```

### 宿主机连接

```bash
virsh console 虚拟机名称
```

退出按 `Ctrl + ]`

---

## 七、常用管理命令

### 虚拟机管理

```bash
# 查看虚拟机列表
virsh list --all

# 启动虚拟机
virsh start 虚拟机名称

# 正常关闭虚拟机
virsh shutdown 虚拟机名称

# 重启虚拟机
virsh reboot 虚拟机名称

# 强制关闭虚拟机
virsh destroy 虚拟机名称

# 删除虚拟机
virsh undefine 虚拟机名称 --remove-all-storage
```

### 虚拟机信息

```bash
# 查看 VNC 端口
virsh vncdisplay 虚拟机名称

# 查看虚拟机 IP
virsh domifaddr 虚拟机名称

# 查看虚拟机详细配置
virsh dumpxml 虚拟机名称

# 编辑虚拟机配置
virsh edit 虚拟机名称
```

### 网络管理

```bash
# 查看网桥状态
bridge link

# 查看网桥 IP
ip addr show br0

# 查看网络连接
nmcli connection show
```

---

## 八、验证清单

| 检查项 | 命令 | 预期结果 |
|--------|------|----------|
| 网桥状态 | `bridge link` | 显示 eth1 和 vnet0 接入 br0 |
| 网桥 IP | `ip addr show br0` | 显示 IPv4 和 IPv6 地址 |
| 虚拟机状态 | `virsh list --all` | 显示虚拟机运行状态 |
| 虚拟机 IP | `virsh domifaddr 虚拟机名称` | 显示虚拟机 IP 地址 |
| VNC 端口 | `virsh vncdisplay 虚拟机名称` | 显示 VNC 端口号 |
| 网络连通性 | `ping 虚拟机IP` | ping 通 |

---

## 九、故障排查

### 虚拟机无法获取 IP

1. 检查网桥配置：

```bash
bridge link
nmcli connection show
```

2. 抓包检查 DHCP：

```bash
tcpdump -i br0 port 67 or port 68 -n
```

3. 检查防火墙：

```bash
sudo firewall-cmd --list-all
```

### 嵌套虚拟化桥接不通

确认 VMware vSwitch 已开启：
- 混杂模式 → 接受
- MAC 地址更改 → 接受
- 伪传输 → 接受

### VNC 无法连接

1. 检查 VNC 监听地址：

```bash
virsh dumpxml 虚拟机名称 | grep -A 5 "graphics"
```

2. 检查防火墙端口：

```bash
sudo firewall-cmd --list-ports
```

3. 使用 SSH 隧道方式连接

---

## 十、参考配置文件

### 网桥配置文件位置

```
/etc/NetworkManager/system-connections/br0.nmconnection
/etc/NetworkManager/system-connections/br0-port1.nmconnection
```

### 虚拟机配置文件位置

```
/etc/libvirt/qemu/虚拟机名称.xml
```

### 虚拟机磁盘默认位置

```
/var/lib/libvirt/images/
```
