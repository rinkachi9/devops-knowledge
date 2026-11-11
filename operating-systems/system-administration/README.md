# System administration

Running a Linux system day to day: boot and service management with systemd, and access control through permissions.

## Map

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["Startup management"]
  n1["Permissions"]
  n0 --> n1
```

## Notes in reading order

1. [Startup management](startup-management.md): the boot sequence, GRUB, initramfs, systemd units and targets.
2. [Permissions](permissions.md): owners, groups, permission bits, special bits, umask and ACLs.

## Planned topics

- Users, groups, sudo and PAM
- Package management across distributions
- Scheduled jobs: cron and systemd timers
- System logging: journald and rsyslog
- Kernel parameters with sysctl
- SELinux and AppArmor
- Backups and restore
