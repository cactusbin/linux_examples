# systemd-boot Emergency Rescue Boot Entries

Complete guide for creating direct partition boot entries in systemd-boot for Alpine Linux and Arch Linux emergency rescue scenarios.

## Overview

Systemd-boot entries allow direct booting to specific partitions without relying on bootloader auto-detection. This is critical for emergency rescue when your primary system fails.

## Prerequisites

```fish
# Verify systemd-boot is installed
bootctl status

# Check partition UUIDs (required for reliable booting)
lsblk -f /dev/sdh2 /dev/sdh4

# Or use blkid for detailed info
blkid /dev/sdh2 /dev/sdh4
```

## Boot Entry Syntax

Systemd-boot entries live in `/boot/loader/entries/` or `/efi/loader/entries/` (depending on your ESP mount point). Each entry is a `.conf` file with this structure:

```conf
title    [Description shown in boot menu]
linux    [Path to kernel relative to ESP]
initrd   [Path to initramfs relative to ESP]
options  [Kernel parameters including root partition]
```

## Alpine Linux Emergency Entry

### Step 1: Identify Kernel and Initramfs

```fish
# Mount the Alpine partition to inspect
mount /dev/sdh2 /mnt

# List available kernels
ls -lh /mnt/boot/vmlinuz-*

# List initramfs images
ls -lh /mnt/boot/initramfs-*

# Note the kernel version (e.g., vmlinuz-lts or vmlinuz-edge)
umount /mnt
```

### Step 2: Copy Kernel to ESP

```fish
# Mount Alpine partition
mount /dev/sdh2 /mnt

# Mount ESP (adjust path as needed)
mount /dev/sdh1 /boot  # or /efi

# Create directory for Alpine kernels
mkdir -p /boot/alpine-rescue

# Copy kernel and initramfs to ESP
cp /mnt/boot/vmlinuz-lts /boot/alpine-rescue/
cp /mnt/boot/initramfs-lts /boot/alpine-rescue/

# Cleanup
umount /mnt
```

### Step 3: Get Partition UUID

```fish
# Get UUID (more reliable than /dev/sdh2)
set ALPINE_UUID (blkid -s UUID -o value /dev/sdh2)
echo $ALPINE_UUID
```

### Step 4: Create Boot Entry

```fish
# Create the entry file
cat > /boot/loader/entries/alpine-rescue.conf << 'EOF'
title   Alpine Linux (Emergency Rescue)
linux   /alpine-rescue/vmlinuz-lts
initrd  /alpine-rescue/initramfs-lts
options root=UUID=YOUR_UUID_HERE rootfstype=ext4 modules=sd-mod,usb-storage,ext4 quiet
EOF

# Replace YOUR_UUID_HERE with actual UUID
sd "YOUR_UUID_HERE" $ALPINE_UUID /boot/loader/entries/alpine-rescue.conf
```

**Key Options Explained:**
- `root=UUID=...`: Specifies root partition by UUID (survives disk reordering)
- `rootfstype=ext4`: Filesystem type (change to btrfs, xfs, etc. if needed)
- `modules=...`: Alpine-specific: explicitly load required modules
- `quiet`: Suppress verbose boot messages (remove for debugging)

## Arch Linux Emergency Entry

### Step 1: Identify Kernel and Initramfs

```fish
# Mount Arch partition
mount /dev/sdh4 /mnt

# List kernels (Arch typically has 'linux' or 'linux-lts')
ls -lh /mnt/boot/vmlinuz-*

# List initramfs
ls -lh /mnt/boot/initramfs-*

# Note: Arch uses initramfs-linux.img and initramfs-linux-fallback.img
umount /mnt
```

### Step 2: Copy Kernel to ESP

```fish
# Mount Arch partition
mount /dev/sdh4 /mnt

# Mount ESP
mount /dev/sdh1 /boot

# Create directory for Arch kernels
mkdir -p /boot/arch-rescue

# Copy kernel and both initramfs images
cp /mnt/boot/vmlinuz-linux /boot/arch-rescue/
cp /mnt/boot/initramfs-linux.img /boot/arch-rescue/
cp /mnt/boot/initramfs-linux-fallback.img /boot/arch-rescue/

# Cleanup
umount /mnt
```

