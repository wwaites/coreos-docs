---
layout: docs
slug: guides
title: Getting Started with systemd
category: launching_containers
sub_category: launching
weight: 5
---

<div class="coreos-docs-banner">
<span class="glyphicon glyphicon-info-sign"></span>These instructions have been updated for <a href="{{site.url}}/blog/new-filesystem-btrfs-cloud-config/">our new images</a>.
</div>

# Getting Started with systemd

systemd is an init system that provides many powerful features for starting, stopping and managing processes. Within the CoreOS world, you will almost exclusively use systemd to manage the lifecycle of your Docker containers.

## Terminology

systemd consists of two main concepts: a unit and a target. A unit is a configuration file that describes the properties of the process that you'd like to run. This is normally a `docker run` command or something similar. A target is a grouping mechanism that allows systemd to start up groups of processes at the same time. This happens at every boot as processes are started at different run levels.

systemd is the first process started on CoreOS and it reads different targets and starts the processes specified which allows the operating system to start. The target that you'll interact with is the `multi-user.target` which holds all of the general use unit files for our containers.

Each target is actually a collection of symlinks to our unit files. This is specified in the unit file by `WantedBy=multi-user.target`. Running `systemctl enable foo.service` creates symlinks to the unit inside `multi-user.target`.

## Unit File

On CoreOS, unit files are located within the R/W filesystem at `/etc/systemd/system`. Let's create a simple unit named `hello.service`:

```
[Unit]
Description=My Service
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/docker run busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"

[Install]
WantedBy=multi-user.target
```

The description shows up in the systemd log and a few other places. Write something that will help you understand exactly what this does later on.

`After=docker.service` and `Requires=docker.service` means this unit will only start after `docker.service` is active. You can define as many of these as you want.

`ExecStart=` allows you to specify any command that you'd like to run when this unit is started.

`WantedBy=` is the target that this unit is a part of.

To start a new unit, we need to tell systemd to create the symlink and then start the file:

```
$ sudo systemctl enable /etc/systemd/system/hello.service
$ sudo systemctl start hello.service
```

To verify the unit started, you can see the list of containers running with `docker ps` and read the unit's output with `journalctl`:

```
$ journalctl -f -u hello.service
-- Logs begin at Fri 2014-02-07 00:05:55 UTC. --
Feb 11 17:46:26 localhost docker[23470]: Hello World
Feb 11 17:46:27 localhost docker[23470]: Hello World
Feb 11 17:46:28 localhost docker[23470]: Hello World
...
```

<a class="btn btn-default" href="{{site.url}}/docs/launching-containers/launching/overview-of-systemctl">Overview of systemctl</a>
<a class="btn btn-default" href="{{site.url}}/docs/cluster-management/debugging/reading-the-system-log">Reading the System Log</a>

## Advanced Unit Files

systemd provides a high degree of functionality in your unit files. Here's a curated list of useful features listed in the order they'll occur in the lifecycle of a unit:

| Name    | Description |
|---------|-------------|
| ExecStartPre | Commands that will run before `ExecStart`. |
| ExecStart | Main commands to run for this unit. |
| ExecStartPost | Commands that will run after all `ExecStart` commands have completed. |
| ExecReload | Commands that will run when this unit is reloaded via `systemctl reload foo.service` |
| ExecStop | Commands that will run when this unit is considered failed or if it is stopped via `systemctl stop foo.service` |
| ExecStopPost | Commands that will run after `ExecStop` has completed. |
| RestartSec | The amount of time to sleep before restarting a serivce. Useful to prevent your failed service from attempting to restart itself every 100ms. |

The full list is located on the [systemd man page](http://www.freedesktop.org/software/systemd/man/systemd.service.html).

Let's put a few of these concepts togther to register new units within etcd. Imagine we had another container running that would read these values from etcd and act upon them.

We can use `ExecStart` to either create a container with the `docker run` command or start a pre-existing container with the `docker start -a` command. We need to account for both because you can't issue multiple docker run commands when specifying a `-name`. In either case we must leave the container in the foreground (i.e. don't run with `-d`) so systemd knows the service is running.

Since our container will be started in `ExecStart`, it makes sense for our etcd command to run as `ExecStartPost` to ensure that our container is started and functioning.

When the service is told to stop, we need to stop the docker container using its `-name` from the run command. We also need to clean up our etcd key when the container exits or the unit is failed by using `ExecStopPost`.

```
[Unit]
Description=My Advanced Service
After=etcd.service
After=docker.service

[Service]
ExecStart=/bin/bash -c '/usr/bin/docker start -a apache || /usr/bin/docker run -name apache -p 80:80 coreos/apache /usr/sbin/apache2ctl -D FOREGROUND'
ExecStartPost=/usr/bin/etcdctl set /domains/example.com/10.10.10.123:8081 running
ExecStop=/usr/bin/docker stop apache
ExecStopPost=/usr/bin/etcdctl rm /domains/example.com/10.10.10.123:8081

[Install]
WantedBy=multi-user.target
```

## Unit Variables

In our last example we had to hardcode our IP address when we announced our container in etcd. That's not scalable and systemd has a few variables built in to help us out. Here's a few of the most useful:

| Variable | Meaning | Description |
|----------|---------|-------------|
| `%n` | Full unit name | Useful if the name of your unit is unique enough to be used as an argument on a command. |
| `%m` | Machine ID | Useful for namespacing etcd keys by machine. Example: `/machines/%m/units` |
| `%b` | BootID | Similar to the machine ID, but this value is random and changes on each boot |
| `%H` | Hostname | Allows you to run the same unit file across many machines. Useful for service discovery. Example: `/domains/example.com/%H:8081` |

A full list is on the [systemd man page](http://www.freedesktop.org/software/systemd/man/systemd.unit.html).

## Instantiated Units

Since systemd is based on symlinks, there are a few interesting tricks you can leverage that are very powerful when used with containers. If you create multiple symlinks to the same unit file, the following variables become available to you:

| Variable | Meaning | Description |
|----------|---------|-------------|
| `%p` | Prefix name | Refers to any string before `@` in your unit name. |
| `%i` | Instance name | Refers to the string between the `@` and the suffix. |

In our earlier example we had to hardcode our IP address when registering within etcd:

```
ExecStartPost=/usr/bin/etcdctl set /domains/example.com/10.10.10.123:8081 running
```

We can enhance this by using `%H` and `%i` to dynamically announce the hostname and port. Specify the port after the `@` by using two unit files named `foo@123.service` and `foo@456.service`:

```
ExecStartPost=/usr/bin/etcdctl set /domains/example.com/%H:%i running
```

This gives us the flexiblity to use a single unit file to announce multiple copies of the same container on a both a single machine (no port overlap) and on multiple machines (no hostname overlap).

#### More Information
<a class="btn btn-default" href="http://www.freedesktop.org/software/systemd/man/systemd.service.html">systemd.service Docs</a>
<a class="btn btn-default" href="http://www.freedesktop.org/software/systemd/man/systemd.unit.html">systemd.unit Docs</a>
<a class="btn btn-default" href="http://www.freedesktop.org/software/systemd/man/systemd.target.html">systemd.target Docs</a>
