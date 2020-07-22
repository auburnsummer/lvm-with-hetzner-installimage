# Notes on using Hetzner `installimage` with LVM

There isn't that much information about this, is there?

I thought I'd write up my own notes. This isn't a tutorial. You will probably
need to adapt stuff to your own situation. But I hope it will be helpful to at
least one other person.

## The Scenario

I have a Hetzner auction server. It has three physical drives:

 - 1x 500 GB SATA SSD
 - 2x 3 TB HDD

The server's doing double duty. There's a Postgres instance and various webapps which I want to run on the SSD. There's also an IPFS instance which I'm
using for data storage. The IPFS data can go on the 6 TB of HDD space. 

## LVM Concepts

 - A **Physical Volume** is a physical partition that additionally has an LVM header
   installed on it.
   [Apparently][1] you _can_ make a Physical Volume that spans more than one physical partition, but that's a Bad Idea.

   The other thing is that you can make a "Physical Volume" on any
   "disk-like" object. The Arch Wiki gives making a Physical Volume on a loopback file
   as an example. Another one is if you're using device mapper to create a RAID array,
   you can then use the RAID array as a Physical Volume.

   The take away is I need to partition my drives with normal physical partitions
   before I can use them for LVM. Okay, so far so good.

 - A **Volume Group** is a group of Physical Volumes. The storage of a Volume Group
   is just the combined storage capacity of the component volumes.

   So this is starting to make sense to me. If I have two 3TB Physical Volumes corresponding to my two drives, I can combine those two Physical Volumes into one 6 TB Volume Group.

 - A **Logical Volume** is the LVM equivalent of a physical partition. You can format them
   with file systems and mount them in places and do all your regular partition stuff with them. A Logical Volume takes up some portion of a Volume Group.

 - Each Physical Volume is split into **Physical extents**. Arch Wiki says the default size
   of each Physical extent is 4 MiB, which implies you can change it if you want.

   Logical Volumes are also split into **Logical extents**. The way LVM works is that it
   maps logical extents to physical extents. The obvious consequence is that the minimum
   size of a Logical Volume is going to be dictated by the size of the extents.

## Applying the LVM Concepts to my situation.

Here's my initial plan:

 - Make three physical partitions, one spanning the entire size of each of the
   disks.
 - Make three Physical Volumes that map to each of the partitions.
 - Make two Volume Groups, one which only has the SSD and the other which spans
   both the HDD Physical Volumes
 - Split the first volume group (SSD) into a `/` and swap Logical Volumes
 - Allocate the second volume group (the two HDDs) as one big Logical Volume

## Making sense of `installimage`

`installimage` is kinda confusing. Here's how we're going to do it:

 - Install LVM on the SSD partition
 - Leave the HDDs alone for now, we can do them later

So I started by commentating out `DRIVE2` and `DRIVE3` at the top, then turning
off `SWRAID` and changing `SWRAIDLEVEL` to something that wasn't 5. 

Next up, Hetzner helpfully informs us the `/boot` partition _cannot_ be on LVM, so
we're going to make a normal `/boot` partition, and everything else will be LVM.

```
PART /boot ext3 512M
PART lvm   vg0  all
```

This creates an ext3 formatted partition in the first 512M of the SSD and assigns
the remainder of it to a Volume Group called `vg0`.

Huh, didn't I just make a Volume Group straightaway without having to make a
Physical Volume? I guess `installimage` must automatically make the partition and
Physical Volume for you.


Then we can assign the logical volumes. I'll make 6 GB of swap and the remainder
just goes to `/`.

```
LV vg0 swap swap swap 6G
LV vg0 root /    ext4 all
```

Then I ran it! Hetzner whirred away for a while and then told me to reboot.

## Adding the other two drives to the LVM

If you look at `/etc/fstab` or `lsblk` you can check that the LVM is what you expected.

Now for the other two drives! They're not even partitioned yet, so first I need to
go into `parted` and make the partitions. The [Arch Wiki page][2] on parted is helpful here.

I made one partition on each of the drives. `lsblk` now looks like this:

```
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda            8:0    0   477G  0 disk
├─sda1         8:1    0   512M  0 part /boot
└─sda2         8:2    0 476.4G  0 part
  ├─vg0-swap 253:0    0     6G  0 lvm  [SWAP]
  └─vg0-root 253:1    0 470.4G  0 lvm  /
sdb            8:16   0   2.7T  0 disk
└─sdb1         8:17   0   2.7T  0 part
sdc            8:32   0   2.7T  0 disk
└─sdc1         8:33   0   2.7T  0 part
```

You can see `sda1` which is the boot physical partition, which isn't part of the
LVM. Then there's `sda2` which you can see is part of a Volume Group `vg0` and has
two Logical Volumes. There are two other partitions `sdb1` and `sdc1` but I haven't
added them to the LVM yet.

Next I looked at the Arch Wiki page for [LVM](https://wiki.archlinux.org/index.php/LVM)
which tells us to use `pvcreate` to create a Physical Volume:

```
# pvcreate /dev/sdb1
# pvcreate /dev/sdc1
```

Then I can stick them in the same volume group:

```
# vgcreate vg1 /dev/sdb1 /dev/sdc1
```

Running `vgs` gives me this:

```
  VG  #PV #LV #SN Attr   VSize   VFree
  vg0   1   2   0 wz--n- 476.43g     0
  vg1   2   0   0 wz--n-  <5.46t <5.46t
```

Dang look at that 5.46 TB volume group! That's a lot of hard drive space!

Finally we make a Logical Volume on that using 100% of the space:

```
lvcreate -l 100%VG vg1 -n data
```

This is what `lsblk` looks like after that:

```
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda            8:0    0   477G  0 disk
├─sda1         8:1    0   512M  0 part /boot
└─sda2         8:2    0 476.4G  0 part
  ├─vg0-swap 253:0    0     6G  0 lvm  [SWAP]
  └─vg0-root 253:1    0 470.4G  0 lvm  /
sdb            8:16   0   2.7T  0 disk
└─sdb1         8:17   0   2.7T  0 part
  └─vg1-data 253:2    0   5.5T  0 lvm
sdc            8:32   0   2.7T  0 disk
└─sdc1         8:33   0   2.7T  0 part
  └─vg1-data 253:2    0   5.5T  0 lvm
```

So there's now a new lvm partition called `vg1-data`. We did it!

The partition is located at `/dev/mapper/vg1-data`. I can make a file system
and mount it, etc as normal.

[1]: https://www.redhat.com/sysadmin/create-physical-volume
[2]: https://wiki.archlinux.org/index.php/parted