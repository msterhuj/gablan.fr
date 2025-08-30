+++
date = '2025-08-31T00:01:05+02:00'
draft = false
title = 'Import CloudInit images into Proxmox'
+++

## Introduction

Proxmox VE is a powerful open-source virtualization platform that allows you to manage virtual machines and containers with ease. One of its great features is the ability to use cloud-init images, which are pre-configured virtual machine images that can be easily deployed and customized using cloud-init.

In this guide, I’ll show you how to import a cloud image into Proxmox VE and create a virtual machine from it. For this example, we’ll use an Ubuntu 24.04 cloud image, but the process is similar for other distributions as long as the image supports cloud-init.

All steps to create this template are done via the CLI using `qm` commands, but you can also achieve this through the web interface.

## Step-by-Step Guide

1. **Download the Cloud Image**: First, download the cloud image you want to use on your Proxmox server. You can find official cloud images for various distributions on their respective websites. For Ubuntu, check [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/).

    ```bash
    wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
    ```

2. **Create a New VM**: Use the `qm create` command to create a new virtual machine. Specify the VM ID, name, memory size, and network configuration.

    ```bash
    qm create 900 --name ubuntu2404 --memory 2048 --net0 virtio,bridge=vmbr0
    ```

3. **Import the Cloud Image**: Use the `qm importdisk` command to import the downloaded cloud image into the newly created VM. Specify the VM ID, image path, storage location, and format.

    > **Tip**: I recommend not expanding the disk of the template itself. It’s better to resize the disk after cloning. This avoids wasting time when creating full clones.

    ```bash
    qm importdisk 900 noble-server-cloudimg-amd64.img local -format qcow2
    ```

4. **Attach the Disk to the VM**: Attach the imported disk to the VM with `qm set`.

    ```bash
    qm set 900 --scsihw virtio-scsi-pci --scsi0 local:vm-900-disk-0
    ```

5. **Create a Cloud-Init Drive**: Configure cloud-init for the VM. This will create and attach a cloud-init drive, set the boot order, and configure console access.

    ```bash
    qm set 900 --ide2 local:cloudinit --boot c --bootdisk scsi0 --serial0 socket --vga serial0
    ```

6. **Configure Cloud-Init Settings**: Set network configuration, SSH keys, and user password.

    a. **Set IP Configuration** (DHCP recommended for templates):

    ```bash
    qm set 900 --ipconfig0 ip=dhcp
    # Or static (not recommended for templates, can cause conflicts if cloned):
    # qm set 900 --ipconfig0 ip=192.168.255.10/24,gw=192.168.255.254
    ```

    b. **Add SSH Key**:

    ```bash
    qm set 900 --sshkey ~/.ssh/id_rsa.pub
    ```

    c. **Set User Password**:

    I personally prefer using an SSH key for authentication, but if you don’t set a password here, you won’t be able to log in via the console.

    ```bash
    qm set 900 --cipassword YourSecurePassword
    ```

7. **Convert to Template**:

    Now that everything is set up we can convert the VM into a template to start using it.

    ```bash
    qm template 900
    ```

8. **Clone and Test the Template**: Create a clone (linked or full) from the template, resize the disk if necessary, and configure the network.

    ```bash
    qm clone 900 901 --name ubuntu2404-clone1
    qm resize 901 scsi0 +30G
    qm set 901 --ipconfig0 ip=192.168.255.10/24,gw=192.168.255.254
    qm start 901
    ```

9. **Access the VM**: Once started, access your VM via SSH.

    ```bash
    ssh ubuntu@192.168.255.10
    ```

## Conclusion

It’s a simple and efficient workflow — I no longer install VMs manually. I use cloud-init for all my quick tests and development setups. For more advanced use cases, you can even write full cloud-init configs and inject them via Proxmox snippets. I’ll cover that in a future article.
