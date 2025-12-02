# 🛠️ Homelab Configuration: K3s + HA (Ansible)

![Ansible](https://img.shields.io/badge/tool-Ansible-EE0000?style=flat&logo=ansible) ![K3s](https://img.shields.io/badge/platform-K3s-FFC61C?style=flat&logo=kubernetes) ![Proxmox](https://img.shields.io/badge/platform-Proxmox-E57000?style=flat&logo=proxmox) ![CI Status](https://github.com/MrStarktastic/homelab-ansible/actions/workflows/validate.yml/badge.svg)

This repository contains the **Ansible** configuration to provision a High-Availability **K3s Kubernetes Cluster** on Proxmox VE.

It is designed as the final stage of a GitOps pipeline. It uses a **Dynamic Inventory** to discover nodes provisioned by [homelab-terraform](https://github.com/MrStarktastic/homelab-terraform) and is automatically triggered via `repository_dispatch` whenever infrastructure changes are applied.

## ✨ Key Features

* **High Availability (HA):** Deploys **Kube-VIP** as a DaemonSet to provide a floating Virtual IP (VIP) for the Kubernetes Control Plane.
* **Dynamic Inventory:** Utilizes the `community.proxmox` plugin to automatically group hosts based on Proxmox Tags (`k3s`, `master`, `worker`). No static `hosts.ini` required.
* **Event-Driven Deployment:** The deployment workflow is triggered automatically by the Terraform repository only when infrastructure changes (Scaling/Re-provisioning) occur.
* **Auto-Upgrades:** The playbook runs the official K3s installer in a way that detects and applies version upgrades automatically on every run.
* **Self-Contained Roles:** Roles manage their own dependencies (downloading scripts, token exchange) to ensure reliability on stateless runners.

## 📂 Repository Structure

```text
.
├── group_vars/
│   └── all.yml              # Global configuration (VIP, User, Interface)
├── inventory/
│   └── proxmox.yml          # Dynamic inventory plugin configuration
├── roles/
│   ├── k3s_init/            # Bootstrap the first Master node + Kube-VIP
│   ├── k3s_masters/         # Join additional Control Plane nodes
│   └── k3s_workers/         # Join Worker nodes
├── .github/
│   └── workflows/           # CI/CD: Validate, Deploy
├── k3s.yml                  # Main Playbook
├── ansible.cfg              # Ansible defaults (pipelining, callbacks)
└── requirements.txt         # Python dependencies for the runner
````

## 🛠️ Prerequisites

  * **Proxmox VE Cluster** (Accessible by the runner).
  * **Self-Hosted GitHub Runner** (Required for network access to Proxmox API and VMs).
  * **Terraform-Provisioned VMs** properly tagged in Proxmox.

## ⚙️ Configuration

### Inventory (`inventory/proxmox.yml`)

Hosts are grouped dynamically using Proxmox tags:

  * **`masters`**: VMs with tags `k3s` AND `master`.
  * **`workers`**: VMs with tags `k3s` AND `worker`.

### Global Variables (`group_vars/all.yml`)

| Variable | Description |
| :--- | :--- |
| `vip_address` | The floating IP for the API Server (Must be reserved/unused in your network). |
| `ansible_user` | SSH User (e.g., `debian`) injected by Cloud-Init. |
| `flannel_iface` | Network interface for K3s/Flannel traffic (default: `eth0`). |

## 🚀 Workflows (GitHub Actions)

### 1\. Validate (`validate.yml`)

  * **Trigger:** Pull Requests.
  * **Action:** Runs `ansible-lint` and validates the syntax of the dynamic inventory config without connecting to hosts.

### 2\. Deploy (`deploy.yml`)

  * **Trigger:**
      * `repository_dispatch` (Event: `infrastructure-changed` from Terraform).
      * Merge to `main`.
      * Manual `workflow_dispatch`.
  * **Action:**
    1.  Installs dependencies (`proxmoxer`, `requests`).
    2.  Queries Proxmox API to build the inventory.
    3.  Runs `k3s.yml` to bootstrap or converge the cluster state.

## 🔐 Secrets

The following GitHub Secrets are required:

| Secret | Description |
| :--- | :--- |
| `PROXMOX_URL` | API Endpoint (e.g., `https://pve:8006/api2/json`) |
| `PROXMOX_USER` | API User (e.g., `root@pam`) |
| `PROXMOX_TOKEN_ID` | API Token ID |
| `PROXMOX_TOKEN_SECRET` | API Token Secret |
| `SSH_PRIVATE_KEY` | Private SSH key matching the public key injected by Terraform |

## 🔗 Related Repositories

  * **Foundation:** [MrStarktastic/homelab-packer](https://github.com/MrStarktastic/homelab-packer)
  * **Infrastructure:** [MrStarktastic/homelab-terraform](https://github.com/MrStarktastic/homelab-terraform)

## 📄 License

This project is licensed under the MIT License.
