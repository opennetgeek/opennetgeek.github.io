---
layout: post
title: "Developing OpenSwitch With Linux VM/OS X Host"
date: 2016-04-19 14:58:23 -0600
comments: true
categories: [Linux, Mac, OpenSwitch, OS X]
---

{% img center /images/ops-mac.jpg 'OpenSwitch on Mac' 'OpenSwitch on Mac' %}

__Note__: This article was [originally published here](https://opennetgeek.wordpress.com/2016/04/19/developing-openswitch-with-linux-vmos-x-host/).

One of the purposes when we designed the build system in OpenSwitch, was to make it possible to develop on as many environments as possible. If you have some background with developing networking firmware, the typical developer love to have this VM where everything works perfectly, but makes it impossible to work in your laptop at 30000 feet. This is not really a sin (as long as you can have the VM hosted in your machine), but the problem is that usually is some IT team on charge of the VMs setup, and the deployment is not handled by some automated/version-controlled code.

So for OpenSwitch, we aimed to at least document the requirements and steps for manual setup of your environment. You can read [this page](http://www.openswitch.net/documents/dev/linux-setup) to get your Linux machine to ready it for OpenSwitch development.

So, why to write an article about my particular setup? Well, I'm a Mac user, so in this article I'm going to detail my setup using a OS X host with a Linux VM. This provides some nice tricks that makes your workflow easier if you are using a similar setup. I will also explain the rationale of the setup.

My use of Linux VM for development is mostly thru the Linux CLI and I use NFS to share files between my Mac and Linux. This allows me to use any graphical tool from the Linux VM if I have to, but also to use tools from the host without major hurdles.Read more...

## Setup the VM

<!--more-->

The first step is that I will use [VirtualBox](https://www.virtualbox.org/) to deploy my Linux VM. You can use any software that you want, but VirtualBox is free and available to any developer, so it's a good target.

Since I'm using a Macbook Pro Retina with a i7, I will use the following setup:

* 6 Cores: the Yocto build system can take advantage of the multicore environment, so the more cores, the faster builds. This leaves 2 cores idle for my Mac, so performance doesn't suffer.
* 4Gb of RAM: A full from-scratch build of the compiler (rare, since we use Yocto's shared states) can consume quite a bit of RAM.
* 20GB of disk: the build directories from Yocto could be big, averaging 6G each, but the figure could easily go up depending on what you are doing. So having some disk space to spare is a good measure (I personally ran with 160GB)
* Two network interfaces:
  * First one will be NAT, so that I can reach the internet independent of the network connection on my Mac (sometimes I use wireless, sometimes I use a thunderbolt dock station).
  * Second one will be host-only adapter, this provides a internal network so that we can easily communicate between the VM and the host. Using multicast DNS (aka Bonjour), we won' even have to worry about IP addresses on this network.

For this VM, I will be using [Ubuntu LTS 14.04 Server ISO](http://mirror.uoregon.edu/ubuntu-releases/14.04.4/ubuntu-14.04.4-server-amd64.iso). Right now OpenSwitch works best with this version of Ubuntu (and provides faster build times). We hope to update to the new LTS soon and once we do, I will update this post to reflect that. __Update__: I have added comments about how to make it work with [Ubuntu 16.04](http://releases.ubuntu.com/16.04/ubuntu-16.04-server-amd64.iso)!.

I'm assuming you are familiar with installing an Ubuntu ISO into a VM and will omit those details. The only important detail from the installation would be to install the SSH server. I would recommend that you setup your user account with the same user id that you have in GitHub (this is a good moment to create one, if you don't). Using the same user account makes life easier with gerrit, but is not an strict requirement.

## Proxy Setup

If you are behind a corporate firewall, be sure to setup your corporate proxies in your user environment:

```
cat <<EOF >>~/.profile
export http_proxy=http://your.proxy.com:port
export https_proxy=http://your.proxy.com:port
EOF
source ~/.profile
```

If my case, I may be roaming between the office and home, so sometimes I don't use a proxy, in which case I find easy to unset the variables and be able to work. So, I don't like to wire the apt configuration with the proxy as [explained here](http://askubuntu.com/questions/257290/configure-proxy-for-apt), instead I like to run the apt-get command with 'sudo -E' parameter so that it inherits my environment variables and uses the proxy if I have it defined.

## Initial Setup

First, I usually disable asking password for my user:

```
sudo visudo -f /etc/sudoers.d/passwordless-user
```

Then enter a line like the following (replace your username):

```
username ALL=(ALL) NOPASSWD:ALL
```

Next, let's update the apt package cache:

```
sudo apt-get update
```

## Networking Setup

The initial installation will configure the machine out of the box to have DHCP on the first network interface, so we will need to add the second interface for automatic startup. So I will add the following lines to /etc/network/interfaces:

```
auto eth1
iface eth1 inet dhcp
```

Then, bring the interface up:

```
sudo ifup eth1
```

__Update__: if you are using 16.04, the network interfaces have different names (to understand why and revert it if you want, take a look here). Run 'ifconfig -a' to find the name of your interfaces.

Next, proceed to install some extra packages and configurations that make networking work better.

dnsmasq: the build system fetches constantly from the servers in order to use the shared states to speed the build. So using a local DNS cache will help reduce the pressure on your DNS (and may save you from raising flags about the high rate of request your machine will be doing).

```
sudo apt-get install dnsmasq
```

__Update__: if you are using 16.04, dnsmasq won't work out of the box since it conflicts with the dnsmasq used by the LXC container daemon. I worked around by disabling LXC (but maybe that solution doesn't work for you):

```
sudo systemctl stop lxc lxc-net
sudo systemctl disable lxc lxc-net
```

nss-mdns: this is the name service resolution to use multicast DNS. This allows the Linux machine to resolve (and as side effect of dependencies, advertise) .local addresses and makes communication with the Mac simpler.

```
sudo apt-get install libnss-mdns
```

After this is installed, you should be able to reach your machine with the .local name. Lets say your Linux machine is named linuxvm, then you should be able to run from your Mac:

```
ping linuxvm.local
```

## Headless VM

After the initial networking setup is ready, I like to run the VM in headless mode from the OS X terminal. To do that, I shutdown the VM from the VirtualBox UI and close the application. Then from an OS X Terminal I can start the VM in the background with (replace the LinuxVM name with the name of your VM:

```
VBoxHeadless --startvm LinuxVM --vrde off &
```

This will run on the background and as long as you don't close the terminal, the VM will run. Next, I usually connect to the VM from the terminal with (using the mDNS .local resolution):

```
ssh -Y username@linuxvm.local
```

The -Y automatically exports X11 apps into my Mac XQuartz, in case you want to use some graphical program from Linux.

## OpenSwitch Development Environment Tweaks

Create a workspace directory on /ws. This is just an standard I follow  of putting all my work on a separate directory. Makes easier to fine-control the NFS exports from my Linux machine into my Mac host.

```
sudo mkdir /ws
sudo chown $USER /ws
```

### NFS Server Share Setup

Installing an NFS server with the proper parameters to allow easy sharing of files between the host an the Mac took me several hours of research. Here is the recipe for it.

Install the NFS server daemon:

```
sudo apt-get install nfs-kernel-server
```

Configure the /etc/exports file with the following line to allow your Mac host to access the files under /ws with the same user permissions as your Linux user (assuming your user ID is 1000, you can find that with the 'id' command).

```
/ws 192.168.56.1(rw,all_squash,anonuid=1000,anongid=1000,sync,no_subtree_check,insecure)
```

Start the NFS server:

```
sudo service nfs-kernel-server start
```

__Update__: if you are using 16.04 use systemctl instead of service, and the service is called nfs-server:

```
sudo systemctl start nfs-server
```

At this point you should be able to mount the NFS directory from your Mac, by going to the Finder Menu Go  -> Connect to Server... (or using the ⌘k shortcut). On the server address box enter:

```
nfs://yourmachine.local/ws
```

That should get your Mac access over NFS to the directory  /ws and will have the right permission mapping so that files could be changed either from the Linux or Mac environment without permissions conflicts.

### Install Docker

We are going to be using docker, so we need to install it following the instructions [here](https://docs.docker.com/engine/installation/linux/ubuntulinux/).

If you are using a proxy, you need to setup it up for the docker daemon on /etc/default/docker:

```
export http_proxy=http://your.proxy.com:port
export https_proxy=http://your.proxy.com:port
```

Then, restart your docker to make the changes take effect:

```
sudo service docker restart
```

### Prepare for  OpenSwitch Build Environment

Install the minimal required packages for OpenSwitch development:

```
sudo apt-get install gawk wget git-core diffstat unzip texinfo gcc-multilib build-essential chrpath screen curl device-tree-compiler git-review
```

OpenSwitch leverages Yocto's shared-state system, so we want to have a global shared state directory that is shared across all the development environments to speed build time. The OpenSwitch build system will get this information automatically from a variable environment, so I will put it on my profile by default:

```
sudo mkdir /ws/openswitch-sstate-cache /ws/openswitch-downloads
sudo chown $USER /ws/openswitch-sstate-cache /ws/openswitch-downloads
echo -e "export SSTATE_DIR=/ws/openswitch-sstate-cache\nexport DL_DIR=/ws/openswitch-downloads" >> ~/.profile
source ~/.profile
```

## Git And Gerrit Setup

Setup some initial git setup for working with [OpenSwitch Gerrit](https://review.openswitch.net/). You will need a GitHub account in order to login into gerrit, so I will assume you already have one, and you know how to upload your ssh key into gerrit. You will also need to sign a [DCO](http://elinux.org/Developer_Certificate_Of_Origin) when signing for Gerrit, otherwise won't be able to upload code for review (if you are going to contribute as part of your job, you need to follow the steps on the OpenSwitch page to get your employer to sing a corporate contribution agreement).

First, setup git (I like to use colors, and use st as an alias for status):

```
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
git config --global alias.st status
git config --global color.ui auto
```

If you are working behind a corporate firewall, it may be that gerrit ports (29418) are blocked. You can test a telnet connection into the port to confirm if you are blocked. A workaround this restriction is to install socat and use a proxy for the ssh connection:

```
sudo apt-get install socat
cat <<EOF >>~/.ssh/config
Host review.openswitch.net
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand socat - PROXY:your.proxy.com:%h:%p,proxyport=<port>
EOF
```

If your github username is different from the username in your machine, you may want to configure git to use the right username when doing git reviews:

```
git config --global gitreview.username <username>
```

You can verify your connection to gerrit works by issuing this command and getting a similar output:

```
ssh -p 29418 <username>@review.openswitch.net
  ****    Welcome to Gerrit Code Review    ****

  Hi John Doe, you have successfully connected over SSH.

  Unfortunately, interactive shells are disabled.

  To clone a hosted Git repository, use:

  git clone ssh://<username>@review.openswitch.net:29418/REPOSITORY_NAME.git

Connection to review.openswitch.net closed.
```

## Building OpenSwitch

Finally, I'm going to clone the OpenSwitch build system, and build an appliance image that I would use in my next post about working OpenSwitch with GNS3.

First I clone the build system:

```
git clone https://git.openswitch.net/openswitch/ops-build ops-appliance
cd ops-appliance
make configure appliance
make
```

Now you can sit a wait for the build to complete. If you have a fast internet connection it will take in the order of 15 minutes to download all the caches and sources to build a full OpenSwitch Virtual Appliance. In  my next post we will be using that.

## Conclusion

I hope the information is useful. Drop me a comment if you think there is any improvement I can do to the environment.
