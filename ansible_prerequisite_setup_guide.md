# Introduction to Ansible Workshop - Document 1: Pre-requisite Setup Guide (Docker-Based Lab)

**Audience:** Linux/DevOps engineers, system administrators, workshop participants  
**Purpose:** Build a fully local Ansible lab on a Windows laptop using WSL2, Docker Desktop (WSL integration), and two Ubuntu container managed nodes.

> **Scope note**
>
> This guide assumes a Windows 10/11 laptop with administrator access and enough local resources to run WSL2 and Docker containers. No cloud account is required.

## 1. Overview

### Lab architecture

```text
Windows 10/11
└── WSL2 Ubuntu
    ├── Ansible control node
    ├── Docker CLI (connected to Docker Desktop)
    ├── Inventory and playbooks
    └── Managed nodes (containers)
        ├── node1 via 127.0.0.1:2221
        └── node2 via 127.0.0.1:2222
```
### Roles in the lab

| Component | Role | Where it runs | What it does |
|---|---|---|---|
| Control node | Ansible controller | WSL2 Ubuntu | Stores inventory, playbooks, roles, and runs Ansible commands |
| Managed node 1 | Target Linux server | Docker container | Receives Ansible tasks over SSH |
| Managed node 2 | Target Linux server | Docker container | Receives Ansible tasks over SSH |
| Container runtime | Lab infrastructure | Docker Desktop on Windows (via WSL2 integration) | Builds and runs container nodes |

### Required software

- WSL2
- Ubuntu 24.04 or 22.04
- Docker Desktop (WSL2 backend enabled)
- Ansible
- OpenSSH

### Lab authentication model

- SSH login to the containers is password-based using the `ansible` user
- `sudo` inside the containers is passwordless for workshop convenience
- This lab choice keeps Ansible `become: true` examples simple without introducing extra prompt-handling steps

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

## 3. Install Docker Desktop on Windows and integrate with WSL2

Install Docker Desktop on Windows (recommended best practice):

- Download and install Docker Desktop for Windows
- In Docker Desktop settings, enable the WSL2 backend
- Under **Resources > WSL Integration**, enable integration for your Ubuntu distro

Then verify Docker from inside WSL Ubuntu:

```bash
docker --version
docker compose version
docker ps
```

> **Best practice note**
>
> Use a single Docker daemon model. For this workshop, use Docker Desktop with WSL2 integration and avoid running a second standalone Docker daemon inside WSL.

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
mkdir -p ~/ansible-workshop-docker/{playbooks,templates,group_vars,host_vars,ssh-keys/node1,ssh-keys/node2}
cd ~/ansible-workshop-docker
```

Generate persistent SSH host keys one time so rebuilt containers keep the same host identity:

```bash
ssh-keygen -t ed25519 -f ~/ansible-workshop-docker/ssh-keys/node1/ssh_host_ed25519_key -N ""
ssh-keygen -t rsa -b 4096 -f ~/ansible-workshop-docker/ssh-keys/node1/ssh_host_rsa_key -N ""
ssh-keygen -t ed25519 -f ~/ansible-workshop-docker/ssh-keys/node2/ssh_host_ed25519_key -N ""
ssh-keygen -t rsa -b 4096 -f ~/ansible-workshop-docker/ssh-keys/node2/ssh_host_rsa_key -N ""
chmod 600 ~/ansible-workshop-docker/ssh-keys/node1/ssh_host_*_key
chmod 600 ~/ansible-workshop-docker/ssh-keys/node2/ssh_host_*_key
```

These keys stay on the WSL filesystem and are mounted into the containers so SSH host fingerprints remain stable across rebuilds.

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
  echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible && \
  chmod 440 /etc/sudoers.d/ansible && \
    sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

This lab image enables passwordless sudo for the `ansible` user so playbooks using `become: true` work without prompting for a sudo password. SSH login is still password-based.

### 5.3 Create Docker Compose file

Create `docker-compose.yml`:

```yaml
services:
  node1:
    build: .
    container_name: node1
    hostname: node1
    ports:
      - "2221:22"
    volumes:
      - ./ssh-keys/node1/ssh_host_ed25519_key:/etc/ssh/ssh_host_ed25519_key:ro
      - ./ssh-keys/node1/ssh_host_ed25519_key.pub:/etc/ssh/ssh_host_ed25519_key.pub:ro
      - ./ssh-keys/node1/ssh_host_rsa_key:/etc/ssh/ssh_host_rsa_key:ro
      - ./ssh-keys/node1/ssh_host_rsa_key.pub:/etc/ssh/ssh_host_rsa_key.pub:ro

  node2:
    build: .
    container_name: node2
    hostname: node2
    ports:
      - "2222:22"
    volumes:
      - ./ssh-keys/node2/ssh_host_ed25519_key:/etc/ssh/ssh_host_ed25519_key:ro
      - ./ssh-keys/node2/ssh_host_ed25519_key.pub:/etc/ssh/ssh_host_ed25519_key.pub:ro
      - ./ssh-keys/node2/ssh_host_rsa_key:/etc/ssh/ssh_host_rsa_key:ro
      - ./ssh-keys/node2/ssh_host_rsa_key.pub:/etc/ssh/ssh_host_rsa_key.pub:ro
