---
name: esxi-management
description: "Use when managing VMware ESXi hosts via govc CLI. Covers VM lifecycle (create/delete/power/snapshot/clone), host monitoring, storage, networking, ISO upload, and multi-host management. Requires govc installed and ESXi credentials."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [esxi, vmware, govc, virtualization, vm-management, infrastructure]
    related_skills: [remote-deployment, windows-ssh-management]
---

# ESXi Management via govc

## Overview

VMware ESXi is a bare-metal hypervisor for running virtual machines. This skill covers managing ESXi hosts and their VMs using **govc**, the official Go CLI from VMware/govmomi. govc communicates with ESXi's vSphere API over HTTPS (port 443) and does not require SSH to be enabled on the host.

This document is written to be self-contained: a model or human reading only this file should be able to install govc, connect to an ESXi host, and perform any common management task without additional references.

## When to Use

- User asks to manage, monitor, or troubleshoot an ESXi host
- User wants to create, delete, start, stop, snapshot, or clone VMs
- User needs to upload ISO images or manage datastores
- User wants to inspect host hardware, resource usage, or network configuration
- User has multiple ESXi hosts and needs batch operations

## When NOT to Use

- Managing vCenter Server clusters (govc supports vCenter, but this skill focuses on standalone ESXi)
- Managing VMware Workstation/Fusion (different product, different tools)
- Managing Proxmox, Hyper-V, or KVM (different hypervisors entirely)

---

## 1. Installation

### Linux (x86_64)

```bash
# Download latest release
cd /tmp
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Linux_x86_64.tar.gz
tar -xzf govc_Linux_x86_64.tar.gz govc

# Install to system path (requires sudo)
sudo mv govc /usr/local/bin/
sudo chmod +x /usr/local/bin/govc

# -- OR -- install to user-local path (no sudo needed)
mkdir -p ~/.local/bin
mv govc ~/.local/bin/
chmod +x ~/.local/bin/govc
# Ensure ~/.local/bin is in PATH
export PATH="$HOME/.local/bin:$PATH"
```

### macOS (Apple Silicon / Intel)

```bash
# Apple Silicon
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Darwin_arm64.tar.gz
tar -xzf govc_Darwin_arm64.tar.gz govc
sudo mv govc /usr/local/bin/

# Intel
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Darwin_x86_64.tar.gz
tar -xzf govc_Darwin_x86_64.tar.gz govc
sudo mv govc /usr/local/bin/
```

### Verify Installation

```bash
govc version
# Expected: govc 0.53.1 (or newer)
```

---

## 2. Connection Configuration

govc uses environment variables for connection details. These must be set before every command, or exported in the shell session.

### Required Environment Variables

```bash
export GOVC_URL='https://<ESXi_IP>'
export GOVC_USERNAME='<username>'        # usually 'root'
export GOVC_PASSWORD='<password>'
export GOVC_INSECURE=true                # skip TLS certificate verification (required for self-signed certs)
```

### Optional Environment Variables

```bash
export GOVC_DATASTORE='datastore1'       # default datastore for operations
export GOVC_NETWORK='VM Network'         # default network/portgroup
export GOVC_RESOURCE_POOL='/'            # resource pool path (ESXi standalone uses '/')
export GOVC_DATACENTER='ha-datacenter'   # datacenter name (ESXi standalone uses 'ha-datacenter')
```

### Persisting Configuration

For a single host, add to `~/.bashrc` or `~/.zshrc`:

```bash
export GOVC_URL='https://10.110.199.218'
export GOVC_USERNAME='root'
export GOVC_PASSWORD='your_password'
export GOVC_INSECURE=true
```

For multiple hosts, do NOT persist globally. Instead, set per-command or use a wrapper script (see Multi-Host Management section).

### Verify Connection

```bash
govc about
# Expected output:
#   FullName:     VMware ESXi 8.0.3 build-XXXXXXXX
#   Name:         VMware ESXi
#   Vendor:       VMware, Inc.
#   Version:      8.0.3
#   API type:     HostAgent
```

