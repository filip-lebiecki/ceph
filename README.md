[Youtube video part1](https://youtu.be/KojO_wjFvP8?si=YwmSWDbY8E1yVyV2)
[Youtube video part2](https://youtu.be/)

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

# OpenStack Nova and Glance

This document provides a step-by-step guide with commands to explore OpenStack services like Glance, Nova, and Cinder, and demonstrates how to integrate them with a Ceph storage backend.
This section covers the basics of creating images with Glance and launching virtual machine instances with Nova using the local file system as the backend.

```bash
# Create a new image in Glance named "cirros" from a local qcow2 file.
# --disk-format: The format of the disk image file.
# --container-format: The container format (usually 'bare' for VM images).
# --public: Makes the image accessible to all projects.
# --file: The path to the image file to be uploaded.
openstack image create --disk-format qcow2 --container-format bare --public --file cirros-0.6.3-x86_64-disk.img cirros

# Find the running Docker container for the Glance API service.
docker ps | grep glance

# Inspect the Glance container to find its volume mounts.
# This helps locate where Glance stores its images on the host filesystem.
# Replace '59e93da85208' with your actual Glance container ID.
docker inspect 59e93da85208 | jq '.[0].Mounts'

# List the contents of the Glance images directory on the host machine.
# This confirms that the image file was uploaded and stored by Glance.
sudo ls -lh /var/lib/docker/volumes/glance/_data/images

# Verify the file type of the stored image to ensure it's a QCOW2 image.
# Replace the UUID with the actual filename from the previous command's output.
sudo file /var/lib/docker/volumes/glance/_data/images/f61e6a77-645d-49fb-bae9-ddcbe4f19d7b

# Launch a new virtual machine (server) instance named "demo1".
# --flavor: Specifies the size of the instance (CPU, RAM, disk).
# --image: The Glance image to boot from.
# --network: The network to attach the instance to.
openstack server create --flavor m1.tiny --image cirros --network demo-net demo1

# Display the details of the "m1.tiny" flavor.
openstack flavor show m1.tiny

# Find the running Docker containers for the Nova API and Scheduler services.
docker ps | grep nova_api
docker ps | grep nova_scheduler

# Display the directory tree for Glance images and Nova instances on the host.
# This shows where the base images and instance-specific files are stored.
sudo tree /var/lib/docker/volumes/glance/_data/images/
sudo tree /var/lib/docker/volumes/nova_compute/_data/instances

# Check the file type of the instance's disk.
# When using a qcow2 image, Nova creates a new qcow2 file for the instance
# that uses the original Glance image as its backing file (copy-on-write).
# Replace the UUID with your instance's UUID.
sudo file /var/lib/docker/volumes/nova_compute/_data/instances/47c6690d-a98a-4a66-abf5-3444ac8d7f0a/disk

# Delete the "demo1" server instance.
openstack server delete demo1

# Re-check the instances directory to confirm the instance's files have been removed.
sudo tree /var/lib/docker/volumes/nova_compute/_data/instances/
```

# OpenStack Cinder

This section explores Cinder for persistent block storage. Unlike an instance's ephemeral disk, a Cinder volume persists even after the instance it's attached to is deleted.

```bash
# Create a new 10 GB Cinder volume named "my-second-disk".
openstack volume create --size 10 my-second-disk

# List all Cinder volumes to see the newly created one.
openstack volume list

# Create a new server instance. We will attach our persistent volume to it.
openstack server create --flavor m1.tiny --network demo-net --image cirros demo1

# List servers to confirm the instance is running.
openstack server list

# Attach the persistent volume "my-second-disk" to the "demo1" instance.
openstack server add volume demo1 my-second-disk

# Create a floating IP from the "public1" pool to allow external access.
openstack floating ip create public1

# Assign the floating IP to the "demo1" instance.
# Replace '192.168.12.101' with the IP generated by the previous command.
openstack server add floating ip demo1 192.168.12.101

# SSH into the instance using its floating IP.
ssh cirros@192.168.12.101

# Inside the VM, list the block devices. You should see /dev/vdb, which is our attached volume.
lsblk

# Format the attached volume with an ext4 filesystem.
sudo mkfs.ext4 /dev/vdb

# Create a directory to use as a mount point.
sudo mkdir /data

# Check the disk usage to confirm the volume is mounted.
df -h

# Create a test file on the mounted volume to verify persistence.
cd /data
sudo vi test.txt # Add some text and save

# Exit the SSH session.
exit

# Delete the server instance. The Cinder volume will NOT be deleted.
openstack server delete demo1

# Confirm the server is gone.
openstack server list

# Confirm the Cinder volume still exists.
openstack volume list

# Create a second, new server instance.
openstack server create --flavor m1.tiny --network demo-net --image cirros demo2

# Attach the SAME persistent volume to the new instance "demo2".
openstack server add volume demo2 my-second-disk

# Assign the floating IP to the new server.
openstack server add floating ip demo2 192.168.12.101

# Since the IP is now associated with a new host, remove the old SSH key from your local known_hosts file.
ssh-keygen -f '/home/ubuntu/.ssh/known_hosts' -R '192.168.12.101'

# SSH into the new instance.
ssh cirros@192.168.12.101

# Mount the volume again inside the new VM.
sudo mkdir /data
sudo mount /dev/vdb /data

# Check the content of the test file. The data should still be there!
sudo cat /data/test.txt

# Exit the SSH session.
exit
```

# OpenStack Cinder Under the Hood

This shows how the default Cinder backend (LVM) maps to Volume Groups (VGs) and Logical Volumes (LVs) on the host machine.

```bash
# On the host, list the LVM Volume Groups. You should see one named 'cinder-volumes'.
sudo vgs

# List the LVM Logical Volumes. You'll see LVs corresponding to the Cinder volumes you created.
sudo lvs

# Compare the output with the list of volumes in OpenStack.
openstack volume list
```

# OpenStack Volumes from Images

You can create a bootable volume directly from a Glance image. This allows you to boot an instance from persistent storage.

```bash
# Create a new 10 GB bootable volume named "boot-volume" using the "cirros" image.
openstack volume create --image cirros --size 10 boot-volume

# List volumes to see the new bootable volume.
openstack volume list

# Create a new server instance, booting it directly from the Cinder volume.
# Note the use of `--volume` instead of `--image`.
openstack server create --flavor m1.tiny --volume boot-volume --network demo-net demo3

# List servers to confirm the instance is running.
openstack server list
```

# OpenStack Snapshots and Clones

You can snapshot a volume and then create new volumes (clones) from that snapshot.

```bash
# Clean up the previous instance.
openstack server delete demo3

# The boot volume still exists.
openstack volume list

# Create a snapshot of the "boot-volume".
openstack volume snapshot create --volume boot-volume snap1

# List all volume snapshots.
openstack volume snapshot list

# Create a new volume, "boot-volume-clone", from the snapshot.
openstack volume create --snapshot snap1 --size 10 boot-volume-clone

# List volumes to see the original and the new clone.
openstack volume list

# Boot a new instance from the cloned volume.
openstack server create --flavor m1.tiny --volume boot-volume-clone --network demo-net demo4
```

# Ceph Configuration for OpenStack

This section details the steps to configure Ceph as a storage backend for Glance, Cinder, and Nova.

```bash
# Enter the Ceph administrative shell.
sudo cephadm shell

# Create Ceph pools to store Cinder volumes, Glance images, Nova ephemeral disks, and Cinder backups.
ceph osd pool create volumes
ceph osd pool create images
ceph osd pool create vms
ceph osd pool create backups

# Enable the 'rbd' (RADOS Block Device) application on each pool.
ceph osd pool application enable volumes rbd
ceph osd pool application enable images rbd
ceph osd pool application enable vms rbd
ceph osd pool application enable backups rbd

# Initialize the pools for use with RBD.
rbd pool init volumes
rbd pool init images
rbd pool init backups
rbd pool init vms

# Create a Ceph user ('client.glance') for Glance with appropriate permissions
# and output its keyring to a file.
ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' -o ceph.client.glance.keyring

# Create a Ceph user ('client.cinder') for Cinder/Nova with appropriate permissions
# and output its keyring to a file.
ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=backups, profile rbd pool=vms, profile rbd pool=images' -o ceph.client.cinder.keyring
```

# OpenStack Configuration for Ceph

This section details how to configure OpenStack (deployed via Kolla-Ansible) to use the prepared Ceph cluster.

```bash
# Edit the Kolla-Ansible global configuration file.
# On the OpenStack deployment node:
vi /etc/kolla/globals.yml

# In globals.yml, set the following to enable Ceph backends:
enable_cinder: "yes"
cinder_backend_ceph: "yes"
enable_cinder_backup: "yes"
cinder_backup_driver: "ceph"
ceph_cinder_backup_user: "cinder"
glance_backend_ceph: "yes"
nova_backend_ceph: "yes"
ceph_nova_user: "cinder"

# Create directories to hold Ceph configuration files.
# Kolla-Ansible will mount these directories into the service containers.
mkdir -p /etc/kolla/config/glance
mkdir -p /etc/kolla/config/nova
mkdir -p /etc/kolla/config/cinder/cinder-backup
mkdir -p /etc/kolla/config/cinder/cinder-volume

# From the Ceph cluster, copy the contents of /etc/ceph/ceph.conf
# into /etc/kolla/config/glance/ceph.conf on the OpenStack node.
# Then, copy this file for the other services.
cp /etc/kolla/config/glance/ceph.conf /etc/kolla/config/nova/
cp /etc/kolla/config/glance/ceph.conf /etc/kolla/config/cinder/cinder-volume
cp /etc/kolla/config/glance/ceph.conf /etc/kolla/config/cinder/cinder-backup

# On the Ceph node, get the key for the glance user.
ceph auth get client.glance

# On the OpenStack node, paste the key into the glance keyring file.
vi /etc/kolla/config/glance/ceph.client.glance.keyring

# On the Ceph node, get the key for the cinder user.
ceph auth get client.cinder

# On the OpenStack node, paste the key into the Cinder keyring file
# and copy it for the other services that use it.
vi /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring
cp /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-backup/
cp /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring /etc/kolla/config/nova/

# Update the Glance API configuration to advertise multiple storage locations.
vi /etc/kolla/config/glance/glance-api.conf
[DEFAULT]
show_multiple_locations = true

# View the final configuration directory structure.
tree /etc/kolla/config/

# Apply the new configuration to the OpenStack services.
kolla-ansible reconfigure -i all-in-one

# Verify that the Cinder services are running and aware of the Ceph backend.
openstack volume service list

# Shell into the nova_compute container to verify the Ceph config files are mounted correctly.
docker exec -it nova_compute /bin/bash
ls -l /etc/ceph
exit
```

# Testing the Ceph Backend

Now we test the full integration by creating images, volumes, and instances.

```bash
# Download a new Cirros image.
wget https://download.cirros-cloud.net/0.6.3/cirros-0.6.3-x86_64-disk.img

# Ceph works best with RAW images. Convert the qcow2 image to raw format.
qemu-img convert -f qcow2 -O raw cirros-0.6.3-x86_64-disk.img cirros.raw
file cirros.raw

# Upload the RAW image to Glance.
openstack image create cirros --file cirros.raw --disk-format raw --container-format bare --public

# List images in OpenStack.
openstack image list

# On the Ceph node, list the RBDs in the 'images' pool. You'll see an object for the new image.
rbd ls images

# Create a bootable volume from the Ceph-backed Glance image.
openstack volume create --image cirros --size 10 boot-volume

# List volumes in OpenStack.
openstack volume list

# On the Ceph node, list the RBDs in the 'volumes' pool. You'll see the new volume.
rbd ls volumes

# Get information about the RBD volume. Note that it has a parent, which is the image it was cloned from.
# Replace the UUID with your volume's UUID.
rbd info volumes/volume-14173e19-b6f5-4eb8-b830-fef17ad9ee25

# Boot an instance from the Ceph-backed Cinder volume.
openstack server create --flavor m1.tiny --network demo-net --volume boot-volume demo1

# List servers in OpenStack.
openstack server list

# On Ceph, check the status of the volume. You will see a "watcher," which is the VM using the volume.
rbd status volumes/volume-14173e19-b6f5-4eb8-b830-fef17ad9ee25

# Create an instance with an ephemeral disk, booting from a Glance image.
openstack server create --flavor m1.tiny --network demo-net --image cirros ephemeral-vm

# On Ceph, list the RBDs in the 'vms' pool. The ephemeral disk for the new instance is stored here.
rbd ls vms
```

# Cinder Backup with Ceph

Ceph can also be used as an efficient backend for Cinder backups.

```bash
# Create a full backup of the 'boot-volume'.
openstack volume backup create --name boot-volume-backup boot-volume --force

# List backups in OpenStack.
openstack volume backup list

# On Ceph, list the RBDs in the 'backups' pool. You will see the backup object.
rbd ls backups

# Get information about the backup object in Ceph.
# Replace with your actual backup object name.
rbd info backups/volume-14173e19-b6f5-4eb8-b830-fef17ad9ee25.backup.d433a255-0a5e-4afd-a27f-0dd0dd2e7053

# Create an incremental backup. This only saves the changes since the last backup.
openstack volume backup create --name boot-volume-backup2 boot-volume --force --incremental

# List the backups again to see both the full and incremental backups.
openstack volume backup list

# On Ceph, list the snapshots for the backup object. You'll see new snapshots
# corresponding to the incremental backup.
rbd snap ls backups/volume-14173e19-b6f5-4eb8-b830-fef17ad9ee25.backup.d433a255-0a5e-4afd-a27f-0dd0dd2e7053
```

