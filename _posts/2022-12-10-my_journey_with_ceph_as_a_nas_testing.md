---
title: My journey with CEPH as my expandable home NAS - Testing
key: 20221210
tags: 
  - sysadmin
  - ceph
  - storage
  - nas
  - linux
mathjax: true
mathjax_autoNumber: true
---

First of all I'd rather not to buy hardware blindly and then start testing the system and it's configurations, so this post will be about testing a single node ceph cluster in a virtual environment.

## My Setup

I'm running everything on the following hardware configuration:
 - Computing: Intel Core i7-12700 16GB RAM DDR4@3200MT/s
 - Storage: 
  - 1x NVME 500GB disk
  - 3x WD-RED 1TB HDD 5400 rpm (WDC WD10EFRX-68F) in RAID5 configuration (Intel RST + mdadm)

### Host performances

Here are some approximate measurements of read/write speeds on the disks done with the commands:
```
sudo hdparm -Tt <disk> #for read performances
dd if=/dev/zero of=test.bin bs=1M count=2048 conv=fsync status=progress
```

HDD RAID speed reported to be

```
READS:
  Timing cached reads:   39396 MB in  2.00 seconds = 19732.09 MB/sec
  Timing buffered disk reads: 862 MB in  3.01 seconds = 286.77 MB/sec
WRITES:
  2048+0 records in
  2048+0 records out
  2147483648 bytes (2,1 GB, 2,0 GiB) copied, 21,1149 s, 102 MB/s
```

NVME (Only writes):

```
  2048+0 records in
  2048+0 records out
  2147483648 bytes (2,1 GB, 2,0 GiB) copied, 1,51033 s, 1,4 GB/s
```

## 1 - Spinning up the VM

I Used a virtual machine with 4 cores and 4096MB of ram allocated and Rocky Linux 9 minimal installation.
For the storage configuration I've gone with the following:
  - 1 boot disk (16GB) on my boot disk (nvme) qcow2
  - 3 disks (8GB each) on the mechanical storage in raw format fully allocated on sata bus (this is done for performance validation, qcow2 is SLOOOW)
  - 2 disks (6GB each) on the nvme disk in raw format fully allocated on SATA bus for testing out cache tiering performances


### Base system setup

```bash
sudo dnf update

#Set hostname to ceph-test
sudo hostnamectl set-hostname ceph-test
```

### Testing speeds

```bash
#Get disks and labels
[stefano@ceph-test ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0    8G  0 disk 
sdb           8:16   0    8G  0 disk 
sdc           8:32   0    8G  0 disk 
sdd           8:48   0    6G  0 disk 
sde           8:64   0    6G  0 disk 
sr0          11:0    1 1024M  0 rom  
vda         252:0    0   16G  0 disk 
├─vda1      252:1    0    1G  0 part /boot
└─vda2      252:2    0   15G  0 part 
  ├─rl-root 253:0    0 13.4G  0 lvm  /
  └─rl-swap 253:1    0  1.6G  0 lvm  [SWAP]

#Testing out speeds
[stefano@ceph-test ~]$ sudo dnf install hdparm

# Boot disk speed (vda)
[stefano@ceph-test ~]$ sudo hdparm -Tt /dev/vda

/dev/vda:
 Timing cached reads:   39310 MB in  2.00 seconds = 19694.27 MB/sec
 Timing buffered disk reads: 7938 MB in  3.00 seconds = 2645.92 MB/sec

[stefano@ceph-test ~]$ dd if=/dev/zero of=test.bin bs=1M count=1024 conv=fsync status=progress
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.836329 s, 1.3 GB/s

# SSDs

[stefano@ceph-test ~]$ sudo hdparm -Tt /dev/sde

/dev/sde:
 Timing cached reads:   37702 MB in  2.00 seconds = 18887.34 MB/sec
 Timing buffered disk reads: 4096 MB in  1.68 seconds = 2434.00 MB/sec
[stefano@ceph-test ~]$ sudo dd if=/dev/zero of=/dev/sde bs=1M count=1024 conv=fsync status=progress
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 1.16804 s, 919 MB/s

#HDDSs

[stefano@ceph-test ~]$ sudo hdparm -Tt /dev/sda

/dev/sda:
 Timing cached reads:   39666 MB in  2.00 seconds = 19871.21 MB/sec
 Timing buffered disk reads: 636 MB in  3.00 seconds = 211.75 MB/sec
[stefano@ceph-test ~]$ sudo dd if=/dev/zero of=/dev/sda bs=1M count=1024 conv=fsync status=progress
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 6.96485 s, 154 MB/s
```

