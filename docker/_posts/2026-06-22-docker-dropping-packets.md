---
layout: post
title: Docker dropping packets and troubleshooting
description: >
  Troubleshooting docker connectivity...
sitemap: false
hide_last_modified: true
comments: true
---

Docker, you fill my life with joy - until you really dont!
I recently span up a new cloud compute instance and leveraging my previous experiences and following the setup already running on other instances (docker + tailscale), I installed Docker.

Great!

Then tailscale, then certificates to control the instance via portainer.
I then logged off.

A few days later, it was the moment to deploy something. Anything really.

Decided to start with `dockurr/windows-arm`: docker compose deployed, container started... service unreachable.
Maybe the issue is with the container. I tried again, but went for something simple this time: `nginx`.

Browser on mac, on phone, ssh on mac, on other containers: timeout!

###### Claude, come to the rescue!

Not going to lie, after looking and realising nothing was wrong in my configuration I turned to AI asking CLaude to help me debug the issue.

It wasn't easy and it took some time but finally the we (it) found a lead:

~~~bash
sudo nft -a list table ip raw
~~~

providing this precious output
```
table ip raw {
	chain PREROUTING {
		type filter hook prerouting priority raw; policy accept;
		iifname != "br-50a1eccc3fa3" ip daddr 172.29.2.2 counter packets 109 bytes 6848 drop
	}
}
```
This is a custom rule in the raw table. It says: "any packet destined for 172.29.2.2 that doesn't arrive on br-50a1eccc3fa3 gets dropped."

It turns out that recent Docker versions (mid-late 2025 onward) added a hardening feature that inserts raw table anti-spoofing rules for user-defined bridge networks — specifically to prevent external hosts from sending traffic directly to a container's internal IP, bypassing the normal published-port/NAT path. 

[This](https://fivenineslab.com/blog/docker-nftables-port-blocking-priority-chains) is the link Claude provided confirming the findings, with possible solutions.

But the solution it provided me was a different one. Running this command
~~~bash
sudo iptables -L ts-forward -n -v
~~~
this is the result
~~~bash
Chain ts-forward (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  tailscale0 *       0.0.0.0/0            0.0.0.0/0            MARK xset 0x40000/0xff0000
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match 0x40000/0xff0000
    0     0 DROP       all  --  *      tailscale0  100.64.0.0/10        0.0.0.0/0           
    0     0 ACCEPT     all  --  *      tailscale0  0.0.0.0/0            0.0.0.0/0
~~~

And the key part is `0x40000/0xff0000`.

Opening (or creating in my case) the file `/etc/docker/daemon.json` and paste the following

~~~json
{
  "bridge-accept-fwmark": "0x40000/0xff0000" 
}
~~~

Restart docker 

~~~bash
sudo systemctl restart docker
~~~

and that's it... !hat way the rule regenerates correctly from the start!