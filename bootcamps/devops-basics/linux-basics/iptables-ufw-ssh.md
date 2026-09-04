# Lesson 11: iptables, ufw & SSH

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-10 (terminal through networking essentials — especially ports/sockets and `ss`)

## Learning Objectives

By end of lesson student can:
- Explain iptables' tables/chains model and how a packet is matched against ACCEPT/DROP/REJECT rules
- Write, list, and delete basic iptables rules, and persist them
- Use `ufw` as a friendlier front-end for the same firewall functionality
- Generate an SSH key pair, deploy it with `ssh-copy-id`, and connect without a password
- Write an SSH client config for frequently-used hosts
- Use `scp` to copy files over SSH, and set up local/remote SSH port forwarding
- Harden `sshd_config` against the most common attack surface

## Topics

- iptables concepts: tables (filter/nat/mangle), chains (INPUT/OUTPUT/FORWARD), ACCEPT/DROP/REJECT
- iptables rules: `-A`, `-D`, `-I`, `-L --line-numbers`, `-F`; allow/block port; `iptables-save`/`restore`
- ufw: enable/disable, allow/deny (port, service, from IP), delete rules, status numbered
- SSH basics: `ssh`, `ssh-keygen` (ed25519), `ssh-copy-id`, `authorized_keys`, `known_hosts`
- SSH config file: `~/.ssh/config` (`Host`, `HostName`, `User`, `IdentityFile`, `Port`); `scp`
- SSH port forwarding: local (`-L localport:host:remoteport`), remote (`-R remoteport:host:localport`)
- SSH hardening: `/etc/ssh/sshd_config` (`PermitRootLogin no`, `PasswordAuthentication no`, `Port`)

## Concepts

### iptables: tables and chains

`iptables` configures the Linux kernel's `netfilter` packet-filtering framework. Every packet that touches the network stack passes through a sequence of **chains**, grouped into **tables** by purpose:

| Table | Purpose |
|---|---|
| `filter` | The default table — decides whether to allow or block traffic (what you'll use almost always) |
| `nat` | Network address translation — rewriting source/destination addresses (e.g. port forwarding) |
| `mangle` | Specialized packet header modification (rarely needed day-to-day) |

Within the `filter` table, three built-in chains matter most:

| Chain | Applies to |
|---|---|
| `INPUT` | Packets destined for this machine |
| `OUTPUT` | Packets originating from this machine |
| `FORWARD` | Packets passing through this machine to somewhere else (routing) |

Each chain holds an ordered list of **rules**. A packet is checked against rules top to bottom; the first matching rule decides its fate, and no further rules in that chain are checked. If nothing matches, the chain's **default policy** applies.

### ACCEPT, DROP, REJECT

| Target | Effect |
|---|---|
| `ACCEPT` | Let the packet through |
| `DROP` | Silently discard the packet — sender gets no response, connection appears to hang/time out |
| `REJECT` | Discard the packet, but send back an error response (e.g. "connection refused") |

`DROP` is often preferred for security (it wastes an attacker's time waiting for a timeout, and doesn't confirm a port even exists). `REJECT` is friendlier for legitimate users/internal networks, since they get an immediate, clear failure instead of a hang.

### ufw: a friendlier front-end

`ufw` (Uncomplicated Firewall) doesn't replace iptables — it configures the same underlying netfilter rules through a much simpler command syntax. Ubuntu ships with `ufw` specifically to make firewall management approachable without writing raw iptables syntax. Most day-to-day firewall work on an Ubuntu server uses `ufw`; raw `iptables` is for cases needing finer control than `ufw` exposes.

### Why SSH key authentication matters

SSH supports two main authentication methods: password and public-key. Public-key authentication uses a **key pair** — a private key (kept secret, never shared, stays on your machine) and a public key (safe to share, placed on servers you want to access). The server challenges your client to prove it holds the private key matching a public key it already trusts; if it can, you're in — no password ever crosses the network. Key-based auth is both more secure (immune to password-guessing/brute-force attacks) and more convenient (no typing a password every connection, once configured).

`authorized_keys` (on the server, under `~/.ssh/`) lists which public keys are allowed to log in as that user. `known_hosts` (on the client) records the server's own public key fingerprint the first time you connect, so future connections can detect if the server's identity has unexpectedly changed (a possible sign of interception).

### SSH port forwarding

SSH can tunnel arbitrary TCP traffic through its already-encrypted connection:

- **Local forwarding** (`-L`) — a port opened on *your* machine forwards to a service reachable from the *remote* machine. Useful for reaching a database or internal service that's only accessible from inside a remote network, as if it were running locally.
- **Remote forwarding** (`-R`) — the reverse: a port opened on the *remote* machine forwards back to a service on *your* machine. Useful for exposing something local to a remote server temporarily.

### SSH hardening basics

The default SSH server configuration is reasonably secure but leaves some common attack surface open by default. A few settings in `/etc/ssh/sshd_config` meaningfully reduce risk:

| Setting | Effect when hardened |
|---|---|
| `PermitRootLogin no` | Root cannot log in directly over SSH — must log in as a normal user then `sudo` |
| `PasswordAuthentication no` | Only key-based login is accepted — eliminates password brute-force attacks entirely |
| `Port <non-default>` | Moves SSH off port 22 — doesn't stop a targeted attacker, but sharply cuts automated scanning noise |

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `iptables -L --line-numbers` | List rules with numbers | `sudo iptables -L --line-numbers` |
| `iptables -A` | Append a rule to end of chain | `sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT` |
| `iptables -I` | Insert a rule at a position (default: top) | `sudo iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT` |
| `iptables -D` | Delete a rule (by spec or line number) | `sudo iptables -D INPUT 3` |
| `iptables -F` | Flush (delete all rules in) a chain | `sudo iptables -F INPUT` |
| `iptables-save` | Dump current rules to stdout | `sudo iptables-save > /tmp/rules.v4` |
| `iptables-restore` | Load rules from a file | `sudo iptables-restore < /tmp/rules.v4` |
| `ufw enable`/`disable` | Turn the firewall on/off | `sudo ufw enable` |
| `ufw allow`/`deny` | Allow/deny a port, service, or source | `sudo ufw allow 22/tcp` |
| `ufw status numbered` | List active rules with numbers | `sudo ufw status numbered` |
| `ufw delete` | Remove a rule by number | `sudo ufw delete 2` |
| `ssh-keygen` | Generate a key pair | `ssh-keygen -t ed25519 -C "me@example.com"` |
| `ssh-copy-id` | Copy public key to a server's `authorized_keys` | `ssh-copy-id user@server` |
| `ssh` | Connect to a remote host | `ssh -i ~/.ssh/id_ed25519 user@server` |
| `scp` | Copy files over SSH | `scp file.txt user@server:/tmp/` |
| `ssh -L` | Local port forward | `ssh -L 8080:localhost:80 user@server` |
| `ssh -R` | Remote port forward | `ssh -R 9090:localhost:3000 user@server` |

## Examples / Walkthrough

```bash
# --- iptables: inspecting and writing rules ---
sudo iptables -L --line-numbers -v      # list INPUT/OUTPUT/FORWARD chains, rule numbers, packet counts

# allow established/related connections back in (so replies to outgoing traffic aren't blocked)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# allow SSH (port 22) in — insert near the top so it's checked before any DROP-everything rule
sudo iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT

# allow HTTP and HTTPS in
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# block a specific IP entirely
sudo iptables -A INPUT -s 203.0.113.55 -j DROP

# reject (not silently drop) traffic on port 8080, so the sender gets an immediate refusal
sudo iptables -A INPUT -p tcp --dport 8080 -j REJECT

sudo iptables -L --line-numbers          # confirm the rules landed in the order expected
sudo iptables -D INPUT 4                 # delete rule number 4 from INPUT
sudo iptables -F INPUT                   # flush (remove) every rule in INPUT — use with care

# persist rules across reboot (iptables rules are otherwise lost on restart)
sudo iptables-save | sudo tee /etc/iptables/rules.v4

# --- ufw: same goals, simpler syntax ---
sudo ufw status                          # check if ufw is active
sudo ufw allow 22/tcp                    # allow SSH by port
sudo ufw allow ssh                       # equivalent, using the service name from /etc/services
sudo ufw allow 80,443/tcp                # allow HTTP and HTTPS together
sudo ufw allow from 203.0.113.10 to any port 22   # allow SSH only from one specific IP
sudo ufw deny 8080                       # deny a port

sudo ufw enable                          # turn the firewall on (careful over an active SSH session —
                                          # confirm rule 22/tcp allow is in place FIRST, or you'll lock yourself out)

sudo ufw status numbered                 # list rules with numbers, for use with delete
sudo ufw delete 3                        # remove rule number 3

# --- SSH keys ---
ssh-keygen -t ed25519 -C "student@devops-basics"
# prompts for a save path (default ~/.ssh/id_ed25519) and an optional passphrase
# creates two files: id_ed25519 (private, keep secret) and id_ed25519.pub (public, safe to share)

cat ~/.ssh/id_ed25519.pub                # view your public key — this is what gets shared

ssh-copy-id student@192.168.1.50         # copies your public key into the remote user's authorized_keys
                                          # (prompts once for the remote password, then key auth works)

ssh student@192.168.1.50                 # connect — no password needed now
cat ~/.ssh/known_hosts                   # each line records a server's host key fingerprint

# --- SSH client config: shortcuts for frequently-used hosts ---
cat >> ~/.ssh/config <<'EOF'
Host devbox
    HostName 192.168.1.50
    User student
    IdentityFile ~/.ssh/id_ed25519
    Port 22
EOF
ssh devbox                               # now equivalent to "ssh -i ~/.ssh/id_ed25519 student@192.168.1.50"

# --- scp: copying files over SSH ---
scp localfile.txt student@devbox:/home/student/          # local -> remote (uses ~/.ssh/config alias)
scp student@devbox:/var/log/app.log ./app.log            # remote -> local
scp -r localdir/ student@devbox:/home/student/remotedir  # -r for recursive directory copy

# --- SSH port forwarding ---
# local: reach a remote-only service (e.g. an internal dashboard on port 8080) via localhost
ssh -L 9000:localhost:8080 student@devbox
# now, on your machine: curl http://localhost:9000  -> reaches devbox's localhost:8080

# remote: expose a local dev server (port 3000) to the remote machine as its own port 9090
ssh -R 9090:localhost:3000 student@devbox
# now, on devbox: curl http://localhost:9090  -> reaches your machine's localhost:3000

# --- SSH server hardening ---
sudo cat /etc/ssh/sshd_config | grep -E "PermitRootLogin|PasswordAuthentication|^Port"
# edit with vim/nano (Lesson 3):
#   PermitRootLogin no
#   PasswordAuthentication no      <- only do this AFTER confirming key-based login works!
#   Port 2222                      <- optional, moves off the default port

sudo systemctl restart sshd              # apply the config change (Lesson 5's systemctl)
# open a SECOND terminal and confirm you can still connect BEFORE closing your first session
```

## Common Pitfalls

- **Locking yourself out with `ufw enable` or an iptables `DROP` policy over SSH** — if the SSH rule isn't in place before you enable the firewall (or before a default-DROP policy takes effect), your own connection gets cut and you may lose remote access entirely. Always add and verify the SSH-allow rule first, and test from a second session before closing the one you're working in.
- **Setting `PasswordAuthentication no` before confirming key login works** — this disables your fallback. Always test `ssh` with the key first, in a separate session, before turning off password auth in `sshd_config`.
- **Forgetting iptables rules don't survive reboot** — unlike `ufw` (which persists automatically), raw `iptables` rules live only in the running kernel's memory unless explicitly saved with `iptables-save` and restored on boot.
- **Rule order mistakes with `-A` vs `-I`** — `-A` appends to the end of the chain, `-I` inserts (default: at the top). A DROP-all rule appended after an ACCEPT rule works fine; but if you use `-A` to add an ACCEPT rule after a DROP-all rule already exists, it never gets reached, since the DROP-all rule matches everything first.
- **Mixing up local (`-L`) and remote (`-R`) SSH forwarding** — `-L` opens a port on *your* machine that reaches into the remote network; `-R` opens a port on the *remote* machine that reaches back into yours. Getting this backwards is a very common source of confusion.
- **Using `DROP` where `REJECT` (or vice versa) would be more appropriate** — `DROP` on a public-facing port makes port scanning slower for attackers but also makes debugging legitimate connectivity issues confusing (a hang gives no information). Choose deliberately based on the situation.

## FAQ

**Q: Should I use raw iptables or ufw?**
A: For most day-to-day Ubuntu server firewall needs, `ufw` — it's simpler, less error-prone, and persists automatically. Reach for raw `iptables` when you need rule logic `ufw` doesn't expose (complex NAT, specific chain ordering, rate limiting).

**Q: What's the practical difference between DROP and REJECT?**
A: `DROP` silently discards the packet — the sender's connection just hangs until it times out. `REJECT` sends back an explicit error (e.g. TCP RST or ICMP port-unreachable) — the sender knows immediately the connection was refused. `DROP` is generally preferred for internet-facing firewalls; `REJECT` is friendlier for internal/trusted networks.

**Q: Do I still need a password if I set up SSH key authentication?**
A: Not for logging in, once `authorized_keys` has your public key. You may still set an optional passphrase on your *private key* itself (asked during `ssh-keygen`) — that only protects the key file locally on your own machine, and is separate from the server's password authentication.

**Q: Why would I ever disable `PasswordAuthentication`?**
A: Password auth is vulnerable to brute-force guessing, especially on internet-facing servers getting constant automated login attempts. Disabling it entirely (once key auth is confirmed working) removes that entire attack surface.

**Q: What does `ssh-copy-id` actually do, technically?**
A: It connects to the remote host (using whatever auth currently works, usually a password), appends the contents of your local `~/.ssh/id_*.pub` to the remote user's `~/.ssh/authorized_keys` file, and fixes file permissions on that directory/file (SSH refuses to use `authorized_keys` if its permissions are too open).

**Q: Why change the SSH port away from 22?**
A: It doesn't stop a determined, targeted attacker — they can scan for open ports easily. It does dramatically cut down noise from automated bots that only ever try the default port 22, which reduces log clutter and wasted resources responding to scans.

## Practice / Exercise

**Core:**
1. On a lab VM (not your primary machine), list current iptables rules with `sudo iptables -L --line-numbers`. Add a rule allowing SSH on port 22, then a rule denying/rejecting port 8080. Verify with `-L` again, then delete the 8080 rule by its line number.
2. On the same VM, install and use `ufw` instead: allow SSH, allow 80/443, enable the firewall, verify with `ufw status numbered`, then delete one rule.
3. Generate an ed25519 SSH key pair. Use `ssh-copy-id` to deploy it to a lab server, then confirm you can `ssh` in without a password prompt.
4. Write an `~/.ssh/config` entry for that lab server with a short `Host` alias, and connect using just the alias.
5. Use `scp` to copy a file to the lab server and back.
6. Set up local port forwarding (`ssh -L`) to reach a service running only on the lab server's `localhost`, and confirm with `curl` on your own machine.
7. Edit `/etc/ssh/sshd_config` on the lab server: set `PermitRootLogin no`. Restart `sshd` and confirm normal user login still works.

**Stretch:**
1. Only after confirming key-based login works reliably (from a second, separate session), set `PasswordAuthentication no` on the lab server and confirm password login is now refused while key login still works.
2. Practice remote port forwarding (`ssh -R`): expose a local port from your own machine to the lab server, and confirm the lab server can reach it.
3. Write out (don't necessarily need to run) the `iptables-save`/persistence workflow you'd use to make hand-written iptables rules survive a reboot, and explain why `ufw` doesn't need this step.

## Further Reading

- `man iptables`, `man ufw`, `man ssh`, `man ssh-keygen`, `man ssh_config`, `man sshd_config`
- [Ubuntu Server Guide: Security — Firewall](https://ubuntu.com/server/docs/security-firewall)
- [OpenSSH manual pages](https://www.openssh.com/manual.html)