```

> **Why port mapping?**
>
> With Docker Desktop + WSL2 integration, container bridge IPs are not always routable from the WSL distro. Publishing SSH ports on `127.0.0.1` is the most reliable workshop default.

> **Why persistent host keys?**
>
> Rebuilt containers normally generate new SSH host keys, which causes `REMOTE HOST IDENTIFICATION HAS CHANGED` warnings for the same mapped port. Mounting stable keys prevents that churn.

### 5.4 Start lab containers

```bash
docker compose up -d --build
```

Verify:

```bash
docker ps
ssh -o StrictHostKeyChecking=no ansible@127.0.0.1 -p 2221
ssh -o StrictHostKeyChecking=no ansible@127.0.0.1 -p 2222
```

If you are switching from an older ephemeral-key setup, clear the old entries once before reconnecting:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R '[127.0.0.1]:2221'
ssh-keygen -f ~/.ssh/known_hosts -R '[127.0.0.1]:2222'
```

---

## 6. Configure Ansible inventory and settings

### 6.1 Create `inventory.ini`

```ini
[nodegroup]
node1 ansible_host=127.0.0.1 ansible_port=2221
node2 ansible_host=127.0.0.1 ansible_port=2222

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Note: SSH user and password are defined in `group_vars/all.yml` (below), not here, so they're centralized and not repeated per host.

### 6.2 Create `group_vars/all.yml`

Create a `group_vars` directory if it doesn't exist:

```bash
mkdir -p group_vars
```

Create `group_vars/all.yml` with centralized credentials:

```yaml
ansible_user: ansible
ansible_password: ansible
```

This file applies the SSH user and password to all hosts in the inventory without repeating them per host.

### 6.3 Create `ansible.cfg`

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

This configuration centralizes Ansible behavior. SSH credentials come from `group_vars/all.yml`, so they're clean and persistent across the project.

---

## 7. Verify the project structure

Your `ansible-workshop-docker` folder should now contain:

```text
ansible-workshop-docker/
├── Dockerfile
├── docker-compose.yml
├── inventory.ini
├── ansible.cfg
├── group_vars/
│   └── all.yml
├── ssh-keys/
│   ├── node1/
│   │   ├── ssh_host_ed25519_key
│   │   ├── ssh_host_ed25519_key.pub
│   │   ├── ssh_host_rsa_key
│   │   └── ssh_host_rsa_key.pub
│   └── node2/
│       ├── ssh_host_ed25519_key
│       ├── ssh_host_ed25519_key.pub
│       ├── ssh_host_rsa_key
│       └── ssh_host_rsa_key.pub
├── playbooks/
├── templates/
├── group_vars/
└── host_vars/
```

---

## 8. Validate the Docker-based lab

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

## 9. Troubleshooting

| Problem | What it usually means | Fix |
|---|---|---|
| `No hosts matched` | Wrong inventory group | Verify group name and inventory path |
| `UNREACHABLE!` with `No route to host` | Trying to use non-routable container bridge IPs | Use `127.0.0.1` with mapped SSH ports (`2221`, `2222`) in inventory |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | Old SSH host keys are cached for rebuilt containers | Remove the old entries with `ssh-keygen -f ~/.ssh/known_hosts -R '[127.0.0.1]:2221'` and `ssh-keygen -f ~/.ssh/known_hosts -R '[127.0.0.1]:2222'` |
| `Permission denied` | Wrong SSH credentials | Confirm `ansible_user` and `ansible_password` |
| `python not found` | Python missing in container | Rebuild image and confirm Python install in Dockerfile |
| `docker: command not found` | Docker CLI not available in distro shell | Ensure Docker Desktop WSL integration is enabled for your Ubuntu distro |
| Docker connection errors | Docker Desktop not running or WSL integration disabled | Start Docker Desktop and re-check WSL integration settings |

Useful commands:

```bash
docker compose ps
docker logs node1
docker logs node2
ansible all -m ping -i inventory.ini -vvv
```

---

## 10. Final lab state checklist

- WSL2 is enabled and Ubuntu 24.04/22.04 is installed
- Docker Desktop is installed on Windows with WSL2 backend and Ubuntu integration enabled
- Ansible is installed in WSL
- `docker compose up -d` brings up `node1` and `node2`
- `docker ps` shows both containers as running
- Inventory uses `127.0.0.1` with `ansible_port=2221` and `ansible_port=2222`
- `ansible all -m ping -i inventory.ini` returns `pong` from both nodes

When all checks pass, the instructor can start Module 1 immediately.
