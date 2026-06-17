# Introduction to Ansible Workshop - Document 1: Pre-requisite Setup Guide

**Audience:** Linux/DevOps engineers, system administrators, workshop participants  
**Purpose:** Build a fully local Ansible lab on a Windows laptop using WSL2, VirtualBox, Vagrant, and two Ubuntu managed nodes.

> **Scope note**
>
> This guide assumes a Windows 10/11 laptop with administrator access and enough local resources to run two small Ubuntu virtual machines. No cloud account is required.

## 1. Overview

### Lab architecture

```text
+---------------------------------------------------------------+
|                         Windows Host                          |
|                                                               |
|  +-------------------+     +-------------------------------+  |
|  |       WSL2        |     |           VirtualBox          |  |
|  |   Ubuntu Linux    |     |   Ubuntu VM 1   Ubuntu VM 2   |  |
|  |                   |     |   192.168.56.11 192.168.56.12 |  |
|  |  Ansible control  |<--->|   managed nodes (SSH)         |  |
|  |  node + SSH client|     |                               |  |
|  +-------------------+     +-------------------------------+  |
|                                                               |
|  Windows 10/11 also hosts: Vagrant, VirtualBox GUI, drivers    |
+---------------------------------------------------------------+
```

### Roles in the lab

| Component | Role | Where it runs | What it does |
|---|---|---|---|
| Control node | Ansible controller | WSL2 Ubuntu | Stores inventory, playbooks, roles, and runs Ansible commands |
| Managed node 1 | Target Linux server | VirtualBox VM | Receives Ansible tasks over SSH |
| Managed node 2 | Target Linux server | VirtualBox VM | Receives Ansible tasks over SSH |
| Hypervisor | Virtual machine engine | Windows host | Runs the Ubuntu VMs locally |
| Provisioning tool | VM orchestration | Windows host / terminal access | Creates, starts, and destroys the lab VMs |

### Hardware requirements

| Item | Minimum | Recommended | Notes |
|---|---:|---:|---|
| CPU | 4 logical cores | 6-8 logical cores | Virtualization must be enabled in BIOS/UEFI |
| Memory | 8 GB RAM | 16 GB RAM or more | Two VMs plus WSL and Windows run more smoothly with 16 GB |
| Disk | 25 GB free | 40 GB free | Vagrant boxes and VM disks consume space quickly |
| Network | Local network adapter | Stable internet for first-time downloads | Internet is useful for WSL, boxes, and package installation |
| Firmware | Virtualization enabled | Hyper-V ready if needed | VirtualBox works best when hardware virtualization is enabled |

### Software requirements

| Software | Required version / state | Purpose |
|---|---|---|
| Windows | Windows 10 version 2004+ or Windows 11 | Host operating system |
| WSL2 | Enabled | Linux control-node environment |
| Ubuntu in WSL | Latest supported Ubuntu from Microsoft Store / `wsl --install` | Shell for Ansible |
| VirtualBox | Latest stable release | Local hypervisor |
| Vagrant | Latest stable release | VM lifecycle automation |
| Python 3 | Installed in WSL and on managed nodes | Required by Ansible modules |
| Ansible | Installed inside WSL | Automation tool |
| SSH client | Available in WSL | Used by Ansible to connect |
| OpenSSH server | Installed on managed nodes | Accepts SSH connections |

> **Best practice**
>
> Keep the entire lab project in one folder that is easy to find from both Windows and WSL, such as `C:\Users\<you>\ansible-workshop`. In WSL this appears under `/mnt/c/Users/<you>/ansible-workshop`.

---

## 2. Installing WSL2

### 2.1 Verify prerequisites

Open **PowerShell as Administrator** and run:

```powershell
wsl --status
```

**What this checks:**  
Shows whether WSL is installed, which default version is active, and whether a kernel update is available.

**Expected output example:**

```text
Default Version: 2
WSL version: 2.x.x.x
Kernel version: 5.x.x
```

If WSL is not present or the output shows errors, continue with the installation steps below.

### 2.2 Install WSL2

Run:

```powershell
wsl --install
```

**What this does:**  
Enables the required Windows features, installs the default Linux distribution, and prepares WSL2.

Restart Windows when prompted.

