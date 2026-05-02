---
name: esxi-management
description: "Use when managing VMware ESXi hosts via govc CLI. Covers VM lifecycle (create/delete/power/snapshot/clone), host monitoring, storage, networking, ISO upload, and multi-host management. Requires govc installed and ESXi credentials."
version: 1.0.1
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [esxi, vmware, govc, virtualization, vm-management, infrastructure]
    related_skills: [remote-deployment, windows-ssh-management]
---

# ESXi Management via govc

> **Language / 语言**: This document is bilingual. English is primary; Chinese (中文) translations follow in collapsible sections or inline.

---

## Overview / 概述

VMware ESXi is a bare-metal hypervisor for running virtual machines on physical hardware. This skill covers managing standalone ESXi hosts and their VMs using **govc**, the official Go CLI from VMware/govmomi. govc communicates with ESXi's vSphere API over HTTPS (port 443) and does not require SSH to be enabled on the host.

> VMware ESXi 是一种裸机虚拟化平台，用于在物理服务器上运行虚拟机。本文档介绍如何使用 VMware 官方 CLI 工具 **govc** 管理独立 ESXi 主机及其虚拟机。govc 通过 HTTPS (443端口) 调用 vSphere API 进行通信，无需在 ESXi 上启用 SSH。

This document is self-contained: a model or human reading only this file should be able to install govc, connect to an ESXi host, and perform any common management task without additional references.

> 本文档是自包含的：无论 AI 模型还是人类，仅阅读此文件即可完成 govc 安装、连接 ESXi 主机及所有常见管理操作。

---

## When to Use / 适用场景

- User asks to manage, monitor, or troubleshoot an ESXi host
- User wants to create, delete, start, stop, snapshot, or clone VMs
- User needs to upload ISO images or manage datastores
- User wants to inspect host hardware, resource usage, or network configuration
- User has multiple ESXi hosts and needs batch operations

> - 用户要求管理、监控或排查 ESXi 主机问题
> - 用户需要创建、删除、启动、停止、快照或克隆虚拟机
> - 用户需要上传 ISO 镜像或管理数据存储
> - 用户需要查看主机硬件、资源使用率或网络配置
> - 用户拥有多台 ESXi 主机需要批量操作

## When NOT to Use / 不适用场景

- Managing vCenter Server clusters (govc supports vCenter, but this skill focuses on standalone ESXi)
- Managing VMware Workstation/Fusion (different product, different tools)
- Managing Proxmox, Hyper-V, KVM, or other hypervisors (completely different platforms)

> - 管理 vCenter Server 集群（govc 支持 vCenter，但本文档专注于独立 ESXi）
> - 管理 VMware Workstation/Fusion（不同产品，使用不同工具）
> - 管理 Proxmox、Hyper-V、KVM 等其他虚拟化平台

---

## 1. Installation / 安装

### Linux (x86_64)

```bash
# Download latest release / 下载最新版本
cd /tmp
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Linux_x86_64.tar.gz
tar -xzf govc_Linux_x86_64.tar.gz govc

# Option A: Install to system path (requires sudo) / 安装到系统路径（需要 sudo）
sudo mv govc /usr/local/bin/
sudo chmod +x /usr/local/bin/govc

# Option B: Install to user-local path (no sudo) / 安装到用户本地路径（无需 sudo）
mkdir -p ~/.local/bin
mv govc ~/.local/bin/
chmod +x ~/.local/bin/govc
export PATH="$HOME/.local/bin:$PATH"
```

### macOS

```bash
# Apple Silicon (M1/M2/M3)
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Darwin_arm64.tar.gz
tar -xzf govc_Darwin_arm64.tar.gz govc
sudo mv govc /usr/local/bin/

# Intel x86_64
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Darwin_x86_64.tar.gz
tar -xzf govc_Darwin_x86_64.tar.gz govc
sudo mv govc /usr/local/bin/
```

### Verify Installation / 验证安装

```bash
govc version
# Expected output / 预期输出: govc 0.53.1 (or newer / 或更新版本)
```

---

## 2. Connection Configuration / 连接配置

govc uses environment variables for connection details. These must be set before every command, or exported in the shell session.

