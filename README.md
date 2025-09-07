[Youtube video](https://youtu.be/KojO_wjFvP8?si=YwmSWDbY8E1yVyV2)

# OpenStack and Ceph Storage Guide

This guide provides a comprehensive walkthrough for setting up a Ceph cluster and integrating it with OpenStack. It covers the fundamentals of Ceph, the installation of a 3-node cluster, and demonstrates key features like volume management, snapshots, cloning, and resilience.

## What is Block Storage?

Instead of storing data as files in a hierarchy, block storage works with raw, uniform chunks of data called blocks. The best analogy is this: block storage gives a Virtual Machine a blank hard drive. The VM's operating system can then put its own file system on top, like EXT4, XFS or NTFS, and control it completely. That's what makes block storage so powerful for things like operating systems and databases, where speed and flexibility are essential.

## Ceph Architecture

Before we start building our cluster, let's understand the three main players that make Ceph work.

* **Monitors**: These are the brains of the operation. They maintain the "cluster map", that's the central point where the cluster's state is stored.
* **Object Storage Daemons (OSDs)**: If the monitors are the brains, then the OSDs are the muscle. They do the heavy lifting: storing the data, replicating it, rebalancing it when needed.
* **Managers**: Managers give you a window into the cluster's health and performance. They run the web dashboard, collect metrics, and basically act as the control tower.

---

## Command Summary

### Ceph Cluster Installation

```bash
# Check Ubuntu version
lsb_release -a

# Check Docker version
docker version

# List block devices
lsblk

# Disable swap
swapoff -a

# Make swap off permanent by commenting it out in /etc/fstab
# vi /etc/fstab

# Check memory
free -m

# Install cephadm
apt install cephadm
```

### Cluster Bootstrap

```bash
# Bootstrap the Ceph cluster on the first node
cephadm bootstrap --mon-ip 192.168.80.65 --cluster-network 192.168.80.0/23 --initial-dashboard-user admin --initial-dashboard-password admin123

# List running Docker containers
docker ps

### SSH Configuration

```bash
# Copy SSH key to other nodes
ssh-copy-id -f -i /etc/ceph/ceph.pub ubuntu@192.168.80.68
ssh-copy-id -f -i /etc/ceph/ceph.pub ubuntu@192.168.80.69

# On each of the other nodes, configure root SSH access
sudo mkdir -p /root/.ssh
sudo chmod 700 /root/.ssh
tail -1 .ssh/authorized_keys | sudo tee -a /root/.ssh/authorized_keys
```

### Cluster Management

```bash
# Launch the Ceph command-line environment
cephadm shell

# Check cluster status
ceph -s

# List hosts in the cluster
ceph orch host ls

# Add hosts to the cluster
ceph orch host add ceph2 192.168.80.68
ceph orch host add ceph3 192.168.80.69

# List available devices for OSDs
ceph orch device ls

# Create OSDs on all available devices
ceph orch apply osd --all-available-devices

# Show the OSD tree
ceph osd tree
```

### Create pool and image

```bash
# Create new pool named my-pool
ceph osd pool create my-pool

# List pools
ceph osd pool ls

# Set pool to RBD (RADOS block device)
ceph osd pool application enable my-pool rbd

# Create new 10GB image named my-image
rbd create my-image --size 10240 --pool my-pool

# Show information about my-image
rbd info my-image --pool my-pool
```

### Client Server Setup

```bash
# List block devices on the client
lsblk

# Install Ceph client utilities
sudo apt install ceph-common

# Create/edit Ceph configuration file on the client (copy content from ceph1)
# vi /etc/ceph/ceph.conf

# Create/edit client admin keyring on the client (copy content from ceph1)
# vi /etc/ceph/ceph.client.admin.keyring

# Check cluster status from the client
ceph -s

# List storage pools
ceph osd pool ls

# List block device images in a pool
rbd ls my-pool

# Map an RBD image to a local block device
rbd map my-pool/my-image

# Create a filesystem on the RBD device
mkfs.ext4 /dev/rbd0

# Mount the RBD device
mount /dev/rbd0 /mnt/ceph-volume
```

### Snapshots

```bash
# Create a snapshot
sync; fsfreeze -f /mnt/ceph-volume; rbd snap create my-pool/my-image@my-snapshot; fsfreeze -u /mnt/ceph-volume

# List snapshots for an image
rbd snap ls my-pool/my-image

# Map snapshot as read-only
rbd map my-pool/my-image@my-snapshot --read-only

# Show mapped
rbd showmapped

# Mount snapshot as read-only
mount /dev/rbd1 /mnt/ceph-snap -o ro,noload
```

### Revert snapshot

```bash
# Unmount volume
umount /mnt/ceph-volume

# Unmap volume
rbd unmap /dev/rbd0

# Rollback the image to a snapshot
rbd snap rollback my-pool/my-image@my-snapshot

# Map volume
rbd map my-pool/my-image

# Mount volume again
mount /dev/rbd0 /mnt/ceph-volume/
```

### Cloning

```bash
# Protect a snapshot
rbd snap protect my-pool/my-image@my-snapshot

# Check if snapshot exists
rbd snap ls my-pool/my-image

# Clone a snapshot to a new image
rbd clone my-pool/my-image@my-snapshot my-pool/my-new-image

# List images
rbd ls my-pool

# Map new image
rbd map my-pool/my-new-image

# Show mapped
rbd showmapped

# Mount clone
mount /dev/rbd2 /mnt/ceph-clone

```

### Resilience Testing

```bash
dd if=/dev/zero bs=1M count=1000 | pv -q -L 1m | dd of=/mnt/ceph-volume/slowfile

watch -n1 ls -lh slowfile

# Watch the Ceph cluster health
watch -n1 ceph -s

# Watch the OSD tree
watch -n1 ceph osd tree

# Stop an OSD
cephadm unit --name osd.1 stop

# Mark an OSD as out
ceph osd out 1

# Stop an OSD
cephadm unit --name osd.2 stop

# Mark an OSD as out
ceph osd out 2

# Stop an OSD
cephadm unit --name osd.4 stop

# Mark an OSD as out
ceph osd out 4

# Start a OSD and bring them in
cephadm unit --name osd.4 start
cephadm unit --name osd.1 start
cephadm unit --name osd.2 start
ceph osd in 4
ceph osd in 1
ceph osd in 2
```

# Check the placement group status

```bash
ceph pg stat
```
