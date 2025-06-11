---
title: "Pelican Panel Ansible Automation"
date: 2025-06-11
weight: 30
draft: false
---

# Pelican Panel Ansible Automation

## Overview

{{< hint info >}}
Automated deployment of <a href="https://pelican.dev">Pelican</a> Panel and Wings daemon using Ansible playbooks.
{{< /hint >}}

## Prerequisites

- **Ansible** installed on your local machine
- **SSH access** to target servers 

## Panel Installation

### Automated Panel Install
[View Panel Ansible Playbook on GitHub](https://github.com/M7mdBinGhaith/homelab/blob/main/iac/ansible/install/pelican-panel.yml)

{{< hint tip >}}
Quick setup using Ansible playbook for complete Panel installation.
{{< /hint >}}

```bash
# Download and run the Panel Ansible playbook
wget https://raw.githubusercontent.com/M7mdBinGhaith/homelab/main/iac/ansible/install/pelican-panel.yml
ansible-playbook -i "your-panel-server-ip," pelican-panel.yml
```



## Wings Node Deployment

[View Wings Node API Deployment Playbook on GitHub](https://github.com/M7mdBinGhaith/homelab/blob/main/iac/ansible/deploy/pelican-node-setup.yml)

### Node Setup with API Integration

{{< hint info >}}
<ul>
<li><b>Panel URL:</b> Your Pelican Panel domain (e.g., http://panel.example.com)</li>
<li><b>API Key:</b> Generate from Panel Admin with appropriate permissions</li>
<li><b>Node Configuration:</b> Automatically created via API</li>
</ul>
{{< /hint >}}

### Basic Usage

```bash
ansible-playbook deploy/pelican-node-setup.yml \
  -e "pelican_url=http://10.30.30.122" \
  -e "pelican_api_key=peli_YOUR_API_KEY" \
  -e "node_name=GameServer01" \
  -e "node_fqdn=10.30.30.123" \
  -e "node_memory=8192" \
  -e "node_disk=50000" \
  -e "wings_target_host=10.30.30.123" \
  --ask-become-pass
```

### Required Variables

- `pelican_url` - Panel server URL
- `pelican_api_key` - API key from Panel admin (starts with `peli_`)
- `node_name` - Display name for the node
- `node_fqdn` - Game server IP/domain
- `node_memory` - Memory allocation in MB
- `node_disk` - Disk allocation in MB

### Optional Variables

- `wings_target_host` - Target server for Wings (defaults to `node_fqdn`)
- `wings_ansible_user` - SSH user (defaults to `debian`)
- `node_scheme` - `http` or `https` (defaults to `http`)
- `node_cpu` - CPU allocation (defaults to `0`)
- `node_location_id` - Panel location ID (defaults to `1`)
- `wings_force_reconfigure` - Override existing Wings config (defaults to `false`)

## Safety Features

### Configuration Protection

{{< hint warning >}}
The playbook automatically detects existing Wings configurations and prevents accidental overwrites.
{{< /hint >}}

**First time installation:**
```bash
ansible-playbook deploy/pelican-node-setup.yml \
  -e "pelican_url=http://10.30.30.122" \
  -e "pelican_api_key=peli_YOUR_API_KEY" \
  -e "node_name=GameServer01" \
  -e "node_fqdn=10.30.30.123" \
  -e "node_memory=8192" \
  -e "node_disk=50000" \
  -e "wings_target_host=10.30.30.123"
```

**Reconfigure existing Wings installation:**
```bash
ansible-playbook deploy/pelican-node-setup.yml \
  -e "pelican_url=http://10.30.30.122" \
  -e "pelican_api_key=peli_YOUR_API_KEY" \
  -e "node_name=GameServer01" \
  -e "node_fqdn=10.30.30.123" \
  -e "node_memory=8192" \
  -e "node_disk=50000" \
  -e "wings_target_host=10.30.30.123" \
  -e "wings_force_reconfigure=true"
```

## Bulk Deployment

### Multiple Server Setup

**New installations:**
```bash
for host in 10.30.30.123 10.30.30.124 10.30.30.125; do
  ansible-playbook deploy/pelican-node-setup.yml \
    -e "pelican_url=http://10.30.30.122" \
    -e "pelican_api_key=peli_YOUR_API_KEY" \
    -e "node_name=GameServer-${host##*.}" \
    -e "node_fqdn=${host}" \
    -e "node_memory=8192" \
    -e "node_disk=50000" \
    -e "wings_target_host=${host}"
done
```

**Reconfigure existing installations:**
```bash
for host in 10.30.30.123 10.30.30.124 10.30.30.125; do
  ansible-playbook deploy/pelican-node-setup.yml \
    -e "pelican_url=http://10.30.30.122" \
    -e "pelican_api_key=peli_YOUR_API_KEY" \
    -e "node_name=GameServer-${host##*.}" \
    -e "node_fqdn=${host}" \
    -e "node_memory=8192" \
    -e "node_disk=50000" \
    -e "wings_target_host=${host}" \
    -e "wings_force_reconfigure=true"
done
```

## Features

### Automation Capabilities

- **API Node Creation** - Creates nodes via Pelican API
- **Token Retrieval** - Automatically retrieves Wings authentication token
- **Safety Checks** - Verifies existing Wings configuration
- **Wings Installation** - Installs and configures Wings daemon
- **Service Management** - Starts and enables Wings service


## Implementation Requirements

### Dependencies

- Common tasks available: `../common/system-update.yml` and `../common/install-docker.yml`
- SSH connectivity to target hosts