> govc 通过环境变量传递连接信息。每次执行命令前必须设置，或在当前 shell 会话中 export。

### Required Environment Variables / 必需环境变量

```bash
export GOVC_URL='https://<your_esxi_ip>'
export GOVC_USERNAME='<username>'        # usually 'root' / 通常为 'root'
export GOVC_PASSWORD='<password>'
export GOVC_INSECURE=true                # skip TLS verification / 跳过 TLS 证书验证（ESXi 自签证书必需）
```

### Optional Environment Variables / 可选环境变量

```bash
export GOVC_DATASTORE='datastore1'       # default datastore / 默认数据存储
export GOVC_NETWORK='VM Network'         # default portgroup / 默认端口组
export GOVC_RESOURCE_POOL='/'            # resource pool path / 资源池路径（独立 ESXi 使用 '/'）
export GOVC_DATACENTER='ha-datacenter'   # datacenter name / 数据中心名称（独立 ESXi 使用 'ha-datacenter'）
```

### Persisting Configuration / 持久化配置

For a single host, add to `~/.bashrc` or `~/.zshrc`:

> 单台主机可将配置写入 `~/.bashrc` 或 `~/.zshrc`：

```bash
export GOVC_URL='https://<your_esxi_ip>'
export GOVC_USERNAME='root'
export GOVC_PASSWORD='your_password'
export GOVC_INSECURE=true
```

For multiple hosts, do NOT persist globally. Use per-command assignment or wrapper functions (see Section 7: Multi-Host Management).

> 多台主机不要全局持久化。使用逐命令赋值或封装函数（见第7节：多主机管理）。

### Verify Connection / 验证连接

```bash
govc about
# Expected output / 预期输出:
#   FullName:     VMware ESXi 8.0.3 build-XXXXXXXX
#   Name:         VMware ESXi
#   Vendor:       VMware, Inc.
#   Version:      8.0.3
#   API type:     HostAgent
```

---

## 3. Host Information and Monitoring / 主机信息与监控

### Basic Host Info / 基本主机信息

```bash
# Human-readable summary / 可读格式
govc host.info

# JSON output for programmatic parsing / JSON 格式（便于程序解析）
govc host.info -json
```

### Detailed Hardware Details / 硬件详情

```bash
govc host.info -json | python3 -c "
import json, sys
d = json.load(sys.stdin)
h = d['hostSystems'][0]
hw = h.get('hardware', {})
ci = hw.get('cpuInfo', {})
cp = hw.get('cpuPkg', [])

print(f'CPU Packages: {ci.get(\"numCpuPackages\", \"?\")}')
print(f'CPU Cores:    {ci.get(\"numCpuCores\", \"?\")}')
print(f'CPU Threads:  {ci.get(\"numCpuThreads\", \"?\")}')
print(f'CPU Freq:     {ci.get(\"hz\", 0) / 1e9:.2f} GHz')
print(f'Memory:       {hw.get(\"memorySize\", 0) // 1024 // 1024 // 1024} GB')
print()
for i, pkg in enumerate(cp):
    print(f'CPU{i}: {pkg.get(\"description\", \"N/A\")}')
"
```

### Real-Time Resource Usage / 实时资源使用率

```bash
# CPU and memory usage / CPU 与内存使用率
govc host.info | grep -E "CPU usage|Memory usage"

# Detailed quick stats via JSON / 通过 JSON 获取详细统计
govc host.info -json | python3 -c "
import json, sys
d = json.load(sys.stdin)
h = d['hostSystems'][0]
qs = h.get('summary', {}).get('quickStats', {})
print(f'CPU Usage:    {qs.get(\"overallCpuUsage\", \"?\")} MHz')
print(f'Memory Usage: {qs.get(\"overallMemoryUsage\", \"?\")} MB')
print(f'Uptime:       {qs.get(\"uptime\", \"?\")} seconds')
"
```

### Health Sensors / 健康传感器

