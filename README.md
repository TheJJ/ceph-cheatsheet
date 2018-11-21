Ceph Cheatsheet
===============

(c) 2018 Jonas Jelten <jj@stusta.net>
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

### Manager Setup
[Manager config documentation](http://docs.ceph.com/docs/master/mgr/administrator)

Run one manager for each monitor.

This creates a access key for cephx for the manager.
```
sudo -u ceph mkdir -p /var/lib/ceph/mgr/ceph-$mgrid
sudo ceph auth get-or-create mgr.$mgrid mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-$mgrid/keyring

# test it:
sudo -u ceph ceph-mgr -i eichhorn -d

# actually activate it
sudo systemctl enable --now ceph-mgr@$mgrid.service
```

The manager provides a [shiny dashboard](http://docs.ceph.com/docs/master/mgr/dashboard).


### Storage

[How to add devices](http://docs.ceph.com/docs/master/ceph-volume/lvm/prepare/).

Before adding OSDs, you can do `ceph osd set noin` so new disks are not filled automatically.
Once you want it to be filled, do `ceph osd in $osdid`.

#### Add BlueStore OSD

Add a [BlueStore](http://docs.ceph.com/docs/master/rados/configuration/bluestore-config-ref/) device, with the help of LVM. In the LVM tags, metainfo is stored.

The data and journal (WAL) and keyvalue-DB can be placed on different devices (HDD and SSD).

Use `--dmcrypt` to encrypt the HDD. This just uses [LUKS](https://en.wikipedia.org/wiki/LUKS)!

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


### Metadata Setup

Metadata servers are needed for the CephFS.

```
sudo -u ceph mkdir -p /var/lib/ceph/mds/ceph-$mdsid
sudo -u ceph ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-$mdsid/keyring --gen-key -n mds.$mdsid
sudo -u ceph auth add mds.$mdsid osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-$mdsid/keyring
```


### Erasure Coding

[RAID6](http://docs.ceph.com/docs/master/rados/operations/erasure-code/) with Ceph.

```
sudo ceph osd erasure-code-profile set standard_8_2 k=8 m=2 crush-failure-domain=osd
```

`crush-failure-domain` ensures that no two chunks of an object are stored on the same `osd`.



### Pools

```
ceph osd pool create lol_data 32 32 erasure standard_8_2
ceph osd pool set lol_data allow_ec_overwrites true

ceph osd pool create lol_root 32
ceph osd pool create lol_metadata 32

ceph osd pool set lol_root size 3
ceph osd pool set lol_root min_size 2
ceph osd pool set lol_metadata size 3
ceph osd pool set lol_metadata min_size 2
```

```
ceph osd erasure-code-profile set backup_7_3 k=7 m=3 crush-failure-domain=osd
ceph osd pool create lol_backup 64 64 erasure standard_8_2
ceph osd pool set lol_backup allow_ec_overwrites true
```

### Crushmap

http://docs.ceph.com/docs/master/rados/operations/crush-map-edits/

```
ceph osd getcrushmap > /tmp/map
crushtool -d /tmp/map -o /tmp/map.txt
# edit
crushtool -c /tmp/map.txt -o /tmp/map_new

ceph osd setcrushmap -i /tmp/map_new
```

In a rule, assign hdds only like this:
```
step take default
=>
step take default class hdd
```

Assign pools to placement rules.
```
ceph osd pool set <poolname> crush_rule <rulename>
```

### CephFS

```
ceph fs new lolfs lol_metadata lol_root
mount -v -t ceph -o name=lolroot,secretfile=lolfs.root.secret 10.0.18.1:6789:/ mnt/
```

#### Add Users

Create an user that has full access to `/`
```
ceph fs authorize lolfs client.lolroot / rw
```

To allow this client to [configure quotas and layouts](http://docs.ceph.com/docs/master/cephfs/client-auth/#layout-and-quota-restriction-the-p-fla):
```
ceph auth caps client.lolroot mon 'allow r' mds 'allow rwp' osd 'allow rw tag cephfs data=cephfsname'
```

#### Subdatapool

In the root of the cephfs, create `foldername` folder.
Now assign the `poolname` pool to this folder.

```
ceph fs add_data_pool fsname poolname
```

```
setfattr -n ceph.dir.layout.pool -v poolname foldername
```

The client needs access to this pool. Either via `allow rw tag cephfs data=lolfs` or explicit `allow rw pool=poolname`.
The tagging and data is done with: `ceph osd pool application set <poolname> cephfs <key> <value>`.


#### Namespace

Objects can be prefixed with a namespace.
Access rights can be restricted to a namespace.

-> CephFS directory contents protected by namespace.



### Tipps

* Per `osd` there should be 100 placement groups in total (see with `ceph osd df tree`).
* `min_size` at least +1 than really required (`ceph osd pool ls detail`)
* if health is warning, fix quickly (not just after one week)!
* Never use `ceph osd crush reweight`, only use `ceph osd reweight`
* Never use `reweight-by-utilization`, instead use **balancer plugin** in `mgr`
* When giving storage to a VM, use [`virtio-scsi`](https://wiki.gentoo.org/wiki/QEMU/Options#Hard_drive) instead of `virtio-blockdevice` and enable discard/unmapping.


### Performance

`debug_ms` is the messenger log (which logs every damn message to ram).
Set it to 0.

Recovery speed settings:
`osd_recovery_sleep`, `osd_recovery_max_active` und `osd_max_backfills`.


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
ceph pg dump

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


### OSD adding and removing

```
# Find the host the OSD is in
ceph osd find $osdid

# See which blockdevices the OSD uses & more
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
# -> then create new osd with same id  (ceph-volume ... --osd-id)

# To Remove HDD completely:
# combines osd destroy & osd rm & osd crush rm
ceph osd purge $osdid

# Remove the volume information from lvm
sudo ceph-volume lvm zap vgid/lvid

# Disable the ceph-volume discovery for the removed osd
sudo systemctl disable ceph-volume@lvm-$osdid-....
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
# don't recover
ceph osd set norecover
# don't move data once device is out
ceph osd set noout
# disable scrubbing
ceph osd set noscrub
# disable deepscrubbing
ceph osd set nodeep-scrub

# to unset, use unset $flag
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
debug osd = 1/20
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
