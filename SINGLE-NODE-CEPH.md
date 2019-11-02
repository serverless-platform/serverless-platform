
## If your using a PC That is directly connected to the internet
Firewall all Ports befor u use this else you expose your Cluster Public!


## Creating Mounting a Raid Array of sparseFiles 
We Need 2 Disks for our OSD's https://wiki.ubuntuusers.de/Software-RAID/





```
sudo apt-get install mdadm parted 
sudo mdadm --create /dev/md0 --auto md --level=1 --raid-devices=2 /dev/sde1 /dev/sdf1
```



## Bootstrap the Cluster With Docker

### Monitor Node
SET IP AND NET
```
sudo docker run -d --net=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-e MON_IP=192.168.0.20 \
-e CEPH_PUBLIC_NETWORK=192.168.0.0/24 \
ceph/daemon mon
```

### Object Storage Daemon Nodes
You Need at last 2
#### Ceph Managed OSD Creation 
best From Virtual Block Device that we created via sparsefile Raid 0 Array mount
```
sudo docker run -d --net=host \
--privileged=true \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-v /dev/:/dev/ \
-e OSD_DEVICE=/dev/vdd \
ceph-daemon osd_ceph_disk
```

#### Pre Populated OSD Mounts from host (Only if you dont used Ceph Managed OSD Creation)
https://ceph.io/geen-categorie/create-a-partition-and-make-it-an-osd/
https://ceph.com/geen-categorie/bootstrap-your-ceph-cluster-in-docker/




### Reconfigure Ceph to run on a Single Node with 2+ OSD's

On the Admin Node
```
# Save Crush Map
ceph osd getcrushmap -o crush_map_compressed
crushtool -d crush_map_compressed -o crush_map_decompressed
```

Replace in crush_map_decompressed
```
step chooseleaf firstn 0 type host
# with
step chooseleaf firstn 0 type osd
```

Save upload changed crush_map
```
crushtool -c crush_map_decompressed -o new_crush_map_compressed
ceph osd setcrushmap -i new_crush_map_compresse
```

Check the result with ceph -s

```
ceph@ceph-admin:~/os-cluster$ ceph -s
cluster 15ac0bfc-9c48-4992-a2f6-b710d8f03ff4
health HEALTH_OK
monmap e1: 1 mons at {Storage-01=192.168.0.30:6789/0}
election epoch 9, quorum 0 Storage-01
osdmap e105: 6 osds: 6 up, 6 in
flags sortbitwise
pgmap v426885: 624 pgs, 11 pools, 77116 MB data, 12211 objects
150 GB used, 22164 GB / 22315 GB avail
624 active+clean
client io 0 B/s rd, 889 B/s wr, 0 op/s rd, 0 op/s wr
```