```bash
govc host.info -json | python3 -c "
import json, sys
d = json.load(sys.stdin)
h = d['hostSystems'][0]
sensors = h.get('runtime', {}).get('healthSystemRuntime', {}) \
    .get('systemHealthInfo', {}).get('numericSensorInfo', [])
for s in sensors[:20]:
    name = s.get('name', 'N/A')
    state = s.get('healthState', {}).get('label', 'N/A')
    reading = s.get('currentReading', '')
    units = s.get('baseUnits', '')
    modifier = s.get('unitModifier', 0)
    if reading and modifier:
        value = reading * (10 ** modifier)
        print(f'{state:6} {name:45} {value} {units}')
    else:
        print(f'{state:6} {name}')
"
```

---

## 4. Virtual Machine Management / 虚拟机管理

### List All VMs / 列出所有虚拟机

```bash
# List VM paths (tree view) / 列出 VM 路径
govc find / -type m

# Human-readable info / 可读信息
govc vm.info '*'

# Single VM / 单台虚拟机
govc vm.info '<VM_NAME>'

# JSON with full details / JSON 完整详情
govc vm.info -json '*' | python3 -c "
import json, sys
d = json.load(sys.stdin)
for vm in d.get('virtualMachines', []):
    name = vm.get('name', 'N/A')
    power = vm.get('runtime', {}).get('powerState', 'N/A')
    cpu = vm.get('config', {}).get('hardware', {}).get('numCPU', '?')
    mem = vm.get('config', {}).get('hardware', {}).get('memoryMB', '?')
    guest = vm.get('config', {}).get('guestFullName', 'N/A')
    ip = vm.get('guest', {}).get('ipAddress', '')
    state = 'ON' if power == 'poweredOn' else 'OFF'
    print(f'[{state:3}] {name:50} CPU:{cpu} RAM:{mem}MB {guest} {ip}')
"
```

### VM Power Operations / 虚拟机电源操作

```bash
# Power on / 开机
govc vm.power -on '<VM_NAME>'

# Power off (graceful, sends ACPI shutdown) / 关机（优雅，发送 ACPI 关机信号）
govc vm.power -off '<VM_NAME>'

# Power off (hard, like pulling power cord) / 强制关机（等同于拔电源）
govc vm.power -off -force '<VM_NAME>'

# Restart (graceful reboot) / 重启
govc vm.power -reset '<VM_NAME>'

# Suspend / 挂起
govc vm.power -suspend '<VM_NAME>'
```

### Create a New VM / 创建虚拟机

```bash
# Minimal example / 最小示例
govc vm.create \
  -ds 'datastore1' \
  -net 'VM Network' \
  -m 1024 \
  -c 1 \
  -disk 16GB \
  -disk.controller pvscsi \
  -g centos7_64 \
  'MyNewVM'

# Full options example / 完整参数示例
govc vm.create \
  -ds 'datastore2' \
  -net 'VM Network' \
  -m 8192 \
  -c 4 \
  -disk 100GB \
  -disk.thin \
  -disk.controller pvscsi \
  -g ubuntu64Guest \
  -firmware efi \
  'ProductionServer'
```

#### Common Guest OS Identifiers / 常用客户机操作系统标识

| OS | Identifier |
|---|---|
| CentOS 7 64-bit | `centos7_64` |
| CentOS 8/9 64-bit | `centos8_64` |
| Ubuntu 64-bit | `ubuntu64Guest` |
| Debian 10 | `debian10_64Guest` |
| Debian 11 | `debian11_64Guest` |
| Debian 12 | `debian12_64Guest` |
| RHEL 7/8/9 | `rhel7_64Guest` / `rhel8_64Guest` / `rhel9_64Guest` |
| Windows Server 2019 | `windows2019srv_64Guest` |
| Windows Server 2022 | `windows2022srv_64Guest` |
| Generic Linux 64-bit | `other5xLinux64Guest` |

### Attach ISO Image / 挂载 ISO 镜像

```bash
# Attach ISO as CD-ROM / 挂载 ISO 为光驱
govc device.cdrom.insert \
  -vm '<VM_NAME>' \
  -ds 'datastore1' \
  'iso/CentOS-7-x86_64-DVD-2009.iso'

# Eject ISO / 弹出 ISO
govc device.cdrom.eject -vm '<VM_NAME>'
```