As we can see times are inline with what resulted on the host.

### Installing ceph

First of all we should install some required stuff for it to work
```bash
sudo dnf install podman python3 llvm2 nano
```

We can now bootstrap our (and only) node
```bash
[stefano@ceph-test ~]$ curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
[stefano@ceph-test ~]$ chmod +x cephadm
[stefano@ceph-test ~]$ sudo ./cephadm add-repo --release quincy
...
Completed adding repo.
[stefano@ceph-test ~]$ sudo ./cephadm install
[stefano@ceph-test ~]$ sudo cephadm bootstrap --mon-ip 192.168.122.17 --allow-fqdn-hostname --skip-monitoring-stack --single-host-defaults
```

Where 192.168.122.17 is the current host ip.
You can now access the control dashboard at `https://<your ip>:8443/`

Enter the ceph shell next:
```bash
[stefano@ceph-test ~]$ sudo cephadm shell
...
[ceph: root@ceph-test /]#
```

## Building a standard pool and fs

Once we are into the shell we can now get all the available devices and create the osds.

```
[ceph: root@ceph-test /]# ceph orch device ls --refresh
HOST       PATH      TYPE  DEVICE ID                   SIZE  AVAILABLE  REFRESHED  REJECT REASONS  
ceph-test  /dev/sda  hdd   ATA_QEMU_HARDDISK_QM00009  8589M  Yes        11s ago                    
ceph-test  /dev/sdb  hdd   ATA_QEMU_HARDDISK_QM00007  8589M  Yes        11s ago                    
ceph-test  /dev/sdc  hdd   ATA_QEMU_HARDDISK_QM00011  8589M  Yes        11s ago                    
ceph-test  /dev/sdd  hdd   ATA_QEMU_HARDDISK_QM00013  6442M  Yes        11s ago                    
ceph-test  /dev/sde  hdd   ATA_QEMU_HARDDISK_QM00015  6442M  Yes        11s ago

#Tip: wait for full osc creation and start
[ceph: root@ceph-test /]# ceph orch daemon add osd ceph-test:/dev/sda
[ceph: root@ceph-test /]# ceph orch daemon add osd ceph-test:/dev/sdb
[ceph: root@ceph-test /]# ceph orch daemon add osd ceph-test:/dev/sdc
```

with the same method create the other two osd and change their class (if not recognized as ssd)

```
[ceph: root@ceph-test /]# ceph orch daemon add osd ceph-test:/dev/sdd
[ceph: root@ceph-test /]# ceph osd crush rm-device-class osd.3
[ceph: root@ceph-test /]# ceph osd crush set-device-class ssd osd.3
[ceph: root@ceph-test /]# ceph orch daemon add osd ceph-test:/dev/sde
[ceph: root@ceph-test /]# ceph osd crush rm-device-class osd.4
[ceph: root@ceph-test /]# ceph osd crush set-device-class ssd osd.4
```

Note that osd.3 and osd.4 are the ids ceph assigned to the newly created osds.

### Testing out: hdd only replicated 2/2

First create the replication rule
```
ceph osd crush rule create-replicated replicated_hdd default osd hdd

ceph osd pool create replicated_no_cache_fs replicated replicated_hdd --autoscale-mode=on
ceph osd pool set replicated_no_cache_fs size 2
ceph osd pool set replicated_no_cache_fs min_size 2
```

#### Testing raw pool speed

