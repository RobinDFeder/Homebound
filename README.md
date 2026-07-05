# Homebound
 
**A self-hosted residential exit-node and travel-privacy stack.**
 
Route all of a travelling device's traffic back through a home box on a residential ISP line, so the public internet sees an ordinary residential IP — never a datacenter or VPN — while the travelling laptop is made physically incapable of determining its own location and blocked from external agents for highly critial security work.
 
---
 
## TL;DR
 
Commercial VPNs solve the wrong half of the privacy problem: they hide your traffic from the network, but their exit IPs are public, catalogued, and flagged by every reputation database, so you trade "invisible" for "obviously on a VPN." **Homebound** inverts that. A retired i7 running Linux, sitting on a home fibre line, acts as a [Tailscale](https://tailscale.com) exit node. A GL.iNet Slate 7 travel router tunnels to it and hands its clients that clean residential egress. A hardened laptop — wired, radios off — closes the location-inference channels that IP masking alone can't touch.
 
The result: to any website, you are a normal residential subscriber at home. To the venue you're actually sitting in, you are an opaque encrypted tunnel. To the laptop itself, the surrounding world is invisible.
 
---
 
## The core insight: two different "VPN detections"
 
Most people conflate two unrelated mechanisms. Getting this distinction right is the whole design.
 
| | **Protocol detection** | **IP-reputation detection** |
|---|---|---|
| **Who does it** | ISPs, firewalls, censors (on-path) | Websites, anti-fraud, streaming services (at the endpoint) |
| **What they inspect** | Packet shape / DPI signatures of the tunnel | The source IP against reputation databases |
| **Defeated by** | Obfuscation (e.g. AmneziaWG, shadowsocks) | A **residential exit IP** |
| **Relevant here?** | No | **Yes** |
 
The goal — "don't look like a VPN to the sites I visit" — is purely an **IP-reputation** problem. No amount of protocol obfuscation changes it, because the website reads your *exit IP*, and every commercial VPN's exit IPs are known. The only lever that flips the verdict is exiting from an IP that lives in a residential ISP range. That rules out every VPN provider **and every VPS** (datacenter IPs are flagged just as hard). It effectively mandates self-hosting on a residential line — which is exactly what this project does.
 
---
 
## What it is / isn't
 
**It is:** a privacy and location-integrity system for legitimate use — accessing home-network services abroad, avoiding false "VPN user" flags and CAPTCHA walls, keeping your physical location out of apps while travelling, and generally presenting as a normal residential user.
 
**It isn't:** an anonymity system. The exit is your *own* residential IP, so this deliberately reveals "a residential user at your home location" — it does not make you untraceable, and it isn't designed to.
 
---
 
## Architecture — shipped design (single travel router)
 
```
[ laptop ]  wired · WiFi + Bluetooth off · location services off
    │  Ethernet
    ▼
[ GL.iNet Slate 7 ]  Tailscale client · Custom Exit Node = home box
    │  takes venue WiFi/Ethernet as uplink; tunnels out over it
    ▼
[ i7 home box ]  Linux · Tailscale exit node · IP forwarding + masquerade
    ▼
[ home ISP → internet ]   ← the world sees THIS residential IP
```
 
**Who sees what, by design:**
 
| Observer | Sees |
|---|---|
| Websites you visit | Your home box's **residential IP** — no VPN flag |
| The venue / hotel network | An encrypted Tailscale tunnel to a residential IP; not your traffic |
| Your home ISP | Normal home usage |
| The laptop itself | A cable, a residential IP, no surrounding radios to scan |
 
---
 
## How it works — the mechanics that matter
 
**Tailscale exit node.** The home box runs `tailscale up --advertise-exit-node`. Tailscale is a WireGuard-based mesh with its own coordination plane and NAT traversal, so the home box needs **no port forwarding and works behind CGNAT** — a decisive advantage over hand-rolled WireGuard, where inbound reachability and dynamic residential IPs are constant friction. The exit node must (a) have `net.ipv4.ip_forward=1` / `net.ipv6.conf.all.forwarding=1`, and (b) be approved *Use as exit node* in the admin console. Without forwarding it accepts traffic and blackholes it.
 
**Subnet routing + IP masquerade (the subtle part).** A single Tailscale node (a phone) "just works" as an exit-node client because it egresses under its own tailnet identity. A **router** serving a whole LAN is different: its clients' packets carry source IPs (`192.168.8.x`) that mean nothing on the far side of the tunnel. Two things must line up for return traffic to survive:
1. The travel router **advertises its LAN subnet**, and that route is **approved** in the console.
2. The travel router **masquerades** LAN client traffic onto its own Tailscale IP as it enters the tunnel (SNAT on `tailscale0`), so replies have a valid return path.
If the masquerade is missing, you get the signature failure: tunnel up, exit node selected, subnet approved — and still no client internet. On a Linux exit node, also run the client-facing side with `--accept-routes` so the exit node actually installs the return route.
 
**DNS across the tunnel.** By default the travel router uses whatever DNS the venue hands it (a private address like `192.168.0.1` or a hotspot's `172.20.10.1`). The instant traffic tunnels home, that local resolver is on the wrong side and unreachable — names stop resolving while raw IPs still work. Fix: override client DNS on the router to a public resolver (Cloudflare DoT, or `1.1.1.1`/`8.8.8.8`). (Diagnostic: if `http://1.1.1.1` loads but `google.com` doesn't, it's DNS.)
 
**Kill switch (fail-closed).** If the tunnel drops, the router must *block* egress, not silently fall back to the raw venue connection. The [`gl-tailscale-fix`](https://remotetohome.io) enhancement enables the `tailscale0` masquerade **and** a kernel-level kill switch that holds even if the `tailscaled` daemon crashes — a real leak window on memory-constrained travel routers. Verified by dropping the exit node (`tailscale down`) on the home box while a client pings: replies must *stop*, and the public IP must fail to load — never reveal the venue IP.
 
---
 
## Location-privacy layer
 
IP masking is the *weakest* location defence, because a laptop barely uses its IP to locate itself. Signals, strongest first:
 
1. **WiFi BSSID scanning — dominant.** The WiFi radio sees the MAC addresses of every nearby access point and looks that constellation up in Google/Apple databases, placing the device to the building. It ignores the IP and the tunnel entirely.
2. **GPS / cellular** (if present).
3. **IP geolocation** — city-level, and the channel this project masks.
4. **Timezone / locale** — soft hints.
So the laptop is **wired to the travel router and its WiFi + Bluetooth radios switched fully off** — not merely disconnected. With the radio dark, there is no scan data to leak regardless of what any app or OS-level telemetry attempts, which is the only guarantee that survives a work-managed/MDM device. Location services off, timezone pinned to home, and WebRTC blocked in the browser (a WebRTC leak exposes the real public IP *around* the tunnel) close the remaining gaps.
 
---
 
## Alternative architectures evaluated
 
The shipped design is the simplest that meets the goal. Two others were designed out, and are documented here because the trade-offs are the interesting part.
 
### A. Two-router VPN stack — residential exit *with* an obfuscated middle hop
 
Adds a layer: the link to the home box is wrapped in a commercial VPN (Mullvad) so the **local network can't even see that you're reaching a residential home IP**. The exit stays residential — Mullvad is a *middle* hop, not the egress.
 
```
[ laptop ] → [ Slate 7: Tailscale ] → [ upstream router: Mullvad ] → Mullvad → [ home box ] → home ISP → internet
             (downstream, clients here)   (faces the venue)                     (exit node)
```
 
**Layering rule (counterintuitive):** the router the clients connect to runs **Tailscale**; the router facing the venue runs **Mullvad**. Mullvad is the lower transport layer, Tailscale rides on top of it, so the Tailscale tunnel's packets egress *inside* the Mullvad tunnel.
 
**Why two physical routers:** GL.iNet's firmware explicitly refuses to run a WireGuard/OpenVPN client and Tailscale's exit-node routing on the **same** device — both contend for the default route and collide. Splitting them across two boxes is what resolves it.
 
**Costs:** a second router, extra latency (a detour through the VPN — mitigated by choosing a VPN server near the home box), and **nested-WireGuard MTU shrinkage** (Tailscale-inside-WireGuard) that manifests as stalled TLS / hanging pages until the outer client's MTU is lowered (~1320, then 1280).
 
### B. Single-router VPN stack via manual policy routing
 
Force both onto one Slate 7 by dropping into OpenWrt/LuCI and using `fwmark` policy routing to push Tailscale's transport out through the Mullvad interface. **Rejected:** unsupported, fights the stock firmware, breaks on updates, and inherits the same MTU pain — a bad bet for a travel router you depend on.
 
### Comparison
 
| Design | Exit IP | Hides home link from venue | Complexity | Robustness |
|---|---|---|---|---|
| **Shipped (single-box Tailscale)** | Residential | No | Low | High |
| **A — two-router stack** | Residential | Yes | Medium | High |
| **B — single-box policy routing** | Residential | Yes | High | Low (fragile) |
| Commercial VPN alone | Datacenter (flagged) | n/a | Low | High |
 
---
 
## Threat model
 
**Defends against:** websites/anti-fraud flagging you as a VPN; apps IP-geolocating you; the laptop self-locating via WiFi/GPS/cellular; a dropped tunnel leaking the raw connection (fail-closed); *(design A only)* the local network learning you reach a home connection.
 
**Does not defend against:** application-layer identification (logins, cookies, browser fingerprinting); a determined adversary correlating your home IP to your identity (it *is* your home IP); anything upstream of your home ISP. This is location/reputation privacy, not anonymity.
 
---
 
## Verification methodology
 
Run from a client behind the travel router, exit node active:
 
1. **Exit IP** → home ISP's IP and city (not venue, not VPN).
2. **IP reputation** → reads *residential*; VPN/proxy/hosting flags clear.
3. **DNS leak test** → resolves via the home connection.
4. **WebRTC leak test** → does not expose the real local/public IP.
5. **`tailscale status`** → prefers `direct`; `relay` (DERP fallback) means hole-punching failed — functional, slower.
6. **Kill-switch drill** → `tailscale down` on the home box; client traffic must halt, not leak.
7. **Reboot survival** → power-cycle the home box; it re-advertises unattended and clients resume.
---
 
## Engineering notes (debugging journey)
 
The failure that mattered was "phone-as-client works, but LAN clients behind the router get no internet," which was chased through three layers before the real cause surfaced:
 
- **DNS red herring.** The venue/hotspot resolver being unreachable through the tunnel is real and *looks* like total breakage, but it only kills name resolution — raw IPs still route. The `http://1.1.1.1` test separates DNS failure from routing failure and saved a lot of wasted effort.
- **Same-LAN false failure.** Testing with the travel router and the home box on the *same* home LAN introduced hairpin-NAT/subnet-overlap artefacts that don't exist in real deployment. A subnet collision (GL's `192.168.8.x` default clashing with a home router on the same range) will hard-break return traffic. **Test on a foreign uplink (phone hotspot) — it is the real deployment in miniature.**
- **Actual root cause: missing IP masquerade.** The phone worked because it egressed under its own identity; LAN clients failed because the router wasn't SNAT-ing their subnet onto its Tailscale IP on the beta firmware. Enabling the `tailscale0` masquerade (native toggle on newer firmware, or the `gl-tailscale-fix` package on older) plus `--accept-routes` on the exit node closed it.
**Takeaway:** in layered tunnels, isolate by identity (single node vs whole subnet) and by layer (routing vs DNS vs NAT) before touching config. The most confident-looking symptom (DNS) was the least important cause.
 
---
 
## Bill of materials
 
| Component | Role |
|---|---|
| Retired i7 desktop/laptop, Linux (Mint) | Always-on residential exit node |
| GL.iNet Slate 7 (GL-BE3600) | Travel router / Tailscale client (dual 2.5G ports) |
| Tailscale (free tier) | Mesh VPN, NAT traversal, exit-node routing |
| `gl-tailscale-fix` | `tailscale0` masquerade + fail-closed kill switch |
| *(Design A)* Second GL.iNet router + Mullvad | Obfuscated middle hop |
| Ethernet cables | Wired laptop leg (radios-off enabler) |
 
---
 
## Performance characteristics
 
Throughput is bounded by the **slowest link**, and for downloads that is usually the home line's **upload** (traffic arrives at the home box on its download, then is pushed back to you on its upload; residential lines are asymmetric). Symmetric fibre makes the penalty vanish. Latency takes a detour (you → home → destination → home → you), negligible when you and the box share a region. The i7 encrypts far faster than any home line moves, so CPU is never the bottleneck; in the two-router stack, nested-tunnel MTU overhead is the main tax.
 
---
 
## Possible extensions
 
- Native AmneziaWG/obfuscation on the home-box hop for deployment in DPI-censoring networks.
- Multiple geographically distributed home boxes as selectable residential exits.
- Prometheus/Grafana telemetry on the exit node (uptime, throughput, direct-vs-relay ratio).
- Automated health-check + alert if the exit node drops off the tailnet.
---
 
## Key takeaways
 
- **Reputation, not obfuscation.** Looking "normal" to websites is an IP-reputation problem; only a residential exit solves it.
- **Self-hosting beats renting.** A home line gives the one thing no provider can sell cleanly: an unflagged residential IP — and Tailscale removes the traditional self-hosting friction (ports, CGNAT, dynamic IPs).
- **The whole chain, or nothing.** IP masking without killing WiFi scanning, DNS override, and a fail-closed kill switch leaves the real leaks wide open.
