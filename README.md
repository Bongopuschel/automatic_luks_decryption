# LUKS-Verschlüsselung mit einem TPM-2.0-Chip

Ein TPM-2.0-Chip kann verwendet werden, um ein mit LUKS2 verschlüsseltes Dateisystem beim Systemstart automatisch zu entsperren. Die Entsperrung per Passwort bleibt dabei weiterhin möglich.
Diese Anleitung wurde unter Debian 12 „Bookworm“ getestet.

# Ausgangssituation
Ausgangspunkt ist ein bereits vorhandenes und mit LUKS verschlüsseltes Dateisystem. In diesem Beispiel ist lediglich die Partition /dev/nvme0n1p3 verschlüsselt:
```bash
~# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1                         259:0    0 476,9G  0 disk
├─nvme0n1p1                     259:1    0   487M  0 part  /boot/efi
├─nvme0n1p2                     259:2    0   4,3G  0 part  /boot
└─nvme0n1p3                     259:3    0 472,2G  0 part
  └─cryptlvm                    253:0    0 472,2G  0 crypt
    ├─vg--system-lv--swap       253:1    0  15,3G  0 lvm   [SWAP]
    ├─vg--system-lv--tmp        253:2    0  19,7G  0 lvm   /tmp
    ├─vg--system-lv--varlog     253:3    0  19,7G  0 lvm   /var/log
    ├─vg--system-lv--audit      253:4    0   9,9G  0 lvm   /var/log/audit
    ├─vg--system-lv--vartmp     253:5    0   9,9G  0 lvm   /var/tmp
    ├─vg--system-lv--quarantine 253:6    0   9,9G  0 lvm   /quarantine
    ├─vg--system-lv--home       253:7    0  98,7G  0 lvm   /home
    └─vg--system-lv--root       253:8    0 281,1G  0 lvm   /
```

# Vorraussetzungen
- TPM-2.0 Chip ist vorhanden und im UEFI aktiviert.
- Das System wird im UEFI-Modus gestartet; der Legacy- beziehungsweise CSM-Modus ist deaktiviert.
- Das LUKS-Volume verwendet LUKS2.
- Root-Rechte sind vorhanden.

# 1.) TPM-2.0 prüfen
Zunächst wird geprüft, ob das TPM-Gerät vom System erkannt wurde:
```bash
~# ls -l /dev/tpm*
```
Bei einem funktionierendem TPM2 sollte beispielsweise vorhanden sein:
```bash
/dev/tpm0
/dev/tpmrm0
```
Anschließend kann die TPM-Version überprüft werden:
```bash
~# cat /sys/class/tpm/tpm0/tmp_version_major
2
```
Die Ausgabe 2 bestätigt, dass ein TPM-2.0-Chip erkannt wurde.

# 2.) Benötigte Pakete installieren
```bash
~# apt install tpm2-tools clevis clevis-luks clevis-tpm2 clevis-initramfs
```

# 3.) Clevis an TPM-2.0 binden
Mit dem folgenden Befehl können die Eigenschaften des TPM-2.0-Chips angezeigt werden:
```bash
~# tpm2_getcap properties-fixed
TPM2_PT_FAMILY_INDICATOR:
  raw: 0x322E3000
  value: "2.0"
TPM2_PT_LEVEL:
  raw: 0
TPM2_PT_REVISION:
  raw: 0x74
  value: 1.16
...
```
Nun wird das LUKS-Volume /dev/nvme0n1p3 an den TPM-2.0-Chip gebunden:
```bash
~# clevis luks bind -d /dev/nvme0n1p3 tpm2 '{}'
...
```
Während des Vorgangs wird das vorhandene LUKS-Passwort abgefragt. Dieses Passwort wird benötigt, um einen neuen LUKS-Keyslot für Clevis anzulegen.
Nach erfolgreicher Einrichtung kann der neue Keyslot beispielsweise wie folgt angezeigt werden:
```bash
2: tpm2 '{"hash":"sha256","key":"ecc"}'
```
Die verwendete Slotnummer kann von diesem Beispiel abweichen.

# 4.) Initramfs aktualisieren
Damit Clevis und die TPM-Unterstützung bereits während des Systemstarts verfügbar sind, muss das Initramfs aktualisiert werden:
```bash
~# update-initramfs -u -k all
```

Optional kann überprüft werden, ob Clevis und die TPM-Unterstützung in das Initramfs eingebunden wurden:
```
~# lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'clevis|tpm2'
```

# Abschluss
Beim nächsten Systemstart bleibt die Entsperrung per LUKS-Passwort weiterhin möglich. Wenn die notwendigen Bedingungen erfüllt sind, versucht Clevis parallel dazu, das LUKS-Volume mithilfe des TPM-2.0-Chips automatisch zu entsperren.
Schlägt die automatische Entsperrung fehl, kann das Volume weiterhin wie gewohnt manuell mit dem LUKS-Passwort entsperrt werden.

# Nützliche Befehle
LUKS Informationen:
```bash
~# cryptsetup luksDump PARTITION
```
LUKS-UUID:
```bash
~# cryptsetup luksUUID PARTITION
```
Clevis LUKS Schlüssel anzeigen:
```bash
~# clevis luks list -d PARTITION
```
Clevis Slots löschen:
```bash
~# clevis luks unbind -d PARTITION -s SLOTNUMMER
```
TPM Informationen:
```bash
~# ls -l /dev/tpm*
~# tpm2_getcap prooerties-fixed
```
LUKS Passwort Entschlüsselung testen:
```bash
~# cryptsetup open --test-passphrase --verbose PARTITION
```
Clevis TPM-2.0 Entschlüsselung testen. Das Ergebnis sollte 0 sein:
```bash
~# clevis luks pass -d /dev/nvme0n1p3 -s 1 > /dev/null ; echo $?
```