---

## 3. Host Information and Monitoring

### Basic Host Info

```bash
# Human-readable summary
govc host.info

# JSON output (for programmatic parsing)
govc host.info -json
```

### Detailed Hardware Info (via JSON parsing)

```bash
govc host.info -json | python3 -c "
import json, sys
d = json.load(sys.stdin)
h = d['hostSystems'][0]
hw = h.get('hardware', {})
ci = hw.get('cpuInfo', {})
cp = hw.get('cpuPkg', [])

print(f'CPU packages: {ci.get(\"numCpuPackages\", \"?\")}')
print(f'CPU cores:    {ci.get(\"numCpuCores\", \"?\")}')
print(f'CPU threads:  {ci.get(\"numCpuThreads\", \"?\")}')
print(f'CPU freq:     {ci.get(\"hz\", 0) / 1e9:.2f} GHz')
print(f'Memory:       {hw.get(\"memorySize\", 0) // 1024 // 1024 // 1024} GB')
print()
for i, pkg in enumerate(cp):
    print(f'CPU{i}: {pkg.get(\"description\", \"N/A\")}')
"
```

### Real-Time Resource Usage

```bash
# CPU and memory usage from host.info
govc host.info | grep -E "CPU usage|Memory usage"

# More detailed stats via metrics (requires vCenter in most cases)
# For standalone ESXi, use the host.summary.quickStats field:
govc host.info -json | python3 -c "
import json, sys
d = json.load(sys.stdin)
h = d['hostSystems'][0]
qs = h.get('summary', {}).get('quickStats', {})
print(f'CPU usage:    {qs.get(\"overallCpuUsage\", \"?\")} MHz')
print(f'Memory usage: {qs.get(\"overallMemoryUsage\", \"?\")} MB')
print(f'Uptime:       {qs.get(\"uptime\", \"?\")} seconds')
"
```

### Host Health / Sensors

```bash
govc host.info -json | python3 -c "
import json, sys
d = json.load(sys.stdin)
h = d['hostSystems'][0]
sensors = h.get('runtime', {}).get('healthSystemRuntime', {}).get('systemHealthInfo', {}).get('numericSensorInfo', [])
for s in sensors[:20]:  # first 20 sensors
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

## 4. Virtual Machine Management

### List All VMs

```bash
# List VM paths (tree view)
govc find / -type m

# Human-readable VM info
govc vm.info '*'

# Single VM
govc vm.info '<VM_NAME>'

# JSON output
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

### VM Power Operations

```bash
# Power on
govc vm.power -on '<VM_NAME>'

# Power off (graceful, sends ACPI shutdown signal)
govc vm.power -off '<VM_NAME>'

# Power off (hard, equivalent to pulling the power cord)
govc vm.power -off -force '<VM_NAME>'

# Restart (graceful reboot)
govc vm.power -reset '<VM_NAME>'

# Suspend
govc vm.power -suspend '<VM_NAME>'
```

### Create a New VM

```bash
# Minimal: 1 CPU, 1GB RAM, 16GB disk, thin provisioned
govc vm.create \
  -ds 'datastore1' \
  -net 'VM Network' \
  -m 1024 \
  -c 1 \
  -disk 16GB \
  -disk.controller pvscsi \
  -g centos7_64 \
  'MyNewVM'

# Full options example
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

# Common guest OS identifiers (-g flag):
#   centos7_64, centos8_64, centos9_64
#   ubuntu64Guest, ubuntu64Guest (Ubuntu 22.04+)
#   debian10_64Guest, debian11_64Guest, debian12_64Guest
#   windows2019srv_64Guest, windows2022srv_64Guest
#   rhel7_64Guest, rhel8_64Guest, rhel9_64Guest
#   other5xLinux64Guest (generic Linux 64-bit)
```

