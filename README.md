# CAPEv2 with Docker-Virt-Manager on Ubuntu 22.04

This guide provides a step-by-step process for setting up [CAPEv2](https://github.com/kevoreilly/CAPEv2) with [Docker-Virt-Manager](https://github.com/m-bers/docker-virt-manager) on **Ubuntu 22.04**.

## Prerequisites
Ensure you are using `Ubuntu 22.04 Live Server`, which can be downloaded from:
[Ubuntu 22.04 Old Releases](https://old-releases.ubuntu.com/releases/22.04.1/).

> ⚠️ **Important:** Running all these commands as `cape` user is recommended. Best is to create `cape` user during OS installation.

- CPU Total Processor Cores: 4 
  - Number of Processors Recommended: 2
  - Number of Cores per Processor Recommended - 2
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

## Step 2: Install KVM and Virtualization Dependencies
```bash
sudo apt update
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
sudo mkdir -p /var/lib/libvirt/images
sudo chown -R libvirt-qemu:kvm /var/lib/libvirt
sudo chmod -R 775 /var/lib/libvirt
ls -lah /var/run/libvirt/libvirt-sock

# Restart libvirtd service
sudo systemctl restart libvirtd
```

## Step 4: Install and Configure Docker-Virt-Manager
```bash
cd /opt
sudo git clone https://github.com/m-bers/docker-virt-manager.git
cd docker-virt-manager

# Create ISO directory
sudo mkdir -p /opt/iso
sudo chmod 777 /opt/iso
```

Follow [docker-compose.yml](docker-virt-manager/docker-compose.yml) and modify as necessary

```bash
docker build -t docker-virt-manager . && docker compose up -d

# Rebuild and deploy updated docker-compose.yml Docker-Virt-Manager
docker compose up --build -d
```

- Download `Windows 10 21H2` ISO into `/opt/iso` directory.
- Access Docker-Virt-Manager Web UI `http://<Ubuntu_22.04_IP_Addr>:8185`
- Create VM name `cuckoo1` with `Windows 10 21H2` ISO and start the VM.

## Step 5: Install CAPEv2
```bash
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
[resultserver]
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