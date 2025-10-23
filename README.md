# Proxmox Per-VM Current Usage Status

Query and report **live CPU and memory usage** for all VMs and LXCs across one or more Proxmox nodes using pure API calls - **no sudo required**.  
This playbook is lightweight, read-only, and ideal for audits, CI/CD checks, dashboards, or quick health inspections.

---

## Overview

This Ansible playbook uses the [`community.proxmox`](https://galaxy.ansible.com/community/proxmox) collection to query the Proxmox REST API and print a one-line status per VM/LXC that includes **CPU%** and **RAM used/total**.  
All information is fetched live via API - it does **not** execute commands on Proxmox nodes.  
Results are ordered by **actual used memory (GiB)** so the largest RAM consumers appear first, regardless of configured limits.

### Key Features

- **Pure API flow** - no `pvesh`, no SSH, no root shell needed  
- **Readable summary** - each VM/LXC on one line + totals  
- **Smart ordering** - running guests first, then sorted by `USED_Gi` (descending)  
- **CPU insights** - per-VM `CPU_pct` plus average CPU across running guests  
- **Idempotent and safe** - read-only, no cluster state modifications  
- **Vault-friendly** - integrates cleanly with Ansible Vault and Proxmox API tokens  
- Supports **single-node or cluster** queries automatically

---

## Project Structure

```

.
├── ansible.cfg
├── collections
│   ├── ansible_collections
│   └── requirements.yml
├── group_vars
│   ├── all
│   └── proxmox-bms.yml
├── host_vars
│   └── localhost.yml
├── inventory
│   └── hosts.ini
├── LICENSE
├── pve_vm_status.yml
└── README.md

```

---

## Installation & Setup

### On the **Proxmox VE node** (API target)

```bash
apt install ansible-core sudo python3 python3-pip
pip install ansible-dev-tools --break-system-packages
```

### On the **controller (your machine)**

```bash
python3 -m pip install --user proxmoxer requests --break-system-packages
```

### Install required collections

```bash
ansible-galaxy collection install -r collections/requirements.yml -p ./collections
```

---

## Authentication Setup

Create a dedicated API token in Proxmox:

* **User:** `root@pam` (or another automation user)
* **Token ID:** `ansible-automation`
* **Permissions:** VM.Audit or VM.Monitor (read-only is enough)

Store secrets safely with Ansible Vault:

```bash
ansible-vault encrypt_string --name 'vault_become_password' --vault-id default@prompt 'your_sudo_password'
ansible-vault encrypt_string --name 'pm_api_token_secret' --vault-id default@prompt 'your_proxmox_token_secret'
```

Add them to:

```bash
group_vars/all/main.yml
```

Enable passwordless playbook execution (optional):

```bash
echo 'yourVaultPassword' > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
echo 'export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt' >> ~/.bashrc
source ~/.bashrc
```

---

## Configuration

### `group_vars/all/main.yml`

```yaml
ansible_become: true
ansible_become_method: sudo
ansible_become_password: "{{ vault_become_password }}"
vault_become_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ......
api_user: "root@pam"
api_token_id: "ansible-automation"
api_token_secret: "{{ pm_api_token_secret | trim }}"
pm_api_token_secret: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ......
pm_api_validate_certs: false
pm_node: "starhaven"
# host/IP only, no scheme, no port
pm_api_host: "starhaven"
pm_api_port: 8006
```

### `inventory/hosts.ini`

```ini
[proxmox-bms]
proxmox-vm-bm-machine ansible_host=192.168.0.222 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
```

---

## Usage

Run the playbook to get live **CPU + memory** stats:

```bash
ansible-playbook -i inventory/hosts.ini pve_vm_status.yml
```

### Example Output

```
starhaven/101  dev.bean                        running  CPU: 12.0%  RAM: 4.5Gi/7.9Gi (56.7%)
starhaven/106  navidrome-bean                  running  CPU:  3.2%  RAM: 1.0Gi/2.0Gi (49.0%)
starhaven/109  rhel-bean                       stopped  CPU:  0.0%  RAM: 0.0Gi/4.0Gi (0.0%)
Total: 9 guests | Running: 6 | Stopped: 3 | RAM Used: 13.1 Gi / 55.7 Gi (23.5%) | Avg CPU (running): 7.1%
```

### Optional JSON output

To get structured JSON output (for dashboards or parsing):

```bash
ansible-playbook -i inventory/hosts.ini pve_vm_status.yml -o json
```

---

## Requirements

| Dependency          | Version | Purpose                      |
| ------------------- | ------- | ---------------------------- |
| `community.proxmox` | 1.3.0  | API modules for Proxmox      |
| `community.general` | 11.0.0 | Utility filters & formatting |
| `proxmoxer`         | latest  | Python API backend           |
| `requests`          | latest  | HTTP transport for proxmoxer |

Defined in [`collections/requirements.yml`](collections/requirements.yml).

---

## Notes & Caveats

* Read-only - **no changes** are made to the Proxmox environment
* Works with both VMs (QEMU) and containers (LXC)
* Metrics reflect the current API snapshot; brief spikes may be averaged by Proxmox
* For multi-node clusters, omit `pm_node` in `main.yml` to auto-collect all

### What this playbook does

* Collects VM/LXC metadata via the Proxmox REST API
* Filters out templates
* Enriches each item with:

  * `USED_Gi` (used memory in GiB)
  * `MAX_Gi` (configured memory in GiB)
  * `USED_pct` (percent of max)
  * `CPU_pct` (current guest CPU usage, 0-100%)
  * `vCPUs` (configured virtual CPUs)
* **Sorts by `USED_Gi` (descending)** - running first - then prints a compact list + totals
* Computes **Avg CPU over running** guests for a quick cluster health signal

---

## Tips

To clean up Ansible's noisy `(item={...})` loop output, this playbook already uses:

```yaml
loop_control:
  label: "{{ item.node }}/{{ item.vmid }}"
```

For prettier and proper looking output (no extra JSON or ok-lines):

```bash
ANSIBLE_STDOUT_CALLBACK=community.general.yaml ansible-playbook -i inventory/hosts.ini pve_vm_status.yml
```

---

## Author

Thomas Mozdren, 2025
