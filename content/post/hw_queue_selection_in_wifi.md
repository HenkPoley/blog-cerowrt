+++
date = "2016-04-22T18:02:58+01:00"
draft = true
tags = [ "wifi", "bufferbloat" ]
title = "802.11e hardware queue selection in wifi"
description = "What's the right way to use the Voice, video, best effort and background queues?"
+++

```
NOTE: This is more or less a refinement of the wifi QoS architecture that
Andrew McGregor and I came up with in late 2011. I can't find the
document we distributed, so this is the best I can recall.
```

It is a refinement on how [per_station_queuing](/post/per_station_queuing) could work
in wifi.

{{< highlight C >}}
struct {
    u8 vo_packets;
    u8 vi_packets;
    u8 vi_packets;
    u8 vi_packets;
};

{{< /highlight >}}

There is another, possibly better way to use this, counting bytes,
rather than packets, but the structure is larger. You can't even use 16
bit values for it as 802.11ac can wedge over a megabyte into a given txop.

{{< highlight C >}}
struct {
u32 vo_bytes;
u32 vi_bytes;
u32 vi_bytes;
u32 vi_bytes;
};
{{< /highlight >}}

Either way, it's simpler to just use

```
u8 queues[4];
```

because you need to consistently use a diffserv mapping table in and
out... Arguably you could use 5 queues (for multicast) or 10 or more,
but, ah, well. In what follows I am assuming that multicast and
management frames have already gone into it's own queue before being
further classified

As packets enter the per-station queues, their dscp values are recorded here
and also feed into a scoreboard that maps into what hw queue "should" be used
for this station barring conflicting traffic.

On dequeue...

You have to carefully account for all drops throughout the system. As
codel is probably the largest "dropper", and buried deeper in the stack,
it too needs to pass up the fact that a drop of a certain kind of packet
happened otherwise our counters get out of wack.

This turns out to be easier than you'd think. With ecn capability, codel
has to look at the dscp on dequeue anyway, so, drop or not, you merely
decrement the relevant counter. If you hit zero, remove the station from
the relevant scoreboard. There *are* several places in codel where drops
can happen, an example of the relevant code would be changing this:

```
Fixme
```

to this:

```
Fixme
```

## Scheduling selection

You have a couple of ways to select the next station to schedule.
You are typically aiming for airtime fairness. Airtime Deficit Round
Robin is a good way to think about it, with, perhaps the DRR++-like
method fq_codel gives to "sparser" stations.

Adding in 802.11e scheduling for each station complicates matters significantly,
and further, the behavior of the 802.11e queues differ between versions
of successor standards.

Gang scheduling


## Promotion/Demotion

This is where the behavior of an AP and a Client could, and perhaps,
should, differ.

A well behaved client should never try to have more than two TXOP
requests outstanding, no matter the queue.

A well behaved AP should never try to have more than two TXOP requests
outstanding per station, and furthermore should never have more than one
when a background

*you could* go further with flow identification - it would be ok to have
 a background TXOP pending with a set of X flows and a foreground one
 with a set of Y, so long as they are different.

An AP could advertise WMM capability to it's clients, so they can
optimize their traffic, but should be *very* careful to not give clients
more than they need. And it's just simpler for the AP to try and
maximize airtime fairness overall

## There are still problems!

As noted in [on the role of network control in wifi](/post/cs6_on_wifi),
the fast queue/slow queue distinction in fq_codel is still not enough to
ensure that VO packets get the airtime they need, thus I favor something
more like an explicit contract, admission control, and a separate queue
entirely for packets in VO. It should be very short, drop from the head,
and yet not grab too much airtime overall compared to the other queues.

Say: no more than 1 servicing per station per 10ms.

The VI queue's bounds are "tighter", and in a world with ever more wifi
devices in it, maximizing the use of what TXOPs you can get and making
for maximal numbers of TXOPs - devices making use of the VI queue more
effectively will "win" more often against those that are not.

If only we do it more right.

...