### Attach ISO Image to VM

```bash
# First, upload ISO to datastore (see Storage section)
# Then attach it as a CD-ROM device
govc device.cdrom.insert \
  -vm '<VM_NAME>' \
  -ds 'datastore1' \
  '/path/on/datastore/iso/CentOS-7-x86_64-DVD-2009.iso'

# Eject ISO
govc device.cdrom.eject -vm '<VM_NAME>'
```

### Delete a VM

```bash
# Power off first if running
govc vm.power -off -force '<VM_NAME>'

# Delete VM and all its files from datastore
govc vm.destroy '<VM_NAME>'
```

### Clone a VM

```bash
# Clone from existing VM (source must be powered off for consistent clone)
govc vm.power -off '<SOURCE_VM>'
govc vm.clone \
  -ds 'datastore1' \
  -vm '<SOURCE_VM>' \
  '<NEW_VM_NAME>'

# Clone with customization (different resources)
govc vm.clone \
  -ds 'datastore2' \
  -vm '<SOURCE_VM>' \
  -c 8 \
  -m 16384 \
  -disk 200GB \
  -disk.thin \
  'ClonedVM'
```

### Snapshot Management

```bash
# Take a snapshot
govc snapshot.create -vm '<VM_NAME>' 'snapshot-name'

# Take snapshot with memory state (allows reverting to running state)
govc snapshot.create -vm '<VM_NAME>' -m 'before-upgrade'

# List snapshots
govc snapshot.tree -vm '<VM_NAME>'

# Revert to a specific snapshot
govc snapshot.revert -vm '<VM_NAME>' 'snapshot-name'

# Revert to current (latest) snapshot
govc snapshot.revert -vm '<VM_NAME>'

# Delete a specific snapshot
govc snapshot.remove -vm '<VM_NAME>' 'snapshot-name'

# Delete all snapshots
govc snapshot.remove -vm '<VM_NAME>' '*'
```

### VM Console Access

```bash
# Get VM console URL (for VNC/VMRC)
govc vm.console -vm '<VM_NAME>'

# Enable VNC access on a VM (requires VM to be powered off)
govc vm.change \
  -e "RemoteDisplay.vnc.enabled=true" \
  -e "RemoteDisplay.vnc.port=5900" \
  -e "RemoteDisplay.vnc.password=yourpassword" \
  -vm '<VM_NAME>'
```

---

## 5. Storage Management

### List Datastores

```bash
# Human-readable
govc datastore.info

# JSON
govc datastore.info -json
```

### Browse Datastore Contents

```bash
# List files in datastore root
govc datastore.ls

# List files in a specific directory
govc datastore.ls 'datastore1'

# Browse a VM's directory
govc datastore.ls 'datastore1/<VM_NAME>'

# Download a file from datastore
govc datastore.download 'datastore1/<VM_NAME>/vmware.log' /tmp/vmware.log
```

### Upload Files to Datastore

```bash
# Upload an ISO image
govc datastore.upload \
  /local/path/to/CentOS-7.iso \
  'datastore1/iso/CentOS-7.iso'

# Upload a VMDK disk image
govc datastore.upload \
  /local/path/to/disk.vmdk \
  'datastore1/templates/disk.vmdk'
```

### Create a Directory on Datastore

```bash
govc datastore.mkdir 'datastore1/iso'
govc datastore.mkdir 'datastore1/templates'
```

### Expand a VM Disk

```bash
# Resize disk to 200GB (VM should be powered off)
govc vm.power -off '<VM_NAME>'
govc vm.disk.change -vm '<VM_NAME>' -size 200GB
govc vm.power -on '<VM_NAME>'
# Note: Guest OS also needs to expand its partition (resize2fs, growpart, etc.)
```

---

## 6. Network Management

### List Networks / Port Groups

```bash
# List available networks
govc ls 'network/'

# Detailed vSwitch info
govc host.vswitch.info

# Port group details
govc host.portgroup.info
```