```
[ceph: root@ceph-test /]# rados bench -p replicated_no_cache_fs 10 write --no-cleanup
hints = 1
Maintaining 16 concurrent writes of 4194304 bytes to objects of size 4194304 for up to 10 seconds or 0 objects
Object prefix: benchmark_data_ceph-test_475
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
...
   14       4        44        40   11.4275        12     5.29316     4.54613
Total time run:         14.1894
Total writes made:      44
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     12.4036
Stddev Bandwidth:       4.35007
Max bandwidth (MB/sec): 20
Min bandwidth (MB/sec): 0
Average IOPS:           3
Stddev IOPS:            1.13873
Max IOPS:               5
Min IOPS:               0
Average Latency(s):     4.56899
Stddev Latency(s):      1.1601
Max latency(s):         6.0285
Min latency(s):         1.27842
```

12MB/s... pretty crappy, that's 1/6 of an USB2.0

### Testing out: hdd only erasure coded k=2 m=1

```
ceph osd erasure-code-profile set k2m1 k=2 m=1 crush-failure-domain=osd crush-device-class=hdd
ceph osd crush rule create-erasure erasure_hdd k2m

ceph osd pool create data erasure k2m1 erasure_hdd --autoscale-mode=on
ceph osd pool set data allow_ec_overwrites true
```

#### Testing raw pool speed

```
[ceph: root@ceph-test /]# rados bench -p data 30 write
...
Total time run:         33.455
Total writes made:      121
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     14.4672
Stddev Bandwidth:       4.41416
Max bandwidth (MB/sec): 20
Min bandwidth (MB/sec): 0
Average IOPS:           3
Stddev IOPS:            1.12057
Max IOPS:               5
Min IOPS:               0
Average Latency(s):     4.23906
Stddev Latency(s):      0.678004
Max latency(s):         5.70135
Min latency(s):         1.37831
Cleaning up (deleting benchmark objects)
Removed 121 objects
Clean up completed and total clean up time :2.4025
```

14MB/s... HOW? It should be slower...
I think I'll keep the 2/2 replication because EC is wess uited on more disks (like k=4 m=2)

## Trying cache...
### Caching pool 2/1

First of all we need to create the cache pool and the relative crush rule
```
ceph osd crush rule create-replicated replicated_cache default osd ssd

ceph osd pool create cache replicated replicated_cache --autoscale-mode=on
ceph osd pool set cache size 2
ceph osd pool set cache min_size 1
```

Now we can add the cache tier

```
ceph osd tier add <data pool name> cache
ceph osd tier cache-mode cache writeback
ceph osd tier set-overlay replicated_no_cache_fs cache
ceph osd pool set cache hit_set_type bloom
ceph osd pool set cache hit_set_count 12
ceph osd pool set cache hit_set_period 14400
ceph osd pool set cache target_max_bytes <0.8*your ssd capacity> 9600000000
```

#### Small parenthesis, testing the cache raw speed

```
[ceph: root@ceph-test /]# rados bench -p cache 10 write
...
Total time run:         10.1919
Total writes made:      1468
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     576.142
Stddev Bandwidth:       125.174
Max bandwidth (MB/sec): 664
Min bandwidth (MB/sec): 288
Average IOPS:           144
Stddev IOPS:            31.2936
Max IOPS:               166
Min IOPS:               72
Average Latency(s):     0.110211
Stddev Latency(s):      0.0341686
Max latency(s):         0.231888
Min latency(s):         0.053774
Cleaning up (deleting benchmark objects)
Removed 1468 objects
Clean up completed and total clean up time :0.234817
```
A good amount of cpu was used and I'm pretty sure the process was cpu bottleneck'd

