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

[Bootbaren USB-Stick erstellen.](docs/02_create_bootmedia.md)

## Livesystem

[Livesystem booten und Grundeinstellungen vornehmen.](docs/03_setup_livesystem.md)

## Datenträger vorbereiten

Hier gibt es zwei Möglichkeiten:

* [Standard (unverschlüsselt)](docs/04a_prepare_disks.md)
* [Verschlüsselte Systempartition](docs/04b_prepare_disks_encrypted.md)

## Grundsystem installieren

[Installation des Grundsystems.](docs/05_install_base_system.md)

## System konfigurieren

[Konfiguration des Grundsystems.](docs/06_configure_base_system.md)

## Neustart

Mit `Ctrl+d` die Chroot-Umgebung verlassen und mit `umount -R /mnt` alle eingehängten Partitionen aushängen. Dann mit `reboot` neustarten.

Jetzt ist beten und hoffen angesagt.

## Post-Install

[Systemkonfiguration und Administration](docs/07_post-install.md)

## Debug

[Mit Livesystem in bestehende Installation booten.](docs/00_archiso-to-chroot.md)