### Change VM Network

```bash
# Move VM's NIC to a different port group
govc device.info -vm '<VM_NAME>' 'ethernet-*'

# Change network assignment
govc vm.network.change \
  -vm '<VM_NAME>' \
  -net 'NewPortGroup' \
  'ethernet-0'
```

### Add a Network Adapter to VM

```bash
govc device.network.add \
  -vm '<VM_NAME>' \
  -net 'VM Network' \
  -adapter vmxnet3
```

---

## 7. Multi-Host Management

When managing multiple ESXi hosts, do NOT rely on global environment variables. Use per-command assignment or wrapper functions.

### Shell Function Approach

```bash
# Define in ~/.bashrc or source before use
esxi() {
  local host="$1"; shift
  GOVC_URL="https://${host}" \
  GOVC_USERNAME='root' \
  GOVC_PASSWORD='your_password' \
  GOVC_INSECURE=true \
  /usr/local/bin/govc "$@"
}

# Usage:
esxi 10.110.199.218 about
esxi 10.110.199.218 vm.info '*'
esxi 10.110.199.220 vm.info '*'
esxi 10.110.199.218 vm.power -on 'MyVM'
```

### Script-Based Approach

```bash
#!/bin/bash
# esxi-manage.sh - Multi-host ESXi management script

HOSTS=("10.110.199.218" "10.110.199.220")
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

### Parallel Operations

```bash
# Run the same command on multiple hosts in parallel
for host in 10.110.199.218 10.110.199.220; do
  GOVC_URL="https://$host" GOVC_USERNAME='root' GOVC_PASSWORD='pw' GOVC_INSECURE=true \
    govc vm.info '*' &
done
wait
```

---

## 8. Guest Operations (VM Tools Required)

These commands require VMware Tools (or open-vm-tools) to be installed and running inside the guest VM.

### Execute Command Inside Guest

```bash
# Run a command in the guest VM (requires guest credentials)
govc guest.run \
  -vm '<VM_NAME>' \
  -l 'root:guest_password' \
  'uname -a'

# Copy a file into the guest
govc guest.upload \
  -vm '<VM_NAME>' \
  -l 'root:guest_password' \
  /local/file.txt /remote/path/file.txt

# Copy a file from the guest
govc guest.download \
  -vm '<VM_NAME>' \
  -l 'root:guest_password' \
  /remote/path/file.txt /local/file.txt
```

### Set Guest Info (IP, hostname)

```bash
# Check if VMware Tools is running
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

## 9. Resource Pool and CPU/Memory Management

### Change VM Resources

```bash
# Change CPU count (VM must be powered off)
govc vm.change -vm '<VM_NAME>' -c 8

# Change memory (VM must be powered off)
govc vm.change -vm '<VM_NAME>' -m 16384

# Change both
govc vm.change -vm '<VM_NAME>' -c 8 -m 16384

# Enable CPU hot-add
govc vm.change -vm '<VM_NAME>' -e "vcpu.hotadd=true"

# Enable memory hot-add
govc vm.change -vm '<VM_NAME>' -e "memory.hotadd=true"
```

### Resource Pool Management

```bash
# ESXi standalone has a single root resource pool '/'
# List resource pools
govc find / -type p

# Create a resource pool
govc pool.create -cpu.shares 4000 -mem.shares 4000 '/ha-datacenter/host/localhost./localhost.localdomain/Resources/MyPool'

# Move VM to a resource pool
govc vm.change -vm '<VM_NAME>' -pool 'MyPool'
```

---

## 10. Advanced Operations

### Export VM as OVF/OVA

```bash
# Export VM to local directory
govc export.ovf -vm '<VM_NAME>' /tmp/export/

# Export as OVA single file
govc export.ovf -vm '<VM_NAME>' -ova /tmp/export/vm.ova
```

### Import OVF/OVA

