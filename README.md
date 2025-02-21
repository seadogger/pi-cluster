# HomeLab Raspberry Pi k3s Deployment Guide

## Architecture Overview

![Pi Cluster Architecture](images/Architecture.png)

## Prerequisites

### Hardware Requirements
- Raspberry Pi5 nodes (1 server node and 3 worker nodes)
- POE switch (recommended: Ubiquiti Dream @Machine SE)
  - Powers Raspberry Pis via POE HAT
  - Simplifies the wiring and setup, but not totally neccessary.  
  - **If you do not use POE, adjust the BoM (e.g. rack mounted solution will be different, likely)**
- Ethernet cables for hardwired connections
  - WiFi is disabled and not recommended for k3s clusters

### Network Setup
- DHCP static IP assignments for all Raspberry Pis
  - Configured on network switch for centralized management
  - Static IPs required for k3s cluster nodes
- DHCP service range configuration
  - Reserve IPs up to 192.168.1.239
  - Leaves space for MetalLB allocation above this range
  - NOTE - If you are using a different subnet, there is a lot of changes to apply throughout the deployment scripts.  
  `TODO: centralize the subnet in the ansible manifest config.yaml`
- WireGuard (optional)
  - Required only for remote access
  - Provides encrypted tunnels for services like OpenWebUI, PiHole when you are not on your network

### Software Requirements
- SSH enabled on all Raspberry Pis
- AWS account with Bedrock API tokens
- Working knowledge of:
  - Docker containers and orchestration
  - Basic AWS services
  - Git and GitHub CLI tools

### Development Environment
- VS Code (recommended) with:
  - Continue Extension
  - SSH Remote Extension
- Alternative: SSH terminal access


# Bill of Materials

