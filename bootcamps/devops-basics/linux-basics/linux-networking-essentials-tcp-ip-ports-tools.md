# Lesson 10: Linux Networking Essentials – TCP/IP, Ports & Tools

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-9 (terminal, filesystem, permissions, find/grep/piping, systemd, users/sudo, packages, process management, scheduled tasks)

## Learning Objectives

By end of lesson student can:
- Explain what an IP address is, the difference between the network and host portion, and read CIDR notation
- Distinguish a switch from a router, and explain what happens differently on each side of that boundary
- Explain what NAT is and why most home/office/cloud-private networks depend on it
- Inspect and configure network interfaces with the `ip` command
- Read and reason about a routing table, and identify the default gateway
- Explain TCP/UDP port ranges and identify what a handful of well-known ports are used for
- Find which process owns a listening port with `ss`/`lsof`
- Test connectivity and diagnose reachability with `ping`, `traceroute`/`mtr`, `curl`, `wget`
- Query DNS records with `dig`/`nslookup`, and explain the roles of `/etc/hosts` and `/etc/resolv.conf`
- Read and modify a basic Ubuntu netplan configuration

## Topics

- Network essentials: IP addresses, network vs host portion, subnet masks, CIDR notation, private vs public ranges
- Network overview: LAN vs WAN, switches (Layer 2) vs routers (Layer 3), how a packet actually gets from your machine to the internet
- NAT: why it exists, source NAT vs destination NAT, private-range translation at the gateway
- Network interfaces: `ip addr show`, `ip link set up/down`, `ip addr add/del`
- Routing: `ip route show`, `ip route add`, default gateway, `ip neigh`
- Ports & sockets: TCP/UDP port ranges (well-known 0-1023, registered, ephemeral 49152+); common ports: 22/80/443/3306/5432/6379; `ss -tuln`, `lsof -i`
- Connectivity tools: `ping` (`-c`, `-i`), `traceroute`/`mtr`, `curl`, `wget`
- DNS tools: `dig` (A, MX, NS, CNAME, PTR), `nslookup`; `/etc/hosts`, `/etc/resolv.conf`
- Ubuntu network config: `/etc/netplan/` structure, `netplan apply`, `netplan try`

## Concepts

### IP addresses: network vs host portion, and CIDR notation

An **IPv4 address** is a 32-bit number, written as four decimal numbers 0-255 separated by dots (e.g. `192.168.1.50`) — each number represents 8 bits (an "octet"). Every address is conceptually split into two parts: a **network portion** (identifies which network the address belongs to) and a **host portion** (identifies one specific device within that network). Where that split happens is described by a **subnet mask** or, more commonly today, **CIDR notation** — a `/N` suffix stating how many leading bits belong to the network portion.

`192.168.1.50/24` means: the first 24 bits (`192.168.1`) are the network portion, the remaining 8 bits are the host portion — so this address belongs to the `192.168.1.0/24` network, which can address 256 hosts (0-255 in that last octet, though `.0` and `.255` are reserved as the network address and broadcast address respectively, leaving 254 usable). A smaller number after the `/` means a *larger* network (more host bits available); `/24` is a common "one office/home LAN" size, `/16` is much larger (65,536 addresses), `/32` addresses exactly one single host.

Two devices can only talk directly (without a router in between) if they're on the same network — meaning their addresses share the same network portion. This is precisely what a device checks before deciding whether to send a packet straight to another device on the local network, or hand it off to a router instead.

Certain IPv4 ranges are reserved as **private** — not routable on the public internet, free for anyone to reuse inside their own network:

| Range | Common use |
|---|---|
| `10.0.0.0/8` | Large private networks (cloud VPCs, big offices) |
| `172.16.0.0/12` | Private networks, common default for Docker/container networking |
| `192.168.0.0/16` | Small private networks (home routers, small offices) |

Every other address (roughly) is **public** — globally unique and routable across the internet. This distinction is exactly why NAT (below) exists.

### Network overview: LANs, switches, and routers

A **LAN** (Local Area Network) is a set of devices on the same local network, typically physically close together (one office, one home, one cloud subnet) — they can reach each other directly. A **WAN** (Wide Area Network) connects separate LANs together across greater distances — the internet is the largest WAN.

Two devices sit at the boundary of "local" vs "beyond local," and they operate at different layers:

| Device | Layer | What it does |
|---|---|---|
| **Switch** | Layer 2 (data link) | Connects devices *within* the same LAN; forwards traffic based on MAC (hardware) addresses; has no concept of IP networks or "outside" |
| **Router** | Layer 3 (network) | Connects *different* networks together; forwards traffic based on IP addresses, using its routing table (as seen with `ip route`) to decide where each packet goes next |

A practical mental model: your laptop plugs into a switch (or a switch is built into your home router/access point) along with other local devices — the switch handles traffic *between* those local devices. When a device needs to reach something outside the local network, the switch simply forwards the packet toward the router, and the router — consulting its own routing table, the same concept covered in "Routing and the default gateway" below — decides the next hop toward the destination. A packet leaving your laptop for a website typically crosses a switch, then a router (often several routers, hop by hop) before ever reaching the destination.

### NAT: sharing one public address across a private network

**NAT (Network Address Translation)** rewrites the source and/or destination IP address (and often port) of a packet as it passes through a device, typically a router/gateway. It exists because private IP ranges (above) aren't routable on the public internet — a home LAN full of `192.168.1.x` devices can't have replies routed back to them directly, since that address means nothing outside the LAN.

The most common form is **source NAT** (also called masquerading): when a private-address device sends a packet to the internet, the gateway rewrites the packet's source address to its own public IP before forwarding it, and remembers the mapping so the reply can be translated back and delivered to the correct original device. This is exactly how an entire household or office shares one public IP address across many devices simultaneously — this is also the `nat` table concept referenced when `iptables` was introduced in Lesson 11.

