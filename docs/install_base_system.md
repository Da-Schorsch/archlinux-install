# Installation und Konfiguration des Grundsystems

## Grundlegende Pakete installieren

Sollten Signaturfehlermeldungen auftauchen, dann ist wahrscheinlich das Livesystem schon nicht mehr "ganz frisch" und der Arch Keyring muss aktualisiert werden.

```bash
pacman -Sy archlinux-keyring
```

Statt des Arch Linux Standardkernels (Paket `linux`) wird gleich der Zen-Kernel installiert. Der Befehl `pacstrap` kann auch gleich weitere Pakete installieren, die später benötigt werden. Damit ist nach dem Booten schon ein "kompletteres" System vorhanden.

```bash
pacstrap /mnt base base-devel linux-zen linux-firmware alsa-firmware intel-ucode iucode-tool btrfs-progs dosfstools e2fsprogs nvme-cli xfsprogs cryptsetup bash-completion lzop nano reflector rsync sbctl sbsigntools terminus-font tmux which
```

## Konfiguration des Grundsystems

### Fstab

Eine `fstab` Datei erstellen basieren auf der Datei des Livesystems (mit Option -U für UUID als Referenz statt des Gerätenamens).

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot

Mit `arch-chroot` in das neue System wechseln.

```bash
arch-chroot /mnt
```

### Zeitzone

Der `hwclock` Befehl setzt voraus, das die Systemzeit auf UTC steht!

```bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
```

### Lokalisierung

Die gewünschten Lokalisierungen konfigurieren und erzeugen.

```text
nano /etc/locale.gen
--------------------
# Die Kommentarzeichen entfernen für:
en_US.UTF-8

--------------------
# Im Anschluss
locale-gen
```

Und anschließend das System entsprechend konfigurieren.

```text
nano /etc/locale.conf
---------------------
LANG=en_US.UTF-8
LC_MESSAGES=C
LC_NUMERIC=C
```

Zum Schluss noch die Tastaturbelegung und Konsolenschriftart (beides optional)

```text
nano /etc/vconsole.conf
-----------------------
FONT=ter-u18n
FONT_MAP=8859-15
KEYMAP=de-latin1
XKBLAYOUT=de
XKBMODEL=pc105
XKBOPTIONS=caps:escape
XKBVARIANT=nodeadkeys
```

### Zusätzliche Software installieren

Mit `tmux` eine neue Tmux-Session öffnen. Das ist wichtig, da bei der Installation seitenweise Ausgaben von Pacman erzeugt werden, unter anderem mit empfohlenen Paketen.

```bash
pacman -S sudo alsa-utils blueman bluez{,-hid2hci,-utils,-tools,-plugins} d-feet dbus-broker doublecmd-qt5 engrampa flatpak foot foot-terminfo fwupd gamemode gamescope gnome-firmware gnome-disk-utility htop intel-gpu-tools irqbalance keepassxc mako mandoc mangohud mc mediainfo-gui mesa mesa-utils mesa-vdpau mousepad mpv networkmanager nm-connection-editor noto-fonts{,-emoji,-cjk,-extras} {otf,ttf}-font-awesome polkit polkit-gnome powertop qt5ct ripgrep ristretto sway{,bg,idle,lock} thermald thunar{,-volman,-{archive,media-tags}-plugin} ttf-croscore usbutils vdpauinfo waybar wireless-regdb wofi wpa_supplicant xdg-user-dirs xorg-xwayland xsettingsd yt-dlp zram-generator
```

Zusätzliche Abhängigkeiten installieren. Als extra Schritt, um den Vorgang mit `pacman` mit der Option `--asdeps` auch als Abhängigkeit deklarieren zu können. Die folgende Liste ist nur ein Startpunkt. Im letzten Schritt sind von Pacman sicherlich deutlich mehr zusätzliche Pakete empfohlen worden. -> Die Vorschläge durchgehen und abwägen.

```bash
pacman -S --asdeps intel-media-driver intel-media-sdk libva-intel-driver libva-mesa-driver libva-utils libvdpau libvdpau-va-gl lm_sensors lsof mesa-vdpau qt5-wayland vulkan-intel vulkan-mesa-layers
```

Für Rechner mit AMD-Grafik `vulkan-intel` durch `vulkan-radeon` ersetzen und die anderen `*intel*` Pakete weglassen.

### Netzwerkkonfiguration

#### Netzwerkschnittstellen finden und Status prüfen/ändern

`ip link` listet die vorhandenen Netzwerkschnittstellen auf und zeigt den Status (DOWN/UP) an. Mit `ip link show dev enp2s0` lässt sich der Status von einem bestimmten Interface (hier *enp2s0* als Beispiel) anzeigen und mit `ip link set enp2s0 up|down` kann diese Schnittstelle aktiviert oder deaktiviert werden.

#### Hostname

Hostnamen setzen. Da im Livesystem `hostnamectl set-hostname` noch nicht funktioniert, muss der Hostname in `/etc/hostname` geschrieben werden.