If Ubuntu is not the distro you want, list available distros:

```powershell
wsl --list --online
```

**Expected output example:**

```text
The following is a list of valid distributions that can be installed.
Ubuntu
Debian
openSUSE-Leap-15.6
...
```

Install Ubuntu explicitly if needed:

```powershell
wsl --install -d Ubuntu
```

**What this does:**  
Downloads and installs Ubuntu as the WSL distribution.

### 2.3 Initial Ubuntu configuration

Launch Ubuntu from the Start menu or PowerShell:

```powershell
wsl
```

**First-time prompts:**  
You will be asked to create a Linux username and password.

Inside Ubuntu, update the package lists:

```bash
sudo apt update
```

**What this does:**  
Refreshes package metadata from Ubuntu repositories.

Expected output includes lines such as:

```text
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Reading package lists... Done
```

Upgrade installed packages:

```bash
sudo apt -y upgrade
```

**What this does:**  
Installs the latest security and bug-fix updates available for the installed packages.

Optional but recommended baseline packages:

```bash
sudo apt install -y curl wget git unzip ca-certificates gnupg lsb-release
```

**What this does:**  
Installs tools commonly needed during workshop preparation.

### 2.4 Verify WSL2

From PowerShell:

```powershell
wsl --list --verbose
```

**What this does:**  
Shows installed distros and whether each is using WSL 1 or WSL 2.

**Expected output example:**

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

Inside Ubuntu, confirm the kernel:

```bash
uname -r
```

**What this does:**  
Confirms that the Linux kernel is running.

**Expected output example:**

```text
5.15.0.XX-microsoft-standard-WSL2
```

### 2.5 Troubleshooting WSL

| Symptom | Likely cause | Fix |
|---|---|---|
| `wsl --install` fails | Virtualization disabled in BIOS/UEFI or unsupported Windows build | Enable virtualization and confirm Windows version supports WSL2 |
| Ubuntu does not start | Distro install did not complete | Run `wsl --list --online` and reinstall Ubuntu |
| WSL shows version 1 | Default version not set to 2 | Run `wsl --set-default-version 2` |
| Slow file access | Working in `/mnt/c` too heavily | Keep active code in WSL home if possible; use `/mnt/c` only where Windows access is needed |
| Kernel update error | WSL kernel outdated | Run `wsl --update` from PowerShell |

Useful commands:

```powershell
wsl --set-default-version 2
wsl --update
wsl --shutdown
```

**What these do:**  
Set WSL2 as the default, update the kernel, and stop all WSL distributions so changes take effect.

---

## 3. Installing VirtualBox

### 3.1 Download

Download VirtualBox for **Windows hosts** from Oracle's VirtualBox downloads page.

**What to download:**  
The current stable VirtualBox installer for Windows hosts.

> **Tip**
>
> If your environment allows package managers, you may also be able to install via `winget`, but the standard installer is the most universally supported approach for workshop participants.

### 3.2 Installation steps

Run the downloaded installer and accept the defaults unless your corporate environment requires changes.

During installation, make sure:

- The VirtualBox networking components are installed
- The VirtualBox USB support components are installed if available
- The installation completes without errors

If Windows asks for driver approval, allow it.

### 3.3 Verification

Open a new **Command Prompt** or **PowerShell** window and run:

```powershell
VBoxManage --version
```

**What this does:**  
Confirms that VirtualBox is installed and its command-line tools are available in `PATH`.

**Expected output example:**

```text
7.1.xrXXXX
```

You can also open the VirtualBox GUI and confirm that it starts without errors.

### 3.4 Troubleshooting VirtualBox

| Symptom | Likely cause | Fix |
|---|---|---|
| `VBoxManage` not recognized | PATH not updated yet | Open a new terminal or log out and back in |
| VM startup fails with Hyper-V errors | Windows Hyper-V conflicts with VirtualBox | Disable Hyper-V or use the organization-approved virtualization configuration |
| Network adapter creation fails | VirtualBox networking components blocked | Re-run installer as Administrator |
| VM crashes immediately | Hardware virtualization disabled | Enable VT-x / AMD-V in BIOS/UEFI |

---

## 4. Installing Vagrant

### 4.1 Download

