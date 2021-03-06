---
layout: docs
title: OpenStack
category: running_coreos
sub_category: platforms
weight: 5
---

<div class="coreos-docs-banner">
<span class="glyphicon glyphicon-info-sign"></span>This image is now way easier to use! Read about our <a href="{{site.url}}/blog/new-filesystem-btrfs-cloud-config/">new file system layout and cloud-config support</a>.
</div>

# Running CoreOS on OpenStack

CoreOS is currently in heavy development and actively being tested.  These
instructions will walk you through downloading CoreOS for OpenStack, importing
it with the `glance` tool and running your first cluster with the `nova` tool.

## Import the Image

These steps will download the CoreOS image, uncompress it and then import it
into the glance image store.

```
$ wget http://storage.core-os.net/coreos/amd64-usr/alpha/coreos_production_openstack_image.img.bz2
$ bunzip2 coreos_production_openstack_image.img.bz2
$ glance image-create --name CoreOS \
  --container-format bare \
  --disk-format qcow2 \
  --file coreos_production_openstack_image.img \
  --is-public True
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 4742f3c30bd2dcbaf3990ac338bd8e8c     |
| container_format | ovf                                  |
| created_at       | 2013-08-29T22:21:22                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | cdf3874c-c27f-4816-bc8c-046b240e0edd |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | coreos                               |
| owner            | 8e662c811b184482adaa34c89a9c33ae     |
| protected        | False                                |
| size             | 363660800                            |
| status           | active                               |
| updated_at       | 2013-08-29T22:22:04                  |
+------------------+--------------------------------------+
```

## Cloud-Config

CoreOS allows you to configure machine parameters, launch systemd units on startup and more via cloud-config. Jump over to the [docs to learn about the supported features]({{site.url}}/docs/cluster-management/setup/cloudinit-cloud-config). We're going to provide our cloud-config to Openstack via the user-data flag. Our cloud-config will also contain SSH keys that will be used to connect to the instance. In order for this to work your OpenStack cloud provider must be running the OpenStack metadata service.

The most common cloud-config for Openstack looks like:

```
#cloud-config

coreos:
  etcd:
    # generate a new token for each unique cluster from https://discovery.etcd.io/new
    discovery: https://discovery.etcd.io/<token>
    # multi-region and multi-cloud deployments need to use $public_ipv4
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
ssh_authorized_keys:
  # include one or more SSH public keys
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0g+ZTxC7weoIJLUafOgrm+h...
```

## Launch Cluster

Boot the machines with the `nova` CLI, referencing the image ID from the import step above and your `cloud-config.yaml`:

```
nova boot \
--user-data ./cloud-config.yaml \
--image cdf3874c-c27f-4816-bc8c-046b240e0edd \
--key-name coreos \
--flavor m1.medium \
--num-instances 3 \
--security-groups default coreos
```

Your first CoreOS cluster should now be running. The only thing left to do is
find an IP and SSH in.

```
$ nova list
+--------------------------------------+-----------------+--------+------------+-------------+-------------------+
| ID                                   | Name            | Status | Task State | Power State | Networks          |
+--------------------------------------+-----------------+--------+------------+-------------+-------------------+
| a1df1d98-622f-4f3b-adef-cb32f3e2a94d | coreos-a1df1d98 | ACTIVE | None       | Running     | private=10.0.0.3  |
| db13c6a7-a474-40ff-906e-2447cbf89440 | coreos-db13c6a7 | ACTIVE | None       | Running     | private=10.0.0.4  |
| f70b739d-9ad8-4b0b-bb74-4d715205ff0b | coreos-f70b739d | ACTIVE | None       | Running     | private=10.0.0.5  |
+--------------------------------------+-----------------+--------+------------+-------------+-------------------+
```

Finally SSH into an instance, note that the user is `core`:

```
$ chmod 400 core.pem
$ ssh -i core.pem core@10.0.0.3
   ______                ____  _____
  / ____/___  ________  / __ \/ ___/
 / /   / __ \/ ___/ _ \/ / / /\__ \
/ /___/ /_/ / /  /  __/ /_/ /___/ /
\____/\____/_/   \___/\____//____/

core@10-0-0-3 ~ $
```

## Adding More Machines

Adding new instances to the cluster is as easy as launching more with the same 
cloud-config. New instances will join the cluster assuming they can communicate 
with the others.

Example:

```
nova boot \
--user-data ./cloud-config.yaml \
--image cdf3874c-c27f-4816-bc8c-046b240e0edd \
--key-name coreos \
--flavor m1.medium \
--security-groups default coreos
```

## Multiple Clusters

If you would like to create multiple clusters you'll need to generate and use a
new discovery token. Change the token value on the etcd discovery parameter in the cloud-config, and boot new instances.

## Using CoreOS

Now that you have instances booted it is time to play around.
Check out the [CoreOS Quickstart]({{site.url}}/docs/quickstart) guide or dig into [more specific topics]({{site.url}}/docs).
