# Bootbaren USB-Stick erstellen

## Download

Zur [Arch Linux Downloadseite](https://archlinux.org/download) gehen. Die aktuelle ISO- (**archlinux-version-x86_64.iso**) und Checksummendatei (**sha256sums.txt**) von einem Mirror herunterladen. Der Mirror sollte HTTPS unterstützen, um Manipulationen der Dateien während der Übertragung zu verhindern.

Außerdem wird zur Verifikation noch die GnuPG-Signatur (**archlinux-version-x86_64.iso.sig**) benötigt. Diese unbedingt direkt von der Arch Downloadseite herunterladen, um auszuschließen, dass das Image, als auch die Signatur, auf dem Mirror manipuliert sind.

## Verifikation

Fehlerfreiheit der ISO-Datei durch Prüfung der SHA256-Checksumme verifizieren. Unter Windows kann für die folgenden Befehle **git bash** verwendet werden.

```bash
sha256sum --ignore-missing -c sha256sums.txt
```

Bei einer fehlerfreien Datei sollte die Ausgabe folgendermaßen aussehen.

```text
archlinux-version-x86_64.iso: OK
```

Außerdem selbst nochmal die SHA256-Checksumme anzeigen lassen und das Ergebnis mit der Prüfsumme auf der Arch Linux Downloadseite vergleichen.

```bash
sha256sum archlinux-version-x86_64.iso
```

Echtheit der ISO-Datei mit GPG überprüfen.

```bash
gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
```

Die Ausgabe sollte ungefähr folgendermaßen aussehen.

```text
gpg: assuming signed data in 'archlinux-2022.08.05-x86_64.iso'
gpg: Signature made Fr 05 Aug 2022 13:13:05 CEST
gpg:                using RSA key 4AA4767BBC9C4B1D18AE28B77F2D434B9741E8AC
gpg:                issuer "pierre@archlinux.de"
gpg: Good signature from "Pierre Schmitz <pierre@archlinux.de>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 4AA4 767B BC9C 4B1D 18AE  28B7 7F2D 434B 9741 E8AC
```

Die Angabe von **Primary key fingerprint** muss zur Angabe auf der Arch Linux Downloadseite passen. (**PGP fingerprint** im Abschnitt **Checksums**. Der Fingerprint erscheint als Tooltip nach Mouseover auf dem Hexwert).

## Image auf USB-Stick schreiben

Pfad für Datenträger herausfinden mit `lsblk`. Der Datenträger darf vor Ausführen des Schreibbefehls *nicht* gemounted sein.

```bash
dd bs=4M if=path/to/archlinux-version-x86_64.iso of=/dev/sdx conv=fsync oflag=direct status=progress
```

Unter Windows kann **Rufus** verwendet werden.