### Delete a VM / 删除虚拟机

```bash
# Power off first if running / 如正在运行则先关机
govc vm.power -off -force '<VM_NAME>'

# Delete VM and all its files / 删除 VM 及其所有文件
govc vm.destroy '<VM_NAME>'
```

### Clone a VM / 克隆虚拟机

```bash
# Clone from existing VM / 从现有 VM 克隆
govc vm.power -off '<SOURCE_VM>'          # power off for consistent clone / 关机以确保一致性
govc vm.clone \
  -ds 'datastore1' \
  -vm '<SOURCE_VM>' \
  '<NEW_VM_NAME>'

# Clone with resource customization / 克隆并自定义资源
govc vm.clone \
  -ds 'datastore2' \
  -vm '<SOURCE_VM>' \
  -c 8 \
  -m 16384 \
  -disk 200GB \
  -disk.thin \
  'ClonedVM'
```

### Snapshot Management / 快照管理

```bash
# Take a snapshot / 创建快照
govc snapshot.create -vm '<VM_NAME>' 'snapshot-name'

# Take snapshot with memory state / 创建包含内存状态的快照
govc snapshot.create -vm '<VM_NAME>' -m 'before-upgrade'

# List snapshots / 列出快照
govc snapshot.tree -vm '<VM_NAME>'

# Revert to a specific snapshot / 恢复到指定快照
govc snapshot.revert -vm '<VM_NAME>' 'snapshot-name'

# Revert to current (latest) snapshot / 恢复到当前快照
govc snapshot.revert -vm '<VM_NAME>'

# Delete a specific snapshot / 删除指定快照
govc snapshot.remove -vm '<VM_NAME>' 'snapshot-name'

# Delete all snapshots / 删除所有快照
govc snapshot.remove -vm '<VM_NAME>' '*'
```

### VM Console Access / 虚拟机控制台访问

```bash
# Get console URL / 获取控制台 URL
govc vm.console -vm '<VM_NAME>'

# Enable VNC access (VM must be powered off) / 启用 VNC 访问（VM 需关机）
govc vm.change \
  -e "RemoteDisplay.vnc.enabled=true" \
  -e "RemoteDisplay.vnc.port=5900" \
  -e "RemoteDisplay.vnc.password=yourpassword" \
  -vm '<VM_NAME>'
```

---

## 5. Storage Management / 存储管理

### List Datastores / 列出数据存储

```bash
# Human-readable / 可读格式
govc datastore.info

# JSON format / JSON 格式
govc datastore.info -json
```

### Browse Datastore Contents / 浏览数据存储内容

```bash
# List files in datastore root / 列出数据存储根目录
govc datastore.ls

# List specific directory / 列出指定目录
govc datastore.ls 'datastore1'

# Browse a VM's directory / 浏览 VM 目录
govc datastore.ls 'datastore1/<VM_NAME>'

# Download a file from datastore / 从数据存储下载文件
govc datastore.download 'datastore1/<VM_NAME>/vmware.log' /tmp/vmware.log
```

### Upload Files to Datastore / 上传文件到数据存储

```bash
# Upload an ISO image / 上传 ISO 镜像
govc datastore.upload \
  /local/path/to/CentOS-7.iso \
  'datastore1/iso/CentOS-7.iso'

# Upload a VMDK disk image / 上传 VMDK 磁盘镜像
govc datastore.upload \
  /local/path/to/disk.vmdk \
  'datastore1/templates/disk.vmdk'
```

### Create Directory on Datastore / 在数据存储上创建目录

```bash
govc datastore.mkdir 'datastore1/iso'
govc datastore.mkdir 'datastore1/templates'
```

### Expand a VM Disk / 扩展虚拟机磁盘

```bash
# Resize disk to 200GB (VM must be powered off) / 扩展磁盘至 200GB（VM 需关机）
govc vm.power -off '<VM_NAME>'
govc vm.disk.change -vm '<VM_NAME>' -size 200GB
govc vm.power -on '<VM_NAME>'

# Note: Guest OS also needs to expand its partition
# 注意：客户机操作系统也需要扩展其分区（使用 growpart、resize2fs 等工具）
```

