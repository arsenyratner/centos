# RAID MD
https://raid.wiki.kernel.org/index.php/A_guide_to_mdadm
https://knowledgebase.starwindsoftware.com/guidance/recommended-raid-settings-for-hdd-and-ssd-disks/
RAID Level   Chunk size for HDD Arrays   Chunk size for SSD Arrays
0                    Disk quantity * 4Kb                Disk quantity * 8Kb
5                    (Disk quantity – 1) * 4Kb        (Disk quantity – 1) * 8Kb
6                    (Disk quantity – 2) * 4Kb        (Disk quantity – 2) * 8Kb
10                  (Disk quantity * 4Kb)/2          (Disk quantity * 8Kb)/2

```
slowdisk1="/dev/sda"
slowdisk2="/dev/sdb"
fastdisk1="/dev/sdc"
fastdisk2="/dev/sdd"
slowmd="/dev/md0"
fastmd="/dev/md1"

mdadm --create --verbose --assume-clean $slowmd --level=1 --raid-devices=2 $slowdisk1 $slowdisk2

```

# LVM Cache

## varaiables
```
lvcache_slowdev="/dev/md127"
lvcache_fastdev="/dev/md126"
lvcache_vgname="vg1-data"
lvcache_lv1_data="lv_vm_data"
lvcache_lv1_data_size="1024G"
lvcache_lv1_cache="lv_vm_cache"
lvcache_lv1_cache_size="200G"
lvcache_lv1_cache_meta="lv_vm_cache_meta"
lvcache_lv1_cache_meta_size="2G"

```
## create VG 
```
pvcreate $lvcache_slowdev
pvcreate "$lvcache_fastdev"
vgcreate $lvcache_vgname $lvcache_slowdev $lvcache_fastdev
```
## create LV
```
lvcreate -L $lvcache_lv1_data_size -n $lvcache_lv1_data $lvcache_vgname $lvcache_slowdev
lvcreate -L $lvcache_lv1_cache_size -n $lvcache_lv1_cache $lvcache_vgname $lvcache_fastdev
lvcreate -L $lvcache_lv1_cache_meta_size -n $lvcache_lv1_cache_meta $lvcache_vgname $lvcache_fastdev
lvconvert --type cache-pool --cachemode writeback --poolmetadata $lvcache_vgname/$lvcache_lv1_cache_meta $lvcache_vgname/$lvcache_lv1_cache
lvs -a -o +devices

lvconvert --type cache --cachepool $lvcache_vgname/$lvcache_lv1_cache $lvcache_vgname/$lvcache_lv1_data
lvs -a -o +devices

```
