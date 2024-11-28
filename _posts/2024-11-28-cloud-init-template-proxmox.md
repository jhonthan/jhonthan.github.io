---
layout: post
title: Cloud-Init Templates in Proxmox
# subtitle: Excerpt from Soulshaping by Jeff Brown
cover-img: /assets/img/cloud_init.png
thumbnail-img: /assets/img/cloud_init_thumb.png
share-img: /assets/img/cloud_init.png
tags: [proxmox, homelab, cloud-init]
author: Jonathan Luiz
---

The cloud-init (cloud image initialization) is an easy and efficient way to create a virtual machine Linux in Proxmox VE.

## Cloud initialization parameters in PVE
The following parameters can be viewed in the cloud initialization unit created in the Proxmox VE interface:

- User (a user is created)
- Password (a password is set)
- DNS domain (DNS domain substitution for default PVE host)
- DNS servers (DNS server replacement for standard PVE host)
- SSH public key (your own personal SSH public key)
- Update packages yes (the image is completely updated when it starts)
- IP Config (net0) (IP configuration for the first network device, static or DHCP)

## Package dependency
To use cloud-init in Proxmox, you must have the dependency package libguestfs-tools. This tool is used to access the guest virtual machine (VM) disk images, perform backups, clone VMs, build VMs, format disks, resize disks, and much more. 
```jsx
apt update -y && apt install libguestfs-tools -y
```

## Prepare the cloud-init as a template
A good way to use the images in Proxmox is to have a template image, because this will be a base image with all the staff necessary created previously. 
In this example we will create a template with ID 9501 based in the Ubuntu 22.04 (Jammy).

Access the SSH in the Proxmox and run the command:
```jsx
cd /var/lib/vz/template/iso/

wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img &&
virt-customize -a /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img --install qemu-guest-agent &&
virt-customize -a /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img --root-password password:ubuntu123 &&
virt-customize -a /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img --run-command "echo -n > /etc/machine-id"

qm create 9501 --name "ubuntu-2204-ci" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 &&
qm importdisk 9501 /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img local-lvm &&
qm set 9501 --scsihw virtio-scsi-single --scsi0 local-lvm:vm-9501-disk-0,cache=writeback,discard=on,ssd=1 &&
qm set 9501 --scsi1 local-lvm:20,cache=writeback,discard=on,ssd=1 &&
qm set 9501 --boot c --bootdisk scsi0 &&
qm set 9501 --scsi2 local-lvm:cloudinit &&
qm set 9501 --agent enabled=1 &&
qm set 9501 --serial0 socket &&
qm set 9501 --vga serial0 &&
qm set 9501 --cpu cputype=host &&
qm set 9501 --ostype l26 &&
qm set 9501 --ciuser ubuntu &&
qm template 9501
rm jammy-server-cloudimg-amd64.img
```

**Explaining the parameters:**

Download the Ubuntu 22.04 (Jammy).

```jsx
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

Install the **qemu-guest-agent**, used by Proxmox to have the capability to send commanders to the virtual machines and collect some information.

```jsx
virt-customize -a /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img --install qemu-guest-agent
```

Define the user password and the machine-id

```jsx
virt-customize -a /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img --root-password password:ubuntu123 &&
virt-customize -a /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img --run-command "echo -n > /etc/machine-id"
```

Create a new VM with VirtIO SCSI controller, define the name of the template as a "ubuntu-2204-ci" and 2GB of memory and 2 cores of CPU.

```jsx
qm create 9501 --name "ubuntu-2204-ci" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
```

Import the downloaded disk to the local-lvm storage, attaching it as a SCSI drive

```jsx
qm importdisk 9501 /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img local-lvm
```

Define the SCSI disk with 20GB as a boot device in the virtual machine

```jsx
qm set 9501 --scsihw virtio-scsi-single --scsi0 local-lvm:vm-9501-disk-0,cache=writeback,discard=on,ssd=1
qm set 9501 --scsi1 local-lvm:20,cache=writeback,discard=on,ssd=1
qm set 9501 --boot c --bootdisk scsi0
qm set 9501 --scsi2 local-lvm:cloudinit
```

Enable the qemu-guest-agent in the virtual machine setting to 1, to disable use 0.

```jsx
qm set 9501 --agent enabled=1
```

For many Cloud-Init images, it is required to configure a serial console and use it as a display. If the configuration doesn’t work for a given image, however, switch back to the default display instead.

```jsx
qm set 9501 --serial0 socket &&
qm set 9501 --vga serial0
```

Define the user inside the template:

```jsx
qm set 9501 --ciuser ubuntu
```

In the last step, it is helpful to convert the VM into a template. From this template, you can then quickly create linked clones. The deployment from VM templates is much faster than creating a full clone (copy).

```jsx
qm template 9501
```

Remove the ISO image in the final of the procedure to convert to a template.

```jsx
rm jammy-server-cloudimg-amd64.img
```

In the final, you have a template image based on the Ubuntu 22.04 (Jammy) that can be create new virtual machines, cloning this template ;)
