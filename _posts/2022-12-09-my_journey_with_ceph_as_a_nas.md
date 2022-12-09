---
title: My journey with CEPH as my expandable home NAS - Thoughts
key: 20221209
tags: 
  - sysadmin
  - ceph
  - storage
  - nas
  - linux
mathjax: true
mathjax_autoNumber: true
---

I'm planning from a long time to build/buy a small little NAS system for my personal storage.
When I first researched the topic and checked for some available solutions I discovered how many solutions are there to be "ready to use", from the prebuilt and ready to use Qnap solutions with two mirror disks to the more DIY and hardware recycling solution of TrueNas.

My first tackle to this topic was an old FreeNAS installation (yep, a long long time ago) on an older HPE server dating back to the 2000's so power hungry and inefficient to be capable of tripping the breakers by powering up.

It was a small, 2GB ram (PC2-FB) and three disks in RAID5 configuration and it lacked all of the wanted features for an home NAS:
  - Compact: It was a rack mountable 5 units by 19 inches weighting over 15kg
  - Energy efficient and silent: 600W Power supply and it's a server... It was fun whenever it rebooted in the middle of the night!
  - Expandable: _Will talk about later_

## My problems with Qnap

_For disclaimer, I've never owned a Qnap NAS, so I may be biased._

Qnap devices look great and maybe they work ok. What scares me is the computing power inside them and the fact that they aren't software customizable.
If you need a basic NAS for storing your photos and documents, go ahead, but you'll be tied to their OS (QTS) and their decisions. You will not be able to choose and configure your software and "feel at home".

I recognize that my thoughts are a little biased by my work/field of study and some other Open Source stuff but I prefer to be able to choose and change.

## My problems with TrueNAS (and ex FreeNAS)

TrueNAS is a very well made system for home and enterprise storage solutions. I have no doubts.
They also offer the Hyperconverged Compute & Storage capable verion TrueNAS Scale which yet I haven't tried but there is still the one little problem: disk scalability.

Last time I used TrueNAS it was an ok experience with everything working fine but with some rough patches. First of all I'm not a fan of FreeBSD with it's strange standards and inner workings. That made everything more complex. Plus add the fact that Bhyve does not work with some installers (\*coff\* \*coff\* ubuntu (: ) and that it is a nightmare...

## The problem with RAID

There are lots of problems with any RAID configuration or ZFS. Before talking about RAID extensibility let's think about how different raid configurations may fail taking away all of the data they contain.

 - **RAID 0**: If you deployed a NAS with a RAID 0 disk configuration you are mad. A lot. **WHYYYYYY**!?
    If any disk fails ALL of your data will be lost. You'd better have a couple recent backup images somewhere...

    I heard some comments talking about RAID 0 failures saying that you'll lose only the files stored on the failed disk but they cannot be more wrong. RAID 0 is fast because it parallelizes writes of adjacent data in different stripes across all the disk pool, it does not concatenates disks.
  
  - **RAID 1**: It's fast and it's the one with the more resilience. I think it is a waste of raw disk data tho.

  - **RAID 5**: One disk can be removed and the data remains still intact. Good.

  - **RAID 6**: Two disk can be removed and the data remains still intact. Good.

    Less probability that you suffer a data loss while rebuilding because of the double parity. Yet if you're using SSD's from the same production batch, chances are that they will fail all at once or very closely. You can get a situation where a disk fails during the rebuilding process, so you swap it and another fails while you're rebuilding the second one and so on until all the disks are changed.

  - **RAID 1+0**: You take some RAID 1 disks and stripe them. You just halved your data capacity.

  - **RAID X+0**: This is a good talk to do. This is one of the most adopted solutions and consists of striping RAID 5's or 6's such that your disks are grouped in "failure domains" generally of 5-10 disks.
    
    First of all, we are definitly exceeding the "Home NAS" thing because 10-20 disks will take a lot of space and processing power.

    You can loose up to 1/2 disks per failure domain and be some what safe. Some people say that in a `(RAID5(4))*3` (_A striping configuration of three RAID 5 failure domains_) you can loose up to three disks. What if you lose two disk in the same failure domain? Say goodby to your data :)

### RAID Extension

Let's take down the elephant in the room:

#### What if I need more space?

Since you cannot extend a RAID configuration you'll need to backup your data (As you should already have done, RAID is not backup -.- ), then you have two options:
  - Destroy your RAID and recreate it with more disks or different disks. This causes a long downtime and it will stop being scalable and affordable.
  - Create a new ride and live with two storages.

#### What if I run out of SATA ports and I want to add a disk?

That's only one option. New NAS or change a disk with a bigger one.
See previous question.

End of the story: you can't expand a RAID configuration while maintaining all of your data.

## Let's talk about Ceph




## Additional resources
 - https://www.abakus.si/download/events/2018_sergej_rozman_raid-ceph.pdf
 - https://blog.devgenius.io/ceph-install-single-node-cluster-55b21e6fdaa2
 - https://balderscape.medium.com/setting-up-a-virtual-single-node-ceph-storage-cluster-d86d6a6c658e
 - https://linoxide.com/hwto-configure-single-node-ceph-cluster/
 - https://vineetcic.medium.com/how-to-remove-add-osd-from-ceph-cluster-1c038eefe522
 - https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/5/html/operations_guide/management-of-osds-using-the-ceph-orchestrator#removing-the-osd-daemons-using-the-ceph-orchestrator_ops
 - https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/operations_guide/handling-a-disk-failure#replacing-a-failed-osd-disk_ops
 - https://arpnetworks.com/blog/2019/06/28/how-to-update-the-device-class-on-a-ceph-osd.html