---
title: Installation des Grundsystems
---

## Grundlegende Pakete installieren

Sollten Signaturfehlermeldungen auftauchen, dann ist wahrscheinlich das Livesystem schon nicht mehr "ganz frisch" und der Arch Keyring muss aktualisiert werden.

```bash
pacman -Sy archlinux-keyring
```

Statt des Arch Linux Standardkernels (Paket `linux`) wird gleich der Zen-Kernel installiert. Der Befehl `pacstrap` kann auch gleich weitere Pakete installieren, die später benötigt werden. Damit ist nach dem Booten schon ein "kompletteres" System vorhanden.

```bash
pacstrap /mnt base linux-zen linux-firmware alsa-firmware intel-ucode iucode-tool btrfs-progs dosfstools e2fsprogs xfsprogs cryptsetup bash-completion which terminus-font nano lzop reflector rsync sbctl sbsigntools
```

## TODO: Zusätzliche Softwareinstallation nach später verschieben

polkit texinfo pigz (für dracut oder booster?)
mpv
