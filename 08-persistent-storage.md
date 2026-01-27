# Add persistent storage to the cluster

1. Decide which Pi gets the SSD
2. SSH into that Pi
3. Plug in SSD (hot-plug)
4. Partition & format SSD
5. Mount it permanently
6. Tell Kubernetes about it
7. Create a StorageClass
8. Test with a PVC
9. (Optional) Move real workloads later


# Current usage, not total capacity

```
# Total memory capacity
kubectl describe nodes
## Or per node:
kubectl describe node 10.0.0.50


# Check current storage capacity (important before adding SSD)
ssh -i $HOME\.ssh\k3s-cluster adsieg@10.0.0.221
```


### View mounted filesystems
```
adsieg@adsieg-k3s-node-1:~$ df -h

Filesystem      Size  Used Avail Use% Mounted on
tmpfs           781M  4.4M  776M   1% /run
/dev/mmcblk0p2   29G  6.3G   22G  23% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/mmcblk0p1  505M  185M  320M  37% /boot/firmware
tmpfs           781M   12K  781M   1% /run/user/1000
```

### View physical disks
```
adsieg@adsieg-k3s-node-1:~$ lsblk

NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0 42.9M  1 loop /snap/snapd/24787
loop1         7:1    0 41.6M  1 loop /snap/snapd/25939
mmcblk0     179:0    0 29.8G  0 disk
├─mmcblk0p1 179:1    0  512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 29.3G  0 part /
```

### What happens after you plug it in
```
adsieg@adsieg-k3s-node-1:~$ lsblk

NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0 42.9M  1 loop /snap/snapd/24787
loop1         7:1    0 41.6M  1 loop /snap/snapd/25939
sda           8:0    0  934G  0 disk
└─sda1        8:1    0  934G  0 part
mmcblk0     179:0    0 29.8G  0 disk
├─mmcblk0p1 179:1    0  512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 29.3G  0 part /
```

### Format the disk (EXT4)
```
adsieg@adsieg-k3s-node-1:~$ sudo mkfs.ext4 /dev/sda
mke2fs 1.47.0 (5-Feb-2023)
Found a dos partition table in /dev/sda
Proceed anyway? (y,N) y
Discarding device blocks: done
Creating filesystem with 244842496 4k blocks and 61210624 inodes
Filesystem UUID: a318cca8-523e-4fe4-88be-cab17b3dba67
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

### Create a permanent mount point
````
# Create directory
sudo mkdir -p /mnt/k3s-ssd

Find UUID (important!)
sudo blkid /dev/sda1

sudo nano /etc/fstab

Add one line:
UUID=abcd-1234  /mnt/k3s-ssd  ext4  defaults,noatime  0  2

adsieg@adsieg-k3s-node-1:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           781M  4.4M  776M   1% /run
/dev/mmcblk0p2   29G  6.3G   22G  23% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/mmcblk0p1  505M  185M  320M  37% /boot/firmware
tmpfs           781M   12K  781M   1% /run/user/1000
/dev/sda        919G   28K  872G   1% /mnt/k3s-ssd
```




PS C:\Users\cloti\Desktop\HandsOn\k3-homelab-n-url-paths-v3> kubectl get storageclass
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  7d4h
local-ssd              rancher.io/local-path   Delete          WaitForFirstConsumer   false                  7s
PS C:\Users\cloti\Desktop\HandsOn\k3-homelab-n-url-paths-v3> 







PS C:\Users\cloti> kubectl label node 10.0.0.221 ssd=true
Error from server (NotFound): nodes "10.0.0.221" not found
PS C:\Users\cloti> kubectl get nodes
NAME                STATUS   ROLES           AGE    VERSION
adsieg-k3s-master   Ready    control-plane   7d4h   v1.34.3+k3s1
adsieg-k3s-node-1   Ready    <none>          7d2h   v1.34.3+k3s1
PS C:\Users\cloti> kubectl label node adsieg-k3s-node-1 ssd=true
node/adsieg-k3s-node-1 labeled




Now your applications can use this storage by creating PVCs with storageClassName: ssd-storage!

Create a Persistent Volume




- Capacity = physical RAM
- Allocatable = what Kubernetes can actually schedule
- Memory vs. storage


ssh -i $HOME\.ssh\k3s-cluster adsieg@10.0.0.221
