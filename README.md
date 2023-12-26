# Archlinux-Installation

## Ziele

- [X] Boot
  - [X] EFI-Boot
  - [ ] Secure-Boot aktiv
  - [X] Systemd-boot als Bootmanager
- [X] BTRFS mit Möglichkeit für Snapshots
  - [X] Verschlüsselte Systempartition
- [X] Swap mit zram statt swap-Partition
- [ ] Initramfs mit `booster` erzeugen statt `mkinitcpio`

## Installationsmedium erzeugen

[Bootbaren USB-Stick erstellen.](docs/create_bootmedia.md)

## Livesystem

[Livesystem booten und Grundeinstellungen vornehmen.](docs/setup_livesystem.md)

## Datenträger vorbereiten

Hier gibt es zwei Möglichkeiten:

* [Standard (unverschlüsselt)](docs/prepare_disks.md)
* [Verschlüsselte Systempartition](docs/prepare_disks_encrypted.md)

## Grundsystem installieren und konfigurieren

[Installation und Konfiguration des Grundsystems.](docs/install_base_system.md)

## Neustart

Mit `Ctrl+d` die Chroot-Umgebung verlassen und mit `umount -R /mnt` alle eingehängten Partitionen aushängen. Dann mit `reboot` neustarten.

Jetzt ist beten und hoffen angesagt.

## Post-Install

[Systemkonfiguration und Administration](docs/post-install.md)

## Debug

[Mit Livesystem in bestehende Installation booten.](docs/archiso-to-chroot.md)