#### Testing the final speed
```
[ceph: root@ceph-test /]# rados bench -p replicated_no_cache_fs 10 write
...
Total time run:         10.1919
Total writes made:      1468
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     576.142
Stddev Bandwidth:       125.174
Max bandwidth (MB/sec): 664
Min bandwidth (MB/sec): 288
Average IOPS:           144
Stddev IOPS:            31.2936
Max IOPS:               166
Min IOPS:               72
Average Latency(s):     0.110211
Stddev Latency(s):      0.0341686
Max latency(s):         0.231888
Min latency(s):         0.053774
Cleaning up (deleting benchmark objects)
Removed 1468 objects
Clean up completed and total clean up time :0.234817
```

## Deleting pools

Pool deletion is disabled by default.
activate it by
```
ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
```

you can now delete the pool
```
ceph osd pool delete <name> <name> --yes-i-really-really-mean-it
```

## Configuring a file system

```
ceph osd pool create metadata 128 replicated replicated_hdd 
ceph osd pool set metadata size 3
ceph osd pool set metadata min_size 2

ceph osd pool application enable data cephfs
ceph osd pool application enable metadata cephfs

ceph orch apply mds ceph-test
ceph orch apply nfs ceph-test

ceph fs new data_fs metadata data
```

now you can deploy a NFS export from the dashboard and mount on a client computer.

### Testing speeds

```
sudo mount -t nfs 192.168.122.17:/ /mnt
sudo dd if=/dev/zero of=/mnt/data/test.bin bs=1M count=2048 conv=fsync status=progress
...
1206910976 bytes (1,2 GB, 1,1 GiB) copied, 4 s, 296 MB/s
1212153856 bytes (1,2 GB, 1,1 GiB) copied, 5 s, 237 MB/s
1217396736 bytes (1,2 GB, 1,1 GiB) copied, 6 s, 198 MB/s
1221591040 bytes (1,2 GB, 1,1 GiB) copied, 7 s, 174 MB/s
1226833920 bytes (1,2 GB, 1,1 GiB) copied, 8 s, 152 MB/s
1233125376 bytes (1,2 GB, 1,1 GiB) copied, 9 s, 137 MB/s
...
2048+0 records in
2048+0 records out
2147483648 bytes (2,1 GB, 2,0 GiB) copied, 119,608 s, 18,0 MB/s
```

As you can see, the copy starts pretty fast and the speeds decays because of buffers and similar things.

## Oh no, A disk is gone!

Simply delete the osd, change the disk and recreate it.
Ceph will take care of moving and rewriting missing pgs

```
ceph osd out osd.0
ceph orch osd rm 0
ceph osd purge 0 --yes-i-really-mean-it
ceph orch daemon rm osd.0 --force
ceph orch device ls --refresh
ceph orch daemon add osd localhost.localdomain:/dev/sdf
```

Once the osd has been recreated, data will start to flow into the disk. remember to change the disk class if you're in a virtual machine!

## Expanding

Add a new osd and the crushrules will do the math and move things around

```
ceph orch daemon add osd localhost.localdomain:/dev/sdf
```

Another way of expanding is to remove an existing disk, wait for cluster rebalance and then add a new bigger disk. ceph should handle this ssituation but you'll get a disk with more IOPS and more data into it. Be sure that all the data in the bigger disck can be moved to the others!

## Additional resources
 - https://blog.devgenius.io/ceph-install-single-node-cluster-55b21e6fdaa2
 - https://balderscape.medium.com/setting-up-a-virtual-single-node-ceph-storage-cluster-d86d6a6c658e
 - https://linoxide.com/hwto-configure-single-node-ceph-cluster/
 - https://vineetcic.medium.com/how-to-remove-add-osd-from-ceph-cluster-1c038eefe522
 - https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/5/html/operations_guide/management-of-osds-using-the-ceph-orchestrator#removing-the-osd-daemons-using-the-ceph-orchestrator_ops
 - https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/operations_guide/handling-a-disk-failure#replacing-a-failed-osd-disk_ops
 - https://arpnetworks.com/blog/2019/06/28/how-to-update-the-device-class-on-a-ceph-osd.html
 - https://docs.ceph.com/en/latest/rados/operations/erasure-code/
 - https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/ZJCYFAIUSPJGJFDIMVYOZ4K4AAM2BLL7/