```text
nano /etc/hostname
------------------
optiplex
```

#### Lokale Namensauflösung

Die lokale Namensauflösung erfolgt durch `nss-myhostname` (gehört zu systemd) und ist standardmäßig aktiviert. `/etc/hosts` muss eigentlich nicht editiert werden, aber um Probleme mit Software zu vermeiden, die doch `/etc/hosts` direkt liest, sollte folgender Eintrag gemacht werden.

```text
nano /etc/hosts
---------------
127.0.0.1        localhost
::1              localhost
127.0.1.1        optiplex
```

Prüfen, ob alles passt. `getent hosts` Sollte folgende Ausgabe liefern.

```text
127.0.0.1       localhost
127.0.0.1       localhost
127.0.1.1       optiplex
```

#### NetworkManager

`NetworkManager` aktivieren.

```bash
systemctl enable NetworkManager.service
```

#### Wifi einrichten

Zuerst die korrekt Ländereinstellung setzen.

```text
nano /etc/conf.d/wireless-regdom
--------------------------------
# Die Kommentarzeichen entfernen für:
WIRELESS_REGDOM="DE"
```

Die Einstellung wird allerdings erst nach einem Neustart aktiv. Für die Konfiguration bis zum ersten Neustart kann noch mit folgendem Befehl nachgeholfen werden.

```bash
iw reg set DE
```

Mit `nmtui` das ncurses Interface für NetworkManager starten. Damit kann nach Netzwerken gesucht und auch damit verbunden werden.

Die Alternative, auf der Kommandozeile `nmcli`, zu verwenden hat den Nachteil, dass das WLAN-Passwort in Klartext eingegeben werden muss und dann unverschlüsselt in der bash history landet.

#### Verbindungsüberprüfung (captive portal)

Unklar, ob diese Funktion nicht auch schon standardmäßig aktiv ist (mit Config unter `/usr/lib/NetworkManager/conf.d/20-connectivity.conf`).

Für alle Fälle:

```ini
nano /etc/NetworkManager/conf.d/20-connectivity.conf
----------------------------------------------------
[connectivity]
enabled=true
uri=http://ping.archlinux.org/nm-check.txt
```

#### DNS Management mit systemd-resolved

NetworkManager kann `systemd-resolved` als DNS-Resolver und -Cache verwenden. `systemd-resolved` wird dabei im stub-Modus betrieben. Als erstes muss der Service aktiviert werden.

```bash
systemctl enable systemd-resolved
```

NetworkManager verwendet automatisch `systemd-resolved` wenn `/etc/resolv.conf` ein Symlink auf `/run/systemd/resolve/stub-resolv.conf` ist. Um den Symlink zu setzen, muss das Chroot verlassen werden da im Chroot `/run` leer ist. Also entweder mit `Ctrl+d` oder `exit` das Chroot verlassen. Dann im Livesystem den Symlink setzen.

```bash
ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
```

Wenn die Konfiguration nicht im Livesystem erfolgt, sondern erst wenn das System schon "richtig" gebootet wurde, dann muss `/mnt/etc/resolv.conf` durch `/etc/resolv.conf` ersetzt werden.

Falls man sich nicht nur auf den Symlink verlassen will, kann NetworkManager auch explizit nochmal darauf hingewiesen werden, dass `system-resolved` verwendet werden soll.

```ini
nano /etc/NetworkManager/conf.d/dns.conf
----------------------------------------
[main]
dns=systemd-resolved
```

#### NTP

```bash
systemctl enable systemd-timesyncd.service
```

#### Bluetooth

Experimentelle DBus-Interfaces in der Bluetoothkonfiguration aktivieren.

```text
nano /etc/bluetooth/main.conf
-----------------------------
Experimental = true
```

Anschließend den Bluetoothdienst aktivieren.

```bash
systemctl enable bluetooth.service
```

#### TODOs und offene Fragen

