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
vmname="r-vm-w10p1"
vmmemory=4096
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

```
vmid=113
qcow2file=/mnt/appc-pc/pub/iso/astra/alse-vanilla-1.7.4uu1-cloud-base-mg12.0.1.qcow2
qcow2storage="rpool"
qcow2options=",cache=writeback"
qm importdisk $vmid $qcow2file rpool; qm set $vmid --virtio0 $qcow2storage:vm-${vmid}-disk-0${qcow2options}; qm set $vmid --boot c --bootdisk virtio0
```
