---
date: "2023-07-23"
title: "Homelab v5: Going Down The Network Stack"
---

<iframe src="https://toot.kif.rocks/@ruhrscholz/110748487026100881/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://toot.kif.rocks/embed.js" async="async"></script>

## The Old Setup

To properly appreciate the upcoming network shenanigans I'll have to give you a bit of context on my current setup first. You see, there is a server VLAN at home where everything is hosted, using normal, local IPv4 addresses. Traffic comes in via a Hetzner VPS which runs a reverse proxy (Caddy), that reverse proxy talks to the local network via a WireGuard VPN between VPS and my router. The reasoning behind an ingress VPS was 1. so I don't have to deal with dynDNS and 2. so I don't leak my private IPs that easily. So far so good.

## Why

My network annoyed me. As it always does.

At first I wanted to access my Nextcloud from my home network. That required me to set up an additional ingress node in my local network, because otherwise all the traffic would go to the Hetzner DC and back. Which is definitely slower than my in-home gigabit. So I did that. Then I remembered that that local ingress node needed certificates to function. Caddy supports the ACME HTTP-01 challenge without issues, *but* that requires the ingress to be publicly reachable which - given that it sits in my local network only -- it is not. Thus I had to set up API access for my domains, which required me to move to a better DNS hosting provider, it was a whole ordeal.

Also, to make all of this locally reachable I had to set up local DNS as well, so that the requests actually arrive at the local ingress.

After that, I wanted to add a service. I believe it was Tautulli. So I had to go through the whole setup again, public DNS, local DNS, public ingress, local ingress, a pain overall.

So I decided to move all problems down the OSI stack which actually solves a lot of things. After all, why use DNS to handle different routing when there is, well, routing.

But where do I get all the IPv4 addresses for that? Do I need a HTTP reverse proxy again? No. Just use IPv6. I wanted to lean on IPv6 more after all, and that felt like the perfect opportunity.

## The Plan

Re-do the network. Completely. Again. But with publicly routable IPv6 addresses for each and every service. I sadly only get assigned a dynamic IPv6 prefix from my ISP (because Germany), but I got another place where I can get IPv6 from: Hetzner (or any other VPS provider). An additional advantage of that option was that I 1. don't leak my local addresses (as before), and 2. that I have full control over reverse DNS entries. I already used the VPS as ingress proxy, and it already had a WireGuard connection to my local network. The ideal starting point. Apart from that, I stumbled upon a really great blog article related to all of that, I'll link it later. That was enough of a plan for me to start tearing down the old setup and to get to work. 

I'll spare you some parts of the journey but rather explain the end result and the decisions behind it.

### Example - IPv6

Let's build a mental image of how a requests to one of my services (say Nextcloud) would actually reach my Nextcloud server. $USER1 types `nextcloud.merlins.domain` into their browser. In a perfect world this resolves via `AAAA` record to `2a01:4f8:xxxx:xxxx:1::7`, where `2a01:4f8::/32` is Hetzner's prefix and `2a01:4f8:xxxx:xxxx::/64` is the prefix Hetzner assigned to my VPS. Now for the fun part. While the whole `/64` gets forwarded to my VPS, it actually listens only on `2a01:4f8:xxxx:xxxx::1/128`. For all other packets that arrive, it just acts as a router (simply enable routing in Linux via `sysctl -w net.ipv4.ip_forward=1` and/or `sysctl -w net.ipv6.conf.all.forwarding=1`).

Flashback to the WireGuard tunnel between VPS and my home network. In WireGuard, you can specify which IP ranges shall be routed through the interface. My config looks like this:

```ini
[Interface]
PrivateKey = HAHALOLNO1
Address = 169.254.1.1/32, fe80::1/128
ListenPort = 51820

[Peer]
PublicKey = HAHALOLNO2
AllowedIPs = 2a01:4f8:xxxx:xxxx:1::/80
```