**Destination NAT** (also called port forwarding) works the other direction: an incoming request to the gateway's public IP on a specific port gets rewritten to be delivered to a specific internal private-address device instead — how you'd expose a single internal server to the outside world without giving it a public IP of its own. `net.ipv4.ip_forward` (Lesson 9's kernel parameters) must be enabled on any Linux box acting as this kind of gateway/router, since it controls whether the kernel is willing to forward packets between interfaces at all.

### The `ip` command family

Modern Linux networking is managed through the `ip` command (from the `iproute2` package), which replaced the older `ifconfig`/`route` tools. It's organized into **objects**: `ip addr` (IP addresses), `ip link` (network interfaces themselves), `ip route` (routing table), `ip neigh` (neighbor/ARP table). Each object supports subcommands like `show`, `add`, `del`, `set`.

An **interface** is a network device the kernel can send/receive packets through — a physical NIC (`eth0`, `enp0s3`), a virtual bridge, or the loopback interface `lo` (always present, always `127.0.0.1`, used for a machine to talk to itself).

### Routing and the default gateway

The kernel keeps a **routing table**: a list of rules mapping destination network ranges to "send it out this interface, via this next-hop." When your machine wants to reach an address, it checks the routing table for the most specific matching entry.

The **default gateway** is the catch-all entry (destination `0.0.0.0/0`) — where to send traffic when no more specific route matches, typically your router. `ip route show` lists the table; the line starting `default via <IP>` identifies the default gateway.

`ip neigh` shows the ARP/neighbor table — the kernel's cache mapping IP addresses on the local network to their hardware (MAC) addresses, learned as the machine talks to its neighbors.

### Ports and sockets

A **port** is a 16-bit number (0-65535) that lets multiple network services share one IP address — the OS routes incoming traffic to the right process based on which port it arrived on. A **socket** is the combination of protocol + local IP + local port + remote IP + remote port that uniquely identifies one active connection.

Port ranges are conventionally divided into three bands:

| Range | Name | Use |
|---|---|---|
| 0-1023 | Well-known ports | Reserved for standard services; binding one usually requires root |
| 1024-49151 | Registered ports | Assigned to specific applications by convention, not enforced |
| 49152-65535 | Ephemeral (dynamic/private) ports | Assigned automatically as the *source* port for outgoing connections |

Common well-known/registered ports worth memorizing:

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |

TCP is connection-oriented — it establishes a handshake, guarantees delivery order, and retransmits lost packets (used for HTTP, SSH, databases). UDP is connectionless — no handshake, no delivery guarantee, lower overhead (used for DNS queries, video streaming, and other latency-sensitive traffic where an occasional dropped packet is fine).

### DNS resolution basics

DNS translates human-readable names (`example.com`) into IP addresses. A handful of record types come up constantly:

| Record | Maps |
|---|---|
| `A` | Hostname → IPv4 address |
| `AAAA` | Hostname → IPv6 address |
| `CNAME` | Hostname → another hostname (alias) |
| `MX` | Domain → mail server(s), with priority |
| `NS` | Domain → authoritative nameservers |
| `PTR` | IP address → hostname (reverse lookup) |

Two local files shape name resolution before any network query happens:

- `/etc/hosts` — a static list of hostname-to-IP mappings, checked first; entries here override DNS entirely for that name.
- `/etc/resolv.conf` — lists which DNS server(s) (`nameserver` lines) the system queries when a name isn't found in `/etc/hosts`.

### Ubuntu network configuration with netplan

Ubuntu configures network interfaces declaratively through YAML files under `/etc/netplan/`. Each file describes which interfaces exist, whether they use DHCP or a static IP, DNS servers, and routes. `netplan apply` renders the YAML into the actual running network configuration (via `systemd-networkd` or `NetworkManager` underneath). `netplan try` applies the configuration temporarily and automatically rolls back if you don't confirm within a countdown — a safety net against locking yourself out with a bad config on a remote machine.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `ip addr show` | List interfaces and their IP addresses | `ip addr show` |
| `ip link set up/down` | Bring an interface up or down | `sudo ip link set eth0 up` |
| `ip addr add/del` | Add/remove an IP address on an interface | `sudo ip addr add 192.168.1.50/24 dev eth0` |
| `ip route show` | Display the routing table | `ip route show` |
| `ip route add` | Add a route | `sudo ip route add 10.0.0.0/24 via 192.168.1.1` |
| `ip neigh` | Show the ARP/neighbor table | `ip neigh show` |
| `ss -tuln` | List listening TCP/UDP sockets | `ss -tuln` |
| `lsof -i` | List processes with open network connections | `sudo lsof -i :80` |
| `ping` | Send ICMP echo requests | `ping -c 4 8.8.8.8` |
| `traceroute` | Show the hop-by-hop path to a host | `traceroute example.com` |
| `mtr` | Live combination of ping + traceroute | `mtr example.com` |
| `curl` | Transfer/request data from a URL | `curl -I https://example.com` |
| `wget` | Download a file from a URL | `wget https://example.com/file.tar.gz` |
| `dig` | Query DNS records | `dig example.com A` |
| `nslookup` | Query DNS (simpler, older) | `nslookup example.com` |
| `netplan apply` | Apply netplan config | `sudo netplan apply` |
| `netplan try` | Apply with auto-rollback if not confirmed | `sudo netplan try` |

## Examples / Walkthrough

```bash
# --- IP addressing and CIDR, read from your own machine ---
ip -brief addr show                     # note the /N suffix after each address, e.g. 192.168.1.50/24

# /24 = 256 total addresses, 254 usable (network + broadcast reserved)
# /16 = 65,536 total addresses
# /32 = exactly one address (a single host route)

# confirm whether two addresses are on the same network by comparing their network portion
# e.g. 192.168.1.50/24 and 192.168.1.75/24 -> same network (both start 192.168.1.)
# e.g. 192.168.1.50/24 and 192.168.2.10/24 -> DIFFERENT networks, need a router in between

ip route show                            # the line for your local network (no "via", just "dev")
                                          # shows the network your machine can reach directly (same LAN)
ip route show default                    # the line WITH "via <gateway>" — everything else goes through the router
```

```bash
# --- interfaces ---
ip addr show                            # all interfaces: name, state, IP/CIDR
ip addr show eth0                       # just one interface (name may differ, e.g. enp0s3)
ip -brief addr show                     # compact one-line-per-interface summary

ip link show                            # interfaces without address detail, shows UP/DOWN state

# --- routing ---
ip route show                           # full routing table
ip route show default                   # just the default gateway line
# example output: default via 192.168.1.1 dev eth0

ip neigh show                           # ARP table: IP -> MAC mappings learned so far

# --- ports and sockets ---
ss -tuln                                # -t TCP, -u UDP, -l listening only, -n numeric (no DNS lookups)
# example line: LISTEN  0  128  0.0.0.0:22  0.0.0.0:*   <- sshd listening on all interfaces, port 22

ss -tulnp                               # add -p to show the owning process (needs sudo for full detail)
sudo ss -tulnp | grep :22               # confirm what's listening on port 22

sudo lsof -i :80                        # which process (if any) owns port 80
sudo lsof -i -P -n | head               # all network connections, numeric ports, no DNS lookups

# --- connectivity testing ---
ping -c 4 8.8.8.8                       # 4 ICMP echo requests to Google's public DNS resolver
ping -c 4 -i 2 8.8.8.8                  # same, but 2 seconds between each request

traceroute example.com                  # hop-by-hop path (may need: sudo apt install traceroute)
mtr example.com                         # live continuously-updating combination of ping + traceroute

curl -I https://example.com             # -I: headers only, fast way to check a service responds
curl -o /tmp/page.html https://example.com   # save response body to a file
wget https://example.com/index.html -O /tmp/index.html   # download, explicit output filename

# --- DNS ---
dig example.com A                       # A record: hostname -> IPv4
dig example.com MX                      # mail servers for the domain
dig example.com NS                      # authoritative nameservers
dig -x 8.8.8.8                          # reverse lookup: IP -> hostname (PTR record)

nslookup example.com                    # simpler alternative to dig

cat /etc/hosts                          # static hostname overrides, checked before DNS
cat /etc/resolv.conf                    # which DNS server(s) this machine queries

# --- netplan (read-only exploration; edits require care on a live/remote machine) ---
ls /etc/netplan/                        # usually one or two YAML files
cat /etc/netplan/*.yaml                 # example structure:
# network:
#   version: 2
#   ethernets:
#     eth0:
#       dhcp4: true

sudo netplan try                        # apply temporarily; auto-reverts if not confirmed in time
sudo netplan apply                      # apply permanently (only after you've verified the config)
```

## Common Pitfalls

- **Editing netplan YAML with bad indentation** — YAML is whitespace-sensitive; a single misaligned space silently breaks the file or changes its meaning. Always run `sudo netplan try` before `netplan apply` on any machine you're connected to remotely, so a bad config auto-reverts instead of locking you out.
- **Confusing "listening" with "established" in `ss` output** — `ss -tuln` with `-l` only shows sockets in `LISTEN` state (waiting for connections). To see active connections, drop `-l` or use `ss -tun` (or add `-a` for all states).
- **Forgetting `ping` uses ICMP, not TCP** — a host can block ICMP (no response to `ping`) while still serving traffic fine on TCP ports like 80/443. "It doesn't respond to ping" does not mean "it's down" — always also test the actual service port with `curl` or `ss`.
- **Assuming `curl`/`wget` failures mean DNS is broken** — a connection can fail for many reasons (firewall, wrong port, service down, DNS). Isolate the layer: `dig` to check DNS resolves, `ping`/`ss` to check the host/port is reachable, then `curl` to check the actual HTTP response.
- **Reading `/etc/resolv.conf` as the source of truth on modern Ubuntu** — on systems using `systemd-resolved`, `/etc/resolv.conf` is often a symlink to a generated stub file, and the real config lives elsewhere (managed by `systemd-resolved`/netplan). Treat it as informational, and prefer `resolvectl status` when digging deeper (outside this lesson's scope, but useful to know exists).
- **Trusting ephemeral port numbers as "the service's port"** — when you make an outgoing connection (e.g. `curl`), your machine picks a random high-numbered ephemeral port as the *source* port. Only the *destination* port (80, 443, etc.) identifies which service you're talking to.
- **Assuming any two private-range addresses can reach each other directly** — being in `10.0.0.0/8` or `192.168.0.0/16` doesn't mean two devices are on the *same* network; the actual CIDR prefix (`/24`, `/16`, etc.) determines that, not just "both private." Two `/24` networks under the same `10.x` umbrella still need a router between them.
- **Confusing switch and router responsibilities when troubleshooting** — "the network is down" can mean very different things depending on which layer is broken. No connectivity to anything, even local devices, points at Layer 2 (switch/cabling/interface down); no connectivity *beyond* the local network, while local devices work fine, points at Layer 3 (routing/gateway).

## FAQ

**Q: What's the difference between `ip addr` and `ip route`?**
A: `ip addr` shows what IP addresses are assigned to your own interfaces. `ip route` shows how the kernel decides where to send traffic based on destination — including the default gateway for everything not on your local network.

**Q: Why does `ping` sometimes work but `curl` to the same host fail?**
A: They test different things. `ping` only confirms ICMP reachability at the network layer. `curl` requires the target port to be open, the service to be running, and (for HTTPS) a valid TLS handshake. A host can be pingable with its web server down, or have ICMP blocked while the web server works fine.

**Q: What's the difference between well-known, registered, and ephemeral ports?**
A: Well-known (0-1023) are reserved for standard services and usually need root to bind. Registered (1024-49151) are conventionally assigned to specific applications but not enforced by the OS. Ephemeral (49152-65535) are handed out automatically as temporary source ports for outgoing connections — you rarely choose these yourself.

**Q: When would I use `dig` vs `nslookup`?**
A: Functionally similar for basic lookups. `dig` gives more detail by default (full record data, TTLs, query timing) and is the standard tool DevOps engineers reach for. `nslookup` is simpler and often available on more minimal systems, useful as a quick sanity check.

**Q: What does `/etc/hosts` actually override?**
A: Any hostname listed in `/etc/hosts` resolves to the IP given there, without ever querying DNS — useful for local testing (e.g. pointing a domain at a local dev server) or overriding a broken/slow DNS entry temporarily.

**Q: Why does `netplan try` matter if I already tested my config carefully?**
A: Because a subtle mistake (wrong gateway, typo'd interface name) can cut off your SSH session entirely on a remote machine, with no way back in except physical/console access. `netplan try`'s automatic rollback is cheap insurance against exactly that scenario.

**Q: Why can't my `192.168.1.x` device connect directly to a `10.0.0.x` device?**
A: Different network portions — they're on entirely separate networks even though both happen to use private address ranges. A router with an interface (or route) on both networks is required to forward traffic between them; two arbitrary private-range devices are not automatically able to reach each other.

**Q: What's the practical difference between a switch and a router I'd actually notice?**
A: A switch just moves traffic between local devices as fast as possible with zero awareness of IP networks — plug more devices into it, and they can all reach each other, no configuration needed. A router is the thing that has to be told about networks and routes (like the routing table this lesson covers) and is the *only* device that decides whether/how traffic leaves the local network at all.

**Q: If NAT rewrites my address, how does the reply ever find its way back to my specific device?**
A: The NAT gateway keeps a translation table mapping each outgoing connection (source IP:port before translation) to what it was rewritten to. When a reply arrives addressed to the gateway's public IP and that specific translated port, the gateway looks up the table and rewrites the destination back to the original private IP:port before forwarding it internally — this is why NAT fundamentally only works cleanly for connections *initiated* from inside the private network, without also configuring destination NAT/port forwarding for the reverse direction.

## Practice / Exercise

**Core:**
1. Run `ip -brief addr show`, identify your machine's IP address and CIDR prefix, and state how many total and usable host addresses that network size has.
2. Given two example addresses (e.g. `192.168.1.20/24` and `192.168.1.240/24`), determine whether they're on the same network, and explain why using the network/host portion split.
3. Run `ip addr show` and identify your machine's IP address(es) and which interface they're on.
4. Run `ip route show` and identify the default gateway; identify the separate line describing your own local network.
5. Run `ss -tuln` and list every port currently in `LISTEN` state on your machine; for at least 2 of them, name the service you'd expect to be using that port (cross-reference against the common ports table).
6. Use `ping -c 4` to test connectivity to `8.8.8.8`, then to a hostname like `example.com`; note the round-trip times.
7. Use `dig example.com A` and `dig example.com MX` and explain the difference in what each returned.
8. Read `/etc/hosts` and `/etc/resolv.conf` on your machine and explain, in your own words, what each controls.
9. Use `curl -I https://example.com` and identify the HTTP status code in the response.

**Stretch:**
1. Use `sudo lsof -i :22` to confirm which process owns the SSH port, and cross-check the PID against `ps -o pid,cmd -p <PID>` (Lesson 8).
2. Compare `traceroute` and `mtr` output to the same destination; explain what extra information `mtr` gives you that a single `traceroute` run doesn't.
3. Read through `/etc/netplan/*.yaml` on your machine (or a lab VM) and diagram, in plain English, what network configuration it describes (DHCP vs static, interface name, any DNS servers listed).
4. Explain, in your own words, why a device behind NAT can freely initiate outbound connections but generally cannot be reached by an unsolicited inbound connection without explicit destination NAT/port forwarding configured on the gateway.

## Further Reading

- `man ip`, `man ss`, `man dig`, `man ping`, `man netplan`
- [Netplan documentation](https://netplan.io/)
