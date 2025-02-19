# CAPEv2 with Docker-Virt-Manager on Ubuntu 22.04

This guide provides a step-by-step process for setting up [CAPEv2](https://github.com/kevoreilly/CAPEv2) with [Docker-Virt-Manager](https://github.com/m-bers/docker-virt-manager) on **Ubuntu 22.04**.

## Prerequisites
Ensure you are using `Ubuntu 22.04 Live Server` like `ubuntu-22.04*live-server-amd64.iso`, which can be downloaded from
[Ubuntu 22.04 Old Releases](https://old-releases.ubuntu.com/releases/22.04.1/).

> ⚠️ **Important:** Running all these commands as `cape` user is recommended. Best is to create `cape` user during OS installation.

- CPU Total Processor Cores: 4 
  - Number of Processors Recommended: 2
  - Number of Cores per Processor Recommended: 2
- CPU with Virtualization Support (VT-x/AMD-V)
- Memory: 8GB RAM (16GB - 32GB Recommended)
- Hard Disk: 200GB

## Step 1: Install Docker and Dependencies
```bash
# Update system and install necessary dependencies
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker and plugins
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Refer to the official Docker documentation for more information: [Docker Installation Guide](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

## Step 2: Install KVM and Virtualization Dependencies
```bash
# Install necessary dependencies
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst uidmap libvirt-dev libguestfs-tools

# Enable and check libvirtd service
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd

# Add current user to libvirt and kvm groups
sudo usermod -aG libvirt,kvm $USER
newgrp libvirt
newgrp kvm

# Verify KVM installation
virsh list --all
sudo kvm-ok
```

## Step 3: Configure Virtualization Directories
```bash
# Create necessary directories and set permissions
sudo mkdir -p /var/lib/libvirt/images
sudo chown -R libvirt-qemu:kvm /var/lib/libvirt
sudo chmod -R 775 /var/lib/libvirt
ls -lah /var/run/libvirt/libvirt-sock

# Restart libvirtd service
sudo systemctl restart libvirtd
```

## Step 4: Install and Configure Docker-Virt-Manager
```bash
# Clone Docker-Virt-Manager repository
cd /opt
sudo git clone https://github.com/m-bers/docker-virt-manager.git
cd docker-virt-manager

# Create ISO directory
sudo mkdir -p /opt/iso
sudo chmod 777 /opt/iso
```

> ⚠️ **Important:** Make sure to have `privileged: true` in `docker-compose.yml` file for KVM support.


Follow [docker-compose.yml](docker-virt-manager/docker-compose.yml) and modify as necessary

```bash
# Build and deploy Docker-Virt-Manager
docker build -t docker-virt-manager . && docker compose up -d

# Only run this cmd if you need to rebuild and deploy updated Docker-Virt-Manager docker-compose.yml file
docker compose up --build -d
```

- Upload/Download `Windows 10 21H2` ISO into `/opt/iso` directory.
- Access Docker-Virt-Manager Web UI `http://<Ubuntu_22.04_IP_Addr>:8185`
- Create VM name `cuckoo1` with `Windows 10 21H2` ISO and start the VM.

Personally I used WinSCP (must have OpenSSH Server running in Ubuntu) to upload the ISO file. If you don't have the Windows 10 ISO, you may choose to git clone [MediaCreationTool.bat](https://github.com/AveYo/MediaCreationTool.bat) and create the ISO.

## Step 5: Install CAPEv2
```bash
# Clone CAPEv2 repository
cd /opt
sudo git clone https://github.com/kevoreilly/CAPEv2.git
cd CAPEv2
```

Follow [cape2.sh](CAPEv2/installer/cape2.sh) and modify as necessary. 
> ⚠️ **Important:** Check OS Network Interface, do not blindly follow!
```plaintext
# Configuration
NETWORK_IFACE=virbr0
IFACE_IP="192.168.122.1"
PASSWD="<cape_user_password>"
USER="cape"
```

```bash
# Run CAPEv2 installer
sudo ./cape2.sh base cape | tee cape.log
```

## Step 6: Install Poetry and Dependencies
```bash
# Install Poetry
curl -sSL https://install.python-poetry.org | python3 -

# Install dependencies
cd /opt/CAPEv2
poetry install
poetry env list
poetry run python3 cuckoo.py
poetry run pip install -r extra/optional_dependencies.txt
poetry run pip install chepy certvalidator asn1crypto mscerts
poetry run extra/libvirt_installer.sh
```

## Step 7: Configure & Verify CAPEv2 Services
> ⚠️ **Important: Restart and Check** to make sure all services are running without missing dependencies or error in `journalctl` logs.

Follow [cuckoo.conf](CAPEv2/conf/cuckoo.conf) and modify as necessary

```bash
# Edit cuckoo configuration file
sudo nano /opt/CAPEv2/conf/cuckoo.conf
```

```plaintext
# Change the following values
[resultserver]
...
ip = 192.168.122.1 # Change to virbr0 IP
```

```bash
# Start CAPEv2
poetry run python3 cuckoo.py

# Restart CAPEv2 services
sudo systemctl restart cape.service
sudo systemctl restart cape-processor.service
sudo systemctl restart cape-web.service
sudo systemctl restart cape-rooter.service

# Check logs for CAPEv2 services
sudo journalctl -u cape.service | tail -n 20
sudo journalctl -u cape-processor.service | tail -n 20
sudo journalctl -u cape-web.service | tail -n 20
sudo journalctl -u cape-rooter.service | tail -n 20
```

That's it for Part 1. Refer to the following links for Part 2 and Part 3.

[[Part2] Installing and Configuring CAPEv2 on Ubuntu 22.04](https://medium.com/@rizqisetyokus/building-capev2-automated-malware-analysis-sandbox-part-2-0c47e4b5cbcd)

[[Part3] Installing and Configuring CAPEv2 on Ubuntu 22.04](https://medium.com/@rizqisetyokus/building-capev2-automated-malware-analysis-sandbox-part-3-d5535a0ab6f6)