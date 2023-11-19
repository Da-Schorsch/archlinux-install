# Datenträger vorbereiten

Da das System im EFI-Modus gebooted werden soll, muss auf dem Datenträger eine GPT-Partitionstabelle erstellt werden. Es werden im Anschluss zwei Partitionen erzeugt. Eine EFI-Systempartition (**E**FI **S**ystem **P**artition, im weiteren Verlauf abgekürzt als `esp`) mit 1024 MiB und eine Systempartition die den Rest des Datenträgers einnimmt (abzüglich ein paar Gigabyte für Overprovisioning). Da Swapspeicher als ZSwap nur im RAM lebt, wird keine Swappartition benötigt.

## Partitionstabelle und Partitionen erstellen

Vorhandene Datenträger/Partitionen auflisten. Es wird anschließend immer `/dev/sda` als Ziel für die Installation angenommen. Bei abweichender Hardware (z.B. M2 SSD) muss das Zielgerät entsprechend angepasst werden.

```bash
fdisk -l
```

Alternativ kann auch `lsblk` verwendet werden.

```bash
lsblk -f
```

Eine eventuell bestehende Partitionstabelle kann mit `wipefs` gelöscht werden.

```bash
wipefs --all /dev/sda
```

Den Datenträger mit `cgdisk` partitionieren.

```bash
cgdisk /dev/sda
```

Folgende Partitionstabelle einrichten.

| Partition   | Partitionstyp | Partitionsname | Größe  | Einhängepunkt | Dateisystem |
| :---------- | :------------ | :------------- | :----- | :------------ | :---------- |
| `/dev/sda1` | `EF00`        | `esp`          | `1G`   | `/efi`        | `FAT32`     |
| `/dev/sda2` | `8300`        | `arch`         | `Rest` | `/`           | `BTRFS`     |

Wichtig: Den Startoffset für die erste Partition so akzeptieren wie vorgeschlagen! Das sorgt dafür, dass die Partitionen richtig an den Sektorgrenzen ausgerichtet sind (Stichwort: 4k-Sektoren).

Anschließend die Ausrichtung nochmal überprüfen. Die Partitionen sind richtig ausgerichtet, wenn die Nummer des Start-Sektors durch 2048 dividiert werden kann.

```bash
fdisk -l -u /dev/sda
```

## Dateisysteme erzeugen

### ESP

Muss wegen UEFI ein FAT-32 Dateisystem sein.

```bash
mkfs.fat -F 32 /dev/sda1
```

### Systempartition

Btrfs Dateisystem. Label: root, Nodegröße 32k, Metadaten zweifach, Daten einfach.

```bash
mkfs.btrfs -L root -n 32k -m dup -d single /dev/sda2
```

#### BTRFS Subvolumes erzeugen

TODO: Hintergründe für das gewählte Subvolume-Layout erklären. (Snapshots mit Snapper; Snapshots reichen nur bis an Subvolumegrenzen; `$HOME` und `/var/log` unverändert, wenn für `/` ein Rollback gemacht wird; `/var/cache/pacman/pkg`, `/var/tmp` und `/srv` sind von Snapshots ausgeschlossen)

Das Dateisystem muss temporär unter `/mnt` einhängt werden, um Subvolumes erzeugen zu können. Das endgültige Layout sieht ein Subvolume auch für das Rootdateisystem vor. Daher muss das Dateisystem später nochmal ausgehängt und anders neu gemountet werden.

```bash
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async /dev/sda2 /mnt
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
```

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
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@ /dev/sda2 /mnt
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
mount /dev/sda1 /mnt/efi
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@snapshots /dev/sda2 /mnt/.snapshots
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@home /dev/sda2 /mnt/home
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=@var_log /dev/sda2 /mnt/var/log
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=srv /dev/sda2 /mnt/srv
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=var/cache/pacman/pkg /dev/sda2 /mnt/var/cache/pacman/pkg
mount -o defaults,rw,ssd,noatime,noautodefrag,compress-force=zstd:2,space_cache=v2,discard=async,subvol=var/tmp /dev/sda2 /mnt/var/tmp
```