---

## 6. Network Management / 网络管理

### List Networks / 列出网络

```bash
# List available networks / 列出可用网络
govc ls 'network/'

# Detailed vSwitch info / vSwitch 详情
govc host.vswitch.info

# Port group details / 端口组详情
govc host.portgroup.info
```

### Change VM Network / 更改虚拟机网络

```bash
# View current NIC info / 查看当前网卡信息
govc device.info -vm '<VM_NAME>' 'ethernet-*'

# Change network assignment / 更改网络分配
govc vm.network.change \
  -vm '<VM_NAME>' \
  -net 'NewPortGroup' \
  'ethernet-0'
```

### Add Network Adapter / 添加网卡

```bash
govc device.network.add \
  -vm '<VM_NAME>' \
  -net 'VM Network' \
  -adapter vmxnet3
```

---

## 7. Multi-Host Management / 多主机管理

When managing multiple ESXi hosts, do NOT rely on global environment variables. Use per-command assignment or wrapper functions.

> 管理多台 ESXi 主机时，不要依赖全局环境变量。使用逐命令赋值或封装函数。

### Shell Function Approach / Shell 函数方式

```bash
# Define in ~/.bashrc / 定义在 ~/.bashrc 中
esxi() {
  local host="$1"; shift
  GOVC_URL="https://${host}" \
  GOVC_USERNAME='root' \
  GOVC_PASSWORD='your_password' \
  GOVC_INSECURE=true \
  govc "$@"
}

# Usage / 用法:
esxi <your_esxi_ip> about
esxi <your_esxi_ip> vm.info '*'
esxi <another_esxi_ip> vm.info '*'
esxi <your_esxi_ip> vm.power -on 'MyVM'
```

### Script-Based Approach / 脚本方式

```bash
#!/bin/bash
# esxi-manage.sh - Multi-host ESXi management / 多主机管理脚本

HOSTS=("<esxi_host_1>" "<esxi_host_2>")
USER="root"
PASS="your_password"

for host in "${HOSTS[@]}"; do
  echo "=== $host ==="
  GOVC_URL="https://$host" \
  GOVC_USERNAME="$USER" \
  GOVC_PASSWORD="$PASS" \
  GOVC_INSECURE=true \
  govc vm.info '*' 2>/dev/null | grep -E "^(Name:|Power state:|IP address:)"
  echo ""
done
```

### Parallel Operations / 并行操作

```bash
# Run same command on multiple hosts in parallel / 在多台主机上并行执行相同命令
for host in <esxi_host_1> <esxi_host_2>; do
  GOVC_URL="https://$host" GOVC_USERNAME='root' GOVC_PASSWORD='pw' GOVC_INSECURE=true \
    govc vm.info '*' &
done
wait
```

---

## 8. Guest Operations / 客户机操作

These commands require VMware Tools (or open-vm-tools) installed and running inside the guest VM.

> 以下命令要求客户机 VM 内已安装并运行 VMware Tools（或 open-vm-tools）。

```bash
# Execute command inside guest / 在客户机内执行命令
govc guest.run \
  -vm '<VM_NAME>' \
  -l 'root:guest_password' \
  'uname -a'

# Upload file to guest / 上传文件到客户机
govc guest.upload \
  -vm '<VM_NAME>' \
  -l 'root:guest_password' \
  /local/file.txt /remote/path/file.txt

# Download file from guest / 从客户机下载文件
govc guest.download \
  -vm '<VM_NAME>' \
  -l 'root:guest_password' \
  /remote/path/file.txt /local/file.txt
```

### Check VMware Tools Status / 检查 VMware Tools 状态

```bash
govc vm.info -json '<VM_NAME>' | python3 -c "
import json, sys
d = json.load(sys.stdin)
vm = d['virtualMachines'][0]
tools = vm.get('guest', {}).get('toolsStatus', 'N/A')
ip = vm.get('guest', {}).get('ipAddress', 'N/A')
print(f'Tools: {tools}')
print(f'IP:    {ip}')
"
```

---

## 9. Resource Pool and CPU/Memory Management / 资源池与 CPU/内存管理

