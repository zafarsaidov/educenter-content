# Lesson 10: Linux Networking Essentials – TCP/IP, Ports & Tools

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-9 (terminal, filesystem, permissions, find/grep/piping, systemd, users/sudo, packages, process management, scheduled tasks)

## Learning Objectives

By end of lesson student can:
- Inspect and configure network interfaces with the `ip` command
- Read and reason about a routing table, and identify the default gateway
- Explain TCP/UDP port ranges and identify what a handful of well-known ports are used for
- Find which process owns a listening port with `ss`/`lsof`
- Test connectivity and diagnose reachability with `ping`, `traceroute`/`mtr`, `curl`, `wget`
- Query DNS records with `dig`/`nslookup`, and explain the roles of `/etc/hosts` and `/etc/resolv.conf`
- Read and modify a basic Ubuntu netplan configuration

## Topics

- Network interfaces: `ip addr show`, `ip link set up/down`, `ip addr add/del`
- Routing: `ip route show`, `ip route add`, default gateway, `ip neigh`
- Ports & sockets: TCP/UDP port ranges (well-known 0-1023, registered, ephemeral 49152+); common ports: 22/80/443/3306/5432/6379; `ss -tuln`, `lsof -i`
- Connectivity tools: `ping` (`-c`, `-i`), `traceroute`/`mtr`, `curl`, `wget`
- DNS tools: `dig` (A, MX, NS, CNAME, PTR), `nslookup`; `/etc/hosts`, `/etc/resolv.conf`
- Ubuntu network config: `/etc/netplan/` structure, `netplan apply`, `netplan try`

## Concepts

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

## Practice / Exercise

**Core:**
1. Run `ip addr show` and identify your machine's IP address(es) and which interface they're on.
2. Run `ip route show` and identify the default gateway.
3. Run `ss -tuln` and list every port currently in `LISTEN` state on your machine; for at least 2 of them, name the service you'd expect to be using that port (cross-reference against the common ports table).
4. Use `ping -c 4` to test connectivity to `8.8.8.8`, then to a hostname like `example.com`; note the round-trip times.
5. Use `dig example.com A` and `dig example.com MX` and explain the difference in what each returned.
6. Read `/etc/hosts` and `/etc/resolv.conf` on your machine and explain, in your own words, what each controls.
7. Use `curl -I https://example.com` and identify the HTTP status code in the response.

**Stretch:**
1. Use `sudo lsof -i :22` to confirm which process owns the SSH port, and cross-check the PID against `ps -o pid,cmd -p <PID>` (Lesson 8).
2. Compare `traceroute` and `mtr` output to the same destination; explain what extra information `mtr` gives you that a single `traceroute` run doesn't.
3. Read through `/etc/netplan/*.yaml` on your machine (or a lab VM) and diagram, in plain English, what network configuration it describes (DHCP vs static, interface name, any DNS servers listed).

## Further Reading

- `man ip`, `man ss`, `man dig`, `man ping`, `man netplan`
- [Netplan documentation](https://netplan.io/)
