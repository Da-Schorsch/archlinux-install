---
title: Livesystem für Installation
---

## Boot

Von USB-Stick im EFI-Modus booten. Secure Boot muss dafür deaktiviert sein.

## Grundeinstellungen für das Livesystem

### Tastaturbelegung ändern (optional)

Optional, da eine **en_US** Tastaturbelegung während der Installations mit ihren vielen Pfadangaben ganz praktisch ist.

```bash
loadkeys de-latin1-nodeadkeys
```

### Terminalschriftart ändern

```bash
setfont ter-u18b
```

### Bootmode überprüfen

Der folgende Befehl muss eine Ausgabe bringen, ansonsten befindet sich das System im Bioskompatibilitätsmodus (CSM).

```bash
ls /sys/firmware/efi/efivars
```

### Netzwerkverbindung überprüfen

Netzwerkgeräte auflisten.

```bash
ip link
```

Funktionierende Verbindung prüfen.

```bash
ping -c3 archlinux.org
```

### Systemzeit aktualisieren

Erst NTP aktivieren und die richtige Zeitzone einstellen. Anschließend den Status prüfen.

```bash
timedatectl set-ntp true
timedatectl set-timezone Europe/Berlin
timedatectl status
```

### Von anderem Rechner mit ssh einloggen (optional)

Um die einzelnen Installationsschritte aus dieser Einleitung per Copy&Paste einfügen zu können, statt sie mühsam abtippen zu müssen, wäre das jetzt ein guter Zeitpunkt sich von einem anderen Rechner per `ssh` einzuloggen.

Zuerst muss ein Passwort für den root Account des Livesystems vergeben werden.

```bash
passwd
```

Um die Installation auch auf dem Bildschirm des zu installierenden Rechners mitverfolgen zu können wird noch `tmux` benötigt. Zuerst die tmux Session auf dem zu installierenden Rechner starten.

```bash
tmux
```

Dann aus der ssh Session heraus verbinden.

```bash
tmux a
```

## Mirrorliste aktualisieren

```bash
reflector --latest 100 --number 30 --connection-timeout 2 --download-timeout 2 --ipv4 --protocol https --sort rate --country 'AT,BE,CH,DE,FR,IT' --save /etc/pacman.d/mirrorlist
```

## Tools nachinstallieren

Diverse Helferlein nachinstallieren, die das Leben im Livesystem bis zum ersten Reboot leichter machen.

```bash
pacman -Sy iotop htop
```

## Alte Booteinträge löschen

Eventuell vorhandene EFI Booteinträge anzeigen lassen und im Anschluss löschen.

```bash
efibootmgr             # listet alle vorhandenen Einträge auf (der letzte Eintrag ist typischerweise der USB-Stick)
efibootmgr -B -b 0000  # löscht Eintrag 0000
```
