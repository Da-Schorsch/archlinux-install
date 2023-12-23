# System einrichten

## Pacman

Parallele Downloads aktivieren. Dazu muss das Kommentarzeichen vor `ParallelDownloads` in der Pacman-Konfig entfernt werden.

```text
nano /etc/pacman.conf
---------------------
ParallelDownloads = 5
````

## Doas einrichten

Anstelle von `sudo` wird `doas` benutzt.

```txt
nano /etc/doas.conf
-------------------
permit persist :wheel

```

Die Configdatei muss mit einer Leerzeile enden! Es ist außerdem notwendig, dass die Konfiguration frei von Fehlern ist. Prüfen mit:

```bash
doas -C /etc/doas.conf && echo "config ok" || echo "config error"
```

Jetzt noch die richtigen Zugriffsberechtigungen setzen.

```bash
chown -c root:root /etc/doas.conf
chmod -c 0400 /etc/doas.conf
```

### Direkter Ersatz für `sudo`

Symlink von `doas` zu `sudo` anlegen

```bash
ln -s $(which doas) /usr/bin/sudo
```

Ersatz für `sudoedit` über systemweites Environment konfigurieren.

```text
nano /etc/bash.bashrc
---------------------
alias sudoedit='doas rnano'
```

### Bash Tab-Vervollständigung

Global über `/etc/bash.bashrc`.

```text
nano /etc/bash.bashrc
---------------------
complete -cf doas
```

## Benutzer und Gruppen

Neuen Benutzer anlegen. Da dieser Benutzer die Berechtigung haben soll, über `doas` Adminrechte zu erlangen, wird er der zusätzlichen Gruppe **wheel** zugeordnet.

```bash
useradd -m -G wheel schorsch
```

## Ab hier TODO

* sudo statt doas
* sudo Konfiguration schon im chroot statt nach Neustart
* Ditto für Anlegen des Benutzerkontos
* Konfiguration des Intelgrafiktreibers (GuC/HuC)
* iommu Konfig in ` /etc/kernel/cmdline`
* Kurze Zeitverzögerung + Menu für Bootloader (für alternativen Kernel oder Fallback)

## smartmontools

Installation evtl. schon im Livesystem!

TODO: `smartd`-Konfiguration

### Netzwerk

Todo: Connman, iwd, ...

#### iwd

Verzeichnis für Konfiguration erstellen. (wireless regdb nicht vergessen)

```bash
mkdir -p /etc/iwd
```

```text
[Scan]
DisablePeriodicScan=true
```

### Bees (BTRFS dedup)

...

### zram

...

### Errors im Syslog

* /boot nach /efi verschieben
  * Pfade in Pacman Hooks anpassen
  * Sonstige Configs prüfen (mkinitcpio, etc)
  * Zugriffsrechte in fstab anpassen
