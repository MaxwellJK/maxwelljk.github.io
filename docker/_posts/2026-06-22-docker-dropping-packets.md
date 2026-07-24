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

Not going to lie, after looking and realising nothing was wrong in my configuration I turned to AI asking Claude to help me debug the issue.

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

<a href="https://fivenineslab.com/blog/docker-nftables-port-blocking-priority-chains" target="_blank">This</a> is the link Claude provided confirming the findings, with possible solutions.

But the solution it provided me was a different one... and guess what?? it didn't work!
Not completely at least: it was going in the right direction for sure, but it wasn't consistent and mostly, it wasn't permanent.

It turns out that there is a much quicker and standard solution: allowing direct routing in docker.

Easy peasy:
add `--allow-direct-routing` to `docker.service`
~~~bash
sudo systemctl edit docker.service --full
ExecStart=/usr/bin/dockerd --tlsverify --tlscacert=... --tlscert=... --tlskey=/home/ubuntu/... -H fd:// -H=tcp://0.0.0.0:2376 --allow-direct-routing
sudo systemctl daemon-reload
sudo systemctl restart docker
~~~

or allow direct routing via `daemon.json` config file

~~~bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "allow-direct-routing": true
}
EOF
sudo systemctl restart docker
~~~

One of them, not both.
Even after reboot, solution is persisted and docker services are accessible by their IP!