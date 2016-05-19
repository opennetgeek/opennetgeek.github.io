---
layout: post
title: "OpenSwitch Meets P4"
date: 2016-05-18 21:02:01 -0600
comments: true
categories: [Barefoot, Mellanox, OpenSwitch, P4, SDN]
---

{% img center /images/ops-p4-sticker.png %}

__Note__: This article was [originally published here](https://opennetgeek.wordpress.com/2016/05/09/openswitch-meets-p4/).

Before moving on the next post to continue our saga of OpenSwitch
[Simulations with
GNS3](http://opennetgeek.github.io/blog/2016/05/05/setting-up-dc-fabric-simulation-with-openswitch-and-gns3/),
I wanted to take a quick deviation to document a subject that gets a lot
of attention these days: [P4](http://p4.org). 

In case you have been missing all the action around P4, the 30,000 feet view is that it's a
language to describe forwarding pipelines (and no, it's not the same as
OpenFlow, that is useful for programming entries in
almost-always-pre-defined pipelines). One of the
([many](http://events.linuxfoundation.org/sites/events/files/slides/4-%20Chang.pdf))
nice things about this is that you can potentially 'compile' your
pipeline definition into an executable program that provides a
functional simulation of a P4-based ASIC. Did I mention the tools for
doing all of this are [available as open source](http://p4.org/code/)?

## What does P4 has to do with OpenSwitch?

<!--more-->

Well, as I mention in previous posts, OpenSwitch provides a simulation
environment that provides a software data path for testing and training
purposes, and we package it as either container or VM appliance. So far
the default software data path is provided by running an stock
OpenVSwitch that's under control of the switchd plugin for simulation.

However, since Barefoot Networks is a contributing member to the
OpenSwitch project, they have been adding support for using a P4 backend
that allows replace the simulation 'ASIC' to a P4 functional model of an
ASIC. Some details about it can be found in their [presentation from the
last OpenSwitch
Meetup](https://archive.openswitch.net/presentations/Meetups/March_2016/OPS_Meetup_March2016_BFN_P4.pdf) (if
you are interested on OpenSwitch, there were several interesting
presentations on that Meetup. You can watch them
[here](https://www.youtube.com/watch?v=u-eWetiNieU)). 

Since OpenSwitch is not married to a particular interface for the ASIC SDK (reports
otherwise are greatly exaggerated!), the P4 backend provides a [native
plugin for
switchd](http://git.openswitch.net/cgit/openswitch/ops-switchd-p4switch-plugin/)
to use their 'PD' APIs directly.

<div class="callout bottom-left">
Exercise to the reader: Barefoot is a big supporter of 
<a href="https://github.com/opencomputeproject/SAI">SAI</a> and they can generate
the SAI shim as well, so it would be interesting to use it with the 
<a href="http://git.openswitch.net/cgit/openswitch/ops-switchd-sai-plugin/">SAI plugin</a>
that Mellanox is contributing to OpenSwitch.
</div>

## Shut Up and Take my Bitcoin!

OK, OK, how to run with the P4 data path? OpenSwitch still doesn't
provide P4 periodic images, so you have to build the code yourself.
These are the steps you need to do to flip into the P4 backend (assuming
you already have a working build environment for either container or
appliance):

```
rm -Rf build/tmp # Flush file system dependencies
echo "EXTRA_IMAGE_FEATURES += \"ops-p4\"" >> build/conf/local.conf
```

Now, run 'make' and go for a coffee while the image get's build. It
wasn't that hard, was it? 

One more thing™, if you are using the
appliance environment, verify your code already includes [this
commit](https://review.openswitch.net/#/c/8590/), or otherwise
cherry-pick the change.

<div class="callout bottom-left">
<i class="fa fa-2x fa-exclamation-triangle" aria-hidden="true"></i><br><br>
Buyer beware: P4 is an experimental feature, so don't expect everything
to be smooth right now ;). That said, the plan for OpenSwitch moving
forward is to deprecated the OpenVSwitch backend in favor of the P4
implementation as the default implementation for simulations.
</div>

## How to use the P4-based Image?

Pretty much the same way that you use the regular simulator image. If
you are using the P4 Image with GNS3, you can download this special
edition
[stencil](https://archive.openswitch.net/logo/OpenSwitchP4Stencil.png) I
did for it. 

One interesting fact that you may notice when using the P4
simulator, is that control packets into the CPU have higher latency.
This is caused because the PDUs are going thru a TAP device from the
switch process (in userspace) into the netdev interfaces in the SW
namespace.

## Final thoughts

This kind of experimentation wasn't possible a couple years ago, and
software stacks like SAI and P4 are changing the landscape along with
OpenSwitch. Think that you can change the forwarding data path
implementation, rebuild the simulator, and integrate back again with
your NOS stack! is mind boggling. 

So, indeed, 'software is eating the world'. Happy hacking!