Download Vagrant for Windows from HashiCorp's Vagrant installation page.

### 4.2 Installation steps

Run the downloaded installer and accept the defaults.

> **Important**
>
> The installer normally adds `vagrant` to your system `PATH`. If the command is not found after installation, open a new terminal window or sign out and sign back in.

### 4.3 Verification

Open PowerShell and run:

```powershell
vagrant --version
```

**What this does:**  
Confirms that the Vagrant executable is installed.

**Expected output example:**

```text
Vagrant 2.4.x
```

Show the available commands:

```powershell
vagrant --help
```

**What this does:**  
Verifies the CLI is functioning and displays the top-level command list.

### 4.4 Troubleshooting Vagrant

| Symptom | Likely cause | Fix |
|---|---|---|
| `vagrant` not recognized | PATH not refreshed | Open a new terminal or log out/in |
| Vagrant cannot use VirtualBox | VirtualBox missing or blocked | Reinstall VirtualBox and verify `VBoxManage --version` |
| Box downloads fail | Network / proxy restrictions | Pre-download boxes or configure the proxy |
| `vagrant up` hangs | Hypervisor conflict or permissions | Check Hyper-V status and run terminals with sufficient permissions |

---

## 5. Installing Ansible inside WSL

### 5.1 Update repositories

Open Ubuntu in WSL and run:

```bash
sudo apt update
```

**What this does:**  
Updates the package index before installing new tools.

### 5.2 Install required packages

Install the baseline utilities:

```bash
sudo apt install -y python3 python3-pip python3-venv pipx openssh-client sshpass git curl
```

**What this does:**  
Installs Python, pipx, SSH client tools, and helper utilities.

> **Why `pipx`?**
>
> `pipx` installs command-line Python applications in isolated virtual environments, which helps keep your control node clean.

### 5.3 Install Ansible

Install Ansible through `pipx`:

```bash
pipx install --include-deps ansible
```

**What this does:**  
Installs the full Ansible command-line suite in an isolated environment.

If `pipx` is not yet available in your shell path, refresh it:

```bash
pipx ensurepath
exec $SHELL
```

**What this does:**  
Adds the pipx binary directory to the shell path and reloads the shell.

### 5.4 Verify installation

Check the version:

```bash
ansible --version
```

**What this does:**  
Shows the installed Ansible version and the Python interpreter it uses.

**Expected output example:**

```text
ansible [core 2.x.x]
  config file = None
  python version = 3.x.x
```

Check the related tooling:

```bash
ansible-playbook --version
ansible-inventory --version
```

**What these do:**  
Confirm that the playbook runner and inventory parser are also installed.

### 5.5 Troubleshooting Ansible installation

| Symptom | Likely cause | Fix |
|---|---|---|
| `ansible: command not found` | pipx path not loaded | Run `pipx ensurepath`, restart shell |
| Python dependency errors | Missing `python3-venv` or stale package cache | Reinstall baseline packages and retry |
| `ansible --version` shows the wrong Python | Multiple Python environments | Use `pipx` and avoid mixing system Python with global `pip` installs |
| SSH tests fail | SSH client missing | Install `openssh-client` inside WSL |

---

## 6. Building the Lab Environment

### 6.1 Create the project structure

From **PowerShell** or **WSL**, create a workspace folder.

Example from WSL:

```bash
mkdir -p /mnt/c/Users/$USER/Documents/ansible-workshop
cd /mnt/c/Users/$USER/Documents/ansible-workshop
```

**What this does:**  
Creates a shared folder accessible from both Windows and WSL.

Recommended structure:

```text
ansible-workshop/
├── Vagrantfile
├── ansible.cfg
├── inventory.ini
├── group_vars/
├── host_vars/
├── files/
├── templates/
├── playbooks/
└── roles/
```

Create the directories:

```bash
mkdir -p group_vars host_vars files templates playbooks roles
```

**What this does:**  
Creates a standard project layout for later playbooks and roles.

### 6.2 Create the Vagrantfile