For the connection between the two WireGuard instances we just use link-local addresses, because that's what they're there for. Note however the IPv6 addresses, `2a01:4f8:xxxx:xxxx:1::/80` is the subnet I chose to use for my servers at home. That leaves me with a maximum of `2^(128-80) = 281 474 976 710 656` servers (per location) -- should be enough for now. Also, this way I can use `2a01:4f8:xxxx:xxxx:2::/80`, `2a01:4f8:xxxx:xxxx:3::/80` and so on for other "DCs", i.e. a friend's house or something.

Back to the packet, Linux sees it arriving, and just pushes it into the WireGuard tunnel, to eventually reach my local router. From there on it get routed as normal to the Nextcloud VPS which has `2a01:4f8:xxxx:xxxx:1::7` as its IP address. Nextcloud does whatever slow PHP and bad decisions do and eventually produces a new IP packet with the IPv6 address of $USER1 as its destination. The local router (MikroTik RouterOS in my case) has a firewall rule that marks any packets coming from the server VLAN (and not destined for my private home VLAN) with a special routing mark. 

```console
[admin@cr1.merlins.domain] > /ipv6/firewall/mangle/print 
Flags: X - disabled, I - invalid; D - dynamic 
 0    chain=prerouting action=mark-routing new-routing-mark=extip1_rt passthrough=no src-address=2a01:4f8:xxxx:xxxx:1::/80 dst-address=!2001:yyyy::/32 log=no log-prefix="" 
```

From there on, a different routing table is being used that routes any packets with that specific routing mark back through the VPS.

```console
[admin@cr1.merlins.domain] > /ipv6/route/print detail where dst-address=::/0 
Flags: D - dynamic; X - disabled, I - inactive, A - active; 
c - connect, s - static, r - rip, b - bgp, o - ospf, d - dhcp, v - vpn, m - modem, g - slaac, y - bgp-mpls-vpn; H - hw-offloaded; + - ecmp 
   DAd + dst-address=::/0 routing-table=main gateway=fe80::3a10:d5ff:feb2:71b8%wan1 immediate-gw=fe80::3a10:d5ff:feb2:71b8%wan1 distance=1 scope=30 target-scope=10 
         vrf-interface=wan1 

 0  As   dst-address=::/0 routing-table=extip1_rt gateway=fe80::1%extip1_wg immediate-gw=fe80::1%extip1_wg distance=20 scope=30 target-scope=10
```

The normal route is at the top (which routes through my ISP-given modem), and the special server rule with the `extip1_rt` routing table is at the bottom. `fe80::1%extip1_wg` is link-local address of the VPS.

The packet goes the whole way back through the tunnel, through Hetzner, adding about 30-35ms of RTT overall, and eventually ends up at whoever wanted to visit Nextcloud. Happy end. 

### Example - IPv4

The world, however, isn't perfect, and thus there are people who only have IPv4 addresses. For example me when I have to use eduroam. This imaginary $USER2 now tries to get some cat photos off of Nextcloud. The DNS A record resolves to the public IPv4 address of my VPS.

