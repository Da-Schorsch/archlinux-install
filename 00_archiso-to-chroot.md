---
title: Mit Archiso auf bestehendes System zugreifen
---

## Zugriff auf bestehendes System zu Reparaturzwecken

* Von Stick booten
* Mit `passwd` neues Passwort für root setzen (wegen SSH-Zugriff)

```bash
loadkeys de-latin1-nodeadkeys
setfont ter-u18b
timedatectl set-ntp true
timedatectl set-timezone Europe/Berlin

passwd
tmux

reflector --latest 100 --number 30 --connection-timeout 2 --download-timeout 2 --ipv4 --protocol https --sort rate --country 'AT,BE,CH,DE,FR,IT' --save /etc/pacman.d/mirrorlist
pacman -Sy archlinux-keyring

cryptsetup open /dev/nvme0n1p2 luks

mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@ /dev/mapper/luks /mnt

mount /dev/nvme0n1p1 /mnt/efi
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@home /dev/mapper/luks /mnt/home
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@var_log /dev/mapper/luks /mnt/var/log
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=srv /dev/mapper/luks /mnt/srv
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=var/cache/pacman/pkg /dev/mapper/luks /mnt/var/cache/pacman/pkg
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=var/tmp /dev/mapper/luks /mnt/var/tmp

arch-chroot /mnt
```