```bash
# Import an OVF template
govc import.ovf -name 'ImportedVM' -ds 'datastore1' /path/to/template.ovf

# Import an OVA
govc import.ova -name 'ImportedVM' -ds 'datastore1' /path/to/template.ova
```

### VM Advanced Settings

```bash
# List all advanced settings
govc vm.info -e '<VM_NAME>'

# Set a specific advanced setting
govc vm.change -vm '<VM_NAME>' -e "sched.cpu.latencySensitivity=high"

# Common useful settings:
#   sched.cpu.latencySensitivity=high     -- low latency mode
#   disk.EnableUUID=TRUE                  -- expose disk UUID to guest
#   RemoteDisplay.vnc.enabled=true        -- enable VNC
#   isolation.tools.copy.disable=false    -- allow copy/paste
#   isolation.tools.paste.disable=false   -- allow copy/paste
```

### Datastore Maintenance

```bash
# Put datastore in maintenance mode (vCenter only, not standalone ESXi)
# For standalone ESXi, just ensure no VMs are using it

# Rescan storage adapters
govc host.storage.info
```

---

## 11. Troubleshooting

### Connection Issues

```bash
# Test basic connectivity
ping -c 3 <ESXi_IP>

# Test HTTPS port
curl -sk https://<ESXi_IP>/ -o /dev/null -w "%{http_code}"
# Expected: 200

# Test SSH (if needed)
nc -zw3 <ESXi_IP> 22

# Common error: "govc: ServerFaultCode: Cannot complete login due to an incorrect user name or password"
# -> Check GOVC_USERNAME and GOVC_PASSWORD

# Common error: "govc: Post \"https://.../sdk\": x509: certificate signed by unknown authority"
# -> Ensure GOVC_INSECURE=true is set
```

### VM Won't Power On

```bash
# Check for errors
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

# Check if disk files exist
govc datastore.ls "datastore1/<VM_NAME>"
```

### VM IP Not Showing

```bash
# VMware Tools may not be installed or not running
# Install open-vm-tools in the guest:
#   CentOS/RHEL: yum install open-vm-tools && systemctl enable --now vmtoolsd
#   Ubuntu/Debian: apt install open-vm-tools && systemctl enable --now vmtoolsd
#   Windows: Install VMware Tools from the ESXi web UI

# Check tools status
govc vm.info -json '<VM_NAME>' | python3 -c "
import json, sys
d = json.load(sys.stdin)
vm = d['virtualMachines'][0]
guest = vm.get('guest', {})
print(f'Tools status: {guest.get(\"toolsStatus\", \"N/A\")}')
print(f'Tools version: {guest.get(\"toolsVersion\", \"N/A\")}')
print(f'IP: {guest.get(\"ipAddress\", \"N/A\")}')
print(f'Guest OS: {guest.get(\"guestFullName\", \"N/A\")}')
"
```

### Host Performance Issues

