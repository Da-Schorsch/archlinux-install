# Archlinux-Installation

## Ziele

* Boot
  * EFI-Boot
  * Secure-Boot aktiv[^1]
  * Systemd-boot als Bootmanager
* BTRFS mit Möglichkeit für Snapshots
* Verschlüsselte Systempartition[^1]
* ZSwap statt Swappartition
* Initramfs mit `booster` erzeugen statt `mkinitcpio` oder `dracut`[^1]

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

[^1]: Erst in einem späteren Anlauf, wenn der Rest funktioniert!
