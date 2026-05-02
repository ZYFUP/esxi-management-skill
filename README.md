# ESXi Management Skill

> A comprehensive, bilingual skill document for managing VMware ESXi hosts and virtual machines using the **govc** CLI tool.

---

## Description

This repository contains a complete reference for managing VMware ESXi standalone hosts via the command line. It is designed for:

- **AI Agents** (Hermes Agent, Claude, GPT, etc.) that need to perform ESXi management tasks
- **Automation scripts** requiring programmatic control of VMs
- **Human operators** who prefer CLI over the web UI

> 本仓库提供通过命令行管理 VMware ESXi 独立主机的完整参考文档，支持中英双语。适用于 AI 代理、自动化脚本及偏好命令行操作的运维人员。

## Coverage

| Topic | Description |
|-------|-------------|
| Installation | govc binary setup on Linux/macOS |
| Connection | Environment variables, TLS, multi-host patterns |
| Host Monitoring | Hardware info, sensors, resource usage |
| VM Lifecycle | Create, power on/off, clone, delete, snapshot |
| Storage | Datastore management, ISO upload, disk resize |
| Networking | vSwitch, port groups, NIC configuration |
| Multi-Host | Shell functions, scripts, parallel operations |
| Guest Operations | VMware Tools command execution, file transfer |
| Resource Tuning | CPU, memory, resource pools |
| Import/Export | OVF/OVA template management |
| Troubleshooting | Connection errors, VM issues, performance |
| Quick Reference | Common commands cheat sheet |

## Prerequisites

- Network access to ESXi host (HTTPS, port 443)
- ESXi root credentials (or equivalent admin account)
- Linux or macOS machine with `wget` or `curl`

> 前置条件：需具备到 ESXi 主机的网络访问（HTTPS 443端口）、root 凭据、以及装有 wget/curl 的 Linux 或 macOS 环境。

## Quick Start

```bash
# Install govc
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Linux_x86_64.tar.gz
tar -xzf govc_Linux_x86_64.tar.gz govc
sudo mv govc /usr/local/bin/

# Configure connection
export GOVC_URL='https://<your_esxi_ip>'
export GOVC_USERNAME='root'
export GOVC_PASSWORD='your_password'
export GOVC_INSECURE=true

# Verify
govc about

# List VMs
govc vm.info '*'
```

## Skill File

See **[SKILL.md](SKILL.md)** for the complete skill document (bilingual, English primary with Chinese translations).

## License

MIT