Create a file named `Vagrantfile` in the project root with the following content:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
    vb.cpus = 1
  end

  nodes = [
    { name: "ansible1", ip: "192.168.56.11" },
    { name: "ansible2", ip: "192.168.56.12" }
  ]

  nodes.each do |node|
    config.vm.define node[:name] do |machine|
      machine.vm.hostname = node[:name]
      machine.vm.network "private_network", ip: node[:ip]

      machine.vm.provider "virtualbox" do |vb|
        vb.name = node[:name]
      end

      machine.vm.provision "shell", inline: <<-SHELL
        set -eux
        sudo apt-get update
        sudo DEBIAN_FRONTEND=noninteractive apt-get install -y python3 python3-apt openssh-server
        sudo systemctl enable ssh || true
        sudo systemctl restart ssh || true
      SHELL
    end
  end
end
```

**What this does:**

- Uses an Ubuntu box as the base image
- Creates two VMs
- Assigns static host-only IP addresses
- Ensures Python and SSH are present on each managed node

> **Why Ubuntu Jammy?**
>
> It is a stable baseline for workshop labs and works well with common Ansible examples.

### 6.3 Deploy the two Ubuntu VMs

From the project directory on the Windows host, run:

```powershell
vagrant up
```

**What this does:**  
Downloads the box if needed, creates both VMs, and runs the provisioning script.

**Expected output example:**

```text
Bringing machine 'ansible1' up with 'virtualbox' provider...
Bringing machine 'ansible2' up with 'virtualbox' provider...
==> ansible1: Machine booted and ready!
==> ansible2: Machine booted and ready!
```

Check status:

```powershell
vagrant status
```

**What this does:**  
Confirms that both VMs are running.

**Expected output example:**

```text
Current machine states:

ansible1                  running (virtualbox)
ansible2                  running (virtualbox)
```

### 6.4 Configure networking

The lab uses:

- NAT for outbound internet access
- Host-only / private network for Ansible SSH traffic

**What this means:**  
The VMs can reach package repositories and can also be reached from the WSL control node using the private IP addresses.

If VirtualBox creates a host-only adapter prompt, accept the default adapter creation.

### 6.5 Configure SSH

Vagrant configures SSH access automatically.

To inspect the SSH settings for a VM:

```powershell
vagrant ssh-config ansible1
```

**What this does:**  
Shows the hostname, port, user, and private key location that Vagrant generated.

**Expected output example:**

```text
Host ansible1
  HostName 127.0.0.1
  User vagrant
  Port 2200
  IdentityFile C:/Users/<you>/Documents/ansible-workshop/.vagrant/machines/ansible1/virtualbox/private_key
```

Repeat for `ansible2` if needed.

---

## 7. Connecting Ansible to Vagrant Machines

### 7.1 Generate inventory

Create `inventory.ini`:

```ini
[ubuntu_nodes]
ansible1 ansible_host=192.168.56.11 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/ansible1/virtualbox/private_key
ansible2 ansible_host=192.168.56.12 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/ansible2/virtualbox/private_key