[Systemd-Resolved Seite im ArchWiki für weitere Recherche](https://wiki.archlinux.org/title/Systemd-resolved)

* systemd-resolvd Konfiguration
  * Normales DNS sollte einfach funktionieren
    * Link für Stub-Resolver: passt das alles?
    * DNS (DNSSEC, DoH/DoT), mDNS und LLMNR; Was davon geht nicht und wird das wirklich benötigt?
* Ifplugd? Oder, wenn Stromsparen gefragt ist, nochmal probieren, ob Wifi sparsamer ist.

### Root Passwort

Das Rootpasswort setzen.

```bash
passwd
```

### Bootloader

#### Installation

Als Bootloader wird systemd-boot verwendet. Wenn die UEFI System Partition (esp) unter `/boot` oder `/esp` eingehängt ist, sind keine weiteren Optionen nötig. Der Bootloader kann direkt mit

```bash
bootctl install
```

installiert werden.

#### Automatisches Update einrichten

Der Bootloader muss bei einem Update von *systemd* (und damit auch *systemd-boot*) aktualisiert werden. Dies lässt sich mit einem pacman hook automatisieren.

Als erstes muss der `systemd-boot-update.service` aktiviert werden.

```bash
systemctl enable systemd-boot-update.service
```

Anschließend muss noch der pacman hook angelegt werden.

```bash
mkdir -p /etc/pacman.d/hooks
```

```ini
nano /etc/pacman.d/hooks/100-systemd-boot.hook
----------------------------------------------
[Trigger]
Type = Package
Operation = Upgrade
Target = systemd

[Action]
Description = Gracefully upgrading systemd-boot...
When = PostTransaction
Exec = /usr/bin/systemctl restart systemd-boot-update.service
```

Wenn Secure Boot aktiviert ist, muss der Kernel und Bootloader bei einem Update außerdem neu signiert werden. Dafür wird nochmal ein pacman hook benötigt. Diesmal unter `/etc/pacman.d/hooks/99-secureboot.hook`

```ini
nano /etc/pacman.d/hooks/99-secureboot.hook
-------------------------------------------
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux
Target = systemd

[Action]
Description = Signing Kernel for Secure Boot
When = PostTransaction
Exec = /usr/bin/find /boot -type f ( -name vmlinuz-* -o -name systemd* ) -exec /usr/bin/sh -c 'if ! /usr/bin/sbverify --list {} 2>/dev/null | /usr/bin/grep -q "signature certificates"; then /usr/bin/sbsign --key db.key --cert db.crt --output "$1" "$1"; fi' _ {} ;
Depends = sbsigntools
Depends = findutils
Depends = grep
```

### Initramfs

#### Kernel-Optionen

Neue `/etc/kernel/cmdline` Datei mit benötigten Kerneloptionen erzeugen. Bei einer Installation ohne Festplattenverschlüsselung reicht die Angabe von `root=...`. Bei einer Installation mit Festplattenverschlüsselung muss dagegen erst die UUID der verschlüsselten Partition ermittelt und dann angegeben werden.

UUID ermitteln.

```bash
blkid /dev/nvme0n1p2

# Ausgabe als Beispiel
# >> /dev/nvme0n1p2: UUID="ebbdcdc8-0edd-4155-8e18-1c10edd99861" TYPE="crypto_LUKS" PARTLABEL="luks" PARTUUID="37b7594a-3108-4b29-bee7-fce2170727e1"
```

`/etc/kernel/cmdline` bearbeiten.

```text
nano /etc/kernel/cmdline
------------------------
# für unverschlüsseltes System (Kommentarzeichen entfernen)
#root=/dev/nvme0n1p2

# für verschlüsseltes System
rd.luks.name=ebbdcdc8-0edd-4155-8e18-1c10edd99861=luks
root=/dev/mapper/luks

# ab hier für alle Varianten übernehmen
rootflags=subvol=@
rw
quiet
bgrt_disable
```

#### Kernelpresets für mkinitcpio

Kernelkonfig für Linux-Zen editieren. Den 'fallback' Preset entfernen, damit `mkinitcpio` später nicht andauernd mit "firmware missing" Meldungen rumnervt.

Die Einträge für **...efi_image** werden benötigt, da ein "Unified Kernel Image" erzeugt werden soll. Damit müssen Einträge unter `/boot/loader/entries` nicht von Hand gemacht werden, sondern *systemd-boot* kümmert sich automatisch darum. Es erleichtert außerdem das Signieren für Secure Boot.

Der Eintrag **ALL_microcode** sorgt dafür, dass beim Start von allen Presets der CPU-Microcode geladen wird.

```text
nano /etc/mkinitcpio.d/linux-zen.preset
---------------------------------------
# mkinitcpio preset file for the 'linux-zen' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-zen"
ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default')

default_uki="/efi/EFI/Linux/archlinux-linux-zen.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

fallback_uki="/efi/EFI/Linux/archlinux-linux-zen-fallback.efi"
fallback_options="-S autodetect"
```

Neue *initramfs* erzeugen, für alle Kernelkonfigurationen.

#### mkinitcpio.conf anpassen

Den **HOOKS** Eintrag in `mkinitcpio.conf` anpassen. Gefordert ist:

* Unterstützung für eine verschlüsselte Rootpartition
* Deutsches Tastaturlayout bei Passworteingabe im frühen Bootprozess
* Systemd init basierter Bootvorgang
* Unbedingte Unterstützung von USB-Eingabegeräten, unabhängig davon, welche Geräte beim Erzeugen der *initrd* gerade angeschlossen sind (wichtig bei Laptops)

Außerdem den Graphictreiber unter **MODULES** angeben für Early-KMS (korrekte Bildschirmauflösung schon beim Booten).

* `i915` für Intel-GPUs
* `amdgpu` für AMD-GPUs

```text
nano /etc/mkinitcpio.conf
-------------------------
MODULES=(i915)
HOOKS=(base systemd keyboard autodetect modconf kms block mdadm_udev sd-vconsole sd-encrypt lvm2 filesystems fsck)
```

```bash
mkinitcpio -P
```