### Change VM Resources / 更改虚拟机资源

```bash
# Change CPU count (VM must be powered off) / 更改 CPU 数量（VM 需关机）
govc vm.change -vm '<VM_NAME>' -c 8

# Change memory (VM must be powered off) / 更改内存（VM 需关机）
govc vm.change -vm '<VM_NAME>' -m 16384

# Change both / 同时更改
govc vm.change -vm '<VM_NAME>' -c 8 -m 16384

# Enable CPU hot-add / 启用 CPU 热添加
govc vm.change -vm '<VM_NAME>' -e "vcpu.hotadd=true"

# Enable memory hot-add / 启用内存热添加
govc vm.change -vm '<VM_NAME>' -e "memory.hotadd=true"
```

### Resource Pool Management / 资源池管理

```bash
# List resource pools / 列出资源池
govc find / -type p

# Create a resource pool / 创建资源池
govc pool.create \
  -cpu.shares 4000 -mem.shares 4000 \
  '/ha-datacenter/host/localhost./localhost.localdomain/Resources/MyPool'

# Move VM to a resource pool / 将 VM 移至资源池
govc vm.change -vm '<VM_NAME>' -pool 'MyPool'
```

---

## 10. Advanced Operations / 高级操作

### Export VM as OVF/OVA / 导出 VM 为 OVF/OVA

```bash
# Export as OVF directory / 导出为 OVF 目录
govc export.ovf -vm '<VM_NAME>' /tmp/export/

# Export as single OVA file / 导出为单个 OVA 文件
govc export.ovf -vm '<VM_NAME>' -ova /tmp/export/vm.ova
```

### Import OVF/OVA / 导入 OVF/OVA

```bash
# Import OVF template / 导入 OVF 模板
govc import.ovf -name 'ImportedVM' -ds 'datastore1' /path/to/template.ovf

# Import OVA / 导入 OVA
govc import.ova -name 'ImportedVM' -ds 'datastore1' /path/to/template.ova
```

### VM Advanced Settings / 虚拟机高级设置

```bash
# List all advanced settings / 列出所有高级设置
govc vm.info -e '<VM_NAME>'

# Set a specific setting / 设置指定参数
govc vm.change -vm '<VM_NAME>' -e "sched.cpu.latencySensitivity=high"
```

#### Common Useful Settings / 常用高级设置

| Setting | Value | Purpose |
|---|---|---|
| `sched.cpu.latencySensitivity` | `high` | Low latency mode / 低延迟模式 |
| `disk.EnableUUID` | `TRUE` | Expose disk UUID to guest / 向客户机暴露磁盘 UUID |
| `RemoteDisplay.vnc.enabled` | `true` | Enable VNC / 启用 VNC |
| `isolation.tools.copy.disable` | `false` | Allow copy/paste / 允许复制粘贴 |
| `isolation.tools.paste.disable` | `false` | Allow copy/paste / 允许复制粘贴 |

---

## 11. Troubleshooting / 故障排查

### Connection Issues / 连接问题

```bash
# Test basic connectivity / 测试基本连通性
ping -c 3 <your_esxi_ip>

# Test HTTPS port / 测试 HTTPS 端口
curl -sk https://<your_esxi_ip>/ -o /dev/null -w "%{http_code}"
# Expected: 200

# Test SSH (if needed) / 测试 SSH（如需要）
nc -zw3 <your_esxi_ip> 22
```

**Common Error: "ServerFaultCode: Cannot complete login due to an incorrect user name or password"**

> Check GOVC_USERNAME and GOVC_PASSWORD. Ensure no trailing whitespace.

> 检查 GOVC_USERNAME 和 GOVC_PASSWORD，确保没有多余的空格。

**Common Error: "x509: certificate signed by unknown authority"**

> Ensure `GOVC_INSECURE=true` is set. ESXi uses self-signed certificates by default.

> 确保已设置 `GOVC_INSECURE=true`。ESXi 默认使用自签证书。

### VM Won't Power On / 虚拟机无法开机

