# Datenträger vorbereiten

ACHTUNG: Veraltet!!!!! Nach Vorgabe von `04a_prepare_disks.md` aktualisieren und dann diesen Text löschen!

Infos zum Durcharbeiten:

* [Data at rest encryption im Archwiki](https://wiki.archlinux.org/title/Data-at-rest_encryption)
* [dm-crypt im Archwiki](https://wiki.archlinux.org/title/Dm-crypt)
* [Sicheres Löschen im Archwiki](https://wiki.archlinux.org/title/Securely_wipe_disk)
* [SSD Besonderheiten im Archwiki](https://wiki.archlinux.org/title/Solid_state_drive/Memory_cell_clearing#Common_method_with_blkdiscard)

Da das System im EFI-Modus gebooted werden soll, muss auf dem Datenträger eine GPT-Partitionstabelle erstellt werden. Es werden im Anschluss zwei Partitionen erzeugt. Eine EFI-Systempartition (**E**FI **S**ystem **P**artition, im weiteren Verlauf abgekürzt als `esp`) mit 1024 MiB und eine Systempartition die den Rest des Datenträgers einnimmt (abzüglich ein paar Gigabyte für Overprovisioning). Da Swapspeicher als ZSwap nur im RAM lebt, wird keine Swappartition benötigt.

## Vorhandene Datenträger und Partitionen auflisten

Vorhandene Datenträger und darauf vorhandene Partitionen lassen sich mit `fdisk` oder mit `lsblk` auflisten.

```bash
fdisk -l
```

Alternativ:

```bash
lsblk -f
```

Es wird anschließend immer `/dev/nvme0n1` als Ziel für die Installation angenommen. Bei abweichender Hardware (z.B. M2 SSD) muss das Zielgerät entsprechend angepasst werden.

## Datenträger sicher löschen

* Bei SATA SSDs können mit `blkdiscard` auch Blöcke gelöscht werden, die vom Garbage Collector erzeugt wurden und sonst nicht zugänglich sind. Diese Funktion muss vom Laufwerk unterstützt werden!

```bash
blkdiscard --secure /dev/device
```

### Recherchieren und dann ausformulieren

* NVMe spezifische Dinge (Erase und Sanitize)

### Weiter im Text

!!Einleitung passt nach Umstrukturierung des vorherigen Absatzes nicht mehr!!

Das ist für ein verschlüsseltes Dateisystem, bei dem kein Unterschied zwischen beschriebenen und freien Stellen auf dem Datenträger erkennbar sein soll, nicht genug! Vielmehr muss der Datenträger vor dem Erstellen von Partitionen und Dateisystemen mit zufälligen Daten beschrieben werden.

Es gibt verschiedene Methoden das zu erreichen. In diesem Fall wird `cryptsetup` selbst verwendet, mit einem zufälligen Schlüssel. Zum Schreiben wird `dd_rescue` (Schreibweise beachten! `ddrescue` ist ein anderes Tool) verwendet, weil es eine Fortschrittsanzeige ermöglicht. `cat` und `dd` könnten alternativ auch verwendet werden.

Als erstes muss `dd_rescue` nachinstalliert werden, da nicht im Archlinux Installer vorhanden.

```bash
pacman -Sy dd_rescue
```

Den verschlüsselten Container mit zufälligem Passwort anlegen, öffnen und unter `/dev/mapper/target` verfügbar machen.

```bash
cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 target
```

Im Anschluss einen Strom von Nullen in den Container schieben, die dann verschlüsselt werden.

```bash
dd_rescue -w /dev/zero /dev/mapper/target
```

## Partitionstabelle und Partitionen erstellen

Den Datenträger mit `cgdisk` partitionieren.

```bash
cgdisk /dev/nvme0n1
```

Folgende Partitionstabelle einrichten.

| Partition        | Partitionstyp | Partitionsname | Größe  | Einhängepunkt      | Dateisystem |
| :--------------- | :------------ | :------------- | :----- | :------------      | :---------- |
| `/dev/nvme0n1p1` | `EF00`        | `esp`          | `1G`   | `/efi`             | `FAT32`     |
| `/dev/nvme0n1p2` | `8309`        | `luks`         | `Rest` | `/dev/mapper/luks` | `-`         |

Wichtig: Den Startoffset für die erste Partition so akzeptieren wie vorgeschlagen! Das sorgt dafür, dass die Partitionen richtig an den Sektorgrenzen ausgerichtet sind (Stichwort: 4k-Sektoren).

Anschließend die Ausrichtung nochmal überprüfen. Die Partitionen sind richtig ausgerichtet, wenn die Nummer des Start-Sektors durch 2048 dividiert werden kann.

```bash
fdisk -l -u /dev/nvme0n1
```

## Systempartition verschlüsseln

Vor dem Einsatz von `cryptsetup` sicherstellen, dass das `dm_crypt` Kernelmodul geladen ist.

```bash
lsmod | grep dm_crypt
```

Falls nicht, dann mit `modprobe` nachladen.

```bash
modprobe dm_crypt
```

Vor dem Anlegen der verschlüsselten Partition erstmal die verfügbaren Chiffren prüfen.

```bash
cryptsetup benchmark
```

Verschlüsselung mit LUKS. Alle Einstellungen, trotz der explizit angegebenen Parameter, entsprechen den Standardeinstellungen (Stand: Juli 2022), bis auf den `iter-time` Parameter. Dieser wurde vom Default 2000 ms auf 8000 ms erhöht. Zum Standard gehört inzwischen auch schon LUKS2.

```bash
cryptsetup --type=luks2 --cipher aes-xts-plain64 --hash sha256 --key-size 512 --iter-time=8000 --verify-passphrase luksFormat /dev/nvme0n1p2 # <- Passworteingabe erforderlich
cryptsetup luksDump /dev/nvme0n1p2 # zum Überprüfen
cryptsetup open /dev/nvme0n1p2 luks # Passwort erforderlich; durch den Paramter "luks" taucht die entsperrte Partition unter /dev/mapper/luks auf.
```

## Dateisysteme erzeugen

### ESP

Muss wegen UEFI ein FAT-32 Dateisystem sein.

```bash
mkfs.fat -F 32 /dev/nvme0n1p1
```

### Systempartition

Btrfs Dateisystem. Label: root, Nodegröße 32k, Metadaten zweifach, Daten einfach.

```bash
mkfs.btrfs -L root -n 32k -m dup -d single /dev/mapper/luks
```

#### BTRFS Subvolumes erzeugen

TODO: Hintergründe für das gewählte Subvolume-Layout erklären. (Snapshots mit Snapper; Snapshots reichen nur bis an Subvolumegrenzen; `$HOME` und `/var/log` unverändert, wenn für `/` ein Rollback gemacht wird; `/var/cache/pacman/pkg`, `/var/tmp` und `/srv` sind von Snapshots ausgeschlossen)

Das Dateisystem muss temporär unter `/mnt` einhängt werden, um Subvolumes erzeugen zu können. Das endgültige Layout sieht ein Subvolume auch für das Rootdateisystem vor. Daher muss das Dateisystem später nochmal ausgehängt und anders neu gemountet werden.

```bash
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async /dev/mapper/luks /mnt
```

Das geplante Dateisystemlayout sieht folgendermaßen aus:

```text
toplevel                   (volume root directory, not to be mounted by default)
  +-- @                    (subvolume root directory, to be mounted at /)
  +-- @home                (subvolume root directory, to be mounted at /home)
  +-- @snapshots           (subvolume root directory, to be mounted at /.snapshots)
  +-- @var_log             (subvolume root directory, to be mounted at /var/log)
  +-- var                  (directory)
  |   \-- log              (subvolume root directory, to be mounted at /var/log)
  |   \-- cache\pacman     (directory)
  |   \-- cache\pacman\pkg (subvolume root directory, to be mounted at /var/cache/pacman/pkg)
  |   \-- tmp              (subvolume root directory, to be mounted at /var/tmp)
  \-- srv                  (subvolume root directory, to be mounted at /srv)
```

Toplevel-Subvolumes erzeugen.

```bash
btrfs subvolume create /mnt/@           # /
btrfs subvolume create /mnt/@home       # /home
btrfs subvolume create /mnt/@snapshots  # /.snapshots
btrfs subvolume create /mnt/@var_log    # /var/log

Jetzt noch die Subvolumes für die, von den Snapshots ausgeschlossenen Verzeichnissen, erzeugen. Um diese Subvolumes erzeugen zu können, müssen zum Teil erst die übergeordneten Verzeichnisse erzeugt werden.

```bash
mkdir -p /mnt/var
btrfs subvolume create /mnt/var/cache

mkdir -p /mnt/var/cache/pacman
btrfs subvolume create /mnt/var/cache/pacman/pkg

btrfs subvolume create /mnt/var/tmp

btrfs subvolume create /mnt/srv
```

Zur Kontrolle alle erzeugten Subvolumes auflisten.

```bash
btrfs subvolume list /mnt # liefert folgende Ausgabe
-------------------------

ID 256 gen 16 top level 5 path @
ID 257 gen 8 top level 5 path @home
ID 258 gen 9 top level 5 path @snapshots
ID 259 gen 10 top level 5 path @var_log
ID 260 gen 12 top level 5 path var/cache
ID 261 gen 12 top level 260 path var/cache/pacman/pkg
ID 262 gen 13 top level 5 path var/tmp
ID 263 gen 14 top level 5 path srv
```

Das temporär eingehängte Root-Dateisystem unter `/mnt` wieder aushängen.

```bash
umount /mnt
```

## Dateisysteme einhängen

Erst das Rootdateisystem.

```bash
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@ /dev/mapper/luks /mnt
```

Verzeichnisse für die weiteren Mountpunkte erzeugen.

```bash
mkdir -p /mnt/.snapshots
mkdir -p /mnt/efi
mkdir -p /mnt/home
mkdir -p /mnt/srv
mkdir -p /mnt/var/cache/pacman/pkg
mkdir -p /mnt/var/log
mkdir -p /mnt/var/tmp
```

Die restlichen Dateisysteme und Subvolumes einhängen.

```bash
mount /dev/nvme0n1p1 /mnt/efi
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@home /dev/mapper/luks /mnt/home
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@var_log /dev/mapper/luks /mnt/var/log
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=srv /dev/mapper/luks /mnt/srv
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=var/cache/pacman/pkg /dev/mapper/luks /mnt/var/cache/pacman/pkg
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=var/tmp /dev/mapper/luks /mnt/var/tmp
```
