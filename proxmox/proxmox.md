zfs create -V 100G cachepool/cache
zfs create -V 100G cachepool/log
zfs list zp0data/zv_vm

zpool create -f -o ashift=12 my-zfs-pool raidz1 /dev/sdb /dev/sdc /dev/sdd cache /dev/sda5 log /dev/sda4


devices="sdc sdd"
for device in $devices; do
  echo $device
  wipefs -a -f /dev/${device}
  dd if=/dev/zero of=/dev/$device bs=1M count=1
  parted --script /dev/$device "mklabel gpt"  
  parted --script /dev/$device "mkpart primary 0% 25%"
  parted --script /dev/$device "mkpart primary 25% 85%"
done

zpool add rpool log mirror /dev/sdc1 /dev/sdd1 cache /dev/sdc2 /dev/sdd2


pvesm add cifs appc-pc --server 192.168.126.5 --share pve --username nmt --password 19871979

vmid="103"
vmname="r-vm-w10p1"
vmmem=4096
vmos="win10"
hddstorage=local-zfs
isostorage="appc-pc"

qm create \
  $vmid \
  --cdrom $isostorage:iso/en-us_windows_10_business_editions_version_22h2_updated_march_2023_x64_dvd_94a1087a.iso \
  --name $vmname \
  --vlan0 virtio=62:57:BC:A2:0E:18 \
  --virtio0 ${hddstorage}:vm-${vmid}-disk-0,cache=writeback,discard=on,iothread=1,size=32G \ 
  --bootdisk virtio0 \
  --ostype $vmos \
  --memory $vmmem \
  --onboot no \
  --sockets 1

qm clone <vmid> <newid>

--format <qcow2 | raw | vmdk>

## create VM
```
vmid="103"
vmname="tmp-alma-9"
vmmemory=1024
vmostype="Linux"
hddstor="local-zfs"
isostor="appc-pc"

qm create ${vmid}
--name $vmname
--boot order=virtio0;ide2;net0
--sockets 1
--cores 3
--ide2 appc-pc:iso/en-us_windows_10_business_editions_version_22h2_updated_march_2023_x64_dvd_94a1087a.iso,media=cdrom
--memory $vmmemory
--net0 virtio,bridge=vmbr0,firewall=1
--ostype $vmostype
--scsihw virtio-scsi-single
--virtio0 $hddstor:vm-${vmid}-disk-0,cache=writeback,discard=on,iothread=1,size=32G

machine: pc-i440fx-8.0
meta: creation-qemu=8.0.2,ctime=1695756298
smbios1: uuid=4b36f4f4-b155-4056-bc5a-f225a89bb63c
vmgenid: 3d36d5cb-ad37-45ec-a601-0529ad33a8ce
--autostart
```

## загрузить диск для шаблона ВМ

```bash
vmid=9008
#qcow2file=/mnt/appc-pc/pub/iso/astra/alse-vanilla-1.7.4uu1-cloud-base-mg12.0.1.qcow2
qcow2file=/mnt/appc-pc/pub/iso/alma/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
qcow2storage="local-zfs"
qcow2options=",cache=writeback"
qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0
```

