---
title: "RSTP: I'm the Root. That's my Proposal !"
date: 2021-03-19T12:49:12-04:00
draft: false
tags: ["Cisco", "Layer 2", "STP"]
summary: "In this post we will talk about RSTP protocol and its rapid convergence."
---

We will implement RSTP in a small topology to observe its rapid convergence.

## Topology

![rstp1](/neve/img/rstp/rstp1.png)

## Root bridge Election

We know that root selection is based on lowest Bridge ID value consisting of:
1. Switch MAC address (6 bytes).
2. Bridge Priority (2 bytes): 32786 + Sys-ext-id (basically the VLAN number).

Sw1 is now started and we monitor its link e0/0 toward SW2 then we enable RSTP.

```s
SW1(config)#spanning-tree mode rapid-pvst
```

> Note: \
> RSTP can be single or multiple instances. In Cisco implementation, Cisco used RSTP as an underlying mechanism for its rapid-pvst which supports RSTP instance per each VLAN.\
> Also RSTP is used as part of 802.1s MST 

Capturing an STP frame sent by SW1 we notice the following:

![rstp2](/neve/img/rstp/rstp2.png)

1. This is the well-known STP multicast address that switches running STP listen to.
2. Here we see the version ID 2 which indicate RSTP, this is sent inside the BPDU and is used to differentiate switches running RSTP from switches running the legacy STP (version 0).
3. Those are the flags used by RSTP inside the BPDU to indicate the port role and state, it also used when topology change happens. Also they're used in synchronization process by using proposal and agreement flags.
4. This is the root bridge ID. It's SW1 because other switches are turned off right now.
5. This is the local bridge ID (SW1)
6. Those are STP timers, which are set by the root bridge.

> ***STP timers refresher:***\
> **Hello timer**: is the time interval between Configuration BPDUs sent by the root bridge.
> **Forward Delay timer**: is the time interval that a switch port spends in both Listening and Learning states.\
> **Max Age timer**: is how long a switch stores a BPDU before it's aged out, and if no BPDUs received during the Max Age timer the switch will know that a topology change occurred.\
> *This will tell you that if a topology change occurred the convergence time is more than 2x Forward Delay timer.*

Now, I know that SW1 was added first to eve-ng topology so I know it's going to be the root because it has the lowest MAC address. Regardless let's start SW3 and configure it to use RSTP.

We can verify who's the root bridge by:
```s
SW1#show spanning-tree
```

![rstp3](/neve/img/rstp/rstp3.png)

Notice that SW1 is the root bridge and that all its ports have the designated role with the state of forwarding.

As soon as SW3 sees SW1 BPDUs it knows it's not the root bridge (SW1 has lower MAC address) but we can observe in the following capture that when RSTP was first enabled on SW3 it sent BPDU declaring itself as the root and proposed that its port should take the designated role.

![rstp4](/neve/img/rstp/rstp4.png)

SW1 did not agree on that so the role of SW3 port changed to root port.

## Switch: RSTP activated! I'm gonna sync now

So as a switch I know who's the root bridge and which of my ports is the root port and the cost toward it. But now because I'm a good switch I'm going to find out what are the roles of my other ports. 

That's what is called **Synchronization**

> ***RSTP Port Roles Refresher***:\
> **Root port:** has the best cost to the root bridge.\
> **Designated port:** port on a network segment that has the best root path cost to the root bridge.\
> **Alternate port:** port that has an alternate path to the root bridge. but less desirable. If root port is down this port becomes the root port and will move to forwarding state immediately\
> **Backup port:** a less desirable port connected to a segment where another port is already connected.

Let's introduce SW2 to the topology and enable RSTP on it.

SW2 will receive a superior BPDU directly from the root bridge (SW1), so SW3 will put all its nonedge ports in discarding states to prevent loops and the root port is moved to forwarding state immediately.

> ***RSTP Ports Types Refresher:*** \
> **Edge**: single host is connected to the port.\
> **Root**: port with best path cost to root bridge.\
> **Point-to-Point**: a port connecting to another switch.

Now for each nonedge port SW2 will send a proposal to its neighbor (SW3) and it will receive an agreement (SW2 has a lower bridge ID than SW3) and the port will be moved to forwarding state immediately.

By doing a packet capture on SW2 e0/1 port we can see its proposal to SW3 declaring its port should be the designated port: 

![rstp5](/neve/img/rstp/rstp5.png)

Here's SW3 sending an agreement to SW2:

![rstp6](/neve/img/rstp/rstp6.png)

So we see that the entire convergence process is happening at the speed of BPDU transmission and all the switches will propagate these handshakes (Proposal & Agreement) on all point-to-point links and using synchronization to prevent loops while the network is converging.

## Let's guess SW4

Let's power-on SW4 and try to guess its ports role and state before we enable RSTP and verify:

- SW4 has a higher bridge ID (its MAC address is higher) than SW1 and it's directly connected to the root bridge.
- SW4 has a higher bridge ID than SW2.
- All ports in this topology has the same speed thus the same STP cost.

So based on that, SW4 ports role and states are:
- e0/0: root, forwarding. It has a lower port ID than e0/1
- e0/1: alternate, discarding.
- e0/2: alternate, discarding.

> Note: Discarding state is indicated as BLK in `show spanning-tree` command in Cisco IOS.

### Verification

All verifications are the output of `show spanning-tree` on all switches.

**SW1:**
![rstp7](/neve/img/rstp/rstp7.png)

**SW2:**
![rstp8](/neve/img/rstp/rstp8.png)

**SW3:**
![rstp9](/neve/img/rstp/rstp9.png)

**SW4:**
![rstp10](/neve/img/rstp/rstp10.png)

## Final State

![rstp11](/neve/img/rstp/rstp11.png)

And happy convergence everyone !
