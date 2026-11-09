---
title: Router on a stick
date: 2026-11-08
layout: post
---

## Overview

I purchased a tp-link L2 smart switch a couple months back and decided it was time to learn inter-VLAN routing. This will be a two part series. This post will focus on enabling communication between devices on two different VLANs, and the subsequent post will be on setting up a basic linux firewall.

## Hardware

Since this is POC for communication between VLANs, we need at least two separate VLANs, which I'll name VLAN 10 and VLAN 20. Each VLAN needs at least one host. [Colorado State University Surplus Property](https://surplus.colostate.edu/) has pretty good deals on hardware and I managed to get an old ThinkPad P50s for about $100, which I decided would go on VLAN 10. The VLAN 20 host was a Raspberry Pi 3B+ that I've owned for awhile.

For the router itself I used an Intel NUC, a small form factor mini pc that has very low power consumption and comes equipped with a Celeron J3060 processor. It only has one ethernet interface (`enp3s0`) which will be split into two sub-interfaces, one for each VLAN.

The last component in my setup was a TL-SG108E switch, an 8-port gigabit switch that supports 802.1Q tagging.

Here's the network topology for this project

![]({{ '/assets/img/router-on-a-stick.png' | relative_url }})

## Switch configuration

The first step was to choose access ports, which have membership in only one VLAN. Frames leaving these ports will not stamped with 802.1Q tags, which is why they are called "untagged". Ports 6 and 7 were chosen for this purpose. 

Next I set up the trunk port, which has membership in both VLANs. Frames leaving these ports get an 802.1Q tag, which tells a layer 3 device which VLAN a frame belongs to. Port 8 was chosen as the trunk port.

![]({{ '/assets/img/8021q-routing.png' | relative_url }})

The last step was to configure the switch to add a PVID to incoming ethernet frames on the untagged ports. I chose PVID 10 for port 7 and PVID 20 for port 6. This allows the trunk port to determine which VLAN a frame belongs to.

![]({{ '/assets/img/PVIDs.png' | relative_url }})

## Router configuration

I needed these devices for another project I had lined up after this POC, so my configuration involves running `ip`, `modprobe`, and `sysctl` commands that don't persist after a reboot.

The `802.1Q` kernel module allows a linux system to split a single physical interface into multiple VLAN-specific subinterfaces.

```
$ sudo modprobe 8021q
```

Next step was to actually create the subinterfaces. The third octect of the static ip addresses were chosen to match the VLAN ID.

```
$ ip link set enp3s0 up

# Create subinterface for VLAN 10
$ ip link add link enp3s0 name enp3s0.10 type vlan id 10
$ ip addr add 192.168.10.1/24 dev enp3s0.10
$ ip link set enp3s0.10 up

# Create subinterface for VLAN 20
$ ip link add link enp3s0 name enp3s0.20 type vlan id 20
$ ip addr add 192.168.20.1/24 dev enp3s0.20
$ ip link set enp3s0.20 up
```

Checking to make sure everything looks good

![]({{ '/assets/img/intel-nuc-interfaces.jpg' | relative_url }})


Now that we have our subinterfaces set up, we need to tell the kernel that we want to be able to route IPv4 packets between interfaces.

```
sudo sysctl -w net.ipv4.ip_forward=1
```

The final step before running the tests is to configure the host devices. 

Since I'm not using a DHCP server, the Thinkpad and the pi need to assign themselves a static ip that's in the subnet associated with its VLAN.

```
# thinkpad
ip addr add 192.168.10.50/24 dev enp0s31f6

# pi
ip addr add 192.168.20.90/24 dev eth0
```

Now we set the default routes on the VLAN 10 and VLAN 20 hosts to the static ips of the Intel NUCs `enp3s0.10` and `enp3s0.20` interfaces, respectively

```
# thinkpad
ip route add default via 192.168.10.1/24 dev enp0s31f6

# pi
ip route add default via 192.168.20.1/24 dev eth0
```

Now, whenever we tell the thinkpad on VLAN 10 to ping the pi on VLAN 20, the thinkpad knows to send the packets to 192.168.10.1 for routing.

![]({{ '/assets/img/ping-from-thinkpad.jpg' | relative_url }})

Similarly, when the pi on VLAN 20 tries to ping the thinkpad on VLAN 10, it sends it to 192.168.20.1.

![]({{ '/assets/img/ping-from-pi.jpg' | relative_url }})