```bash
sh user@your-proxmox-server

su - root
export IMAGES_PATH="/mnt/appc-pc/pub/iso/alma/" # defines the path where the images will be stored and change the path to it.
cd $IMAGE_PATH
wget https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
## Amazon Linux 2
## https://cdn.amazonlinux.com/os-images/latest/
# wget https://cdn.amazonlinux.com/os-images/2.0.20230727.0/kvm/amzn2-kvm-2.0.20230727.0-x86_64.xfs.gpt.qcow2
# wget https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2
# wget https://download.fedoraproject.org/pub/fedora/linux/releases/38/Cloud/x86_64/images/Fedora-Cloud-Base-38-1.6.x86_64.qcow2

# wget https://yum.oracle.com/templates/OracleLinux/OL9/u2/x86_64/OL9U2_x86_64-kvm-b197.qcow
## Converting image to qcow2 format
# qemu-img convert -O qcow2 -o compat=0.10 OL9U2_x86_64-kvm-b197.qcow OL9U2_x86_64-kvm-b197.qcow2
# rm OL9U2_x86_64-kvm-b197.qcow
# wget https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2
# wget https://cloud-images.ubuntu.com/lunar/current/lunar-server-cloudimg-amd64.img 

#Create VM
export QEMU_CPU_MODEL="host" # Specifies the CPU model to be used for the VM according your environment and the desired CPU capabilities.
export VM_CPU_SOCKETS=1
export VM_CPU_CORES=2
export VM_MEMORY=2048
#export VM_RESOURCE_POOL="CustomResourcePool" # Assigns the VM to a specific resource pool for management.
#Define the Cloud-init configuration. The specified user will be created, and its public key will be defined as an authorized key, enabling remote access using the user's private key.
export CLOUD_INIT_USER="appc" # Specifies the username to be created using Cloud-init.
export CLOUD_INIT_SSHKEY="/root/.ssh/appc.pub" # Provides the path to the SSH public key for the user.
export CLOUD_INIT_IP="dhcp"

export TEMPLATE_ID=9008
export TEMPLATE_NAME="tmp-alma-9"
export VM_DISK_IMAGE="/mnt/appc-pc/pub/iso/alma/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2"
export TEMPLATE_STORAGE="local-zfs"
# Create VM. Change the cpu model or clone from tmp
#qm create ${TEMPLATE_ID} --name ${TEMPLATE_NAME} --cpu ${QEMU_CPU_MODEL} --sockets ${VM_CPU_SOCKETS} --cores ${VM_CPU_CORES} --memory ${VM_MEMORY} --numa 1 --net0 virtio,bridge=vmbr0 --ostype l26 --agent 1 --pool ${VM_RESOURCE_POOL} --scsihw virtio-scsi-single
qm create ${TEMPLATE_ID} --name ${TEMPLATE_NAME} --cpu ${QEMU_CPU_MODEL} --sockets ${VM_CPU_SOCKETS} --cores ${VM_CPU_CORES} --memory ${VM_MEMORY} --numa 1 --net0 virtio,bridge=vmbr0 --ostype l26 --agent 1 --scsihw virtio-scsi-single
qm set ${TEMPLATE_ID} --virtio0 $TEMPLATE_STORAGE:0,import-from=${VM_DISK_IMAGE}
qm set ${TEMPLATE_ID} --ide2 $TEMPLATE_STORAGE:cloudinit --boot order=virtio0
qm set ${TEMPLATE_ID} --ipconfig0 ip=${CLOUD_INIT_IP}
qm set ${TEMPLATE_ID} --ciupgrade 1 --ciuser ${CLOUD_INIT_USER} --sshkeys ${CLOUD_INIT_SSHKEY}
qm cloudinit update ${TEMPLATE_ID}

#OR clone from tmp-9000-lin
qm clone 9000 ${TEMPLATE_ID} --name ${TEMPLATE_NAME} --full
#qm set ${TEMPLATE_ID} --scsi0 local-zfs:0,import-from=${VM_DISK_IMAGE}
#qm set ${TEMPLATE_ID} --boot order=scsi0
qm set ${TEMPLATE_ID} --virtio0 local-zfs:0,import-from=${VM_DISK_IMAGE}
qm set ${TEMPLATE_ID} --boot order=virtio0

#convert to template
qm template ${TEMPLATE_ID}

export VM_ID=$(pvesh get /cluster/nextid)
export TEMPLATE_ID=9008
export VM_NAME="r-vm-vpn2"
qm clone ${TEMPLATE_ID} ${VM_ID} --name ${VM_NAME} --full
qm start ${VM_ID}

```

