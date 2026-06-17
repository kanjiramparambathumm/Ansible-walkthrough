# Introduction to Ansible Workshop - Document 1: Pre-requisite Setup Guide (Docker-Based Lab)

**Audience:** Linux/DevOps engineers, system administrators, workshop participants  
**Purpose:** Build a fully local Ansible lab on a Windows laptop using WSL2, Docker, and two Ubuntu container managed nodes.

> **Scope note**
>
> This guide assumes a Windows 10/11 laptop with administrator access and enough local resources to run WSL2 and Docker containers. No cloud account is required.

## 1. Overview

### Lab architecture

```text
Windows 10/11
└── WSL2 Ubuntu
    ├── Ansible control node
    ├── Docker Engine
    ├── Inventory and playbooks
    └── Managed nodes (containers)
        ├── node1
        └── node2
```

### Key changes from previous VM lab

- VirtualBox removed
- Vagrant removed
- Entire lab runs inside WSL2
- Docker containers act as managed nodes
- SSH connectivity is between the Ansible control node and containers

### Roles in the lab

| Component | Role | Where it runs | What it does |
|---|---|---|---|
| Control node | Ansible controller | WSL2 Ubuntu | Stores inventory, playbooks, roles, and runs Ansible commands |
| Managed node 1 | Target Linux server | Docker container | Receives Ansible tasks over SSH |
| Managed node 2 | Target Linux server | Docker container | Receives Ansible tasks over SSH |
| Container runtime | Lab infrastructure | WSL2 Ubuntu | Builds and runs container nodes |

### Required software

- WSL2
- Ubuntu 24.04 or 22.04
- Docker Engine
- Ansible
- OpenSSH

---

## 2. Install WSL2 and Ubuntu

### 2.1 Verify WSL

Open **PowerShell as Administrator**:

```powershell
wsl --status
```

Install WSL if needed:

```powershell
wsl --install
```

List distributions:

```powershell
wsl --list --online
```

Install Ubuntu if needed:

```powershell
wsl --install -d Ubuntu
```

### 2.2 Verify Ubuntu version inside WSL

```bash
lsb_release -a
```

Use Ubuntu 24.04 or 22.04 for this workshop.

---

## 3. Install Docker Engine inside WSL Ubuntu

Update package metadata:

```bash
sudo apt update
sudo apt -y upgrade
```

Install Docker:

```bash
sudo apt install -y docker.io docker-compose-v2
```

Enable and start Docker:

```bash
sudo systemctl enable --now docker
```

Allow your user to run Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Validate Docker:

```bash
docker --version
docker compose version
docker ps
```

---

## 4. Install Ansible and SSH tooling

Install dependencies:

```bash
sudo apt install -y python3 python3-pip python3-venv pipx openssh-client sshpass git curl
```

Install Ansible:

```bash
pipx install --include-deps ansible
pipx ensurepath
exec $SHELL
```

Validate Ansible:

```bash
ansible --version
ansible-playbook --version
```

---

## 5. Create Docker-based managed nodes

### 5.1 Create project structure

```bash
mkdir -p ~/ansible-workshop-docker/{playbooks,templates,group_vars,host_vars}
cd ~/ansible-workshop-docker
```

### 5.2 Create Dockerfile for SSH-enabled nodes

Create `Dockerfile`:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y openssh-server python3 sudo && \
    mkdir /var/run/sshd && \
    useradd -m -s /bin/bash ansible && \
    echo 'ansible:ansible' | chpasswd && \
    usermod -aG sudo ansible && \
    sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

### 5.3 Create Docker Compose file

Create `docker-compose.yml`:

```yaml
services:
  node1:
    build: .
    container_name: node1
    hostname: node1
    networks:
      ansible_net:
        ipv4_address: 172.20.0.11

  node2:
    build: .
    container_name: node2
    hostname: node2
    networks:
      ansible_net:
        ipv4_address: 172.20.0.12

networks:
  ansible_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

### 5.4 Start lab containers

```bash
docker compose up -d --build
```

Verify:

```bash
docker ps
```

---

## 6. Configure Ansible inventory and settings

### 6.1 Create `inventory.ini`

```ini
[nodegroup]
node1 ansible_host=172.20.0.11 ansible_user=ansible ansible_password=ansible ansible_port=22 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
node2 ansible_host=172.20.0.12 ansible_user=ansible ansible_password=ansible ansible_port=22 ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### 6.2 Create `ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
interpreter_python = auto_silent
stdout_callback = yaml
retry_files_enabled = False
nocows = True

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

---

## 7. Validate the Docker-based lab

Run:

```bash
docker ps
ansible all -m ping -i inventory.ini
```

Expected:

```text
node1 | SUCCESS => {"ping":"pong"}
node2 | SUCCESS => {"ping":"pong"}
```

Optional detailed checks:

```bash
ansible all -m setup -a "filter=ansible_distribution*" -i inventory.ini
ansible-inventory -i inventory.ini --graph
```

---

## 8. Troubleshooting

| Problem | What it usually means | Fix |
|---|---|---|
| `No hosts matched` | Wrong inventory group | Verify group name and inventory path |
| `UNREACHABLE!` | SSH or container networking issue | Confirm containers are up and reachable on the configured IPs |
| `Permission denied` | Wrong SSH credentials | Confirm `ansible_user` and `ansible_password` |
| `python not found` | Python missing in container | Rebuild image and confirm Python install in Dockerfile |
| `docker: permission denied` | User not in docker group | Run `sudo usermod -aG docker $USER` and relogin/newgrp |

Useful commands:

```bash
docker compose ps
docker logs node1
docker logs node2
ansible all -m ping -i inventory.ini -vvv
```

---

## 9. Final lab state checklist

- WSL2 is enabled and Ubuntu 24.04/22.04 is installed
- Docker Engine is installed and running in WSL
- Ansible is installed in WSL
- `docker compose up -d` brings up `node1` and `node2`
- `docker ps` shows both containers as running
- `ansible all -m ping -i inventory.ini` returns `pong` from both nodes

When all checks pass, the instructor can start Module 1 immediately.
