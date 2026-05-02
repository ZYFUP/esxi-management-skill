# ESXi Management Skill

A comprehensive skill document for managing VMware ESXi hosts and virtual machines using the **govc** CLI tool. Designed for AI agents, automation scripts, and human operators who need a complete reference for ESXi management via command line.

## What This Covers

- Installation and configuration of govc
- Connection to ESXi hosts (standalone, not vCenter)
- Host information and monitoring (hardware, sensors, resource usage)
- Virtual machine lifecycle: create, power on/off, clone, delete, snapshot
- Storage management: datastores, ISO upload, disk resize
- Network management: vSwitches, port groups, NIC configuration
- Multi-host management patterns
- Guest operations via VMware Tools
- Resource pool and CPU/memory tuning
- Import/export OVF/OVA templates
- Troubleshooting common issues

## Prerequisites

- Network access to ESXi host (HTTPS, port 443)
- ESXi root credentials (or equivalent admin account)
- Linux/macOS machine with wget/curl for govc installation

## Quick Start

```bash
# Install govc
wget -q https://github.com/vmware/govmomi/releases/latest/download/govc_Linux_x86_64.tar.gz
tar -xzf govc_Linux_x86_64.tar.gz govc
sudo mv govc /usr/local/bin/

# Configure connection
export GOVC_URL='https://10.110.199.218'
export GOVC_USERNAME='root'
export GOVC_PASSWORD='your_password'
export GOVC_INSECURE=true

# Verify
govc about

# List VMs
govc vm.info '*'
```

## Skill File

See [SKILL.md](SKILL.md) for the complete skill document. This is the primary reference file designed to be loaded by AI agents (Hermes Agent, Claude, etc.) to perform ESXi management tasks.

## License

MIT