### Step 3: Get Partition UUID

```fish
# Get UUID for Arch partition
set ARCH_UUID (blkid -s UUID -o value /dev/sdh4)
echo $ARCH_UUID
```

### Step 4: Create Boot Entries

```fish
# Primary entry (standard initramfs)
cat > /boot/loader/entries/arch-rescue.conf << 'EOF'
title   Arch Linux (Emergency Rescue)
linux   /arch-rescue/vmlinuz-linux
initrd  /arch-rescue/initramfs-linux.img
options root=UUID=YOUR_UUID_HERE rw
EOF

# Fallback entry (larger initramfs with more modules)
cat > /boot/loader/entries/arch-rescue-fallback.conf << 'EOF'
title   Arch Linux (Emergency Rescue - Fallback)
linux   /arch-rescue/vmlinuz-linux
initrd  /arch-rescue/initramfs-linux-fallback.img
options root=UUID=YOUR_UUID_HERE rw
EOF

# Replace UUIDs in both files
sd "YOUR_UUID_HERE" $ARCH_UUID /boot/loader/entries/arch-rescue.conf
sd "YOUR_UUID_HERE" $ARCH_UUID /boot/loader/entries/arch-rescue-fallback.conf
```

**Key Options Explained:**
- `root=UUID=...`: Root partition identifier
- `rw`: Mount root as read-write (Arch default)
- Fallback initramfs includes ALL kernel modules for maximum hardware compatibility

## Complete One-Liner Setup Script

```fish
# Alpine setup (portable bash/fish)
mount /dev/sdh2 /mnt && mount /dev/sdh1 /boot && mkdir -p /boot/alpine-rescue && cp /mnt/boot/vmlinuz-lts /boot/alpine-rescue/ && cp /mnt/boot/initramfs-lts /boot/alpine-rescue/ && printf "title   Alpine Linux (Emergency Rescue)\nlinux   /alpine-rescue/vmlinuz-lts\ninitrd  /alpine-rescue/initramfs-lts\noptions root=UUID=%s rootfstype=ext4 modules=sd-mod,usb-storage,ext4 quiet\n" (blkid -s UUID -o value /dev/sdh2) > /boot/loader/entries/alpine-rescue.conf && umount /mnt

# Arch setup (portable bash/fish)
mount /dev/sdh4 /mnt && mkdir -p /boot/arch-rescue && cp /mnt/boot/vmlinuz-linux /mnt/boot/initramfs-linux*.img /boot/arch-rescue/ && printf "title   Arch Linux (Emergency Rescue)\nlinux   /arch-rescue/vmlinuz-linux\ninitrd  /arch-rescue/initramfs-linux.img\noptions root=UUID=%s rw\n" (blkid -s UUID -o value /dev/sdh4) > /boot/loader/entries/arch-rescue.conf && printf "title   Arch Linux (Emergency Rescue - Fallback)\nlinux   /arch-rescue/vmlinuz-linux\ninitrd  /arch-rescue/initramfs-linux-fallback.img\noptions root=UUID=%s rw\n" (blkid -s UUID -o value /dev/sdh4) > /boot/loader/entries/arch-rescue-fallback.conf && umount /mnt
```

## Verification

```fish
# List all boot entries
bootctl list

# Verify entry files exist
ls -lh /boot/loader/entries/*rescue*.conf

# Check entry syntax
cat /boot/loader/entries/alpine-rescue.conf
cat /boot/loader/entries/arch-rescue.conf

# Verify kernels copied to ESP
ls -lh /boot/alpine-rescue/
ls -lh /boot/arch-rescue/
```

## Testing Boot Entries

```fish
# Set next boot entry (without reboot persistence)
bootctl set-oneshot alpine-rescue.conf

# Or set default entry
bootctl set-default arch-rescue.conf

# Reboot to test
reboot
```

## Maintenance

### Updating Kernels

When Alpine or Arch updates kernels, refresh the ESP copies:

```fish
# Alpine kernel update
mount /dev/sdh2 /mnt && cp /mnt/boot/vmlinuz-lts /mnt/boot/initramfs-lts /boot/alpine-rescue/ && umount /mnt

# Arch kernel update
mount /dev/sdh4 /mnt && cp /mnt/boot/vmlinuz-linux /mnt/boot/initramfs-linux*.img /boot/arch-rescue/ && umount /mnt
```

### Automation with Systemd Service

Create a service to auto-sync kernels after updates:

```fish
# Create sync script
cat > /usr/local/bin/sync-rescue-kernels.sh << 'EOF'
#!/bin/sh
mount /dev/sdh2 /mnt 2>/dev/null && cp /mnt/boot/vmlinuz-lts /mnt/boot/initramfs-lts /boot/alpine-rescue/ && umount /mnt
mount /dev/sdh4 /mnt 2>/dev/null && cp /mnt/boot/vmlinuz-linux /mnt/boot/initramfs-linux*.img /boot/arch-rescue/ && umount /mnt
EOF

chmod +x /usr/local/bin/sync-rescue-kernels.sh

# Create systemd path unit (triggers on ESP changes)
cat > /etc/systemd/system/sync-rescue-kernels.path << 'EOF'
[Unit]
Description=Monitor ESP for kernel changes

[Path]
PathChanged=/boot/

[Install]
WantedBy=multi-user.target
EOF

# Create systemd service unit
cat > /etc/systemd/system/sync-rescue-kernels.service << 'EOF'
[Unit]
Description=Sync rescue kernels to ESP

[Service]
Type=oneshot
ExecStart=/usr/local/bin/sync-rescue-kernels.sh
EOF

# Enable monitoring
systemctl enable sync-rescue-kernels.path
systemctl start sync-rescue-kernels.path
```

## Troubleshooting

### Boot Entry Not Appearing

```fish
# Check entry file syntax
bootctl list

# Verify ESP is mounted
mount | grep -E '(boot|efi)'

# Refresh boot entries
bootctl update
```

### Kernel Panic on Boot

```fish
# Use fallback initramfs (Arch)
bootctl set-oneshot arch-rescue-fallback.conf

# Check root UUID matches
blkid /dev/sdh4
cat /boot/loader/entries/arch-rescue.conf

# Add debug options (remove 'quiet', add 'debug')
# Edit: options root=UUID=... rw debug
```

### Wrong Root Filesystem

```fish
# Verify filesystem type
blkid -s TYPE /dev/sdh2

# Update rootfstype in boot entry
# Alpine: rootfstype=btrfs (if using btrfs)
# Arch: typically doesn't need explicit rootfstype
```

## Advanced Options

### Additional Kernel Parameters

```conf
# Enable serial console (emergency remote access)
options root=UUID=... console=tty0 console=ttyS0,115200n8

# Single-user mode (root shell, no services)
options root=UUID=... single

# Emergency mode (minimal mount, root shell)
options root=UUID=... systemd.unit=emergency.target

# Disable graphics (text mode only)
options root=UUID=... nomodeset

# Maximum verbosity (for debugging)
options root=UUID=... debug loglevel=7
```

## Idiom Summary

1. **UUID over device names**: `/dev/sdh2` can become `/dev/sdi2` after hotplug; UUIDs are stable
2. **ESP copies**: Systemd-boot requires kernels on the ESP (FAT32 partition), not on root partitions
3. **Fallback initramfs**: Arch's fallback image includes ALL modules; use when hardware detection fails
4. **Module specification**: Alpine requires explicit `modules=` parameter; Arch auto-detects
5. **Portable scripting**: Using `printf` and `$()` ensures compatibility across bash, sh, dash, and fish

## References

- [systemd-boot documentation](https://www.freedesktop.org/software/systemd/man/systemd-boot.html)
- [bootctl man page](https://www.freedesktop.org/software/systemd/man/bootctl.html)
- [Arch Wiki: systemd-boot](https://wiki.archlinux.org/title/systemd-boot)
- [Alpine Wiki: Boot Process](https://wiki.alpinelinux.org/wiki/Boot_Process)