This is where the magic happens. I don't know what I searched back then but I stumbled upon a great blog, with this great article: [How I'm Using SNI Proxying and IPv6 to Share Port 443 Between Webapps](https://www.agwa.name/blog/post/using_sni_proxying_and_ipv6_to_share_port_443) by Andrew Ayer. In there, he presents his great tool [snid](https://github.com/AGWA/snid). I couldn't believe it when I read the article and the tool description, but it solves all my problems in the most elegant way. You can read his whole article, I highly recommend it, but in short, (in my configuration, ) the tool listens on IPv4 on ports you specify like `0.0.0.0:443`, checks the TLS SNI extension, does a AAAA DNS request and forwards the whole TLS stream to its new destination. I quickly wrote a systemd service file:

```ini
#merlin@ir1:~$ cat /etc/systemd/system/snid.service
[Unit]
Description=snid
Documentation=https://github.com/AGWA/snid
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=exec
User=snid
Group=snid
ExecStart=/usr/local/bin/snid -listen tcp:443 -listen tcp:8006 -listen tcp:5432 -mode nat46 -nat46-prefix 64:ff9b:1:: -backend-cidr-cidr 2a01:4f8:xxxx:xxxx:1::/80
TimeoutStopSec=5s
LimitNOFILE=1048576
LimitNPROC=512
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

In this configuration, snid runs as non-root user (hence the extra capabilities needed to listen on privileged ports) and listens on ports 443 (HTTPS), 8006 (Proxmox), and 5432 (Postgres). It then does a sneaky little bit of NAT46, embeds the source IPv4 address into the end of the NAT46 prefix `64:ff9b:1::`, and sends the packets on their way. We do not need to add the NAT46 prefix to the VPS WireGuard config, as that is the new source address, not the destination address. Rather we have to configure the VPS to listen on that whole prefix for the packages that our Nextcloud server eventually sends back:

```console
merlin@ir1:~$ cat /etc/network/interfaces
auto lo
iface lo inet loopback
iface lo inet6 loopback
	up ip route add local 64:ff9b:1::/96 dev lo

auto eth0
iface eth0 inet static
	address 142.132.xxx.xxx/32
	gateway 172.31.1.1
iface eth0 inet6 static
	address 2a01:4f8:xxx:xxx::1/128
	gateway fe80::1
```

As soon as the loopback interface is up, we add a route to redirect all the NAT46 traffic back there.

All of this works beautifully.

## Advantages

This whole new setup comes with a bunch of advantages, to name a few of them:

### DNS

Since we moved down one OSI layer, we use IP routing for routing instead of DNS based routing. This in turn means, that all IP addresses throughout the whole network are the same, no matter where we want to access anything from. This, in combination with the fact that there are no internal IPv6 addresses in this setup, allows us to just put a single authoritative DNS server into the cloud (I use Hetzner's free DNS) and be done with everything. Even reverse DNS works perfectly this way. "It's always DNS" -- no more.

### ACME

We do not have a single TLS terminating proxy anymore. Instead, all TLS connections are being terminated at their respective services (again, so that we can take different routes to them). A neat little detail about Let's Encrypt or ACME in general is, that it always tries IPv6 first. This is important, since the ACME HTTP-01 challenge works over plain HTTP, which does not have SNI headers and thus can not be forwarded via snid. But since IPv6 doesn't go through snid but rather to the server directly, everything works as it should. Services that can use ACME themselves (like Proxmox) can set themselves up, for everything else there's Caddy.

<iframe src="https://toot.kif.rocks/@ruhrscholz/110720251329511174/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://toot.kif.rocks/embed.js" async="async"></script>

No need for DNS-01 challenges anymore.

### Adding services

Adding services has become easier as well. I just have to spin up the VM, assign the IP address and configure DNS. During install, the hostname get automatically configured via reverse DNS. Snid pulls from DNS the moment it processes a request, and that's it.

<iframe src="https://toot.kif.rocks/@ruhrscholz/110719939842806708/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://toot.kif.rocks/embed.js" async="async"></script>

### No More Port Forwarding

Since every server has a publicly routable IPv6 address, I can just SSH into them from anywhere, without jumpbox or port forwarding or anything. The same holds true for other stuff: Multiple Minecraft servers? Just give them different IPs, no custom ports necessary anymore. Databases, proprietary protocols, everything is directly routable -- I just have to be careful with the firewall now as not to open up my Windows VMs to the whole world.

### Tunnel Everything TLS

Last but not least: Snid can tunnel everything that uses TLS with SNI headers. Mind you, this is only necessary for IPv4-only clients, but has saved me a few times since I set it up. Most recent example: Postgres uses TLS with SNI. I had to access my database from eduroam, and it worked perfectly, without any need for configuration.

## Some Pitfalls

It wouldn't be a proper networking setup if everything worked perfectly fine -- there are some things I had to pay attention to.

### Pitfall 1: Need For IPv4

A lot of stuff still needs IPv4. In the beginning my server subnet was IPv6-only but that quickly turned out to not be feasible in the long run. Some examples include:

- Plex sends some kind of ping every few minutes or hours to plex.tv to show the system that your server is still alive. That was a particularly annoying issue, as my local Plex server seemed to work fine, but would just disappear completely after a few hours, even from its own web interface. Took some time to find that issue.
- GitHub. When you set up servers, you sooner or later have to pull something from GitHub. Be it Nextcloud plugins, Uptime Kuma, Tautulli, everything needs GitHub. Guess which VCS provider doesn't have IPv6 enabled on their apex domain? Exactly. But at least they offer IPv6 for GitHub Pages on apex domains, that's why I moved my sites there from Netlify and Vercel.

<iframe src="https://toot.kif.rocks/@ruhrscholz/110719187528314497/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="400" allowfullscreen="allowfullscreen"></iframe><script src="https://toot.kif.rocks/embed.js" async="async"></script>

- Akkoma. I tried to set up Akkoma (same issues with Pleroma). First of all there were problems with the connection to my (IPv6-only) database. By default, Akkoma doesn't check AAAA records. That issue has been solved by checking [Pleroma's GitLab issues](https://git.pleroma.social/pleroma/pleroma/-/issues/214#note_9438). I was happy to have a working Fedi server, but of course there were troubles again. Akkoma, being an ActivityPub server, has to connect to other instances to actually do useful stuff. But exactly like with the database connection, it only checks for A records. Meaning federation was completely broken, in a way that could not be solved by NAT64. Checking the source code for an easy fix is somewhere on my TODO list.

You see, there is no way to not use IPv4. Sadly. I could think of two workarounds:
1. Enable IPv4 completely. I did that with DHCP and very short lease times, so that I don't accidentally use an IPv4 address permanently. IPv4 is meant to be used only for outgoing traffic. This setup adds the appropriate MASQUERADE and NAT rules to the VPS, so that outgoing traffic can be NATed there.
2. Use any public NAT64 provider. I chose [https://nat64.net](https://nat64.net) simply because they had the lowest latency. I don't need much (or any) bandwidth, since, again, this is only used for emergency IPv4 requests to GitHub, Plex and friends. Not a single big transfer happens over NAT64.

The first approach works, and solved the Akkoma federation issues, but for some reason it was slow as hell and constantly broke on larger file transfers like pulling a git repo. That's why I ultimately switched to NAT64, and am considering hosting it myself. But that's something to do in the future.

### Pitfall 2: Non-TLS traffic

One other problem I noticed is when I simply type `ssh server.merlins.domain`. There's a fifty fifty chance of connecting to the right host. Why? The A records point to the VPS (so that incoming traffic can be NAT46ed there). It basically boils down to "IPv4 and IPv6 record don't point to the same machine". I could solve that one by adding IPv4-only or IPv6-only DNS records specifically for that purpose, but for now I simply type `ssh -6 server.merlins.domain`.

### Pitfall 3: Local routing

The last problem has to do with the dynamic IPv6 prefix given to me by my ISP. As noted earlier, I added a firewall rule to route all traffic that comes from the server subnet and is destined for everything but my home network through Hetzner. The issue is - what is my home network? That one changes every few days, and I haven't found a way to use that prefix in my firewall rules automatically. Current solution: Route everything but my ISP's owned addresses through Hetzner. That may lead to issues when trying to access stuff from other networks that use my ISP, but since they aren't that big, that risk is really low. Yet another thing to properly fix in the future.

Even with those pitfalls, I am pretty happy with my new setup.

## Bonus: Hardware updates

Small little hardware bonus: My main (ZFS-backed) hypervisor has had 32GB of memory, which, for a ZFS-backed hypervisor, isn't that much. So I ordered an additional 64GB (4x 16GB) DDR4 ECC online for 96GB in total. They arrived, I shut the server down, plugged the DIMMs into the motherboard, and booted back up. Only to be greeted by a wonderful "Memory Mismatched" error. Maybe I mixed LRDIMM and RDIMM? I opened the server back up to compare the DIMMs. Turns out, the company where I ordered the RAM packed the wrong sticks. Instead of 64GB in total they sent me 4x 64GB of RAM. So now I'm running on 256GB instead of 32GB, and am hosting so much more stuff. Enough for me to run out of time to install more VMs.