```bash
# Check for errors / 检查错误信息
govc vm.info -json '<VM_NAME>' | python3 -c "
import json, sys
d = json.load(sys.stdin)
vm = d['virtualMachines'][0]
issues = vm.get('configIssue', [])
state = vm.get('runtime', {}).get('powerState', 'N/A')
reason = vm.get('runtime', {}).get('powerOffReason', 'N/A')
print(f'State: {state}')
print(f'Reason: {reason}')
if issues:
    for issue in issues:
        print(f'Issue: {issue}')
"

# Check disk files exist / 检查磁盘文件是否存在
govc datastore.ls "datastore1/<VM_NAME>"
```

### VM IP Not Showing / 虚拟机 IP 未显示

> VMware Tools may not be installed or not running. Install open-vm-tools in the guest:
>
> 可能未安装或未运行 VMware Tools。在客户机中安装 open-vm-tools：

```bash
# CentOS/RHEL
yum install open-vm-tools && systemctl enable --now vmtoolsd

# Ubuntu/Debian
apt install open-vm-tools && systemctl enable --now vmtoolsd

# Windows: Install VMware Tools from ESXi web UI
# Windows：从 ESXi Web 界面安装 VMware Tools
```

```bash
# Check tools status / 检查 Tools 状态
govc vm.info -json '<VM_NAME>' | python3 -c "
import json, sys
d = json.load(sys.stdin)
vm = d['virtualMachines'][0]
guest = vm.get('guest', {})
print(f'Tools Status:  {guest.get(\"toolsStatus\", \"N/A\")}')
print(f'Tools Version: {guest.get(\"toolsVersion\", \"N/A\")}')
print(f'IP Address:    {guest.get(\"ipAddress\", \"N/A\")}')
print(f'Guest OS:      {guest.get(\"guestFullName\", \"N/A\")}')
"
```

### Host Performance Issues / 主机性能问题

```bash
# Check overall resource usage / 检查整体资源使用率
govc host.info | grep -E "CPU usage|Memory usage"

# List VMs sorted by resource usage / 按资源使用率排序列出 VM
govc vm.info -json '*' | python3 -c "
import json, sys
d = json.load(sys.stdin)
vms = []
for vm in d.get('virtualMachines', []):
    name = vm.get('name', 'N/A')
    quick = vm.get('summary', {}).get('quickStats', {})
    cpu = quick.get('overallCpuUsage', 0)
    mem = quick.get('guestMemoryUsage', 0)
    vms.append((name, cpu, mem))
vms.sort(key=lambda x: x[1] + x[2], reverse=True)
print(f'{\"VM Name\":50} {\"CPU (MHz)\":>10} {\"MEM (MB)\":>10}')
print('-' * 75)
for name, cpu, mem in vms:
    print(f'{name:50} {cpu:>10} {mem:>10}')
"
```

---

## 12. Quick Reference Table / 命令速查表

| Task | Command |
|------|---------|
| Test connection / 测试连接 | `govc about` |
| List VMs / 列出虚拟机 | `govc find / -type m` |
| VM details / 虚拟机详情 | `govc vm.info '<VM>'` |
| Power on / 开机 | `govc vm.power -on '<VM>'` |
| Power off (graceful) / 优雅关机 | `govc vm.power -off '<VM>'` |
| Power off (force) / 强制关机 | `govc vm.power -off -force '<VM>'` |
| Create VM / 创建虚拟机 | `govc vm.create -ds DS -net NET -m MB -c CPU -disk SIZE '<VM>'` |
| Delete VM / 删除虚拟机 | `govc vm.destroy '<VM>'` |
| Clone VM / 克隆虚拟机 | `govc vm.clone -vm SRC 'DST'` |
| Take snapshot / 创建快照 | `govc snapshot.create -vm '<VM>' 'NAME'` |
| List snapshots / 列出快照 | `govc snapshot.tree -vm '<VM>'` |
| Revert snapshot / 恢复快照 | `govc snapshot.revert -vm '<VM>' 'NAME'` |
| Delete snapshot / 删除快照 | `govc snapshot.remove -vm '<VM>' 'NAME'` |
| List datastores / 列出数据存储 | `govc datastore.info` |
| Upload file / 上传文件 | `govc datastore.upload LOCAL REMOTE` |
| Attach ISO / 挂载 ISO | `govc device.cdrom.insert -vm '<VM>' -ds DS 'ISO_PATH'` |
| Change CPU / 更改 CPU | `govc vm.change -vm '<VM>' -c N` |
| Change RAM / 更改内存 | `govc vm.change -vm '<VM>' -m MB` |
| Host info / 主机信息 | `govc host.info` |
| Network info / 网络信息 | `govc host.vswitch.info` |