vyos 1.4
```bash
#Create VM
export QEMU_CPU_MODEL="host" # Specifies the CPU model to be used for the VM according your environment and the desired CPU capabilities.
export VM_CPU_SOCKETS=1
export VM_CPU_CORES=2
export VM_MEMORY=1024
#export VM_RESOURCE_POOL="CustomResourcePool" # Assigns the VM to a specific resource pool for management.
#Define the Cloud-init configuration. The specified user will be created, and its public key will be defined as an authorized key, enabling remote access using the user's private key.
export CLOUD_INIT_USER="appc" # Specifies the username to be created using Cloud-init.
export CLOUD_INIT_SSHKEY="/root/.ssh/appc.pub" # Provides the path to the SSH public key for the user.
export CLOUD_INIT_IP="dhcp"
export TEMPLATE_ID=9010
export TEMPLATE_NAME="tmp-vyos-1.4"
export VM_DISK_IMAGE="/mnt/appc-pc/pub/iso/vyos/vyos-1.4-rolling-202401230212-cloud-init-4G-qemu.qcow2"
export TEMPLATE_STORAGE="local-zfs"

#clone from tmp-9000-lin
qm clone 9000 ${TEMPLATE_ID} --name ${TEMPLATE_NAME} --full
qm set ${TEMPLATE_ID} --virtio0 $TEMPLATE_STORAGE:0,import-from=${VM_DISK_IMAGE}
qm set ${TEMPLATE_ID} --boot order=virtio0

#convert to template
qm template ${TEMPLATE_ID}

#create new vm
export VM_ID=$(pvesh get /cluster/nextid)
export TEMPLATE_ID=9010
export VM_NAME="r-vm-vyos14"
qm clone ${TEMPLATE_ID} ${VM_ID} --name ${VM_NAME} --full
qm start ${VM_ID}

```

vyos 1.3
```bash
#Create VM
export QEMU_CPU_MODEL="host" # Specifies the CPU model to be used for the VM according your environment and the desired CPU capabilities.
export VM_CPU_SOCKETS=1
export VM_CPU_CORES=2
export VM_MEMORY=512
#export VM_RESOURCE_POOL="CustomResourcePool" # Assigns the VM to a specific resource pool for management.
#Define the Cloud-init configuration. The specified user will be created, and its public key will be defined as an authorized key, enabling remote access using the user's private key.
export CLOUD_INIT_USER="appc" # Specifies the username to be created using Cloud-init.
export CLOUD_INIT_SSHKEY="/root/.ssh/appc.pub" # Provides the path to the SSH public key for the user.
export CLOUD_INIT_IP="dhcp"
export TEMPLATE_ID=9011
export TEMPLATE_NAME="tmp-vyos-1.3"
export VM_DISK_IMAGE="/mnt/appc-pc/pub/iso/vyos/vyos-1.3-rolling-202401222052-cloud-init-4G-qemu.qcow2"
export TEMPLATE_STORAGE="local-zfs"

#clone from tmp-9000-lin
qm clone 9000 ${TEMPLATE_ID} --name ${TEMPLATE_NAME} --full
qm set ${TEMPLATE_ID} --memory ${VM_MEMORY} --virtio0 $TEMPLATE_STORAGE:0,import-from=${VM_DISK_IMAGE} --boot order=virtio0
#qm set ${TEMPLATE_ID} 

#convert to template
qm template ${TEMPLATE_ID}

#create new vm
export VM_ID=$(pvesh get /cluster/nextid)
export TEMPLATE_ID=9011
export VM_NAME="r-vm-vyos13"
qm clone ${TEMPLATE_ID} ${VM_ID} --name ${VM_NAME} --full
qm start ${VM_ID}

```

