---
layout: post
title: "Setting Up DC Fabric Simulation With OpenSwitch and GNS3"
date: 2016-05-05 19:31:32 -0600
comments: true
categories: [Appliance, GNS3, Linux, OpenSwitch]
---

{% img center /images/google-datacenter-tech-03.jpg %}

__Note__: This article was [originally published here](https://opennetgeek.wordpress.com/2016/05/05/setting-up-dc-fabric-simulation-with-openswitch-and-gns3/).


In the [previous
post](http://opennetgeek.github.io/blog/2016/04/20/using-openswitch-appliance-with-gns3/) I
covered the basics about setting up the OpenSwitch Appliance using GNS3.
The setup was fairly simple: two switches connected to each other and
exchanging LLDP packets. In this post we will setup a more elaborate
network to simulate a DC fabric (although it may be a bit [overkill of a
setup](https://kontrolissues.net/2015/03/27/sometimes-size-matters-im-sorry-but-youre-just-not-big-enough/)).
The setup will be the basis for the next posts about configuring this
fabric using Ansible.

One of the first questions when setting up a complex topology with GNS3
that most people will do is: how do I connect
it to the external world outside of the simulation? For VirtualBox
machines that we are using, the options are limited. The one I found to
work reliably across platforms was to use a NAT connection. This has the
disadvantage that we have limited connectivity from the external world
toward the internal network, but this could be also a security advantage
to prevent accidental propagation of control protocols from our
simulated environment.

Since the purpose of this lab is going to be to play with Ansible, we are
going to need a Linux machine to run it. So, we will setup the following network:

{% img center no-border /images/mgmt1.png %}

Let's elaborate on the setup details:

<!--more-->

-   The 'Internet' cloud is a GNS3's cloud element, that I configured
    with a NAT entry that I named 'nat'. The name doesn't really matter,
    as GNS3 just need to see a NAT interface on the cloud element, so
    that the connection to the VM called 'ubuntu-netlab' is configured
    using VirtualBox's NAT networking.
-   The 'ubuntu-netlab' is a virtual machine that have Ubuntu 16.04
    Server installed on it (instructions about this setup below). This
    VM will have two NICs: one attached to the NAT connection
    and another one to the SW1 that will be the management network of
    the switches. What we will be doing with this machine? Several
    things:
    -   Run a DHCP server to provide IP connectivity to the management
        network
    -   Run a DNS relay to minimize disruptions on the configuration (I
        may be roaming between home/office).
    -   You may optionally configure IP masquerading (aka NAT) on this
        box to allow the OpenSwitch boxes to reach the external world,
        but is not really required.
        <div class="callout bottom-left">
        You may be wondering why not connect the management switch
        directly to the NAT port on the cloud element? I tried this, and
        found the DHCP server shipping in VirtualBox was not really
        good, and sometimes will assign the same IP address to different
        boxes if restarted at a given time (bug with the leases?)
        </div>
    -   We will install Ansible on this box, and use it as control
        machine to configure the fabric
-   The SW1 is a GNS3' ethernet switch with 8 ports. One will be
    connected to the Ubuntu VM, and the other ports to each of the
    management ports of the OpenSwitch boxes.
-   We will use 5 OpenSwitch boxes to create a simple datacenter fabric.
    I'm calling them spines/leaves for convenience, but I will do
    examples with plain L2, and other L3 topologies that may not be a
    CLOS.

So, let's get started with the setup!

## Setup the Management Network

You should start by dragging a cloud element and configuring it to have
a network interface of type NAT:

{% img center no-border /images/cloud_nat.png %}

Now we need to create the template for the Ubuntu VM that we will
connect to it.

## Configuring the GNS3 Template for the Ubuntu VM

As I mention, we will be using an Ubuntu 16.04 (state of the art!)
Server VM. Don't worry, you won't have to install it. I have pre-package
it into a nice OVA (sorry tested only with VirtualBox, you may need to
do some work if want to use VMware). Download it from
[here](https://archive.openswitch.net/demos/ubuntu-netlab.ova).

### Importing the Image

After downloading the OVA file, import it into VirtualBox, and then
proceed to create a template in GNS3 for the machine like we [did for
OpenSwitch in the previous
post](http://opennetgeek.github.io/blog/2016/04/20/using-openswitch-appliance-with-gns3/#install_and_configure).
Here is the summary of the steps and configuration:

-   Preferences -\> VirtualBox VMs -\> New
    -   Select the machine you just imported
    -   ☑︎ Use as linked base VM
-   After created click on 'Edit'
    -   'General settings' tab:
        -   Template Name: *ubuntu-netlab*
        -   Default name format: *{name}-{0}*
        -   Symbol: *:/symbols/vbox\_guest.svg*
        -   Category: *End devices*
        -   RAM: *512 MB*
        -   ☑︎ Enable remote console
        -   ☐ Enable ACPI shutdown
        -   ☑︎ Start VM in headless mode
        -   ☑︎ Use as a linked base VM
    -   'Network' tab:
        -   Adapters: *2*
        -   First port name: *leave empty*
        -   Name format: *eth{0}*
        -   Segment size: 0
        -   Type: *PCNet-FAST III (Am79c973)*
        -   ☑︎ Allow GNS3 to use any configured VirtualBox adapter

Now you can drag an instance of it into your project and setup the
connections as detailed previously. This image is configured with serial
port console enabled, so after starting the VM, you can open the console
from GNS3 and get access. The default user/password is ubuntu/ubuntu.

<div class="callout bottom-left">
Be sure to connect this VM to the NAT interface of the cloud element
first, so that it gets a DHCP response, otherwise it will be blocked
during the boot process until it times out, and you won't see any output
on the serial console. If you have doubt your setup is working, you can
disable the 'headless' setting on the machine and look at the VGA
console of the box.
</div>

## Setting up the Ubuntu VM

### Initial Setup

Now that you have access to the console, let's do some initial setup.
First you may want to tell the serial terminal the size so that the
output is not messed up. In my case I just sized my console window to
120 columns and 35 lines (I use the OS X Terminal app, and the tittle
usually says the dimensions):

```
stty cols 120 rows 35
```

Next we want to install the latest software updates for this image:

<div class="callout top-left">
For the sake of simplicity I assume you are not behind a corporate
firewall, but if you do, you want to set the http_proxy and
http_proxy variables, and when invoking sudo, pass the -E parameter
to inherit these variables.
</div>

```
sudo apt update
sudo apt upgrade
```

### Network Setup

Next, we want to configure the second network interface facing towards
SW1. You will have to run the ifconfig command to find the name of the
second interface:

```
ifconfig -a
```

On this case my second interface is called enp0s8. I will then configure
the interface to an static IP address and bring the interface up:

```
sudo su -
cat <<EOF >>/etc/network/interfaces

auto enp0s8
iface enp0s8 inet static
   address 192.168.1.1
   netmask 255.255.255.0

EOF
exit #leave sudo

sudo ifup enp0s8
```

### DHCP Server and DNS Relay

Next we want to install the DHPC server and DNS relay (DNSmasq)

```
sudo apt install isc-dhcp-server dnsmasq
```

Then proceed to configure the dhcp server modifying the file
*/etc/dhcp/dhcpd.conf* with your favorite editor (if you are not a *vim*
expert, the *nano* text editor could be easier). This are the contents
you will need:

```
option domain-name "dc-lab";
option domain-name-servers 192.168.1.1;
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.10 192.168.1.50;
  option routers 192.168.1.1;
}
```

Now you can start the server:

```
systemctl start isc-dhcp-server
```

### IP Masquerading (NAT)

If you want the OpenSwitch boxes to have access to the external
network over the management port, you may want to enable IP masquerading
on the box. Adjust the following script to the name of your network
interfaces:

```
public_if=enp0s3
private_if=enp0s8
sudo -E iptables -t nat -A POSTROUTING -o $public_if -j MASQUERADE
sudo -E iptables -A FORWARD -i $public_if -o $private_if -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo -E iptables -A FORWARD -i $private_if -o $public_if -j ACCEPT
sudo sysctl -w net.ipv4.ip_forward=1
```

## Setting up the Fabric Switches

Now that we have the Cloud element and the Ubuntu VM, you need to drag
the GNS3 ethernet switch element to connect the second interface from
the Ubuntu VM with it, along with all the management ports of the
OpenSwitch instances. You can now proceed to power up the OpenSwitch
instances and login into their consoles (user/password is netop), and
verify the management interface got an IP address from our control VM:

```
switch# show interface mgmt 
  Address Mode : dhcp
  IPv4 address/subnet-mask : 192.168.1.11/24
  Default gateway IPv4 : 192.168.1.1
  IPv6 address/prefix : 
  IPv6 link local address/prefix: fe80::a00:27ff:fe3a:efdd/64
  Default gateway IPv6 : 
  Primary Nameserver : 192.168.1.1
  Secondary Nameserver : 
switch#
```

<div class="callout bottom-left">
Unfortunately OpenSwitch Simulator still has the <a href="https://tree.taiga.io/project/openswitch/issue/840">bug I mentioned
previously</a> and sometimes it may fail during initialization. If you stumble with this,
just restart the VM.
</div>

You may also test login with user root (no password) into a bash shell
and the networking of the management domain should be able to reach the
external network.

## Next Steps

Now that we have a management network up an running, we want to
configure the fabric to move packets around. At this point most
networking engineers will log into the CLI and start configuring, but we
went thru all this setup of a management network to do something more
interesting...

In our next post we will do this instead using Ansible,
which is a pretty good tool for managing
[cattle](http://www.theregister.co.uk/2013/03/18/servers_pets_or_cattle_cern/).