---

## Common Pitfalls / 常见问题

1. **Forgot GOVC_INSECURE=true.** ESXi uses self-signed certificates. Without this flag, all commands fail with x509 errors. Always set it.

   > **未设置 GOVC_INSECURE=true。** ESXi 使用自签证书，缺少此标志所有命令都会报 x509 错误。

2. **Password with special characters.** If the password contains `$`, `!`, or other shell metacharacters, use single quotes: `GOVC_PASSWORD='p@$$w0rd!'`. Double quotes will cause variable expansion.

   > **密码含特殊字符。** 如密码含 `$`、`!` 等 shell 元字符，使用单引号包裹：`GOVC_PASSWORD='p@$$w0rd!'`。双引号会导致变量展开。

3. **VM name with spaces.** Always quote VM names: `govc vm.info 'My VM Name'`. Unquoted names cause "not found" errors.

   > **VM 名称含空格。** 始终用引号包裹 VM 名称，否则会报 "not found" 错误。

4. **govc vm.info '*' returns nothing.** Use `govc find / -type m` as a fallback to list VMs.

   > **govc vm.info '*' 返回空。** 使用 `govc find / -type m` 作为备选方案列出虚拟机。

5. **Cannot clone a running VM consistently.** Power off the source VM before cloning, or take a snapshot first.

   > **无法一致地克隆运行中的 VM。** 克隆前先关机，或先创建快照。

6. **Disk resize requires guest-side action.** `govc vm.disk.change -size` only expands the virtual disk. The guest OS must also expand its filesystem (growpart, resize2fs, diskmgmt.msc, etc.).

   > **磁盘扩容需要客户机侧操作。** govc 只扩展虚拟磁盘，客户机操作系统还需扩展文件系统。

7. **VMware Tools not installed.** Guest IP, guest operations, and graceful shutdown all require VMware Tools. Install `open-vm-tools` on Linux guests.

   > **未安装 VMware Tools。** 客户机 IP 显示、客户机操作、优雅关机等都依赖 VMware Tools。

8. **Datastore path format.** Use `datastore_name/path/to/file`, NOT `/vmfs/volumes/...`. govc handles the translation.

   > **数据存储路径格式。** 使用 `datastore_name/path/to/file`，不要使用 `/vmfs/volumes/...`。

9. **SSH is not required.** govc uses the vSphere API over HTTPS (port 443). SSH does not need to be enabled for normal management.

   > **无需 SSH。** govc 通过 HTTPS (443端口) 调用 vSphere API，常规管理无需启用 SSH。

10. **Standalone ESXi vs vCenter.** This skill targets standalone ESXi. When connecting to vCenter, path structure changes (e.g., `/Datacenter/vm/...` instead of `/ha-datacenter/vm/...`).

    > **独立 ESXi vs vCenter。** 本文档针对独立 ESXi 主机。连接 vCenter 时路径结构不同。

---

## Verification Checklist / 验证清单

After any ESXi management task, verify:

> 完成任何 ESXi 管理任务后，验证以下项目：

- [ ] `govc about` returns expected ESXi version and host / 返回预期的 ESXi 版本和主机信息
- [ ] VMs are in expected power state / 虚拟机处于预期电源状态
- [ ] No error messages in govc output / govc 输出无错误信息
- [ ] If creating/cloning VM, verify it appears in `govc find / -type m` / 创建/克隆后确认 VM 出现
- [ ] If deleting VM, verify it no longer appears and datastore space is reclaimed / 删除后确认 VM 消失且存储空间已回收
- [ ] If changing resources, verify with `govc vm.info` / 更改资源后用 `govc vm.info` 确认

---

## License

MIT