CHR 7 Mikrotik RouterOS 7.13.2
```bash
#Create VM
#export VM_RESOURCE_POOL="CustomResourcePool" # Assigns the VM to a specific resource pool for management.
#Define the Cloud-init configuration. The specified user will be created, and its public key will be defined as an authorized key, enabling remote access using the user's private key.
export CLOUD_INIT_USER="appc" # Specifies the username to be created using Cloud-init.
export CLOUD_INIT_SSHKEY="/root/.ssh/appc.pub" # Provides the path to the SSH public key for the user.
export CLOUD_INIT_IP="dhcp"

export VM_DISK_IMAGE="/mnt/appc-pc/pub/iso/redos/redos.qcow2"
export VM_CPU_SOCKETS=1
export VM_CPU_CORES=4
export VM_MEMORY=4096
export TEMPLATE_ID=9009
export TEMPLATE_NAME="tmp-redos73"
export TEMPLATE_STORAGE="local-zfs"

#clone from tmp-9000-lin
qm destroy ${TEMPLATE_ID}
qm clone 9000 ${TEMPLATE_ID} --name ${TEMPLATE_NAME} --full
qm set ${TEMPLATE_ID} --memory ${VM_MEMORY} --virtio0 $TEMPLATE_STORAGE:0,import-from=${VM_DISK_IMAGE} --boot order=virtio0
#qm set ${TEMPLATE_ID} 

#convert to template
qm template ${TEMPLATE_ID}

#create new vm
export VM_ID=$(pvesh get /cluster/nextid)
export TEMPLATE_ID=9012
export VM_NAME="r-vm-redos2"
qm clone ${TEMPLATE_ID} ${VM_ID} --name ${VM_NAME} --full
qm start ${VM_ID}


```



















devices="sdc sdd"
for device in $devices; do
  echo $device
  wipefs -a -f /dev/${device}
  dd if=/dev/zero of=/dev/$device bs=1M count=1
  parted --script /dev/$device "mklabel gpt"  
  parted --script /dev/$device "mkpart primary 0% 25%"
  parted --script /dev/$device "mkpart primary 25% 85%"
done

zpool add rpool log mirror /dev/sdc1 /dev/sdd1 cache /dev/sdc2 /dev/sdd2

qpvesm add cifs zz-backup --server 192.168.126.5 --share backup --subdir /r-pve1 --username nmt --password 19871979
qpvesm add cifs zz-images --server 192.168.126.5 --share pub --subdir /iso --username nmt --password 19871979
# --smbversion 3

vmid="103"
vmname="r-vm-w10p1"
vmmem=4096
vmos="win10"
hddstorage=local-zfs
isostorage="appc-pc"

qm create \
  $vmid \
  --cdrom $isostorage:iso/en-us_windows_10_business_editions_version_22h2_updated_march_2023_x64_dvd_94a1087a.iso \
  --name $vmname \
  --vlan0 virtio=62:57:BC:A2:0E:18 \
  --virtio0 ${hddstorage}:vm-${vmid}-disk-0,cache=writeback,discard=on,iothread=1,size=32G \ 
  --bootdisk virtio0 \
  --ostype $vmos \
  --memory $vmmem \
  --onboot no \
  --sockets 1

qm clone <vmid> <newid>

--format <qcow2 | raw | vmdk>

## create tmp

```shell
vmid="9000"
vmname="tmp"
vmmemory="512"
vmbridge="vmbr0"
vmostype="l26"
#qm destroy $vmid
qm create ${vmid} \
--name $vmname \
--template 1 \
--sockets 1 \
--cores 2 \
--memory 512 \
--scsihw virtio-scsi-pci \
--net0 virtio,bridge=$vmbridge,firewall=1

#convert to template
qm template $vmid





```

## create VM
```shell
vmid="1900"
vmname="tmp"
vmmemory=512
vmostype="win10"
hddstor="local-zfs"
isostor="appc-pc"

qm create ${vmid}
--name $vmname
--boot order=virtio0;ide2;net0
--sockets 1
--cores 3
--ide2 appc-pc:iso/en-us_windows_10_business_editions_version_22h2_updated_march_2023_x64_dvd_94a1087a.iso,media=cdrom
--memory $vmmemory
--net0 virtio,bridge=vmbr0,firewall=1
--ostype $vmostype
--scsihw virtio-scsi-single
--virtio0 $hddstor:vm-${vmid}-disk-0,cache=writeback,discard=on,iothread=1,size=32G

machine: pc-i440fx-8.0
meta: creation-qemu=8.0.2,ctime=1695756298
smbios1: uuid=4b36f4f4-b155-4056-bc5a-f225a89bb63c
vmgenid: 3d36d5cb-ad37-45ec-a601-0529ad33a8ce
--autostart
```