```bash
# Check overall resource usage
govc host.info | grep -E "CPU usage|Memory usage"

# List VMs sorted by resource usage
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

## 12. Quick Reference Table

| Task | Command |
|------|---------|
| Test connection | `govc about` |
| List VMs | `govc find / -type m` |
| VM details | `govc vm.info '<VM>'` |
| Power on | `govc vm.power -on '<VM>'` |
| Power off (graceful) | `govc vm.power -off '<VM>'` |
| Power off (force) | `govc vm.power -off -force '<VM>'` |
| Create VM | `govc vm.create -ds DS -net NET -m MB -c CPU -disk SIZE '<VM>'` |
| Delete VM | `govc vm.destroy '<VM>'` |
| Clone VM | `govc vm.clone -vm SRC 'DST'` |
| Take snapshot | `govc snapshot.create -vm '<VM>' 'NAME'` |
| List snapshots | `govc snapshot.tree -vm '<VM>'` |
| Revert snapshot | `govc snapshot.revert -vm '<VM>' 'NAME'` |
| Delete snapshot | `govc snapshot.remove -vm '<VM>' 'NAME'` |
| List datastores | `govc datastore.info` |
| Upload file | `govc datastore.upload LOCAL REMOTE` |
| Attach ISO | `govc device.cdrom.insert -vm '<VM>' -ds DS 'ISO_PATH'` |
| Change CPU | `govc vm.change -vm '<VM>' -c N` |
| Change RAM | `govc vm.change -vm '<VM>' -m MB` |
| Host info | `govc host.info` |
| Network info | `govc host.vswitch.info` |

---

## Common Pitfalls

1. **Forgot GOVC_INSECURE=true.** ESXi uses self-signed certificates. Without this flag, all commands fail with x509 errors. Always set it.

2. **Using single quotes for GOVC_PASSWORD with special characters.** If the password contains `$`, `!`, or other shell metacharacters, use single quotes: `GOVC_PASSWORD='p@$$w0rd!'`. Double quotes will cause variable expansion.

3. **VM name with spaces.** Always quote VM names: `govc vm.info 'My VM Name'`. Unquoted names with spaces cause "not found" errors.

4. **govc vm.info '*' returns nothing.** This can happen if the JSON parser fails silently. Use `govc find / -type m` as a fallback to list VMs, then query individually.

5. **Cannot clone a running VM consistently.** Power off the source VM before cloning, or take a snapshot first for a consistent clone.

6. **Disk resize requires guest-side action too.** `govc vm.disk.change -size` only expands the virtual disk. The guest OS must also expand its filesystem (growpart, resize2fs, etc.).

7. **VMware Tools not installed.** Guest IP address, guest operations (run/upload/download), and graceful shutdown all require VMware Tools. Install `open-vm-tools` on Linux guests.

8. **Datastore path format.** When referencing datastore paths, use `datastore_name/path/to/file`, NOT `/vmfs/volumes/...`. govc handles the translation.

9. **SSH is not required.** govc uses the vSphere API over HTTPS (port 443). SSH does not need to be enabled for normal management. Only enable SSH for advanced troubleshooting.

10. **Standalone ESXi vs vCenter.** This skill targets standalone ESXi hosts. If connecting to vCenter, the path structure changes (e.g., `/Datacenter/vm/...` instead of `/ha-datacenter/vm/...`) and some commands behave differently.

---

## Verification Checklist

After any ESXi management task, verify:

- [ ] `govc about` returns expected ESXi version and host
- [ ] VMs are in expected power state (`govc vm.info '*' | grep "Power state"`)
- [ ] No error messages in govc output
- [ ] If creating/cloning VM, verify it appears in `govc find / -type m`
- [ ] If deleting VM, verify it no longer appears and datastore space is reclaimed
- [ ] If changing resources, verify with `govc vm.info '<VM>'` that CPU/memory reflect changes

---

## ESXi Host Reference (Example Configuration)

This section documents a real-world ESXi setup for reference:

### Host A: 10.110.199.218

- VMware ESXi 8.0.3 (build-24585383)
- Vendor: UNISINSIGHT (New H3C)
- CPU: 2x Intel Xeon Silver 4314 @ 2.40GHz (32 cores / 64 threads)
- Memory: 256 GB
- Storage: datastore1 (430.8 GB VMFS), datastore2 (7.7 TB VMFS)
- Network: vSwitch0 with Management Network, VM Network, Proxy_Network(Special)
- Boot time: 2025-09-19

### Host B: 10.110.199.220

- VMware ESXi 8.0.3 (build-25067014)
- Vendor: New H3C
- CPU: 2x Intel Xeon Gold 5220R @ 2.20GHz (48 cores / 96 threads)
- Memory: 256 GB
- Storage: datastore1 (2.1 TB VMFS)
- Network: vSwitch0
- Boot time: 2026-04-14

### Credentials

- Username: root
- Password: [stored separately, do not commit to repository]
- Both hosts use the same credentials.
