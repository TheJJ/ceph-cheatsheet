Ceph Cheatsheet
===============

(c) 2018-2022 Jonas Jelten <jj@sft.lol>

Released under GPLv3 or any later version.

<!-- markdown-toc start -->
**Table of Contents**

- [Ceph Cheatsheet](#ceph-cheatsheet)
  - [What?](#what)
  - [Components](#components)
  - [Principle](#principle)
    - [Data placement](#data-placement)
  - [Setup](#setup)
    - [Architecture](#architecture)
    - [Setup Links](#setup-links)
    - [Monitor Setup](#monitor-setup)
      - [Settings](#settings)
      - [Monmap](#monmap)
    - [Manager Setup](#manager-setup)
      - [Crash dump collection](#crash-dump-collection)
    - [Storage](#storage)
      - [Add BlueStore OSD](#add-bluestore-osd)
        - [Adding many OSDs at once](#adding-many-osds-at-once)
      - [Separate OSD Database and Bulk Storage](#separate-osd-database-and-bulk-storage)
        - [Migrate of OSD journal and database](#migrate-of-osd-journal-and-database)
    - [Metadata Setup](#metadata-setup)
    - [Balancer](#balancer)
    - [Erasure Coding](#erasure-coding)
    - [Pools](#pools)
      - [Placement group autoscaling](#placement-group-autoscaling)
    - [Crushmap](#crushmap)
    - [CephFS](#cephfs)
      - [CephFS Setup](#cephfs-setup)
        - [Add Users](#add-users)
          - [CephFS Quotas and Layouts](#cephfs-quotas-and-layouts)
          - [Snapshot Permissions](#snapshot-permissions)
        - [Subdatapool](#subdatapool)
        - [RBD datapool for client](#rbd-datapool-for-client)
        - [Namespace](#namespace)
        - [Quota Config](#quota-config)
        - [CephFS Snapshots](#cephfs-snapshots)
      - [CephFS Status](#cephfs-status)
      - [Tuning CephFS](#tuning-cephfs)
    - [RADOS Block Devices RBD](#rados-block-devices-rbd)
      - [Kernel RBD client](#kernel-rbd-client)
        - [Tuning KRBD](#tuning-krbd)
      - [Useful RBD commands](#useful-rbd-commands)
      - [Automatic mapping](#automatic-mapping)
      - [Manual mapping](#manual-mapping)
      - [RBD Status](#rbd-status)
    - [Cluster Performance](#cluster-performance)
    - [Random infos](#random-infos)
    - [Minimal client config](#minimal-client-config)
  - [Operation](#operation)
    - [Cluster status](#cluster-status)
    - [Utilization](#utilization)
    - [Daemon control and info](#daemon-control-and-info)
    - [OSD adding and removing](#osd-adding-and-removing)
    - [Device control](#device-control)
    - [Device performance](#device-performance)
      - [Raw device speed](#raw-device-speed)
      - [OSD speed](#osd-speed)
      - [RADOS speed](#rados-speed)
      - [RBD speed](#rbd-speed)
      - [CephFS speed](#cephfs-speed)
    - [PG Autoscale](#pg-autoscale)
    - [PGs not starting](#pgs-not-starting)
      - [Stuck after adding new OSDs](#stuck-after-adding-new-osds)
    - [MDS](#mds)
    - [Recovery](#recovery)
    - [Data Corruption](#data-corruption)
      - [OSD crashes](#osd-crashes)
      - [Incomplete PGs](#incomplete-pgs)
    - [Decrypt OSDs](#decrypt-osds)
  - [Kernel Feature List](#kernel-feature-list)
  - [Tricks](#tricks)
    - [Combine Multiple RBDs](#combine-multiple-rbds)
    - [RBD client local cache](#rbd-client-local-cache)
    - [Available Space](#available-space)
    - [Tipps](#tipps)

<!-- markdown-toc end -->


What?
-----

[Ceph](https://ceph.com/) is a distributed storage cluster.

In this file, I try to compress my knowledge and recommendations about operating Ceph clusters.

I'm sorry for the chaos in this cheatsheet, it basically serve(s/d) as my searchable command reference...
If you think it's worth a shot, please submit pullrequests.

[How to debug](https://docs.ceph.com/en/latest/rados/troubleshooting/log-and-debug/).


Components
----------

Component | Description
----------|------------
Client | Something that connects to the Ceph cluster to access data.
[Cluster](https://docs.ceph.com/en/latest/rados/) | All Ceph components as a whole, providing storage (RADOS)
[Object Store Device (OSD)](https://docs.ceph.com/en/latest/rados/configuration/common/#osds) | Actually stores data on single drive (key-value db)
[Monitor (MON)](https://docs.ceph.com/en/latest/rados/configuration/common/#monitors) | Coordinates the cluster (odd number, >=1), stores state on local disk
[Metadata Server (MDS)](https://docs.ceph.com/en/latest/cephfs/) | Handles filesystem inode transactions, stores its state on OSDs
[Manager (MGR)](https://docs.ceph.com/en/latest/mgr/) | Collects statistics, balances, hosts webui, collects crashes, ...

Thing | Description
------|------------
Object | Data stored under a key (name), like a C++ `unordered_map` or Python `dict`
Pool | Group of objects to store in the same way (redundancy, placement), access realm
Namespace | Object name prefix to create access realms
[Placement Group](https://docs.ceph.com/en/latest/rados/operations/placement-groups/) | Partition of a pool by object key (name) hashes


Principle
---------

The gist of how Ceph works:

All services store their data as "objects", usually 4MiB size.
A huge file or a block device is thus split up into 4MiB pieces.

An object is "randomly" placed on some OSDs, depending on placement rules to ensure desired redundancy.

Ceph provides basically 4 services to clients:
* Block device ([RBD](https://docs.ceph.com/en/latest/rbd/))
* Network filesystem ([CephFS](https://docs.ceph.com/en/latest/cephfs/))
* Object gateway ([RGW, S3, Swift](https://docs.ceph.com/en/latest/radosgw/))
* Raw key-value storage via [(lib)rados](https://docs.ceph.com/en/latest/man/8/rados/)

A ceph-client is e.g.
* Linux kernel that has CephFS mounted, e.g. at `/srv/hugestorage`
  * which provides you a mounted directory where you can store whatever you want
* Linux kernel that has a RBD mapped as `/dev/rbd42`, on top of which can be LVM, a filesystem, ...
  * which you can then use like a regular block device/filesystem for whatever Linux service you want
* QEMU that uses a RBD as virtual disk for the VM
* NFS Ganesha that converts a CephFS directory to NFS
* Samba that uses `vfs_ceph` to provide CephFS directories via CIFS
* The `ceph`, `rbd`, `rados`, ... commandline tools (yes, they're just clients).


### Data placement

Why does Ceph **scale**? Why is it [secure™ and safe™](https://github.com/SFTtech/sticker/raw/master/sicher/sicher.pdf)?

Basically, data is stored on multiple OSDs, while respecting constraints.
When OSDs fail, the missing copy is automatically recreated somewhere else.

Cluster partitioning in pools, PGs and shards:
* A cluster consists of pools, each can have custom redundancy and placement settings
* A pool is partitioned in `2^x` placement groups (PG) - an object is stored in one of its pools PGs
* The chosen pool redundancy now affects each PG e.g. "3 copies on separate servers" (3 OSDs) or "raid 6 on 12 servers" (12 OSDs)
* Each OSD participating in serving a PG is called a PG shard. Which OSD should serve what PG shard is determined by CRUSH, respecting the desired redundancy and topology constraints

To read/write an object:
* Needed: Pool of the object, object's key ("name")
* Hash the object's key
* Take the last `x` bits of the hash and choose the placement group (that's why we have `2^x` PGs)
* Use the CRUSH algorithm (and upmap updates) to find the primary OSD id of this placement group
* Look in the "OSDMap" to figure out the for the IP address of the PG's primary OSD
* Establish a connection and talk to the primary OSD of this placement group and retrieve the object data
  * In case the pool stores data as erasure code ("RAID"), the primary OSD contacts the remaining needed OSDs in its PG for reconstructing the data
  * When writing, the PG's primary OSD contacts all other OSDs in the same PG to let them write, and then waits until they have acknowledged, and then confirms the write to the client

Since **all data is chunked** into (usually) 4MiB blocks, each of the blocks of a file is in a different PG, i.e. on different OSDs, thus we talk to a different primary OSD for each block.

-> All requests are spread accross all OSDs of the whole cluster

* Recovery: When the desired object redundancy is no longer met (due to unavailable OSDs - drive-, server-, rack-failure), Ceph recreates the missing data automatically
* Adding OSDs: move ("backfill") some of the existing placement groups to the new OSDs so every OSD hopefully stores roughly the same amount of data.
* Removing OSDs: move ("backfill") the placement groups existing on the to-be-removed OSDs to others that are not removed.


Setup
-----

Ceph generally **works very well** if you use it like others are using it.
Once you deviate from the default, [prepare for unforseen consequences](https://www.youtube.com/watch?v=xtrqYdvZ29E).

But *I*'ve never lost a bit of data so far, I had to apply extensive massage a few times, but I got it to recover every time (so far).


### Architecture

Ceph is pretty flexible, and things work "better" the more uniform your setup is.
All in all you have to run Ceph's components on the machines you have, so storage is created magically.

As a reminder, the components:
* client: e.g. a Linux kernel, a QEMU process, a samba or NFS server to provide storage for a VM, a webserver, BigBlueButton recordings, whatever.
* MON: cluster synchronizer, you should have `2n+1` many of them, usually 3 or 5 so you can tolerate 1 or 2 outages (or rather: maintenances).
* OSD: a single key-value storage device, which actually stores all data
* MGR: statistics collector, where e.g. OSDs report to
* MDS: CephFS inode metadata service, which uses OSDs to store its metadata, and clients write and retrieve data pointed to by the MDS metadata directly from OSDs.

How you create it, is up to you, but know this:

* OSD:
  * Each OSD requires >1GiB RAM, I recommend 4GiB. It's mainly for caching, configurable with `osd_memory_target`.
  * It needs a quite capable CPU (e.g. the erasure coding, compression, and of course the regular storage request path).
* MON:
  * The CPU doesn't need to be too crazy, but should be good enough.
  * The MON storage should be fast: All management actions are quicker then.
* MDS:
  * The CPU should be quite capable when handling lots of requests, otherwise not super important.
  * The MDS metadata-pool (the CephFS metadata) should be very fast.
  * The MDS itself needs lots of ram, depending on your filesystem open file count (usually >4GiB, but can be >32GiB for millions of files).
* MGR:
  * Doesn't need any storage, just a medium-grade CPU and 2G RAM maybe.

You can run these services on any device that's suitable, combine them, provide the actual storage drives over FibreChannel to a Linux server running the Ceph OSDs, whatever.

For example:
* 1 Server. Run one MON, one MGR and OSDs, and two MDS (if you use CephFS).
  * I'd say if you want at least some performance, have at least 6 OSDs
  * You can store data in replicated or erasure coded pools
  * In such a "cluster" you can still store >500T, I know such a thing...
* 3 Servers. All have a MON and a couple of OSDs. You store data by in a replicated pool, each data copy on a separate server.
* Many Servers. Each has some OSDs (4 to 64) and all are clustered together.
  * Separate servers for MONs and OSDS.
  * Ideally also independent MDS servers (if you use CephFS), but you can run them on the MON server with enough RAM.
  * You can store data in erasure coded (EC, a bit like a RAID5/6/...) pools, but you need sufficiently many servers then, or else it has to behave like one big server and things go down whenever you restart only one of them.
  * That's the "standard" setup.

Each server should have 2x10GiB bond/link aggregation or at least 10GiB connectivity.

I wouldn't create separate "public" and "cluster" networks, since having one big link provides more peak performance for both scenarios - more internal and external traffic bandwidth.


### Setup Links

What [hardware?](https://docs.ceph.com/en/latest/start/hardware-recommendations/)

Ceph needs an odd number of >=1 MONs to [get a quorum](https://en.wikipedia.org/wiki/Paxos_(computer_science)).

For better understanding the setup, I recommend the [manual method](https://docs.ceph.com/en/latest/install/index_manual/).

For config options, see the [upstream documentation](https://docs.ceph.com/en/latest/rados/configuration/ceph-conf/).


### Monitor Setup
[Monitor config documentaion](https://docs.ceph.com/en/latest/rados/configuration/mon-config-ref/)

Follow the 'manual method' above to add a `ceph-$monid` monitor, where `$monid` usually is a letter from `a-z`,
but we use creative names (the host name).

Monitors don't use `ceph.conf` for their addressing, they store it internally.
[IP changing guide](https://docs.ceph.com/en/latest/rados/operations/add-or-rm-mons/#changing-a-monitor-s-ip-address).

For more hosts, [monitors can be added](https://docs.ceph.com/en/latest/rados/operations/add-or-rm-mons/).


#### Settings

http://docs.ceph.com/docs/mimic/rados/configuration/mon-osd-interaction/


```
# if too many osds are out, don't operate.
mon osd min in ratio = 100
```

#### Monmap

To edit the monitor map (e.g. to change names and IP addresses of monitors):

```
ceph-mon --extract-monmap --name mon.monname /tmp/monmap
monmaptool --print /tmp/monmap
monmaptool --help   # edit the monmap
ceph-mon --inject-monmap --name mon.monname /tmp/monmap
```


### Manager Setup
[Manager config documentation](https://docs.ceph.com/en/latest/mgr/administrator/)

You should run one manager for each monitor, but having more doesn't hurt.
Offloads work from MONs and allows scaling beyond 1000 OSDs (statistics and other unimportant stuff like disk usage)
One MGR is active, all others are on standby.

This creates a access key for cephx for the manager.
```
sudo -u ceph mkdir -p /var/lib/ceph/mgr/ceph-$mgrid
ceph auth get-or-create mgr.$mgrid mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-$mgrid/keyring

# test it:
sudo -u ceph ceph-mgr -i $mgrid -d

# actually activate it
sudo systemctl enable --now ceph-mgr@$mgrid.service
```

The manager provides a [shiny dashboard](https://docs.ceph.com/en/latest/mgr/dashboard/) and other plugins (e.g. the [balancer](https://docs.ceph.com/en/latest/mgr/balancer/))


#### Crash dump collection

The manager has a [crash collection module](https://docs.ceph.com/en/latest/mgr/crash/).

Enable it:

```
ceph mgr module enable crash
```

Setup crash collection with `ceph-crash.service`:

On each of your servers with OSDs etc, deploy `/etc/ceph/ceph.client.crash.keyring`, containing the key for `client.crash`, created like this:

```
ceph auth get-or-create client.crash mon 'profile crash' mgr 'profile crash'

# or, restrict the crash reports to a specific subnet!
ceph auth get-or-create client.crash mon 'profile crash' mgr 'profile crash network 1337.42.42.0/24'
```

Then enable and start `ceph-crash.service`.
`ceph-crash.service` runs on on each server periodically looks inside `/var/lib/ceph/crash` for new-to-report crashes, and then uses `ceph crash post` to send them to the MGR.
In order to post, it tries `client.crash` as username, and to submit we need both `mgr` and `mon` `crash`-caps, otherwise the upload will fail with `[errno 13] error connecting to the cluster` and `Error EACCES: access denied: does your client key have mgr caps?`.

The `profile crash` allows you running `ceph crash post` (which the `ceph-crash` uses to actually report stuff).

New crashes appear in `ceph status`.
Details: `ceph crash`


### Storage

[How to add devices](https://docs.ceph.com/en/latest/ceph-volume/lvm/prepare/).

Before adding OSDs, you can do `ceph osd set noin` so new disks are not filled automatically.
Once you want it to be filled, do `ceph osd in $osdid`.

#### Add BlueStore OSD

Add a [BlueStore](https://docs.ceph.com/en/latest/rados/configuration/bluestore-config-ref/) device, with the help of LVM. In the LVM tags, metainfo is stored.

The data and journal (WAL) and keyvalue-DB can be placed on different devices (HDD and SSD).

Use `--dmcrypt` to encrypt the HDD. This just uses [LUKS](https://en.wikipedia.org/wiki/LUKS)! The key are stored in the MONs.

Use `--crush-device-class somename` to assign a device class (any name is possible), autodetected are `hdd`, `ssd` and `nvme`.

```
sudo ceph-volume lvm create --dmcrypt --data /dev/partition
```

If you use `/dev/partition`, an LVM LV is created:
* The VG is `ceph-$(uuidgen)` (generated with `python3 -c "import uuid; print(uuid.uuid4())"`)
* The LV is `osd-block-$(uuidgen)`

If you use `vg/lvname` instead of `/dev/partition`, the lv is `luksFormated` and then initialized.

SSDs should be set up with `noop` ioscheduler, HDDs with `deadline`. These are settings of Linux.

[Adding/removing OSDs](https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/).

Failed OSDs can be removed with `ceph osd purge-new $id`.


Information about the drive is placed in `LVM` in its tags:
```
sudo lvs --readonly --separator=" " -o lv_tags,lv_path,lv_name,vg_name,lv_uuid,lv_size [optional_path_to_lv]
```

Device setup (opening, decryption, ...) is done with `ceph-volume` and `ceph-volume-systemd`.

The **secret hdd keys** for `--dmcrypt` are stored in the `config-key` database in the mons.

```
ceph config-key dump | grep dm-crypt
```

##### Adding many OSDs at once

* Ensure the cluster is healthy (`HEALTH_OK`)
* `ceph osd set norebalance nobackfill`
* Add the OSDs with normal procedure as above
* Let all OSDs peer, this might take a few minutes
* `ceph osd unset norebalance nobackfill`
  * now the cluster fills up the new OSDs
* Everything's done once cluster is on `HEALTH_OK` again


#### Separate OSD Database and Bulk Storage

You can store the metadata and data of an OSD on different devices.

Usually, you store the database and journal on a `fastdevice`:
* `ceph-volume lvm create --data /dev/slowdevice --block.db /dev/fastdevice`

If your `fastdevice` is too small, you can store only the journal on it:
* `ceph-volume lvm create --data /dev/slowdevice --block.wal /dev/fastdevice`

You can even store on three different blockdevices: `data`, `block.db` and `block.wal`.

Because a fast device (SSD, NVMe is usually so much faster than a HDD),
you can use multiple partitions and place one DB on each.

If the "external" DB is full, the data-device will be used to store the remaining information.
BlueStore will automatically relocate often-used data to the fast device then.


##### Migrate of OSD journal and database

With [`ceph-bluestore-tool`](https://docs.ceph.com/en/latest/man/8/ceph-bluestore-tool/), you can create, migrate expand and merge OSD block devices.

* To move the block.wal from an all-in-one OSD to a new device with target full partition size:
```
ceph-bluestore-tool --command bluefs-bdev-new-wal --dev-target /dev/system/osdwal$id --path /var/lib/ceph/osd/ceph-$id
```

* The same is possible with DB (i.e. wal + most hot rocksdb data):
```
ceph-bluestore-tool --command bluefs-bdev-new-db --dev-target /dev/system/osdwal$id --path /var/lib/ceph/osd/ceph-$id
```

* For example, **view the sizes** of all involved BlueFS block devices:
```
ceph-bluestore-tool --command bluefs-stats --path /var/lib/ceph/osd/ceph-$i
```

* You can pass some arguments via env-variables if needed: `CEPH_ARGS="--bluestore-block-db-size 2147483648" ceph-bluestore-tool ...`
* To resize a `block.db`, use `bluefs-bdev-expand` (e.g. when the underlying partition size was increased)
* To merge separate `block.db` or `block.wal` drives onto the slow disk, use `bdev-migrate`
* Details for all the commands are in the [manpage](https://docs.ceph.com/en/latest/man/8/ceph-bluestore-tool/)


### Metadata Setup

Metadata servers (MDS) are needed for the CephFS.

```
sudo -u ceph mkdir -p /var/lib/ceph/mds/ceph-$mdsid
sudo -u ceph ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-$mdsid/keyring --gen-key -n mds.$mdsid
sudo -u ceph ceph auth add mds.$mdsid osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-$mdsid/keyring
# test with sudo ceph-mds -f --cluster ceph --id $mdsid --setuser ceph --setgroup ceph
sudo systemctl enable --now ceph-mds@$mdsid.service
```

Multiple MDS servers can be active, they distribute the inode workload. Kernel clients support this [since Linux 4.14](#Kernel feature list).

To assign a hot-standby to every active MDS:

```
ceph fs set $fsname allow_standby_replay <true|false>
```


### Balancer

This is super-important to use when you have different-sized OSDs!

Also, make sure for right balancing that big pools have enough PGs, otherwise each shard of the PG is very big already.
If there's too less PGs for a larger pool, the PGs are balanced correctly, but **one** might already fill up a OSD by 200G.

```
# shard size calculation
if pool is replica:
    shardsize = pool_size / pg_num
elif pool is erasurecoded(n+m):
    shardsize = pool_size / (pg_num * n)
```

For ideal balancing, the shard sizes of all pools should be equal!


```
ceph tell 'mgr.*' injectargs -- --debug_mgr=4/5    # for: `tail -f ceph-mgr.*.log | grep balancer`
ceph balancer status
ceph balancer mode upmap         # upmap items as movement method, not reweighting.
ceph balancer eval               # evaluate current score
ceph balancer optimize myplan    # create a plan, don't run it yet
ceph balancer eval myplan        # evaluate score after myplan. optimal is 0
ceph balancer show myplan        # display what plan would do
ceph balancer execute myplan     # run plan, this misplaces the objects
ceph balancer rm myplan

# view auto-balancer status and durations
ceph tell 'mgr.$activemgrid' balancer status
```

Use `upmap` mode: relocate single PGs as "override" to CRUSH.
Needs Ceph Luminous and [Linux kernel 4.13](#Kernel feature list).
```
# If you have Luminous and kernel >=4.13 it may still complain about a too old client,
# but we know what we're doing :)
ceph osd set-require-min-compat-client luminous --yes-i-really-mean-it
```

To further optimize placement and really adjust the equal PG-count to be within a bound of 1.
```
ceph config set mgr mgr/balancer/upmap_max_deviation 1
```

### Erasure Coding

[RAID6](https://docs.ceph.com/en/latest/rados/operations/erasure-code/) with Ceph.

Create a new profile, `standard_8_2` is the arbitrary name.
```
ceph osd erasure-code-profile set standard_8_2 k=8 m=2 crush-failure-domain=osd
```

```
# show the how pools are configured, expecially which ec-profile was assigned to a pool
ceph osd pool ls detail --format json | jq -C .
```

`crush-failure-domain` ensures that no two chunks of an object are stored on the same `osd`.



### Pools

```
## For CephFS:

# erasure coding pool
ceph osd pool create lol_data 32 32 erasure standard_8_2
ceph osd pool set lol_data allow_ec_overwrites true

# replicated pools
ceph osd pool create lol_root 32 replicated
ceph osd pool create lol_metadata 32 replicated

# min_size: minimal osd count (per PG) before a PG goes offline
ceph osd pool set lol_root size 3
ceph osd pool set lol_root min_size 2
ceph osd pool set lol_metadata size 3
ceph osd pool set lol_metadata min_size 2
```

```
# in the same CephFS, there can be multiple storage policies
# for example, this 8+3 pool can be to store some directories 'more safe'
ceph osd erasure-code-profile set backup_8_3 k=8 m=3 crush-failure-domain=osd
ceph osd pool create lol_backup 64 64 erasure backup_8_3
ceph osd pool set lol_backup allow_ec_overwrites true
```

Pool quotas
```
# set max storage bytes to 1TiB (uses shell-calculation)
ceph osd pool set-quota funny_pool_name max_bytes $((1 * 1024 ** 4))

# limit number of objects
ceph osd pool set-quota funny_pool_name max_objects 1000000
```

#### Placement group autoscaling

`nautilus` support automatic creation and pruing of placement groups.

```
ceph mgr module enable pg_autoscaler

# view autoscale information and what the autoscaler would do
ceph osd pool autoscale-status

# policy for newly created pools
# I recommend setting warn, and _not_ on.
ceph config set global osd_pool_default_pg_autoscale_mode <mode>

# policy per-pool
# warn, on or off. recommended by me: warn.
ceph osd pool set $pool pg_autoscale_mode <mode>
```

```
# help the autoscaler by providing target_ratio:
# the fraction of total cluster size this pool is expected to consume.
ceph osd pool set foo target_size_ratio .2

# only warn that the pg-count is suboptimal
ceph osd pool set $poolname pg_autoscale_mode warn

# enable automatic pg adjustments on the given pool
ceph osd pool set $poolname pg_autoscale_mode on
```

### Crushmap

* https://docs.ceph.com/en/latest/rados/operations/crush-map/
* https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/

```
# get and decompile
ceph osd getcrushmap > /tmp/map
crushtool -d /tmp/map -o /tmp/map.txt

# remember the returned crushmap version
# edit /tmp/map.txt

# compile and update crushmap
crushtool -c /tmp/map.txt -o /tmp/map_new
ceph osd setcrushmap -i /tmp/map_new previous_crushmap_version
```

compare crushmaps:
```
crushtool -i crushmap --compare crushmap.new
```

In a rule, device classes can be used to select OSDs for a CRUSH rule.
You can define and assign arbitrary device classes, e.g. use `huge`, `fast`...
`ssd`, `hdd` and `nvme` detected automatically, but any class can be assigned:
```
ceph osd crush set-device-class $deviceclass $osdid
```

To select just OSDs of a given class in a CRUSH rule:
```
step take default
=>
step take default class hdd
```

Assign pools to placement rules.
```
ceph osd pool set <poolname> crush_rule <rulename>
```

CRUSH rule commands:

```
rule rulename {
    # unique id
    id 1
    type replicated
    # which pools can use this rule?
    min_size 1    # -> all pools
    max_size 10

    # now the device selection steps
    # start at the default bucket, but only for ssd devices
    step take default class ssd

    # chooseleaf firstn: recursively explore bucket to look for single devices
    # choose firstn: select bucket for next step
    # 0: choose as many buckets as needed for copies (-1: one less than needed, 3: exactly three)
    # host: bucket type to choose for the next step
    step chooseleaf firstn 0 type host

    # the set of osds was selected
    step emit
}
```

### CephFS

On top of RADOS, Ceph provides a [POSIX filesystem](https://docs.ceph.com/en/latest/cephfs/posix/).


#### CephFS Setup

```
ceph fs new lolfs lol_metadata lol_root
mount -v -t ceph -o name=lolroot,secretfile=lolfs.root.secret 10.0.18.1:6789:/ mnt/
```

newer way:
```
ceph fs volume create ...
```

##### Add Users

Create an user that has full access to `/`
```
ceph fs authorize lolfs client.lolroot / rw
```

You can restict access to a path (this is only for Metadata, i.e. inode infos! use [namespaces](#namespace) to restrict data access).
```
ceph fs authorize lolfs client.some.weird_name /lol/ho_ho_ho rw /stuff r
```

These commands just create regular `auth` keys, you can view its (effective) permissions with:
```
ceph auth get client.some.weird_name
```

###### CephFS Quotas and Layouts

To allow this client to [configure quotas and layouts](https://docs.ceph.com/en/latest/cephfs/client-auth/#layout-and-quota-restriction-the-p-flag), it needs the `p` flag. Beware that this **overwrites** existing caps, so make sure to `auth get` first, and then update.
```
ceph auth caps client.lolroot mon 'allow r' mds 'allow rwp' osd 'allow rw tag cephfs data=cephfsname'
```

###### Snapshot Permissions

To allow a key to manage snapshots, it needs the `s` permission flag for MDS.
```
ceph auth caps client.blabla ... mds 'allow rw path=/some/dir, allow rws path=/some/dir/snapshots_allowed, ...' ...
```

This client can now access `/some/dir` and can manage snapshots in `/some/dir/snapshots_allowed` or any folder below.


##### Subdatapool

New data can be written to a different pool (e.g. an ec-pool).

Any folder can be assigned a new custom pool.
New file content is then written to the new pool (new == file has had size 0 before).
Existing file content will stay in their old pool.
Only when the content is written again from 0 (e.g. copy), the new pool is used.

```
# this automatically sets the application tag for 'cephfs' to data=$cephfsname
# so a client automatically has access!
ceph fs add_data_pool fsname poolname
```

To assign this pool to a folder:

```
setfattr -n ceph.dir.layout.pool -v poolname foldername
```

The client needs access to this pool.
The "simple" way is matching to the cephfs name in the auth key:
Clients will automatically have access to all pools through the `data=cephfsname` tag, if the access key has the `osd 'allow rw tag cephfs data=cephfsname'` capability.
You can set the tag manually (but why would you do that?) with: `ceph osd pool application set <poolname> cephfs <key> <value>`.

Alternatively, you can grant access to pools explicitly: `allow rw pool=poolname, allow r pool=someotherpool, ...`.

Access is better restricted by [namespace](#namespace), though: Namespaces are pool-independent and there can be many namespaces per pool.


##### RBD datapool for client

Some client tools don't support specifying an RBD data pool.

With this "trick", you can set a RBD data pool with erasure coding in OpenStack Cinder, Proxmox, ...

By selecting one `ceph.conf` and a client name, we set the data pool name as default.
If needed, you can have more ceph.conf files, as long as your client tool allows specifying at least the config file name...

```
[global]
fsid = your-fs-id
mon_host = mon1.rofl.lol, mon2.rofl.lol, mon3.rofl.lol

[client.your-awesome-user]
rbd default data pool = your-ec-poolname
```

##### Namespace

OSD access would allow reading (and writing!) CephFS objects directly in the pool, even though the MDS prevents mounts through the path restriction.

To actually restrict access, objects can be prefixed with a namespace so the OSDs can check access through namespace restriction.

Set the namespace on a directory of the CephFS
```
sudo setfattr -n ceph.dir.layout.pool_namespace -v $namespacename /mnt/cephfs/directory/name
```

Change auth caps for a client to only mount `/directory/name` (mds restriction) and read osd data from namespace `$namespacename` on cephfs `lolfs`.
```
ceph auth caps client.somename mds 'allow rw path=/directory/name' mon 'allow r' osd 'allow rw namespace=$namespacename tag cephfs data=lolfs'
```

You can grant access to multiple namespaces to a CephFS client with `allow rw namespace=$ns1, allow r namespace=$ns2, allow rw ....`.

CephFS namespaces are supported on kernel clients since [Linux 4.8](#Kernel feature list) (https://github.com/torvalds/linux/commit/72b5ac54d620b29cae23d25f0405f2765b466f72).


##### Quota Config

To set a [quota](https://docs.ceph.com/en/latest/cephfs/quota/) on a CephFS subdirectory, use:
```
setfattr -n ceph.quota.max_bytes -v 20971520 /a/directory   # 20 MiB
setfattr -n ceph.quota.max_files -v 5000 /another/dir       # 5000 files
```

To remove the quota, set the value to `0`.

CephFS quotas work since [Linux 4.17](https://github.com/torvalds/linux/commit/b284d4d5a6785f8cd07eda2646a95782373cd01e).


##### CephFS Snapshots

A client with the `s` permission for MDS can manage snapshots.
A [snapshot](https://docs.ceph.com/en/latest/dev/cephfs-snapshots/) is created by creating a directory: `dir/to/backup/.snap/snapshot_name`.


#### CephFS Status

```
# show status overview
ceph fs status

# dump all filesystem info
ceph fs dump

# get info for specific fs
ceph fs get lolfs
```

Show connected CephFS clients and their IPs
```
ceph tell mds.$mdsid client ls
```

#### Tuning CephFS

* Enable `fast_read` on pools (see below)
* Enable inline data: Store content for (default <4KB) in inode: `ceph fs set lolfs inline_data yes`

#### MDS Slow ops
```
mds.mds1 [WRN] slow request 30.633018 seconds old, received at 2020-09-12 17:38:03.970677: client_request(client.148012229:9530909 getattr AsLsXsFs #0x100531c49cf
                caller_uid=3860, caller_gid=3860{}) currently failed to rdlock, waiting

rados -p $cephfs-data-pool getxattr 100531c49cf.00000000 parent | ceph-dencoder type inode_backtrace_t import - decode dump_json
```

### RADOS Block Devices RBD

* Create pools: One non-EC pool for metadata first, optionally more data pools
* If a data pool is an EC-Pool, allow `ec_overwrites` on it
  * `ceph osd pool set lol_pool allow_ec_overwrites true`
  * [Linux 4.11](#Kernel feature list) is required if the Kernel should map the RBD
* Prepare all pools for RBD usage: `rbd pool init $pool_name`
* If you have a pool named `rbd`, it's the default (metadata) rbd pool
* You can store rbd data and metadata on separate pools, see the `--data-pool` option below

Now, images can be created in that pool:

* Optionally, create a rbd namespace to restrict access to that image (supported since Nautilus 14.0 and [Kernel 4.19](#Kernel feature list)):
  * `rbd --pool $poolname --namespace $namespacename namespace create`

* **Create** an image: `rbd create --pool $metadata_pool_name --data-pool $storage_pool_name --namespace $namespacename --size 20G $imagename`

* Display namespaces: `rbd --pool $metadata_pool_name namespace ls`
* Display images in a namespace: `rbd --pool $metadata_pool_name --namespace $namespacename ls`

* Create access key for whole pools: `ceph auth get-or-create client.$name mon 'profile rbd' osd 'profile rbd pool=$metadata_pool_name, profile rbd pool=$storage_pool_name, profile rbd-read-only pool=$someotherpoolname'`
* Create access key for a specific rbd namespace: `ceph auth get-or-create client.$name mon 'profile rbd' osd 'profile rbd pool=$metadata_pool_name namespace=$namespacename, profile rbd pool=$storage_pool_name namespace=$namespacename'`
  * Important: restrict the namespace on both the storage and metadata pool! Otherwise this key can read other images' data.

To map an image on a client to `/dev/rbdxxx` (using monitor addresses from `/etc/ceph/ceph.conf`):

* `sudo rbd device map --name client.$name -k keyring [$metadata_pool_name/[$namespacename/]]$imagename[@$snapshotname]`
  * `-t nbd` to mount as nbd-device
  * `--namespace $namespacename` to specify the rbd namespace (as an alternative to the image "path" above)


#### Kernel RBD client

Some Linux kernel `rbd` clients ("krbd") don't support all features of `rbd` images.

Available `rbd` features declared [here](https://github.com/ceph/ceph/blob/master/src/include/rbd/features.h) and listed [here](https://docs.ceph.com/en/latest/man/8/rbd/#cmdoption-rbd-image-feature) and defined [in the kernel code](https://github.com/torvalds/linux/blob/master/drivers/block/rbd.c).

krbd kernel features: [see in the kernel feature list](#Kernel feature list)

```
# if you get this dmesg-output:
rbd: image $imagename: image uses unsupported features: 0x38
# view enabled features of this image:
rbd --pool $meta_data_pool --namespace $namespacename info $imagename
# then disable the unsupported features:
rbd --pool $meta_data_pool --namespace $namespacename feature disable $imagename $unavailable_feature_name $anotherfeature...
```

##### Tuning KRBD

You get the most out of your RBD if you start tweaking some knobs.

Generally: Your client is in control - it has to begin every request to the Ceph cluster
- writes: writes can be parallelized easily (write-cache), even if they occur sequentially from your actual program
- reads: reads can't be parallelized easily, since many programs just read in a sequential fashion (if they read in parallel, it works just like parallel writes)
  - One solution is to "guess" that the program will want to read more data: configure `read_ahead`
  - Another solution is to do **read-caching** with `lvmcache` on a client-local SSD
- have a look at your metrics: `ceph_osd_op_r_process_latency` (and `w`), `ceph_rbd_read_latency` (and `write`), and `ceph_osd_commit_latency_ms`
  - when sequential read ops are requested, you won't get more than `1/ceph_osd_op_r_process_latency in seconds` ops per second (say 15ms -> max 66 iops)
  - given a HDD-pool with read-latency of 10ms average (which is very good), you'll get max 100 random read ops/s from one client.


suggested `/etc/udev/rules.d/80-rbd.rules` for tweaking read and write
```
# parallel requests - this only works if the rbd was mapped with at least queue_depth=512!
ACTION=="add", KERNEL=="rbd*", ATTR{queue/nr_requests}="512"
# read up to 8 ceph objects in advance when the fs on the rbd decides for it
ACTION=="add", KERNEL=="rbd*", ATTR{bdi/read_ahead_kb}="32768"
```

- read tuning: read ahead
  - set the maximum amount of bytes that can be pre-read (i.e. read in parallel!)
  - the actual number is determined on the fly e.g. by the filesystem depending on the read pattern.
  - configurable in: `/sys/block/rbdxxx/queue/read_ahead_kb`
    - to read up to 8 Ceph RBD objects in advance, we have to read `8 * 4096KiB = 32768 KiB`
    - `echo 32768 > /sys/block/rbdxxx/queue/read_ahead_kb`
  - you can't get faster than the OSDs - but `lvmcache` on SSDs and the client's page cache can help drastically with read performance


- write tuning: IO queue optimizations
  - once your filesystem/... is done (it merges tiny-files into bigger blocks, does journaling and metadata), somewhere the raw io requests will be submitted for the block device
  - these requests are then placed in the block device queue, which is handled by your io scheduler (usually mq-deadline)
  - this block device IO operation queue has a depth, configurable in `/sys/block/rbdxxx/queue/nr_requests` and is 256 by default (this number allocates slots for the block io queue). I think this is a good default. If you have lots of small IOPS, increase this to 512.
    - [the Linux documentation](https://docs.kernel.org/admin-guide/abi-stable.html?highlight=block#abi-sys-block-disk-queue-nr-requests) says `"the total allocated number may be twice this amount, since it applies only to reads or writes"`.
      My experiment on Linux 5.4 with mq-deadline RBD has shown `fio` with `rw direct=1 sync=0 iodepth=512` with `nr_requests=256` produces only 256 ceph ops (mixed r and w).
      Setting `nr_requests` to 512 yields 512 ceph ops (mixed r and w)
      Hence, `nr_requests` seems to be the total amount of pending IOPS for the `mq-deadline` Ceph RBD.
  - in the block-io-queue, operations are selected by some algorithm (e.g. deadline) and merged with each other because they are adjacent. The resulting 'optimized' operations are then given to the block driver (rbd in our case)
  - the RBD driver converts the IO requests to Ceph `ops`, i.e. selects which OSDs to send the ops over network
  - the RBD has its own ceph op queue, which is allocated when the device is mapped, set by `queue_depth` (e.g. in `/etc/ceph/rbdmap` or `rbd device map your/namespaced/rbd -o queue_depth=256`.
    It's 128 by default, but I would set it to 256 (since 256 slots are allocated for the io queue above (`nr_requests`). Use 512 if you increased `nr_requests` to 512)
  - the RBD driver then submits the operations, you can see those in tcpdump or `/sys/kernel/debug/ceph/$fsid-$yourclientid/osdc` (get the clientid by `rbd status your/namespaced/rbd`)
  - [since kernel 5.7](#kernel-feature-list) krbd processes the mq-deadline queue with multi-threading (i.e. picking ops from the io-queue and sending/receiving ops over network) for even more speed

- `ext4` optimizations:
  - usually your rbd objects will be 4 MiB (with 64KiB minmal allocations). so `ext4` can respect this and distribute allocations across RADOS objects if you create the fs the following way:
    - `mkfs -E nodiscard,stride=1024,stripe_width=1024 /dev/rbd/your/rbd`
    - why 1024? the rbd has 4096 byte blocks, i.e. `1024 * 4096 = 4MiB` = the object size
    - it makes no sense also considering the EC sharding sizes, since all requests end up at the primary OSD of a PG anyway
  - you can also specify the stripe width when mounting the fs (it will take `stripe_width` by default):
    - `mount -o defaults,noauto,stripe=1024 /dev/rbd/your/rbd /your/mountpoint`
  - to see the 'default' stripe width chosen when you don't specify it as mount option, you can inspect `dumpe2fs /dev/rbd/your/rbd` and look for `RAID stripe width`;


Tests with `fio` on a mounted filesystem on a RBD:

- make sure
  - io scheduler's queue size is 512 (`nr_requests`)
  - you allocated 512 RBD "hardware" queue slots (`queue_depth`)
  - the ext4 uses 1024-block stripe size (`stripe`)
  - open another terminal where you do `watch -n 0.5 cat osdc` to see the `osdc` contents quickly

- non-cached queued writes:
  - since we use `O_DIRECT`, Linux's write cache and io scheduler (`mq-deadline`) queue is bypassed
  - You should see around 512 outstanding ops in the `osdc` file (limited by `nr_requests`, `queue_depth` and the fio `iodepth`)
  - `fio --filename=/mnt/rbd-fs/file --size=20G --direct=1 --sync=0 --iodepth=512 --runtime=15 --ioengine=libaio --time_based --rw=rw --bs=64K --numjobs=1 --group_reporting --name=test`

- cached queued writes
  - we don't use `O_DIRECT` here and thus use Linux's write cache
  - you should see much less ops in `osdc`, but a bit more performance in `fio` since the the cache and io scheduler merges all the ops
  - in `iostat -xm 2` you can see how many ops the io scheduler merges (`wrqm`/`rrqm`)
  - `fio --filename=/mnt/rbd-fs/file --size=20G --direct=0 --sync=0 --iodepth=1024 --runtime=15 --ioengine=libaio --time_based --rw=rw --bs=64K --numjobs=1 --group_reporting --name=test`

- cached but synched writes
  - now we make sure every single operation is synced to disk (`sync=1`), but we still use the io scheduler (`direct=0`).
  - we allow to submit 512 operations from fio in the io scheduler queue, but we sync the `ext4` after each one, forcing the io scheduler queue to flush - thus we only allow 1 op at a time.
  - this is really slow, since it causes `ext4` and the io scheduler to guarantee a transaction for every 64KiB write
  - `fio --filename=/mnt/rbd-fs/file --size=20G --direct=0 --sync=1 --iodepth=1 --runtime=15 --ioengine=libaio --time_based --rw=rw --bs=64K --numjobs=1 --group_reporting --name=test`

- non-cached and synched writes
  - we now bypass the cache (`direct=1`) and submit 512 operations, and sync after each one (`sync=1`)
  - i'm really not sure why, but we see max 512 ops at a time in `osdc` - the cache bypass seems to enable parallel synching somehow?
  - `fio --filename=/mnt/rbd-fs/file --size=20G --direct=1 --sync=1 --iodepth=1 --runtime=15 --ioengine=libaio --time_based --rw=rw --bs=64K --numjobs=1 --group_reporting --name=test`

- non-cached read test
  - we don't use the cache and io scheduler and submit 512 read requests
  - we will see up to 512 parallel reads in `osdc` (`sync` doesn't matter for reads)
  - `fio --filename=/mnt/rbd-fs/file --size=20G --direct=1 --sync=0 --iodepth=512 --runtime=15 --ioengine=libaio --time_based --rw=read --bs=64K --numjobs=1 --group_reporting --name=test`

- cached read test
  - we use the io scheduler and cache and submit 512 read requests
  - interestingly, and I can't explain this, this is much slower than with `direct=1`, and in `osdc` there's a masimum of 3 read ops only?
  - `fio --filename=/mnt/rbd-fs/file --size=20G --direct=0 --sync=0 --iodepth=512 --runtime=15 --ioengine=libaio --time_based --rw=read --bs=64K --numjobs=1 --group_reporting --name=test`

- you can also use `--rw=randrw`/`randread` for random access, since `--rw=rw` tests sequentially


#### Useful RBD commands

https://docs.ceph.com/en/latest/rbd/rados-rbd-cmds/

* When you have a fresh rbd device, use `mkfs.ext4 -E nodiscard` to skip the discard step
* List namespaces of pool: `rbd namespace ls $pool`
* List images in namespace: `rbd ls $pool/$namespace`
* List images without namespace: `rbd ls $pool`
* Show info: `rbd info $pool/$namespace/$image`
* Show clients (IP, id) who have mapped the RBD: `rbd status $pool/$namespace/$image`
* Show pending deleted images: `rbd trash ls [$pool]`
* Size increase: `rbd resize --size 9001T $pool/$img`
* Size decrease: `rbd resize --size 20M $pool/$img --allow-shrink`
* QoS on RBD pool, restrict to 1000 ops: `rbd config pool set $pool rbd_qos_iops_limit 1000`
* Remove QoS on RBD: `rbd config pool rm $pool rbd_qos_iops_limit`
* QoS on RBD image, restrict to 1000 ops: `rbd config image set $pool/$img rbd_qos_iops_limit 1000`
* Remove QoS on RBD images: `rbd config image rm $pool/$img rbd_qos_iops_limit`
* Show performance of RBD pool images: `rbd perf image iotop --pool $pool`


#### Automatic mapping

Automatic RBD mapping with [`rbdmap.service`](https://docs.ceph.com/en/latest/man/8/rbdmap), configured in `/etc/ceph/rbdmap`:

```
$poolname/$namespacename/$imagename name=client.$username,keyring=/etc/ceph/ceph.client.$username.keyring
```

I really recommend putting LVM on the RBD, because then you can decide to do [local caching with `lvmcache`](#) someday.

In `/etc/fstab`, note the `noauto` - we need it because the `rbdmap` tool and not `systemd` shall mount the filesystem.
When you use `ext4`, add option `stripe=1024` ([see above for explaination](#tuning-krbd)).

- either mount the RBD directly
```
/dev/rbd/$metadata_pool_name/$namespacename/$imagename /your/$mountpoint $filesystem defaults,noatime,noauto 0 0
```

- or if you used LVM
```
/dev/yourvg/yourlv /your/$mountpoint $filesystem defaults,noatime,noauto 0 0
```

These entries (due to the `noauto`) won't mount at boot.
This is good, since the RBD is not mapped at that point anyway.


To actually mount when the RBD was mapped with `rbdmap.service`, create a mount script (don't forget to mark it *executable*):

`/etc/ceph/rbd.d/$rbd_metadata_pool/$namespacename/$imagename`
```bash
#!/bin/bash

mountpoint -q /your/mount || mount /your/mount
```


#### Manual mapping

Instead of `rbdmap.service` we can directly use `sysfs` to map:

```
echo $ceph_monitor_ip name=$username,secret=$secret,queue_depth=512 $pool $image > /sys/bus/rbd/add
```


#### RBD Status

Show connected RBD clients and their IPs
```
rbd status $pool/$namespace/$image
```


### Cluster Performance

Each OSD should serve **50 to 150 placement groups in total** (see with `ceph osd df tree`).


`debug_ms` is the messenger log (which logs every damn network message to ram by default).
Set it to 0 in your `ceph.conf`:
```
[global]
debug ms = 0/0
```

To speed up [pool reads](https://docs.ceph.com/en/latest/rados/operations/pools/#fast-read), the primary OSD queries **all** shards (not only the non-parity ones) of the data, and take the fastest reply.
This is useful for reading small files with CephFS, but of course is a trade-off for more network traffic and iops from OSDs.
Since all your disks now reply to read requests - you have to benchmark and try if it helps in your current situation:
Only if disks have more IO capacity, but their latency is quite high, `fast_read` will help.
```
ceph osd pool set cephfs_data_pool fast_read 1
```

Faster recovery speed:
```
osd recovery sleep hdd = 0
osd recovery max active = 5
osd max backfills = 16

# wait some time for data to maybe become available again
osd recovery delay start = 30
```


Get current crush tunables profile:
```
ceph osd crush show-tunables
```

Set it to `optimal`:
```
ceph osd crush tunables optimal
```

Set the amount of memory one OSD should occupy. It will adjust its cache sizes automatically.
```
[osd]
osd_memory_target = 4294967296   # 4GiB
```


### Random infos

* [Network config](https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/)
* [Authentication config](https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/)


### Minimal client config

The minimal config required e.g. for mounting CephFS or mapping a RBD:

```
[global]
fsid = your-fs-id
mon_host = mon1.rofl.lol, mon2.rofl.lol, mon3.rofl.lol, ...
```

Operation
---------

[How to operate the cluster](https://docs.ceph.com/en/latest/rados/operations/)

### Cluster status

Status:
```
ceph -s         # current status
ceph -w         # status change log, maybe even to ceph -w | tee -a /var/log/cephhealth.log
iostat -Pxm 5   # io status, see last column: %util
iotop           # io status
htop            # osd i/o stats per thread
```

https://docs.ceph.com/en/latest/rados/operations/monitoring-osd-pg/


### Utilization

```
# cluster usage
ceph df
ceph df detail

# Before Ceph warns about Pools being nearfull
ceph osd dump | grep nearfull_ratio

# pool usage
rados df
ceph osd pool stats

# osd usage
ceph osd tree
ceph osd df tree
ceph osd df tree name $bucket

# osd commit latency performance
ceph osd perf

# pg status and performance
ceph pg stat
ceph pg ls
ceph pg dump
ceph pg $pgid query

# what pgs are on this osd?
ceph pg ls-by-osd $osdid

# list all pgs where the primary is the given osd
ceph pg ls-by-primary $osdid

# What PGs are remapped?
ceph pg ls remapped
```

### Daemon control and info

You have to issue `ceph daemon` commands on the machine where the daemon is running, since it uses its "admin socket" (asok).
The "remote" variant is to use `ceph tell` instead of `ceph daemon`, which works for most commands.

```
ceph daemon $daemonid help

# performance info
ceph daemon $daemonid perf dump

# config differences
ceph daemon $daemonid config diff

# monitor status
ceph daemon mon.$id mon_status

# daemon performance watch
ceph daemonperf $daemontype.$mdsid

# mds sessions
ceph daemon mds.$id session ls

# view cache usage
ceph daemon mds.$id cache status
```

```
# view configuration differences etc
ceph daemon $daemontype.$id config diff|get|set|show
```

```
# inject any ceph.config option into running daemons
# can also be done with dashboard under "Cluster" -> "Configuration Doc."
ceph tell 'osd.*' injectargs -- --debug_ms=0 --osd_deep_scrub_interval=5356800
ceph tell 'mon.*' injectargs -- --debug_ms=0

# This can be done on modern clusters with ceph config set
# ceph central configs
ceph config dump

# set OSD 0 in debug mode
ceph config set osd.0 debug_osd 20/20
```

```
# show daemon versions to find outdated ones ;)
ceph versions

# show cluster topology, find nodes
ceph node ls {all|osd|mon|mds|mgr}

# show daemon version for concrete hosts
ceph tell 'osd.*' version
ceph tell 'mds.*' version
ceph tell 'mon.*' version
```


### OSD adding and removing

```
# Find the host the OSD is in
ceph osd find $osdid

# See which blockdevices the OSD uses & more
# see links in `/dev/mapper/` and `lsblk` to correlate ids with blockdevices etc
ceph osd metadata [$osdid]

# To migrate data off it:
ceph osd out $osdid

# To let objects be placed on OSD:
ceph osd in $osdid

# Check if device can be removed from cluster
ceph osd ok-to-stop $osdid

# Take it down
sudo systemctl stop ceph-osd@$osdid.service

# OSD id recycling for replacement HDD:
ceph osd destroy $osdid
# -> then create new osd with new hdd and reuse the id  (ceph-volume ... --osd-id $osdid)

# To Remove HDD completely:
# combines osd destroy & osd rm & osd crush rm
ceph osd purge $osdid

# Remove the volume information from lvm
sudo ceph-volume lvm zap vgid/lvid

# Disable the ceph-volume discovery for the removed osd
sudo systemctl disable ceph-volume@lvm-$osdid-....

# If desired, purge the LVM from the device
sudo lvchange -a n vgid/lvid
sudo vgremove vgid
sudo pvremove /dev/device1
```


### Device control

https://docs.ceph.com/en/latest/rados/operations/control/

```
# stop client traffic
ceph osd pause

# start it again
ceph osd unpause

# block a specific client
ceph osd blacklist add $client_addr
```

```
# let an OSD repeer again
# you leave the osd running during this command, it will do peering again.
# if you have stuck PGs it often helps to repeer its primary OSD!
ceph osd down $osdid
```

```
# osd control flags:
# don't recover data: norecover
# don't mark device out if it is down: noout
# don't mark new OSDs as in: noin
# disable scrubbing: noscrub
# disable deepscrubbing: nodeep-scrub

# global flag set:
ceph osd set $flag

# per-daemon flag unset:
ceph osd set-group $flag $daemonname $daemonname...

# global flag unset:
ceph osd unset $flag

# per-daemon flag unset:
ceph osd unset-group $flag $daemonname $daemonname...

# alternative per-daemon commands:
# works for noout, nodown, noup, noin
ceph osd add-$flag 0 1 2 ...
ceph osd rm-$flag 0 1 2 ...
```

```
# same weight-OSDs receive roughly the same amount of objects.
ceph osd reweight $osdid $weight
```

```
# set device class to some identifier (builtin are hdd, ssd or nvme)
ceph osd crush set-device-class nvme osd.$osdid
```


### Device performance

#### Raw device speed

[Collection of benchmarked devices](https://github.com/TheJJ/ceph-diskbench)

To benchmark the raw device write operation speed:

```
# 4k sequential write with sync
fio --filename /dev/device --numjobs=1 --direct=1 --fdatasync=1 --ioengine=pvsync --iodepth=1 --runtime=20 --time_based --rw=write --bs=4k --group_reporting --name=ceph-iops
```

When `ceph-iops` results are shown, look at `write: IOPS=XXXXX`.

* SSDs should have >10k iops
* HDDs should have >100 iops
* Bad SSDs have <200 iops => >5ms latency


#### OSD speed

Before taking an OSD `in`, check its speed! Otherwise it can slow down your whole cluster.

Benchmark a running OSD how many object writes it can do:
```
# let the osd write objects of given size until amount is reached
ceph tell osd.$osdid bench $data_amount $object_size
```

Some examples with "expected" values for "good" performance:
* 256MiB with 128KiB objects: `ceph tell osd.$osdid bench $((256 * 1024**2)) $((128 * 1024**1))`
  * HDD: >300 iops
  * SSD: >1500 iops
* 1GiB with 1MiB objects: `ceph tell osd.$osdid bench $((1 * 1024**3)) $((1 * 1024**2))`
  * HDD: >50 iops
  * SSD: >200 iops


#### RADOS speed

Use `rados bench`!


#### RBD speed

see [KRBD tuning](#tuning-krbd).

```
rbd bench --io-type rw $poolname/$imagename
# other io types: read, write, rw
# --io-pattern seq rand
# --io-size $oneiosize (with B/K/M/G/T suffix)
# --io-total $totalbytecount (with suffix)
# --io-threads $threadcount
```

#### CephFS speed

Benchmark a single file within CephFS, or use Bonnie++.


### PG Autoscale

```
ceph osd pool autoscale-status
```

The PG autoscaler can increase or decrease the number of PGs.

Modes: `on`, `warn`, `off`

I strongly _recommend against_ `on`: I had a funny outage because too many PGs were created and the OSDs then rejected them (solution: increase `mon_max_pg_per_osd`, see [PGs not starting](#pgs-not-starting)).

Default policy for new pools:
```
ceph config set global osd_pool_default_pg_autoscale_mode $mode
```

Change policy for a pool
```
ceph osd pool set $pool pg_autoscale_mode $mode
```

### PGs not starting

It very often helps to **restart or repeer the acting primary** of a problematic PG.
Also, if you do `ceph pg $pgid` query, you can see what peers the PG interacts with (primary, backfill-target, ...). Restarting those can also help.
For example, I got a funny `active+remapped` when several PGs chose an OSD as backfill target. After restarting that, the PGs became active again.

If the PG hangs in `down` or `unknown`, you can figure out their 'last primary' with `ceph pg map $pgid`.

If a PG is stuck `activating`, the involved OSDs may have too many PGs and refuses accepting them:

* Soft limit: `mon_max_pg_per_osd = 250`: You'll see a warning.
* Hard limit: `osd_max_pg_per_osd_hard_ratio = 3`, i.e. `3x250 = 750` PGs per OSD.

At least in versions `<= 14.2.8`, the ONLY the soft-limit will display a warning! When there's PGs **over the hard limit**, NO WARNING is issued (to be fixed).

#### Stuck after adding new OSDs

Newly added OSDs may be the problem. This may be the case when a PG is stuck `activating+remapped`, and in `ceph pg $pgid query` the backfill target is a new OSD. Have a look at `ceph daemon osd.$id status` and observe if its `"num_pgs"` is at the limit. If this is indeed the problem, increase the PG limit and repeer the new OSD.

To lift the limit **temporarily**, tell it all OSDs and MONs:
```
ceph tell 'osd.*' injectargs -- --mon_max_pg_per_osd=2000
ceph tell 'mon.*' injectargs -- --mon_max_pg_per_osd=2000
```
This config variable [basically blocks accepting PGs in OSDs](https://github.com/ceph/ceph/blob/master/src/osd/OSD.cc).

To then revive the stuck pgs, repeer the newly added OSDs with `ceph osd down $osdid`.
If this doesn't fix it, try to repeer primary OSDs of stuck pgs.

Afterwards, make sure to reduce the number of PGs!


### MDS

Online scrub:

```
# flush journal
ceph daemon mds.$id flush journal

# online scrub
ceph daemon mds.$id scrub_path /path/on/fs recursive

# tell ceph that mds $0 has been repaired
ceph mds repaired $cephfs_name:0
```

### Recovery

Data movement/recreation can be done as `backfill` or `recovery`.
* `backfill`: Handles a whole PG
* `recovery`: Handles small parts of a PG (e.g. a few objects)

Of course this utilizes the OSD, so it's slower for your client requests.

```
# speed improvements (or reduction)
# set number of active pg-backfills for one osd
ceph tell 'osd.*' injectargs -- --osd_max_backfills=4

# forced sleeptime in ms between recovery operations
ceph tell 'osd.*' injectargs -- --osd_recovery_sleep_hdd=0
```

### Data Corruption

https://docs.ceph.com/en/latest/rados/troubleshooting/troubleshooting-pg/

```
# check for pg status
ceph pg $pgid query | jq -C . | less
```

To start repair when health is `PG_DAMAGED`, do:
```
ceph health detail
ceph pg repair $pgid  # e.g. 1.2b, from the health detail
```

To enable automatic repair of placement groups, set config option:
```
# automatically try to fix scrub errors on corrupted pgs
osd scrub auto repair = true
```

Control the scrub intervals:
```
[global]  # so the mons can check the intervals!
# every week if load is low enough
osd scrub min interval = 604800
# every two weeks even if the load is too high
osd scrub max interval = 2678400
# deep scrub once every month (60*60*24*31*1)
osd deep scrub interval = 2678400
# time to sleep for group of chunks
# to reduce client latency impact
osd scrub sleep = 0.05
# no scrub while there is recovery (performance)
osd scrub during recovery = false
```


#### OSD crashes

Increase log level to 20 when an OSD crashes:
```
[osd]
debug osd = 5/20
```

When PGs are corrupted (so they prevent an OSD boot), remove the broken ones from an OSD.
There's [a script](https://gist.github.com/TheJJ/c6be62e612ac4782bd0aa279d8c82197) for automatic fixing for selected breakages.


```
# turn off the OSD so we can work on its store directly!

# export a pg from an OSD
# to delete it from OSD: --op export-remove ...
ceph-objectstore-tool --op export --data-path /var/lib/ceph/osd/ceph-$id --pgid $pgid --file $pgid-bup-osd$id

# import into an OSD:
ceph-objectstore-tool --op import --data-path /var/lib/ceph/osd/ceph-$id --file saved-pg-dump

# other useful ops: trim-pg-log

# compact bluestore:
ceph-kvstore-tool <rocksdb|bluestore-kv> /var/lib/ceph/osd/ceph-$id compact
```


#### Incomplete PGs

[`incomplete`](https://docs.ceph.com/en/latest/rados/operations/pg-states/) PG state means Ceph is afraid of starting the PG.

If the `ceph pg $pgid query` for a `incomplete` pg says `les_bound` blocked, the following might help.

```
[...]
    "peering_blocked_by_detail": [
        {
          "detail": "peering_blocked_by_history_les_bound"
        }
[...]
```

Dump all affected pg variants from OSDs (see where they are in `ceph pg dump`) with `objectstore-tool`.
After dumping all pg variants, the largest dump file size is likely the most recent one that should be active.
To proceed, remove other variants from OSDs so the largest pg variant is remaining.

Then, to tell Ceph to just use what's there:

```
# JUST FOR RECOVERY set for the right OSD (or inject the arg)
[osd.1337]
osd find best info ignore history les = true
```

When recovery is done, remove the flag!


### Decrypt OSDs

To migrate a crypted OSD to a noncrypted one, have a second same-size HDD ready.
Alternatively, copy the block data on a free HDD with a real filesystem.
Then play it back from there to the `lv` directly, overwriting the crypted data and the cryptheader.

First, make sure you save the `lvm` tags of the encrypted `lv`. You need them for the new device then.
```
# see tags of encrypted volume:
lvs -o lv_tags /dev/path-to-encrypted-lv
```

Prepare the new disk with a new full-size `pv` and `vg` and `lv`.
Name the `vg` `ceph-$(uuidgen)` and the `lv` `osd-block-$(uuidgen)`.

Copy the data, raw. This does the decryption.
```
dd if=/dev/mapper/encrypteddevice of=/dev/decrypted-lv bs=4096 status=progress
```

Add lvm tags so that ceph-volume can find the new device.

```
# set all these tags for the decrypted volume:
ceph.block_device=/dev/path-to-now-decrypted/vg
ceph.block_uuid=name_of_block_uuid                  # see blikd
ceph.cephx_lockbox_secret=                          # leave empty
ceph.cluster_fsid=cluster_fs_id
ceph.cluster_name=ceph
ceph.crush_device_class=None
ceph.encrypted=0                                    # indicates the device is not crypted
ceph.osd_fsid=stufstuf-stufstuf-stuf-stuf-stuf      # get from crypted device tags
ceph.osd_id=$correctosdid
ceph.type=block
ceph.vdo=0

lvchange --addtag $tag_to_set /dev/path-to-now-decrypted-vg
```

Kernel Feature List
--

Notable Linux kernel feature changes:

* [Linux 5.16](https://github.com/torvalds/linux/commit/0ecca62beb12eeb13965ed602905c8bf53ac93d0) [CephFS default `async dirops`](https://github.com/torvalds/linux/commit/f7a67b463fb83a4b9b11ceaa8ec4950b8fb7f902) [(talk)](https://www.usenix.org/sites/default/files/conference/protected-files/vault20_slides_layton.pdf) (open, unlink, ... speedup) (feature available since Linux 5.7)
* [Linux 5.7](https://github.com/torvalds/linux/commit/fcc95f06403c956e3f50ca4a82db12b66a3078e0) [krbd multi blk-mq](https://github.com/torvalds/linux/commit/f9b6b98d24f7cec5b8269217f9d4fdec1ca43218), [CephFS `async dirops`](https://github.com/torvalds/linux/commit/a25949b99003b7e6c2604a3fc8b8d62385508477) feature (`nowsync` mount flag + Octopus needed)
* [Linux 5.3](https://github.com/torvalds/linux/commit/d9b9c893048e9d308a833619f0866f1f52778cf5) [krbd support for `object-map` and `fast-diff`](https://github.com/torvalds/linux/commit/22e8bd51bb0469d1a524130a057f894ff632376a), [selinux xattr support](https://github.com/torvalds/linux/commit/ac6713ccb5a6d13b59a2e3fda4fb049a2c4e0af2)
* [Linux 5.1](https://github.com/torvalds/linux/commit/2b0a80b0d0bb0a3db74588279bf851b28c6c4705) [krbd support for `deep-flatten`](https://github.com/torvalds/linux/commit/b9f6d447a6f67b2acc3c4a9d9adc2508986e8df9)
* [Linux 4.19](https://github.com/torvalds/linux/commit/0a78ac4b9bb15b2a00dc5a5aba22b0e48834e1ad) [krbd support for `namespace`](https://github.com/torvalds/linux/commit/b26c047b940003295d3896b7f633a66aab95bebd) (needs Nautilus)
* [Linux 4.17](https://github.com/torvalds/linux/commit/b284d4d5a6785f8cd07eda2646a95782373cd01e) [CephFS quotas](https://github.com/torvalds/linux/commit/fb18a57568c2b84cd611e242c0f6fa97b45e4907) and snapshot support (recommended)
* [Linux 4.14](https://github.com/torvalds/linux/commit/cdb897e3279ad1677138d6bdf1cfaf1393718a08) CephFS multiple active MDS ([recommended in docs](https://docs.ceph.com/en/latest/cephfs/kernel-features/?highlight=kernel#multiple-active-metadata-servers))
* [Linux 4.11](https://github.com/torvalds/linux/commit/b2deee2dc06db7cdf99b84346e69bdb9db9baa85) [krbd support for `data-pool` (EC storage)](https://github.com/torvalds/linux/commit/7e97332ea9caad3b7c6d86bc3b982e17eda2f736)
* [Linux 4.9](https://github.com/torvalds/linux/commit/8dfb790b15e779232d5d4e3f0102af2bea21ca55) [krbd support for `exclusive-lock`](https://github.com/torvalds/linux/commit/ed95b21a4b0a71ef89306cdeb427d53cc9cb343f)
* [Linux 4.8](https://github.com/torvalds/linux/commit/72b5ac54d620b29cae23d25f0405f2765b466f72) [CephFS `namespace` support](https://github.com/torvalds/linux/commit/779fe0fb8e1883d5c479ac6bd85fbd237deed1f7)
* [Linux 4.7](https://github.com/torvalds/linux/commit/a10c38a4f385f5d7c173a263ff6bb2d36021b3bb) [CephFS multiple active filesystems](https://github.com/torvalds/linux/commit/235a09821c2bc71d9d07f12217ce2ac00db99eba)
* [Linux 3.19](https://github.com/torvalds/linux/commit/57666509b70030a9483d13222bfec8eec5db07df) [CephFS inline data](https://github.com/torvalds/linux/commit/65a22662bfe1a84d72b9bbd9146b6782b9e53478)
* [Linux 3.10](https://github.com/torvalds/linux/commit/91f8575685e35f3bd021286bc82d26397458f5a9) [krbd support for `striping`](https://github.com/torvalds/linux/commit/5cbf6f12c48121199cc214c93dea98cce719343b) and layering



Tricks
--

### Combine Multiple RBDs

Apart from general [KRBD tuning](#tuning-krbd), you can group together multiple RBDs.
You can use LVM (or MD) to bundle multiple RBDs to one device. But [Since Linux 5.7, krbd does per-cpu queue processing so this method likely no longer helps](#Kernel feature list).

See the configured io sizes with `lsblk --topology`. The larger, the better, as Ceph doesn't like small IO.

### RBD client local cache

You can use `lvmcache` to cache a RBD on e.g. local SSD storage (which should be a md RAID or otherwise secured!).
The cachepool size has no hard requirement, but the more the better.

This is very useful for speeding up HDD-Ceph-Pools because the on-RBD filesystem commit latency is reduced drastically,
but especially read performance can benefit when data is cached and there's no need to fetch it while waiting ~15ms.


### Available Space

* Use Erasure Coding to trade-off latency and speed with usable space.

* Activate the [`ceph balancer`](https://docs.ceph.com/en/latest/rados/operations/balancer/).
  If it doesn't help, try my balancer: https://github.com/TheJJ/ceph-balancer


### Tipps

* Data safety: Always have `min_size` at least +1 more than needed for minimal reachability
  * That means good combinations are, at least:
    * Replica: `n>=3: size=n, min_size=2`
    * Erasure code: `n>=2, m>=2, i>=1: EC=n+m => size=n+m, min_size=n+i`
  * see current values in `ceph osd pool ls detail`
  * Why? Every write should have at least one redundant OSD, even when you're down to `min_size`. Because if another disk dies when you're at `min_size` without a redundant OSD _everything_ is lost. Every write should be backed by at least one additional drive, even if you are already degraded.
* If health is warning, fix it quickly (not just after one week) (enable auto-repair)!
* A single dying (SATA) disk can slow down the whole cluster because of slow ops! Replace them!
  * Use SMART and `ceph osd perf` to find them (and prometheus to see read/write op latencies, and use prometheus node-exporter to see high io-loads)
  * Each operation handled by this disk will have to complete, so if it has write times of 1s, things will slow down considerably.
* Don't set `ceph osd reweight` to values other than 0 and 1 (= `ceph osd out/in`), except when you know what you're doing.
  * The problem is that the bucket (e.g. host) weight is unaffected by the reweight, thus probability of placing a PG on the host on which you just reweighted a OSD remains the same, so other OSDs in the same host will get more PGs than they would get by their size.
  * A value of 0 also leads to this behavior.
  * To artificially shrink devices, use `ceph osd crush reweight` instead
* Activate the **balancer plugin** in `mgr` or use [jj's ceph balancer](https://github.com/TheJJ/ceph-balancer) to optimize storage + load distribution.
* When giving storage to a VM, use [`virtio-scsi`](https://wiki.gentoo.org/wiki/QEMU/Options#Hard_drive) instead of `virtio-blockdevice` and enable discard/unmapping.
