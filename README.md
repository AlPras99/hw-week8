# Complete Intro Containers
## Containers
The core of what containers are is just a few features of the Linux kernel duct-taped together, it's just using a few features of Linux together to achieve isolation. Containers give us many of the security and resource-management features of VMs but without the cost of having to run a whole other operating system. It instead usings chroot, namespace, and cgroup to separate a group of processes from each other.
## Chroot
chroot often called as "cha-root" and "change root", it's a Linux command that allows you to set the root directory of a new process. In our container use case, we just set the root directory to be where-ever the new container's new root directory should be. And now the new container group of processes can't see anything outside of it, eliminating our security problem because the new process has no visibility outside of its new root.
## Namespaces
While chroot is a pretty straightforward, namespaces and cgroups are a bit more nebulous to understand but no less important. Both of these next two features are for security and resource management. Namespaces allow you to hide processes from other processes. If we give each chroot'd environment different sets of namespaces, now users can't see each others' processes (they even get different process PIDs, or process IDs, so they can't guess what the others have) and you can't steal or hijack what you can't see!

`unshare` creates a new isolated namespace from its parent and all other future tenants. Run this:
```
exit # from our chroot'd environment if you're still running it, if not skip this

# install debootstrap
apt-get update -y
apt-get install debootstrap -y
debootstrap --variant=minbase bionic /better-root

# head into the new namespace'd, chroot'd environment
unshare --mount --uts --ipc --net --pid --fork --user --map-root-user chroot /better-root bash # this also chroot's for us
mount -t proc none /proc # process namespace
mount -t sysfs none /sys # filesystem
mount -t tmpfs none /tmp # filesystem
```

## Cgroups
Every isolated environment has access to all physical resources of the server. There's no isolation of physical components from these environments.

Enter the hero of this story: cgroups, or control groups. Google saw this same problem when building their own infrastructure and wanted to protect runaway processes from taking down entire servers and made this idea of cgroups so you can say "this isolated environment only gets so much CPU, so much memory, etc. and once it's out of those it's out-of-luck, it won't get any more."

This is a bit more difficult to accomplish but let's go ahead and give it a shot.
```
# outside of unshare'd environment get the tools we'll need here
apt-get install -y cgroup-tools htop

# create new cgroups
cgcreate -g cpu,memory,blkio,devices,freezer:/sandbox

# add our unshare'd env to our cgroup
ps aux # grab the bash PID that's right after the unshare one
cgclassify -g cpu,memory,blkio,devices,freezer:sandbox <PID>

# list tasks associated to the sandbox cpu group, we should see the above PID
cat /sys/fs/cgroup/cpu/sandbox/tasks

# show the cpu share of the sandbox cpu group, this is the number that determines priority between competing resources, higher is is higher priority
cat /sys/fs/cgroup/cpu/sandbox/cpu.shares

# kill all of sandbox's processes if you need it
# kill -9 $(cat /sys/fs/cgroup/cpu/sandbox/tasks)

# Limit usage at 5% for a multi core system
cgset -r cpu.cfs_period_us=100000 -r cpu.cfs_quota_us=$[ 5000 * $(getconf _NPROCESSORS_ONLN) ] sandbox

# Set a limit of 80M
cgset -r memory.limit_in_bytes=80M sandbox
# Get memory stats used by the cgroup
cgget -r memory.stat sandbox

# in terminal session #2, outside of the unshare'd env
htop # will allow us to see resources being used with a nice visualizer

# in terminal session #1, inside unshared'd env
yes > /dev/null # this will instantly consume one core's worth of CPU power
# notice it's only taking 5% of the CPU, like we set
# if you want, run the docker exec from above to get a third session to see the above command take 100% of the available resources
# CTRL+C stops the above any time
```

And now we can call this a container.
## Docker
Docker is a commandline tool that made creating, updating packaging, distributing, and running containers significantly easier which in turns allowed them become very popular with not just system administraters but the programming populace at large. At its heart, it's a command line to achieve what we were doing with cgroups, namespaces, and chroot but in a much more convenient way.
## Docker Desktop
Go ahead and install [Docker](https://www.docker.com/products/docker-desktop) Desktop right now. It will work for both Mac and Windows. If you're on Mac, you'll see a cute little whale icon in your status bar. Feel free to poke around and see what it has. It will also take the liberty of installing the `docker` commandline tool so we can start doing all the fun things with Docker.
## Docker Hub
Docker Hub is a public registry of pre-made containers, you can visit it [here](https://hub.docker.com/search?q=&type=image). Instead of having to handcraft everything yourself, you can start out with a base container from Docker Hub and start from there.
## Docker Images without Docker
These pre-made containers are called images. They basically dump out the state of the container, package that up, and store it so you can use it later. So let's go nab one of these image and run it! We're going to do it first without Docker to show you that you actually already knows what's going on. Let's grab the latest Node.js container that runs Ubuntu.
