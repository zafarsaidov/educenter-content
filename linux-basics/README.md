# Linux Basic

<!-- TODO: add a 2–4 sentence course description here. -->

## Curriculum

<!-- 8 sections · 41 themes -->

### Introduction & Essential Commands

Covers Linux fundamentals: distributions, the terminal, filesystem navigation, and core file operations.

- [Linux Distributions](./initial-commands/initial-commands-theme.md)
- [Terminal Usage and Getting Help](./initial-commands/terminal-and-help.md)
- [Absolute and Relative Paths](./initial-commands/paths.md)
- [Navigating the Filesystem](./initial-commands/navigation.md)
- [File Operations](./initial-commands/operations.md)
- [The Three Standard Streams](./initial-commands/stdin-stdout-stderr.md)
- [Piping Command Output](./initial-commands/piping.md)

### Working with Files and Directories

Files and Directories

- [File Types in Linux](./files-and-directories/file-types.md)
- [Viewing File Content](./files-and-directories/viewing-content.md)
- [File Details with file and stat](./files-and-directories/file-details.md)
- [Inodes](./files-and-directories/inodes.md)
- [Hard Links and Soft Links](./files-and-directories/hard-and-soft-links.md)

### Text Editors and awk/sed

Text editors

- [Vim Basics](./text-editors/vim.md)
- [Nano Text Editor](./text-editors/nano.md)
- [sed — Stream Editor](./text-editors/sed.md)
- [awk — Text Processing](./text-editors/awk.md)

### File Permissions & Ownership

File permissions

- [File Permissions](./file-permissions/permissions.md)
- [Changing Ownership](./file-permissions/ownership.md)
- [Default Permissions with umask](./file-permissions/umask.md)
- [Access Control Lists (ACL)](./file-permissions/acl.md)
- [SUID, SGID, and Sticky Bit](./file-permissions/special-permissions.md)

### Find, Grep, and Text Utilities

Find and grep utilities

- [Finding Files with find](./find-grep-utilities/find.md)
- [Searching with grep](./find-grep-utilities/grep.md)
- [Regex Basics](./find-grep-utilities/regex.md)
- [Text Processing Utilities](./find-grep-utilities/text-utilities.md)

### System Services and Systemd

- [Managing Services with systemctl](./system-services/systemctl.md)
- [Viewing Logs with journalctl](./system-services/journalctl.md)
- [systemd Unit Types](./system-services/service-units.md)
- [Creating a Custom systemd Service](./system-services/custom-service.md)
- [Archiving and Compression](./system-services/archiving.md)
- [File Synchronization with rsync](./system-services/rsync.md)

### User & Group Management

- [Creating and Deleting Users](./user-management/creating-users.md)
- [Modifying Users](./user-management/modifying-users.md)
- [Groups](./user-management/groups.md)
- [Switching Users and sudo](./user-management/switching-users.md)
- [User Home, Shell, UID, and GID](./user-management/user-home-shell.md)
- [User Limits](./user-management/user-limits.md)

### Linux Package Management

- [Package Management on Debian/Ubuntu (apt)](./package-management/debian-apt.md)
- [Package Management on RHEL/CentOS (yum/dnf)](./package-management/rhel-yum-dnf.md)
- [Repositories and Caching](./package-management/repositories.md)
- [Local Packages and Version Locking](./package-management/local-packages.md)