[ubuntu_nodes:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args=-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```

**What this does:**  
Tells Ansible where the managed nodes are and which SSH identity file to use.

> **Tip**
>
> If you prefer, you can also use `vagrant ssh-config` output to build inventory entries automatically. For a beginner workshop, a static inventory is easier to read.

### 7.2 Configure ansible.cfg

Create `ansible.cfg`:

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
```

**What this does:**  
Sets the local inventory file, disables host-key prompts for the lab, and standardizes output.

### 7.3 SSH key usage

Vagrant creates a private key for each machine. Ansible uses that key to connect.

Useful command to inspect the key file path:

```powershell
vagrant ssh-config ansible1
```

**What this does:**  
Confirms the SSH key path and connection settings.

### 7.4 Testing connectivity

From the project directory inside WSL:

```bash
ansible -i inventory.ini ubuntu_nodes -m ping
```

**What this does:**  
Uses the `ping` module to verify SSH access and Python availability on each node.

**Expected output example:**

```text
ansible1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
ansible2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Gather facts:

```bash
ansible -i inventory.ini ubuntu_nodes -m setup
```

**What this does:**  
Collects system information from each managed node.

A shorter version:

```bash
ansible -i inventory.ini ubuntu_nodes -m setup -a "filter=ansible_distribution*"
```

**What this does:**  
Returns only distribution-related facts.

---

## 8. Validation Checklist

Use the following checklist before the workshop begins.

| Check | Command | Expected result |
|---|---|---|
| WSL installed | `wsl --list --verbose` | Ubuntu appears with version `2` |
| Ubuntu starts | `wsl` | Linux shell opens successfully |
| VirtualBox installed | `VBoxManage --version` | Version number prints |
| Vagrant installed | `vagrant --version` | Version number prints |
| Ansible installed | `ansible --version` | `ansible [core ...]` prints |
| VMs created | `vagrant status` | Both VMs show `running` |
| SSH works | `vagrant ssh-config ansible1` | SSH settings and key path display |
| Ansible ping works | `ansible -i inventory.ini ubuntu_nodes -m ping` | `pong` from both hosts |

### Expected output examples

#### WSL verification

```powershell
wsl --list --verbose
```

Expected:

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

#### Vagrant verification

```powershell
vagrant --version
```

Expected:

```text
Vagrant 2.4.x
```

#### VM verification

```powershell
vagrant status
```

Expected:

```text
ansible1                  running (virtualbox)
ansible2                  running (virtualbox)
```

#### SSH verification

```powershell
vagrant ssh-config ansible1
```

Expected includes:

```text
User vagrant
IdentityFile ...
```

#### Ansible ping verification

```bash
ansible -i inventory.ini ubuntu_nodes -m ping
```

Expected:

```text
ansible1 | SUCCESS => {"ping": "pong"}
ansible2 | SUCCESS => {"ping": "pong"}
```

---

## 9. Common Issues and Fixes

| Issue | Symptom | Fix |
|---|---|---|
| SSH failures | `UNREACHABLE!` or `Permission denied` | Confirm the inventory uses the correct private key and username |
| Inventory issues | `No hosts matched` | Check group names and the `-i inventory.ini` flag |
| Python not installed on managed node | `Failed to create temporary directory` or module failures | Re-run `vagrant provision` or SSH into the node and install `python3` |
| Host key issues | SSH warns about changed host keys | Disable host key checking for the lab or clear old `known_hosts` entries |
| Vagrant VM startup issues | VM never reaches `running` | Check Hyper-V conflicts, disk space, and VirtualBox installation |
| VirtualBox network issues | `host-only adapter` or IP conflict errors | Recreate the host-only network or choose unused static IPs |

### SSH failure recovery commands

Clear old host keys if you changed the lab IPs:

```bash
ssh-keygen -R 192.168.56.11
ssh-keygen -R 192.168.56.12
```

**What this does:**  
Removes stale SSH fingerprints from `known_hosts`.

Re-test with verbose SSH:

```bash
ssh -vvv -i .vagrant/machines/ansible1/virtualbox/private_key vagrant@192.168.56.11
```

**What this does:**  
Shows the full SSH handshake so you can see where the connection is failing.

Check Vagrant box health:

```powershell
vagrant global-status
vagrant box list
```

**What these do:**  
Show existing environments and installed boxes.

---

## 10. Final Lab State

### Expected directory structure

```text
ansible-workshop/
├── Vagrantfile
├── ansible.cfg
├── inventory.ini
├── group_vars/
├── host_vars/
├── files/
├── templates/
├── playbooks/
├── roles/
└── .vagrant/
    └── machines/
        ├── ansible1/
        │   └── virtualbox/
        │       └── private_key
        └── ansible2/
            └── virtualbox/
                └── private_key
```

### Successful output examples to capture

Capture screenshots or paste output for the following:

1. `wsl --list --verbose`
2. `VBoxManage --version`
3. `vagrant status`
4. `vagrant ssh-config ansible1`
5. `ansible -i inventory.ini ubuntu_nodes -m ping`

### What success looks like

- WSL launches Ubuntu cleanly
- Vagrant reports both machines as running
- SSH from WSL to each VM succeeds
- Ansible ping returns `pong` from both nodes
- The workshop instructor can start Module 1 immediately

---

## References

- Microsoft Learn: Install WSL
- Oracle VirtualBox: Downloads
- HashiCorp Vagrant: Install Vagrant
- Ansible Core Documentation: Installing Ansible
- Ansible Documentation: Inventory, Playbooks, Loops, Conditionals, Handlers