| Item | Quantity | Cost per Unit | Total Cost |
|------|----------|--------------|------------|
| [Raspberry Pi 5 8GB](https://www.digikey.com/en/products/detail/raspberry-pi/SC1112/21658257?s=N4IgjCBcpgrAnADiqAxlAZgQwDYGcBTAGhAHsoBtEAJngBYwwB2EAXRIAcAXKEAZS4AnAJYA7AOYgAviQC0dFCHSRs%2BYmUrgAzAAYdW5CUbwmYeGykyamwVjwcARgUGCAngAIOw2BaA) | 4 | $80 | $320 |
| Raspberry Pi Rack | 1 | $53 | $53 |
| GPIO Header with shorter standoff | 1 | $10 | $10 |
| Raspberry Pi 5 POE HAT with PCIe | 4 | $37 | $148 |
| [Crucial P3 4TB NVMe SSD](https://www.newegg.com/crucial-4tb-p3/p/N82E16820156298?Item=9SIA12KJ9P1073) | 3 | $225 | $675 |
| [Crucial P3 500GB NVMe SSD](https://www.newegg.com/crucial-500gb-p3-nvme/p/N82E16820156295) | 1 | $38 | $38 |
| Nylon Standoff Kit | 1 | $13 | $13 |
| **Total Cost (Excludes POE Switch)** | **-** | **-** | **$1257** |

# Get Remote PC ready for Ansible Deployment 
Clone the Ansible tasks to your remote PC to that will manage the k3s cluster
   ```bash
   git clone https://github.com/seadogger/seadogger-homelab.git
   ```

# Raspberry Pi 5 Setup

## POE HAT and Drive install

Install POE HAT with NVMe PCIe adapter on all Raspberry Pi 5s

`TODO - Put pictures of install`

## Raspberry Pi Setup

This setup assumes a Raspberry Pi5 64 bit OS Lite (no desktop) will be setup in a k3s cluster with a single server node and 3 worker nodes with 4TB in each worker node. 

The approach taken to get to here included flashing the Raspberry Pi OS (64-bit, lite) to the storage devices using Raspberry Pi Imager onto a 64GB sdCard.  This was then installed into each Pi and transferred to the NVMe in the previous section.  This should be successful before proceeding into this step.

To make network discovery and integration easier, I edit the advanced configuration in Imager, and set the following options:

  - Set hostname: `node1.local` (set to `2` for node 2, `3` for node 3, etc.)
  - Enable SSH
  - Disable WiFi

After setting all those options, and the hostname is unique to each node (and matches what is in `hosts.ini`), I inserted the microSD cards into the respective Pis, or installed the NVMe SSDs into the correct slots, and booted the cluster.

![Hardware Build](images/Single-Node-Mounted-1.jpeg)
![Hardware Build](images/Single-Node-Mounted-2.jpeg)
![Hardware Build](images/Rack-Mounted-Pi5-Nodes.jpeg)

## SSH connection test

To test the SSH connection from the host or PC you intend to run Ansible from, connect to each server individually, and accepted the hostkey.  This must be done for all nodes so password is not requested by the nodes during the Ansible playbook run:

```
ssh-copy-id pi@node[X].local
ssh pi@192.168.1.[X]
ssh pi@node[X].local
```

This ensures Ansible will also be able to connect via SSH in the following steps. You can test Ansible's connection with:

```
ansible all -m ping
```

It should respond with a 'SUCCESS' message for each node.

## NVMe Boot and 4TB NVMe Drive Setup  

This guide will help you set up and use a **4TB NVMe drive** on a **Raspberry Pi 5**. This process involves partitioning, formatting, cloning partitions, updating `/etc/fstab`, and troubleshooting common issues.

`rpi-clone` seems to reset the partition table to MBR vs. GPT so we lose storage space as MBR is confined to 2TB.  These scripts are very specific the hardware stack choosen in the BoM for partition setup and reformats these partitions and you will lose any data on them.  **Use at your own risk!**

I am using a 64GB sdCard and transitioning the `/boot` and `/` mounts to a `4TB NVMe` using the `52Pi POE w/PCIe HAT`.  This will partition the NVMe just slightly larger than the partitions on the sdCard so the rsync transfers complete without error but we do not waste a ton of space.  

> **Note**: All the sector definitions are based on this size sdCard for gdisk.  If you are using a smaller or larger sdCard you will need to modify the partition tables settings in these scripts.  

### Prepare the 512GB NVMe Drive (Master Node)
```bash
   git clone https://github.com/seadogger/seadogger-homelab.git
   cd seadogger-homelab/useful-scripts
   sudo ./partition_and_boot_NVMe_master_node.sh
```

### Prepare the 4TB NVMe Drive (Worker / Storage Node)
```bash
   git clone https://github.com/seadogger/seadogger-homelab.git
   cd seadogger-homelab/useful-scripts
   sudo ./partition_and_boot_NVMe_worker_node.sh
```

## Script Summary

1. Partition the NVMe drive using GPT.
2. Format partitions (`vfat` for EFI, `ext4` for root).
3. Clone partitions from the SD card using `rsync`.
4. Update `/etc/fstab`, `/boot/firmware/config.txt`, and `/boot/firmware/cmdline.txt`
5. Cloning the sdCard to the NVMe partition structure
6. Running disk checks on the NVMe `e2fsck` and `fsck.vfat`
7. Reload systemd daemon.
8. Change boot order, shutdown and reboot

> **Note**: Before you reboot after the above you need to setup for NVMe to boot first in the boot order
- Set the NVMe first in the boot order 
    ```bash
     sudo raspi-config
    ``` 
Under advanced options set the boot order to boot the NVMe first.  
> **Note**: When prompted to reboot **`decline`.  We will reboot in the next step.**

- Set the NVMe first in the boot order and tell the bootloader to detect PCIE
    ```bash
    sudo rpi-eeprom-config --edit
    ```

- Add and modify the following to set the boot order:
    ```
    PCIE_PROBE=1
    BOOT_ORDER=0xf416
    ```

- Shutdown:
   ```bash
   sudo shutdown now
   ```
- Pull the sdCard out and reboot

- Verify partitions are correctly mounted:
   ```bash
   lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
   ```
![Partition Info](images/Partition-Map-NVMe.png)
![Partition Info](images/NVMe-Performance-Compare.png1)

# Raspberry Pi Cluster Mangement with Ansible

## Usage

  1. Make sure [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) is installed on your remote PC (in my case it is a Macbook Air and is attached on the same subnet as my cluster).
  
  2. Copy the `example.hosts.ini` inventory file to `hosts.ini`. Make sure it has the `control_plane` and `node`(s) configured correctly.
  
  3. Copy the `example.config.yml` file to `config.yml`, and modify the variables to your setup.


## Cluster configuration, K3s, App deployment

Run the playbook:

```
ansible-playbook main.yml
```

   - Updates apt package cache
   - Configures cgroups are configured correctly in cmdline.txt.
   - Installs necessary packages
   - Enables and starts iscsid service
   - Configures PCIe settings exist in config.txt
   - Loads dm_crypt kernel module
   - Loads rbd kernel module
   - Appends dm_crypt and rbd to /etc/modules
   - Updates Raspberry Pi firmware to rpi-6.6.y
   - Setup/deploy k3s to the control_plane (e.g. server node)
   - Setup/deploy k3s to the worker node(s)

   `NOTE - if you get an error about cgroups you should perform a reboot and run the ansible script again` 

   - TODO - Need to deploy Ceph-Rook separately between these deployments
   - Deploy ArgoCD for GitOps

   - Deploy PODs/Apps thru ArgoCD  

      - MetalLB
      - AWS Bedrock
      - PiHole
      - Prometheus and Grafana
      - OpenWeb UI
      - Plex Media Server

> **Note**: Applications are deployed declaratively through ArgoCD, ensuring infrastructure-as-code best practices

## Upgrading the cluster

Run the upgrade playbook:

```
ansible-playbook upgrade.yml
```

## Benchmarking the cluster

See the README file within the `benchmarks` folder.  **Credit and Thanks to [Jeff Geerling](https://www.jeffgeerling.com)**

![Benchmark Results](images/IO-Benchmark-2.png)

#### SDCard Vs. NVMe
![Benchmark Results](images/NVMe-Performance-Compare.png)


## Shutting down the cluster

The safest way to shut down the cluster is to run the following command:

```
ansible all -m community.general.shutdown -b
```

You can reboot all the nodes with:

```
ansible all -m reboot -b
```

# Learning Outcomes

## Core Technologies
- [Kubernetes (k3s)](https://docs.k3s.io/architecture) architecture and deployment
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) for cluster management
- [Helm](https://helm.sh) package management
- [Longhorn](https://longhorn.io) distributed storage
- [Prometheus](https://prometheus.io) & [Grafana](https://grafana.com) monitoring stack 
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) for GitOps
- [Ansible](https://docs.ansible.com) for Infrastructure as Code

## Essential Reading
- [Docker Containers for Beginners](https://www.freecodecamp.org/news/a-beginner-friendly-introduction-to-containers-vms-and-docker-79a9e3e119b)
- [Network Chuck's Raspberry Pi Cluster Guide](https://youtu.be/Wjrdr0NU4Sk)
- [Jeff Geerling's Homelab Blog](https://www.jeffgeerling.com/blog)
- [Alex's Cloud Infrastructure Guide](https://alexstan.cloud/posts/homelab/homelab-setup/)
- [Deploying Plex on Kubernetes](https://www.debontonline.com/2021/01/part-14-deploy-plexserver-yaml-with.html)

## Author
The repository was forked from [Jeff Geerling](https://www.jeffgeerling.com)'s Pi-Cluster project and was modified by [seadogger]().