# GRE Tunnel IP Transit: What the 24-Byte Header Costs, Whether Your Carrier Load-Balances on the Inner Packet, and How to Choose a Transit Plan (With a Pre-Order Checklist)

Three numbers and one question decide whether a GRE tunnel over IP transit behaves. The numbers: a 24-byte GRE header, a 64-byte MTU reduction, and an MSS of 1,436 on a standard 1,500-byte path. The question: does the network in the middle hash the inner header when it spreads traffic across ECMP links, or only the outer one?

Get those right and GRE is unremarkable plumbing. Get them wrong and you end up with a tunnel that answers pings but drops every full-size packet, or one that saturates a single member of a bundled transit link while the rest sit idle, and you spend a week blaming the carrier for something you can measure in ten minutes.

Below: the overhead math you can put in a spreadsheet, the hashing question to ask before signing anything, the unidirectional encap/decap behaviour that stops a surprising number of first builds, and five configuration traps that only show up after cutover. The last part covers where to terminate the tunnel if you need a box in Los Angeles, using SixtyNet's current listings as the reference.

> **Quick pick:** SixtyNet's Los Angeles China Unicom-Premium listing is the only one on their price list that mentions a free BGP session setup and 95th-percentile billing: **$2,500/yr**, listed down from $3,000, with an advertised 100M base rate and 3× burst on AS9929. It's the one worth looking at if the GRE endpoints are your own hardware and you want to announce prefixes from the same box. If you only need a stable endpoint with no BGP, LAX-STARTER at **$200/yr** covers 5 TB of traffic. 👉 [See SixtyNet's current Los Angeles plan pricing](https://sixtynet.net/vps-hosting?aff=39)

## All SixtyNet plans currently listed

| Plan | What you get | Price (USD) | Billing | Order |
| --- | --- | --- | --- | --- |
| Los Angeles China Unicom-Premium | 2 cores / 2 GB / 40 GB NVMe, 2G premium network, 5 TB traffic, AS9929, free BGP session setup, 100M base with 3× burst, 95th-percentile billing | $2,500/yr (list $3,000) | Annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |
| LAX-STARTER | 2 cores / 2 GB / 40 GB NVMe, 2G network, 5 TB traffic, DDoS protection | $200/yr (list $240) | Annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |
| LAX-MICRO | 4 cores / 4 GB / 80 GB NVMe, 4G network, 10 TB traffic, DDoS protection | $400/yr (list $480) | Annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |
| LAX-MEDIUM | 8 cores / 8 GB / 160 GB NVMe, 10G network, 20 TB traffic, DDoS protection | $800/yr (list $960) | Annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |
| LAX-LARGE | 8 cores / 16 GB / 320 GB NVMe, 10G network, 40 TB traffic, DDoS protection | $1,600/yr (list $1,920) | Annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |
| Residential IP configurations (2 / 4 / 8 / 16 cores) | 2G to 10G premium network, 5 to 40 TB traffic, residential IP, ports 80 and 443 closed | Shown at checkout | Monthly or annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |
| "Maximum security" residential configurations (2 / 4 cores) | Same shape, rejects all external probes and scans, access only via whitelist or port forward, refunds issued as account balance only | Shown at checkout | Monthly or annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |
| Storage configuration | 4 cores / 4 GB / 300 GB plus 200 GB bonus in RAID 10, 10G network, 10 TB traffic, DDoS protection | Shown at checkout | Monthly or annual | [Check this plan](https://sixtynet.net/vps-hosting?aff=39) |

The four LAX tiers and the China Unicom-Premium tier carry published annual prices. The residential, maximum-security and storage configurations are listed with specs but without a price in the plan summary, so treat those as quote-on-checkout until you see the cart.

Two restrictions apply across the residential lines and matter more than they look: **ports 80 and 443 are closed** on residential IP products for risk-control reasons, and the maximum-security variants state that **refunds go to account balance**, not back to your card.

## What GRE actually adds to every packet

The GRE header is 24 bytes. On an IPv4 path that comes to 64 bytes off the top of a 1,500-byte MTU (40 bytes of IP header plus the 24-byte GRE header), which is why Palo Alto puts the default MSS on a GRE tunnel interface at **1,436 bytes** (1,500 − 40 − 24).

The packet-count effect is smaller than the header size suggests, but it is not zero. Cloudflare's worked example: to deliver 150,000 bytes of content over a 1,500-byte MTU, you need **103 packets without GRE and 105 with it**, because the usable payload per packet drops from 1,460 to 1,436 bytes. That is roughly 2% more packets for identical data.

On flat-rate billing, 2% is noise. On 95th-percentile billing, it is a line item. If your commit is sized to the exact payload figure your capacity planning produced, encapsulation pushes the measured 95th percentile above it every month. Leave a few percent of headroom, or account for the overhead in the commit calculation from the start.

One more trap worth checking on any firewall terminating the tunnel: the MSS Adjustment Size is a separate knob from the MTU. Palo Alto's documentation shows what happens when someone sets it to 300 bytes "to be safe": the MSS falls to **1,176 bytes** (1,500 − 300 − 24). Throughput drops by roughly a fifth, nothing errors out, and the ticket says "the tunnel is slow".

A 1G port is never 1G of payload. With GRE in the path, it is a bit less than that, and the gap widens with every additional encapsulation layer you stack.

## The hashing question that decides whether one tunnel can fill your transit links

This is the part most GRE write-ups skip, and it is the part that determines whether your money buys 10G or 40G.

A GRE tunnel is a point-to-point virtual link between two addresses. Every packet inside it shares the same outer source and destination. If a transit provider's routers hash only the outer header when choosing among equal-cost links, all traffic from that one tunnel collapses onto a single member of the bundle. Four 10G links, one GRE tunnel, 10G of usable capacity, and no amount of tuning on your side changes it.

Cisco's documentation for the 8000 Series describes the capability that fixes this: a transit node distributes incoming GRE traffic evenly across all available ECMP links in the tunnel topology, and the hashing function uses **GRE outer and inner header tuples, including source IP, destination IP, protocol and router ID**, to derive traffic entropy. Inner conversations survive encapsulation and land on different members.

Read that carefully, though. It documents what one platform can do. It says nothing about whether the transit provider you are about to pay has that behaviour enabled on your ports, or whether their policy is to hash outer-only for encapsulated traffic. Ask the question in writing before you sign:

- Do you hash inner headers for GRE-encapsulated traffic on my ingress and egress ports?
- Is that enabled by default, or is it a per-port or per-service setting?
- Does it apply to both directions?

You can verify the answer yourself once the link is up: run several parallel flows with distinct inner five-tuples and watch whether the aggregate exceeds what a single member link can carry. If it plateaus at one member's worth of throughput, you have your answer regardless of what the sales deck said.

There is a second lever entirely on your side. Palo Alto recommends splitting multiple connections to a firewall across **multiple GRE tunnels rather than a single tunnel**, each with its own tunnel interface, for performance and to avoid a single point of failure. More tunnels means more distinct outer flows, which means more entropy for whatever hashing the middle network does, even if it only looks at the outer header.

If you need an endpoint in Los Angeles with a BGP session included, the China Unicom-Premium listing is the one on SixtyNet's price list that mentions free session setup and 95th-percentile billing: 👉 [review the plan and its limits](https://sixtynet.net/vps-hosting?aff=39). Note the wording on their page, "limited BGP technical support", and price your own competence accordingly.

## Unidirectional encap and decap: why your tunnel won't come up

Most engineers assume a GRE tunnel is symmetric: same source, same destination, both directions. On a lot of platforms it is, but not all, and the difference explains a category of builds that never pass traffic.

On Cisco 8000 Series routers, GRE is configured **one direction at a time** through `tunnel mode gre ipv4 encap` or `tunnel mode gre ipv4 decap`. An encapsulation tunnel takes both a source and a destination. A decapsulation tunnel takes **only a source and no destination**, and behaves like a point-to-multipoint tunnel: incoming GRE packets can arrive from any source address, as long as the destination matches the configured tunnel source.

That design has a consequence people hit on their second or third tunnel. Once a source IP address has been used with one decapsulation P2MP tunnel, **it cannot be reused with another decapsulation tunnel**. If you are consolidating several peers onto one box and reuse the loopback address out of habit, the second tunnel simply will not work, and there is nothing in the config that looks wrong.

The scale limits are worth writing down too: GRE and IP-in-IP tunnels support **16 unique source addresses**, and those 16 addresses can back up to **1,000 encapsulation tunnels** or **64 decapsulation tunnels**. If your design needs more than 64 decap tunnels on one box, you need a different box or a different design.

Two related rules from the same documentation: don't create multiple GRE tunnels with the same source/destination pair or the same interface name, and single-ended tunnels (source only or destination only) still need source-destination pairs that are unique across all encap and decap tunnels.

Do not generalise the unidirectional behaviour. Cisco IOS XE on Catalyst 9000 configures ordinary bidirectional point-to-point GRE tunnels with VRF membership, and Red Hat's NetworkManager builds standard bidirectional GRE and GRETAP interfaces with `nmcli`. The unidirectional split is a platform trait, not a property of GRE.

## Five things to check before you order transit for a GRE tunnel

**1. TTL handling.** By default, the TTL of an incoming packet is decremented by one before GRE encapsulation on Cisco 8000. A packet that arrives with TTL 1 becomes 0, gets dropped, and generates an ICMP message back to the source. The `tunnel ttl disable` command, introduced in Release 7.3.2, keeps the original TTL instead. If traceroute output through the tunnel looks truncated or you see odd ICMP unreachables from low-TTL sources, this is the reason.

**2. Unique tunnel endpoints.** Covered above, but it applies to both directions and to single-ended tunnels. Duplicate source/destination pairs are a silent failure.

**3. VRF membership, and the two VRFs that mean different things.** On IOS XE, the VRF attached with `tunnel vrf` governs routing of the **outer** IP packet, the one carrying the tunnel to a physical interface. The VRF attached with `ip vrf forwarding` governs the **inner** packet after it exits the tunnel. Both ends must sit in the same VRF. VRF-aware GRE does not support ISIS, does not support fragmentation of multicast packets through a multicast tunnel, and **does not support keepalive**.

**4. MACsec on the interconnect.** There is a documented topology where Host1 connects to Switch1, Switch1 to an ISR, the ISR to Switch2, and Switch2 to Host2, with GRE transit traffic crossing between two Catalyst 9000 switches. Enable MACsec on the interconnect links through the next-hop device and **the GRE traffic fails**. The documented fix is to disable MACsec between the first switch and the next hop. If you are rolling out link encryption and GRE at the same time, roll them out separately.

**5. The firewall in the middle.** Palo Alto's NGFW has a list of restrictions that matter if the GRE endpoint is a firewall: GRE tunnel interfaces don't support QoS, a single interface can't be both a GRE tunnel endpoint and a decryption broker, NAT isn't supported between GRE tunnel endpoints, and you can route a GRE tunnel into an IPsec tunnel but not GRE or IPsec back into GRE. There is also a policy subtlety: traffic the firewall encapsulates and sends out a GRE tunnel doesn't create a session, so it needs no rule, but **inbound** GRE does create a session and gets the full policy treatment applied to both the GRE IP header and the inner traffic. If you've written your own intrazone deny rule, or the tunnel interface sits in a different zone from the ingress interface, you must explicitly allow GRE.

If you're not terminating the tunnel on the firewall and only want to inspect what passes through it, Palo Alto's guidance is to use Tunnel Content Inspection rather than build a GRE tunnel you don't need.

## GRE does not encrypt anything

This is stated plainly in Red Hat's documentation for both IPIP and GRE: data travelling in the tunnel is not encrypted, so the tunnel should only carry traffic that is already encrypted, such as HTTPS. GRETAP, the layer-2 variant that encapsulates Ethernet frames, has the same property and should run on top of a VPN or another encrypted connection.

Palo Alto's guidance for connecting to another vendor's network is to set up an IPsec tunnel, and to use GRE **only if GRE is the only point-to-point mechanism that vendor supports**. If the remote end requires traffic to be GRE-encapsulated before IPsec encrypts it, which some implementations require for multicast, you configure GRE over IPsec instead.

The practical version: GRE gives you a virtual point-to-point link across a third-party network and carries any layer-3 protocol with a valid EtherType. It gives you nothing else. Confidentiality, integrity and peer authentication are somebody else's job.

## Who shouldn't buy GRE-shaped transit at all

If your only requirement is an encrypted link between two sites, IPsec by itself is less work: no 24-byte header, no MTU reduction to chase, no second encapsulation layer, and a mature story for key exchange. Adding GRE underneath buys you protocol transparency and multicast support you may not need.

If the peer at the other end only speaks IPsec, GRE isn't an option regardless of what you buy.

And if you're shopping for a real transit port with a committed rate in the tens of gigabits, SixtyNet's price list isn't that product. The listings are VM-shaped, and the China Unicom-Premium entry advertises a 100M base rate with 3× burst on 95th-percentile billing. That's fine for a GRE endpoint, a route reflector, or a small edge box. It isn't a 10G commit with a cross-connect and a looking glass.

## Questions that come up before and after cutover

**How much extra bandwidth does GRE consume?**
About 24 bytes of GRE header plus the outer IP header per packet. On a 1,500-byte MTU the MSS lands at 1,436, which means roughly 2% more packets for the same payload, and slightly more bytes on the wire once the outer header is counted. Budget a few percent of headroom on any commit you're sizing.

**Can one GRE tunnel use multiple transit links?**
Only if the provider hashes inner headers for encapsulated traffic. Otherwise the tunnel maps to one member of the ECMP group. Multiple tunnels, each with its own source/destination pair, give the hashing function more to work with and also remove a single point of failure.

**Why does my decap tunnel have no destination configured?**
Because on platforms that split encap and decap, the decap side is point-to-multipoint by design: it accepts GRE from any source as long as the destination matches its own source address. The catch is that a given source IP can only be claimed by one decap tunnel.

**Should I turn on GRE keepalive?**
VRF-aware GRE tunnels don't support keepalive at all, so if you're running VRF membership, that question is settled. On older IOS releases, keepalive was introduced for point-to-point GRE tunnels and could shut the tunnel interface when probes failed; don't assume that behaviour exists on your platform or your release. Check the documentation for the exact model and version you're deploying.

**Does SixtyNet support GRE on their network?**
Their plan pages document AS9929 routing, a free BGP session setup and 95th-percentile billing. They don't document GRE handling, MTU policy, or whether protocol 47 is filtered anywhere in the path. Ask before you order, and ask specifically about protocol 47 and the MTU on the access port.

## Picking a plan against your actual design

If you're terminating a single point-to-point GRE tunnel and don't need BGP, LAX-STARTER at $200/yr with 5 TB of traffic covers the job, and the DDoS protection included on the LAX line is worth having on a tunnel endpoint that advertises its address.

If you're running several parallel tunnels to spread load and want to announce your own prefixes, the China Unicom-Premium listing is the only one on the page with BGP session setup and 95th-percentile billing mentioned, at $2,500/yr annual. Read "limited BGP technical support" as exactly that, and keep someone on your side who can debug a session.

If what you actually need is a residential-looking egress rather than a transit endpoint, none of the GRE material above applies to your problem, and you should know that ports 80 and 443 are closed on those plans.

Before you check out, confirm two things in writing: whether protocol 47 passes unfiltered on the access port, and what MTU the port presents so you can set your tunnel MTU and MSS clamp correctly the first time rather than the third.

👉 [Compare the SixtyNet Los Angeles plans and current annual pricing](https://sixtynet.net/aff.php?aff=39)
