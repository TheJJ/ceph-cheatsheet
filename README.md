Ceph Cheatsheet
===============

(c) 2018-2019 Jonas Jelten <jj@stusta.net>

Released under GPLv3 or any later version.

If you ever need professional Ceph support, try https://croit.io!


What?
-----

[Ceph](https://ceph.com/) is a distributed storage cluster.

[Debugging](http://docs.ceph.com/docs/master/rados/troubleshooting/log-and-debug/).

[Other Cheatsheet](https://sabaini.at/pages/ceph-cheatsheet.html).


Components
----------

Component | Description
----------|------------
Client | Something that connects to the Ceph cluster to access data.
[Cluster](https://wiki.gentoo.org/wiki/Ceph/Cluster) | All Ceph components as a whole, providing storage
[Object Store Device (OSD)](https://wiki.gentoo.org/wiki/Ceph/Object_Store_Device) | Actually stores data on disks
[Monitor (MON)](https://wiki.gentoo.org/wiki/Ceph/Monitor) | Coordinates the cluster (odd number, >=1)
[Metadata Server (MDS)](https://wiki.gentoo.org/wiki/Ceph/Metadata_Server) | Handles inode transactions, stored on OSDs
Public Network | Network accessed by Ceph clients
Cluster Network | Network by which Ceph OSD nodes sync data
Pool | Logical partition to store objects, similar to a LVM LV
[Placement Group](http://docs.ceph.com/docs/hammer/rados/operations/placement-groups/) | Group of objects within a pool


Setup
-----

What [hardware?](http://docs.ceph.com/docs/master/start/hardware-recommendations/)

Ceph needs an odd number of >=1 MONs to [get a quorum](https://en.wikipedia.org/wiki/Paxos_(computer_science)).

For better understanding the setup, I recommend the [manual method](http://docs.ceph.com/docs/master/install/manual-deployment/).

For config options, see the [upstream documentation](http://docs.ceph.com/docs/master/rados/configuration/ceph-conf/).


### Monitor Setup
[Monitor config documentaion](http://docs.ceph.com/docs/master/rados/configuration/mon-config-ref/)

Follow the 'manual method' above to add a `ceph-$monid` monitor, where `$monid` usually is a letter from `a-z`,
but we use creative names (the host name).

Monitors don't use `ceph.conf` for their addressing, they store it internally.
[IP changing guide](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/#changing-a-monitor-s-ip-address).

For more hosts, [monitors can be added](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/).


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
[Manager config documentation](http://docs.ceph.com/docs/master/mgr/administrator)

Run one manager for each monitor.
Offloads work from MONs and allows scaling beyond 1000 OSDs (statistics and other unimportant stuff like disk usage)
One MGR is active, all others are on standby.

This creates a access key for cephx for the manager.
```
sudo -u ceph mkdir -p /var/lib/ceph/mgr/ceph-$mgrid
ceph auth get-or-create mgr.$mgrid mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-$mgrid/keyring

# test it:
sudo -u ceph ceph-mgr -i eichhorn -d

# actually activate it
sudo systemctl enable --now ceph-mgr@$mgrid.service
```

The manager provides a [shiny dashboard](http://docs.ceph.com/docs/master/mgr/dashboard) and other plugins (e.g. the [balancer](http://docs.ceph.com/docs/master/mgr/balancer/))


### Storage

[How to add devices](http://docs.ceph.com/docs/master/ceph-volume/lvm/prepare/).

Before adding OSDs, you can do `ceph osd set noin` so new disks are not filled automatically.
Once you want it to be filled, do `ceph osd in $osdid`.

#### Add BlueStore OSD

Add a [BlueStore](http://docs.ceph.com/docs/master/rados/configuration/bluestore-config-ref/) device, with the help of LVM. In the LVM tags, metainfo is stored.

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

[Adding/removing OSDs](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-osds/).

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

With [`ceph-bluestore-tool`](https://docs.ceph.com/docs/master/man/8/ceph-bluestore-tool/), you can create, migrate expand and merge OSD block devices.

* To move the block.db from an all-in-one OSD to a separate device, you need to overwrite the
  default size of `1G` if your new `block.db` should be bigger than that (example with a new `2G` DB):

```
CEPH_ARGS="--bluestore-block-db-size 2147483648" ceph-bluestore-tool --path /var/lib/ceph/osd/ceph-42 bluefs-bdev-new-db --dev-target /dev/newfastdevice
```

* To resize a `block.db`, use `bluefs-bdev-expand` (e.g. when the underlying partition size was increased)
* To merge separate `block.db` or `block.wal` drives onto the slow disk, use `bdev-migrate`
* Details for the commands are in the [manpage](https://docs.ceph.com/docs/master/man/8/ceph-bluestore-tool/)


### Metadata Setup

Metadata servers (MDS) are needed for the CephFS.

```
sudo -u ceph mkdir -p /var/lib/ceph/mds/ceph-$mdsid
sudo -u ceph ceph ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-$mdsid/keyring --gen-key -n mds.$mdsid
sudo -u ceph ceph auth add mds.$mdsid osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-$mdsid/keyring
```

Multiple MDS servers can be active, they distribute the inode workload. Kernel clients support this since Linux 4.14.


### Balancer

- `set debug_mgr=4/5`                # for: `tail -f ceph-mgr.*.log | grep balancer`
- `ceph balancer mode upmap`
- `ceph balancer eval`               # evaluate current score
- `ceph balancer optimize myplan`    # create a plan, don't run it yet
- `ceph balancer eval myplan`        # evaluate score after myplan. optimal is 0
- `ceph balancer show myplan`        # display what plan would do
- `ceph balancer execute myplan`     # run plan, this misplaces the objects
- `ceph balancer rm myplan`          # remove the plan

`upmap` mode
```
# to use upmap as balancer mode, the client must be luminous!
# kernel >=4.13 supports this (even though the command complains about a too old client)
ceph osd set-require-min-compat-client luminous --yes-i-really-mean-it
```


### Erasure Coding

[RAID6](http://docs.ceph.com/docs/master/rados/operations/erasure-code/) with Ceph.

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
# for example, this 7+3 pool can be to store some directories 'more safe'
ceph osd erasure-code-profile set backup_7_3 k=7 m=3 crush-failure-domain=osd
ceph osd pool create lol_backup 64 64 erasure backup_7_3
ceph osd pool set lol_backup allow_ec_overwrites true
```

#### Placement group autoscaling

`nautilus` support automatic creation and pruing of placement groups.

```
ceph mgr module enable pg_autoscaler

# view autoscale information and what the autoscaler would do
ceph osd pool autoscale-status
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

* http://docs.ceph.com/docs/master/rados/operations/crush-map/
* http://docs.ceph.com/docs/master/rados/operations/crush-map-edits/

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

On top of RADOS, Ceph provides a [POSIX filesystem](http://docs.ceph.com/docs/master/cephfs/posix/).


#### Setup

```
ceph fs new lolfs lol_metadata lol_root
mount -v -t ceph -o name=lolroot,secretfile=lolfs.root.secret 10.0.18.1:6789:/ mnt/
```

##### Add Users

Create an user that has full access to `/`
```
ceph fs authorize lolfs client.lolroot / rw
```

To allow this client to [configure quotas and layouts](http://docs.ceph.com/docs/master/cephfs/client-auth/#layout-and-quota-restriction-the-p-fla):
```
ceph auth caps client.lolroot mon 'allow r' mds 'allow rwp' osd 'allow rw tag cephfs data=cephfsname'
```

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

CephFS namespaces are supported on kernel clients since [Linux 4.8](https://github.com/torvalds/linux/commit/72b5ac54d620b29cae23d25f0405f2765b466f72).


##### Quota

To set a [quota](https://docs.ceph.com/docs/master/cephfs/quota/) on a CephFS subdirectory, use:
```
setfattr -n ceph.quota.max_bytes -v 20971520 /a/directory   # 20 MiB
setfattr -n ceph.quota.max_files -v 5000 /another/dir       # 5000 files
```

To remove the quota, set the value to `0`.

CephFS quotas work since [Linux 4.17](https://github.com/torvalds/linux/commit/b284d4d5a6785f8cd07eda2646a95782373cd01e).


#### Status

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

#### Tuning

* Enable `fast_read` on pools (see below)
* Enable inline data: Store content for (default <4KB) in inode: `ceph fs set lolfs inline_data yes`


### RADOS Block Devices RBD

* Create pools: One non-EC pool for metadata first, optionally more data pools
* If a data pool is an EC-Pool, allow `ec_overwrites` on it
  * `ceph osd pool set lol_pool allow_ec_overwrites true`
  * [Linux 4.11](https://github.com/torvalds/linux/commit/b2deee2dc06db7cdf99b84346e69bdb9db9baa85) is required if the Kernel should map the RBD
* Prepare all pools for RBD usage: `rbd pool init $pool_name`
* If you have a pool named `rbd`, it's the default (metadata) rbd pool
* You can store rbd data and metadata on separate pools, see the `--data-pool` option below

Now, images can be created in that pool:

* Optionally, create a rbd namespace to restrict access to that image (supported since Nautilus 14.0 and Kernel 4.19):
  * `rbd --pool $poolname --namespace $namespacename namespace create`

* **Create** an image: `rbd create --pool $metadata_pool_name --data-pool $storage_pool_name --namespace $namespacename --size 20G $imagename`

* Display namespaces: `rbd --pool $metadata_pool_name namespace ls`
* Display images in a namespace: `rbd --pool $metadata_pool_name --namespace $namespacename ls`

* Create access key for whole pools: `ceph auth get-or-create client.$name mon 'profile rbd' osd 'profile rbd pool=$metadata_pool_name, profile rbd pool=$storage_pool_name, profile rbd-read-only pool=$someotherpoolname'`
* Create access key for a specific rbd namespace: `ceph auth get-or-create client.$name mon 'profile rbd' osd 'profile rbd pool=$metadata_pool_name namespace=$namespacename, profile rbd pool=$storage_pool_name namespace=$namespacename'`
  * Important: restrict the namespace on both the storage and metadata pool! Otherwise this key can read other images' data.

To map an image on a client to `/dev/rbdxxx` (using monitor addresses from `/etc/ceph/ceph.conf`):

* `sudo rbd map --name client.$name -k keyring [$metadata_pool_name/[$namespacename/]]$imagename[@$snapshotname]`
  * `-t nbd` to mount as nbd-device
  * `--namespace $namespacename` to specify the rbd namespace (as an alternative to the image "path" above)

#### Benchmarking

```
rbd bench --io-type rw $poolname/$imagename
# other io types: read, write, rw
# --io-pattern seq rand
# --io-size $oneiosize (with B/K/M/G/T suffix)
# --io-total $totalbytecount (with suffix)
# --io-threads $threadcount
```

#### Kernel client

Some Linux kernel `rbd` clients ("krbd") don't support all features of `rbd` images.

Available `rbd` features declared [here](https://github.com/ceph/ceph/blob/master/src/include/rbd/features.h) and listed [here](http://docs.ceph.com/docs/master/man/8/rbd/#cmdoption-rbd-image-feature) and defined [in the kernel code](https://github.com/torvalds/linux/blob/master/drivers/block/rbd.c).

Supported krbd features:
* Since [Linux 5.3](https://github.com/torvalds/linux/blob/v5.3/drivers/block/rbd.c#L113) support for `object-map` and `fast-diff`
* Since [Linux 5.1](https://github.com/torvalds/linux/blob/v5.1/drivers/block/rbd.c#L113) support for `deep-flatten`
* Since [Linux 4.11](https://github.com/torvalds/linux/blob/v4.11/drivers/block/rbd.c#L113) support for `data-pool`
* Since [Linux 4.9](https://github.com/torvalds/linux/blob/v4.9/drivers/block/rbd.c#L113) support for `exclusive-lock`
* Since [Linux 3.10](https://github.com/torvalds/linux/blob/v4.9/drivers/block/rbd.c#L113) support for `striping`

```
# if you get this dmesg-output:
rbd: image $imagename: image uses unsupported features: 0x38
# view enabled features of this image:
rbd --pool $meta_data_pool --namespace $namespacename info $imagename
# then disable the unsupported features:
rbd --pool $meta_data_pool --namespace $namespacename feature disable $imagename $unavailable_feature_name $anotherfeature...
```



#### [More stuff](http://docs.ceph.com/docs/master/rbd/rados-rbd-cmds/)

* When you have a fresh rbd device, use `mkfs.ext4 -E nodiscard` to skip the discard step
* List images: `rbd ls $meta_pool_name`
* Show info: `rbd info $pool/$image`
* Show pending deleted images: `rbd trash ls`, optionally append the pool name
* Size increase: `rbd resize --size 9001T $pool/$img`
* Size decrease: `rbd resize --size 20M $pool/$img --allow-shrink`


#### Automatic mapping

Automatic RBD mapping with [`rbdmap.service`](http://docs.ceph.com/docs/master/man/8/rbdmap), configured in `/etc/ceph/rbdmap`:

```
$poolname/$namespacename/$imagename name=client.$username,keyring=/etc/ceph/ceph.client.$username.keyring
```

In `/etc/fstab`, note the `noauto`:
```
/dev/rbd/$metadata_pool_name/$imagename $mountpoint $filesystem defaults,noatime,noauto 0 0
```

(it's [a bug](http://tracker.ceph.com/issues/40247) that images are missing the namespace name in their `/dev` path).


### Tipps

* If health is warning, fix quickly (not just after one week) (enable auto-repair)!
* A single dying (SATA) disk can slow down the whole cluster because of slow ops! Replace them! Use SMART and `ceph osd perf` to find them.
* More safety: `min_size` at least +1 than really required (`ceph osd pool ls detail`)
* Never use `ceph osd crush reweight`, only use `ceph osd reweight`: crush rules are restored again on restart and absolute to device size, osd weight is persistent and from 0 to 1!
* Never use `reweight-by-utilization`, instead use **balancer plugin** in `mgr`
* When giving storage to a VM, use [`virtio-scsi`](https://wiki.gentoo.org/wiki/QEMU/Options#Hard_drive) instead of `virtio-blockdevice` and enable discard/unmapping.


### Performance

Each OSD should server **50 to 150 placement groups in total** (see with `ceph osd df tree`).


`debug_ms` is the messenger log (which logs every damn network message to ram by default).
Set it to 0 in your `ceph.conf`:
```
[global]
debug ms = 0/0
```

To speed up [pool reads](http://docs.ceph.com/docs/master/rados/operations/pools/#fast-read), the primary OSD queries **all** shards (not only the non-parity ones) of the data, and take the fastest reply.
This is useful for reading small files with CephFS, but of course is a tradeoff for more network traffic between OSDs.
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
osd memory target = 4294967296   # 4GiB
```


### Random infos

* [Network config](http://docs.ceph.com/docs/master/rados/configuration/network-config-ref/)
* [Authentication config](http://docs.ceph.com/docs/master/rados/configuration/auth-config-ref/)


Operation
---------

[How to operate the cluster](http://docs.ceph.com/docs/master/rados/operations/)

### Cluster status

Status:
```
ceph -s         # current status
ceph -w         # status change log, maybe even to ceph -w | tee -a /var/log/cephhealth.log
iostat -Pxm 5   # io status, see last column: %util
```

http://docs.ceph.com/docs/master/rados/operations/monitoring-osd-pg/

### Utilization

```
# cluster usage
ceph df

# pool usage
rados df
ceph osd pool stats

# osd usage
ceph osd tree
ceph osd df tree

# pg status and performance
ceph pg stat
ceph pg ls
ceph pg dump
ceph pg $pgid query

# osd performance
ceph osd perf
```

### Daemon control and info

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

Inject configuration options (see options from Dashboard under "Cluster" -> "Configuration Doc.") at runtime.

```
ceph daemon $daemontype.$id config
```

```
# inject any ceph.config option into running daemons
ceph tell 'osd.*' injectargs '--debug_ms 0'
ceph tell 'mon.*' injectargs '--debug_ms 0'
```

```
# show daemon versions to find outdated ones ;)
ceph versions
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

http://docs.ceph.com/docs/master/rados/operations/control/

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
# don't recover
ceph osd set norecover

# don't move data once device is out
ceph osd set noout

# don't mark new OSDs as in
ceph osd set noin

# disable scrubbing
ceph osd set noscrub

# disable deepscrubbing
ceph osd set nodeep-scrub

# to unset, use unset $flag
```

maintenance for single OSDs instead of the whole cluster:
```
ceph osd add-noout 0
ceph osd rm-noout 0

same goes for: nodown, noup, noin
```

```
# same weight-OSDs receive roughly the same amount of objects.
ceph osd reweight $osdid $weight
```

```
# set device class to hdd, ssd or nvme
ceph osd crush set-device-class nvme osd.$osdid
```


### Device performance

[Collection of benchmarked devices](https://github.com/TheJJ/ceph-diskbench)

To benchmark device speed:

```
# 4k sequential write with sync
fio --filename=/dev/device --direct=1 --sync=1 --iodepth=1 --runtime=60 --time_based --rw=write --bs=4k --numjobs=1 --group_reporting --name=ceph-journal-write-test
```

When `ceph-journal-write-test` results are shown, look at `iops=XXXXX`.

* SSDs should have >10k iops
* HDDs should have >10 iops
* Bad SSDs have <100 iops => >10ms latency


### Data Corruption

http://docs.ceph.com/docs/master/rados/troubleshooting/troubleshooting-pg/

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

### Recovery

```
# speed improvements
# more backfills for one osd (increase even more, if neccessary)
ceph tell 'osd.*' injectargs '--osd_max_backfills 16'

# no forced sleeptime for recovery
ceph tell 'osd.*' injectargs '--osd_recovery_sleep_hdd 0'
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
# export a pg from an OSD
# to delete it from OSD: --op export-remove ...
ceph-objectstore-tool --op export --data-path /var/lib/ceph/osd/ceph-$id --pgid $pgid --file $pgid-bup-osd$id

# import into an OSD:
ceph-objectstore-tool --op import --data-path /var/lib/ceph/osd/ceph-$id --file saved-pg-dump
```


#### Incomplete PGs

[`incomplete`](http://docs.ceph.com/docs/master/rados/operations/pg-states/) PG state means Ceph is afraid of starting.

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
# JUST FOR RECOVERY set for the right OSD (or all)
[osd.1337]
osd find best info ignore history les = true
```

To speed up the recovery/backfill:
```
[osd]
osd recovery sleep hdd = 0
```

When recovery is done, remove the flag!


### MDS

Online scrub:

```
# flush journal
ceph daemon mds.$id flush journal

# online scrub
ceph daemon mds.$id scrub_path /path/on/fs recursive

# tell ceph that mds $0 has been repaired
ceph mds repaired cephfs_name:0
```

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
ceph.osd_fsid=asdfasdf-asdfasdf-asdf-asdf-asdf      # get from crypted device tags
ceph.osd_id=$correctosdid
ceph.type=block
ceph.vdo=0

lvchange --addtag $tag_to_set /dev/path-to-now-decrypted-vg
```
