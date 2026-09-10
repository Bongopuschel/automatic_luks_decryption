# Nachträgliche Verschlüsselung eine bestehenden Systems

# Ausgangslage
Die Systempartition nvme0n1p3 soll verschlüsselt werden. Die EFI und boot Partition bleiben unverschlüsselt. Die Systempartitionen sind alle BTRFS formatiert:
```bash
~# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1                         259:0    0 476,9G  0 disk
├─nvme0n1p1                     259:1    0   487M  0 part  /boot/efi
├─nvme0n1p2                     259:2    0   4,3G  0 part  /boot
└─nvme0n1p3                     259:3    0 472,2G  0 part
    ├─vg--system-lv--swap       253:1    0  15,3G  0 lvm   [SWAP]
    ├─vg--system-lv--tmp        253:2    0  19,7G  0 lvm   /tmp
    ├─vg--system-lv--varlog     253:3    0  19,7G  0 lvm   /var/log
    ├─vg--system-lv--audit      253:4    0   9,9G  0 lvm   /var/log/audit
    ├─vg--system-lv--vartmp     253:5    0   9,9G  0 lvm   /var/tmp
    ├─vg--system-lv--quarantine 253:6    0   9,9G  0 lvm   /quarantine
    ├─vg--system-lv--home       253:7    0  98,7G  0 lvm   /home
    └─vg--system-lv--root       253:8    0 281,1G  0 lvm   /
```
# 1.) Livesystem
Ausführende Arbeiten müssen von einem Live System durchgeführt werden!

Platz am Ende des LVM-PV schaffen und Dateisystem um 1GiB verkleinern:
```bash
~# vgchange -ay vg-system
~# mount /dev/vg-system/lv-root /mnt
~# btrfs filesystemm resize -1G /mnt
~# umount /mnt
~# lvredurce -L -1G /dev/vg-system/lv-root
~# pvresize --setphisicalvolumesize 450G /dev/nvme0n1p3
~# vgchange -an
```
LVM Verschluesseln:
```bash
~# cryptsetup reencrypt --encrypt --type luks2 --cipher aes-xts-plain64 --key-size 512 --pbkdf argon2id --iter-time 3000 --reduce-device-size 32M /dev/nvme0n1p3
```
LVM an die neue Mapper Größe anpassen:
```bash
~# cryptsetup open /dev/nvme0n1p3 cryptlvm
~# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1                         259:0    0 476,9G  0 disk
├─nvme0n1p1                     259:1    0   487M  0 part  /boot/efi
├─nvme0n1p2                     259:2    0   4,3G  0 part  /boot
└─nvme0n1p3                     259:3    0 472,2G  0 part
  └─cryptlvm                    253:0    0 472,2G  0 crypt
~# pvresize --setphisicalvolumesize 450G /dev/mapper/cryptlvm
```
System bootbar machen. Dazu muss das System gechrootet, die Komponenten installiert und eingerichtet werden:
```bash
~# vgchange -ay
~# mount -o subvol=@rootfs /dev/vg-system/lv-root /mnt
~# mount /dev/nvme0n1p2 /mnt/boot
~# mount /dev/nvme0n1p1 /mnt/boot/efi
~# mount /dev/vg-system/lv-tmp /mnt/tmp
~# mount --bind /dev /mnt/dev
~# mount --bind /run /mnt/run
~# mount -t proc proc /mnt/proc
~# mount -t sysfs sys /mnt/sys
~# chroot /mnt
...
~# apt update && apt install cryptsetup cryptsetup-initramfs -y
~# printf "cryptlvm UUID=$(cryptsetup luksUUID /dev/nvme0n1p3) none luks" > /etc/crypttab
~# update-initramfs -u -k all
~# exit
...
~# reboot
```

