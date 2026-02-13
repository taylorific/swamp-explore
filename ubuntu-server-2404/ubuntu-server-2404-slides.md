---
layout: section
---

# Create virtual machine

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

```bash
$ virsh nodeinfo
CPU model:           x86_64
CPU(s):              12
CPU frequency:       3916 MHz
CPU socket(s):       1
Core(s) per socket:  6
Thread(s) per core:  2
NUMA cell(s):        1
Memory size:         65627760 KiB
```

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           6.3G  2.4M  6.3G   1% /run
/dev/nvme2n1p2  1.8T   14G  1.7T   1% /
tmpfs            32G     0   32G   0% /dev/shm
tmpfs           5.0M   12K  5.0M   1% /run/lock
efivarfs        192K   70K  118K  38% /sys/firmware/efi/efivars
tmpfs            32G     0   32G   0% /run/qemu
/dev/nvme2n1p1  1.1G  6.2M  1.1G   1% /boot/efi
tmpfs           6.3G  112K  6.3G   1% /run/user/1000
/dev/sda1        58G   53G  5.0G  92% /media/automat/Ventoy
```

---
hideInToc: true
---

# Download cloud image template and resize

```bash
sudo apt-get update
sudo apt-get install curl
```

```bash
mkdir swamp
cd swamp
```

```bash
curl -LO https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

```bash
$ qemu-img info noble-server-cloudimg-amd64.img
$ sudo qemu-img convert \
    -f qcow2 -O qcow2 \
    noble-server-cloudimg-amd64.img \
    /var/lib/libvirt/images/swamp.qcow2
$ sudo qemu-img resize -f qcow2 \
    /var/lib/libvirt/images/swamp.qcow2 \
    64G
````

---
hideInToc: true
---

# Define login parameters for cloud-init ISO (1 of 2)

```bash
# Required for NoCloud module to function, uniquely identifies instance
cat >meta-data <<EOF
instance-id: swamp
local-hostname: swamp
EOF
```

---
hideInToc: true
---

# Define login parameters for cloud-init ISO (2 of 2)

```bash
cat >user-data <<EOF
#cloud-config
hostname: swamp
users:
  - name: autobot
    uid: 63112
    primary_group: users
    groups: users
    shell: /bin/bash
    plain_text_passwd: superseekret
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
chpasswd: { expire: False }
ssh_pwauth: True
package_update: False
package_upgrade: false
packages: ['qemu-guest-agent']
growpart: { mode: auto, devices: ['/'] }
power_state: { mode: reboot }
EOF
```

---
hideInToc: true
---

```bash
sudo apt-get update
sudo apt-get install genisoimage
```

```bash
$ genisoimage \
    -input-charset utf-8 \
    -output swamp-cloud-init.img \
    -volid cidata -rational-rock -joliet \
    user-data meta-data

sudo cp swamp-cloud-init.img \
  /var/lib/libvirt/boot/swamp-cloud-init.iso
```

---
hideInToc: true
---

```bash
virt-install \
  --connect qemu:///system \
  --name swamp \
  --boot uefi \
  --memory 4096 \
  --vcpus 4 \
  --os-variant ubuntu24.04 \
  --disk /var/lib/libvirt/images/swamp.qcow2,bus=virtio \
  --disk /var/lib/libvirt/boot/swamp-cloud-init.iso,device=cdrom \
  --network network=host-network,model=virtio \
  --graphics none \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
hideInToc: true
---

# Virtual machine console

```bash
# Command line console
virsh console swamp
```

---
hideInToc: true
---

# Disable cloud-init and remove cloud-init ISO

In the guest:

```bash
# login with autobot user
$ cloud-init status --wait
status: done

# Disable cloud-init
$ sudo touch /etc/cloud/cloud-init.disabled
$ cloud-init status
status: disabled

$ sudo shutdown -h now
```

On the host
```bash
$ virsh domblklist swamp
$ virsh change-media swamp sda --eject
$ sudo rm /var/lib/libvirt/boot/swamp-cloud-init.iso
```

---
hideInToc: true
---

# Snapshots

```bash
$ virsh snapshot-create-as --domain swamp --name clean --description "Initial install"
$ virsh snapshot-list swamp
$ virsh snapshot-revert swamp clean
$ virsh snapshot-delete swamp clean
```

# Cleanup
```bash
$ virsh shutdown swamp
$ virsh undefine swamp --nvram --remove-all-storage
```

# Get IP of virtual machine
```bash
$ virsh domifaddr swamp --source agent
```
