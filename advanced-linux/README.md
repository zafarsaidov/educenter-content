# Linux Advanced

<!-- TODO: add a 2–4 sentence course description here. -->

## Curriculum

<!-- 10 sections · 42 themes -->

### Shell Scripting: Basics & Syntax

- [Shebang, Script Execution, and eval](./shell-scripting-basic/shebang-and-execution.md)
- [Variables, Scope, Quoting, and Expansion](./shell-scripting-basic/variables-and-quoting.md)
- [Conditional Statements: if, else, elif](./shell-scripting-basic/conditionals.md)
- [Exit Codes and Testing with [ ] and [[ ]]](./shell-scripting-basic/exit-codes-and-testing.md)
- [Script Parameters: $1, $2, $@, $#](./shell-scripting-basic/script-parameters.md)
- [Brace Expansion, Guard Clauses, and source](./shell-scripting-basic/brace-guard-source.md)
- [pipefail and Parameter Expansion](./shell-scripting-basic/pipefail-parameter-expansion.md)

### Shell Scripting: Loops & Functions

- [Loops: for, while, until](./shell-scripting-advanced/loops.md)
- [Reading Files Line by Line](./shell-scripting-advanced/reading-files.md)
- [Functions and Return Values](./shell-scripting-advanced/functions.md)
- [Trap Signals and Debugging](./shell-scripting-advanced/trap-and-debugging.md)
- [Working with case and select](./shell-scripting-advanced/case-and-select.md)

### Shell Scripting: Practice 1

- [Practice: Write a Shell Script for System Monitoring](./system-monitoring-script/system-monitoring-script.md)

### Shell Scripting: Practice 2

- [Practice: Install Script for a Monitoring Service](./install-monitoring-service/install-monitoring-service.md)

### Process & Job Management

- [Foreground/Background Jobs: &, jobs, fg, bg](./process-job-management/foreground-background-jobs.md)
- [Monitoring Processes: ps, top, htop, nice, renice](./process-job-management/monitoring-ps-top.md)
- [Zombie and Orphaned Processes](./process-job-management/zombie-orphaned-processes.md)
- [kill, pkill, killall and Signals](./process-job-management/kill-signals.md)
- [Resource Control with ulimit and cgroups](./process-job-management/ulimit-resource-control.md)

### Scheduled Tasks & Kernel Parameters

- [Crontab: User vs System Cron and Schedule Keywords](./scheduled-tasks-kernel/crontab.md)
- [One-Time Scheduled Tasks](./scheduled-tasks-kernel/at-jobs.md)
- [Scheduled Tasks with systemd Timers](./scheduled-tasks-kernel/systemd-timers.md)
- [Kernel Parameters: sysctl, /proc/sys/, sysctl.conf](./scheduled-tasks-kernel/kernel-parameters.md)

### Linux Networking

- [IP & Interface Tools: ip a, ip r, ip link](./linux-networking/ip-interface-tools.md)
- [ping, traceroute, dig, nslookup](./linux-networking/ping-traceroute-dns.md)
- [Port Checks: ss, netstat, telnet, nc](./linux-networking/port-checks.md)
- [Downloading Tools: wget and curl](./linux-networking/wget-curl.md)
- [Firewall: ufw and Intro to iptables](./linux-networking/firewall-ufw-iptables.md)
- [Network Namespaces](./linux-networking/network-namespaces.md)

### SSH & Remote Access

- [SSH Key-Based Authentication](./ssh-remote-access/key-based-auth.md)
- [Configuring sshd_config](./ssh-remote-access/sshd-config.md)
- [SSH Client Config: ~/.ssh/config](./ssh-remote-access/ssh-client-config.md)
- [SSH Port Forwarding](./ssh-remote-access/ssh-port-forwarding.md)
- [Remote Execution, scp, and rsync over SSH](./ssh-remote-access/remote-execution-scp-rsync.md)

### Storage Management: Partitions and File Systems

- [Disks and Block Devices: lsblk, blkid, cfdisk](./storage-partitions/disks-lsblk-blkid.md)
- [Mounting, Unmounting, and /etc/fstab](./storage-partitions/mounting-fstab.md)
- [Filesystem Types: ext4 and xfs](./storage-partitions/filesystem-types.md)
- [Swap Space](./storage-partitions/swap-space.md)

### Storage Management: LVM

- [LVM Concepts: PV, VG, LV](./storage-lvm/lvm-concepts.md)
- [Creating, Resizing, and Removing LVMs](./storage-lvm/creating-resizing-lvm.md)
- [LVM Snapshots, Thin Provisioning, and Striping](./storage-lvm/advanced-lvm-features.md)
- [Storage Monitoring Commands](./storage-lvm/storage-monitoring.md)
