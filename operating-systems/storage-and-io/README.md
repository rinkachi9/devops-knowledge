# Storage and I/O

How data is stored and moved: file system architecture and types, file descriptors, and device and I/O management.

## Map

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["File systems"]
  n1["File descriptors"]
  n0 --> n1
  n2["I/O management"]
  n1 --> n2
```

## Notes in reading order

1. [File systems](file-systems.md): VFS layers, inodes, journaling and a comparison of ext4, XFS, Btrfs and others.
2. [File descriptors](file-descriptors.md): descriptor, open file and inode tables, redirection and limits.
3. [I/O management](io-management.md): device categories, allocation, I/O scheduling and the I/O interface.

## Planned topics

- Disk partitioning (MBR, GPT)
- Logical Volume Manager (LVM)
- RAID levels
- Mounting, fstab and automount
- Network storage: NFS, SMB, iSCSI