## загрузить диск для шаблона ВМ

```shell
vmid=113
qcow2file=/mnt/appc-pc/pub/iso/astra/alse-vanilla-1.7.4uu1-cloud-base-mg12.0.1.qcow2
qcow2storage="rpool"
qcow2options=",cache=writeback"
qm importdisk $vmid $qcow2file rpool; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0

--bios ovmf

#alt
qcow2storage="local-zfs"; qcow2options=",cache=writeback"
cipass='$6$bzSRia4pRwcBEZyY$eVroEqcDq8T8nglHMbhHs1bbsvfNWjR.5.qSbLo44O9/YWRMzT4sDkVyiMOMwOUVN.JXq39zos35dVHgVvYOI1'; ciuser='appc'; cipubkey='/var/tmp/appc.pub'

vmid="9004"
vmname="tmp-alse-1.7.4"
qcow2file=/mnt/appc-pc/pub/iso/ubuntu/22.04-jammy-server-cloudimg-amd64.img

qm create $vmid \
  --name $vmname \
  --template 1 \
  --sockets 1 \
  --cores 2 \
  --memory 2048 \
  --machine q35 \
  --bios seabios \
  --net0 virtio,bridge=vmbr0 \
  --boot c \
  --bootdisk virtio0 \
  --scsihw virtio-scsi-pci \
  --ostype l26 \
  --vga qxl,memory=16 \
  --ide2 $qcow2storage:cloudinit \
  --sshkeys $cipubkey --citype nocloud --ciuser $ciuser --cipass $cipass --ciupgrade 0 \
  --virtio0 $qcow2storage:0${qcow2options},import-from=$qcow2file


sleep 10
mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp
echo "" >> /mnt/tmp/etc/issue
echo "\4" >> /mnt/tmp/etc/issue
echo "\6" >> /mnt/tmp/etc/issue
echo "" >> /mnt/tmp/etc/issue
umount /mnt/tmp
sleep 10
qm resize $vmid virtio0 +7G

vmid="9000"; vmname="tmp"
qm destroy $vmid
qm create $vmid --name $vmname --template 1 --sockets 1 --cores 2 --memory 2048 --machine q35 --bios seabios --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci --ostype l26 --ide2 $qcow2storage:cloudinit --vga qxl,memory=16

qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9009"; vmname="tmp-ros7"; qcow2file=/mnt/appc-pc/pub/iso/mikrotik/chr-7.12.img
qm create $vmid --name $vmname --template 1 --sockets 1 --cores 2 --memory 256 --machine q35 --bios seabios --net0 virtio,bridge=vmbr0 --boot c --bootdisk virtio0 --scsihw virtio-scsi-pci --ostype l26 --ide2 $qcow2storage:cloudinit --vga qxl,memory=16 --virtio0 $qcow2storage:0${qcow2options},import-from=$qcow2file
sleep 10

vmid="9001"; vmname="tmp-alt-p10"; qcow2file=/mnt/appc-pc/pub/iso/alt/alt-p10-cloud-x86_64.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9002"; vmname="tmp-alt-p10-srv"; qcow2file=/mnt/appc-pc/pub/iso/alt/alt-server-p10-cloud-x86_64.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9003"; vmname="tmp-alt-p10-ws"; qcow2file=/mnt/appc-pc/pub/iso/alt/alt-p10-workstation-cloud-x86_64.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9004"; vmname=""; qcow2file=
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9005"; vmname="tmp-deb11"; qcow2file=/mnt/appc-pc/pub/iso/debian/debian-11-generic-amd64.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9006"; vmname="tmp-deb12"; qcow2file=/mnt/appc-pc/pub/iso/debian/debian-12-generic-amd64.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9007"; vmname="tmp-u2204"; qcow2file=/mnt/appc-pc/pub/iso/ubuntu/22.04-jammy-server-cloudimg-amd64.img
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9012"; vmname="tmp-redos73"; qcow2file=/mnt/appc-pc/pub/iso/redos/redos-7.3.4.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; qm template $vmid

#qcow2storage="local-zfs"; qcow2options=",cache=writeback"; 
#cipass='$6$bzSRia4pRwcBEZyY$eVroEqcDq8T8nglHMbhHs1bbsvfNWjR.5.qSbLo44O9/YWRMzT4sDkVyiMOMwOUVN.JXq39zos35dVHgVvYOI1'; ciuser='appc'; cipubkey='/var/tmp/appc.pub'
vmid="9013"; vmname="tmp-alse-1.7.4uu1-base"; qcow2file=/mnt/appc-pc/pub/iso/astra/alse-vanilla-1.7.4uu1-cloud-base-mg12.0.1.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9014"; vmname="tmp-alse-1.7.4uu1-adv"; qcow2file=/mnt/appc-pc/pub/iso/astra/alse-vanilla-1.7.4uu1-cloud-adv-mg12.0.1.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9015"; vmname="tmp-alse-1.7.4uu1-max"; qcow2file=/mnt/appc-pc/pub/iso/astra/alse-vanilla-1.7.4uu1-cloud-max-mg12.0.1.qcow2
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9016"; vmname="tmp-alse-1.7.4uu1-base-gui"; qcow2file=/mnt/appc-pc/pub/iso/astra/alse-vanilla-gui-1.7.4uu1-qemu-base-mg11.3.0.qcow2; qcow2storage="local-zfs"; qcow2options=",cache=writeback"
qm destroy $vmid; qm clone 9000 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; sleep 10; mount /dev/zvol/rpool/data/vm-${vmid}-disk-0-part1 /mnt/tmp; echo "" >> /mnt/tmp/etc/issue; echo "\4" >> /mnt/tmp/etc/issue; echo "\6" >> /mnt/tmp/etc/issue; echo "" >> /mnt/tmp/etc/issue; chroot /mnt/tmp apt update; chroot /mnt/tmp apt install -y python3-apt aptitude astra-update mc cloud-init spice-vdagent; userdel -f -r astra; umount /mnt/tmp; sleep 10; qm resize $vmid virtio0 +17G; qm template $vmid

vmid="9101"; vmname="tmp-w10pro"; qcow2file=/mnt/appc-pc/pub/iso/w10pro-image.qcow2
qm destroy $vmid; qm clone 9100 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9102"; vmname="tmp-w11pro"; qcow2file=/mnt/appc-pc/pub/iso/w11pro-image.qcow2
qm destroy $vmid; qm clone 9100 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9103"; vmname="tmp-w2019std-core"; qcow2file=/mnt/appc-pc/pub/iso/w2019std-core-image.qcow2
qm destroy $vmid; qm clone 9100 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9104"; vmname="tmp-w2019std"; qcow2file=/mnt/appc-pc/pub/iso/w2019std-image.qcow2
qm destroy $vmid; qm clone 9100 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9105"; vmname="tmp-w2022std-core"; qcow2file=/mnt/appc-pc/pub/iso/w2022std-core-image.qcow2
qm destroy $vmid; qm clone 9100 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; qm resize $vmid virtio0 +7G; qm template $vmid

vmid="9106"; vmname="tmp-w2022std"; qcow2file=/mnt/appc-pc/pub/iso/w2022std-image.qcow2
qm destroy $vmid; qm clone 9100 $vmid --full 1 --name $vmname; qm importdisk $vmid $qcow2file $qcow2storage; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0; qm resize $vmid virtio0 +7G; qm template $vmid

/mnt/appc-pc/pub/iso/w10pro-image.qcow2
/mnt/appc-pc/pub/iso/w11pro-image.qcow2
/mnt/appc-pc/pub/iso/w2019std-core-image.qcow2
/mnt/appc-pc/pub/iso/w2019std-image.qcow2
/mnt/appc-pc/pub/iso/w2022std-core-image.qcow2
/mnt/appc-pc/pub/iso/w2022std-image.qcow2

```
