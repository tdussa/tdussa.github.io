# Server Setup: Mac Pro with Ubuntu 20.04 LTS

## Situation

There's this old Mac Pro with two hard drives (2 TB and 1 TB) that I want
to install Ubuntu on.  Thing is, for some reason I cannot select the boot
order (yes, I have tried holding down `Alt` (or the `Win` key) during
power-up, doesn't do anything).  It always boots off the HDDs, and if
there's nothing to boot, then it boots off its optical drive.


## Goal

So I want to have some redundancy, security, and flexibility while being
able to boot headlessly.  So, here's the idea:
 - The first partitions on both HDDs are 1 GB B in size and form a RAID-1.
   Mounted at `/root`.
 - The second partitions on both HDDs are 999 GB in size and form a RAID-1.
   This RAID is encrypted with LUKS and holds LVM volumes, one of which is
   mounted at `/`.
 - On the first HDD, the remainder of the drive resides in a third
   partition for non-RAID use.
 - Decryption should take place via `clevis` at boot time.


## Implementation

Unfortunately, the installer of Focal does not allow you to do anything
fancy at all in way of my design, so I have to install by hand.  That
should be fun... :)  So here goes.


### Prepping the System

Wipe the partition table somehow and boot into a Focal installer.  Then
select language, keymap, and network config.  Before touching the drives,
spawn a shell and set the password for the `ubuntu-server` user so we can
work remotely:
```
passwd ubuntu-server
```

Log into the system and partition the drives:
```
fdisk /dev/sda
fdisk /dev/sdb
```

Create the RAID arrays with sensible names.  The metadata version is
selected so the metadata block is put at the end of the volume.  This
allows the partitions to be used directly without any RAID in case of
emergency:
```
mdadm --create /dev/md/boot --level 1 --raid-devices 2 --metadata 1.0 /dev/sda1 /dev/sdb1 --name boot --homehost phanes
mdadm --create /dev/md/main --level 1 --raid-devices 2 --metadata 1.0 /dev/sda2 /dev/sdb2 --name main --homehost phanes
```
Note that this will trigger a syncing process that will take a bit for the
`main` RAID.

Create the LUKS container:
```
cryptsetup luksFormat /dev/md/main
```

Open the LUKS container and create the LVM setup:
```
cryptsetup luksOpen /dev/md/main main_crypt
pvcreate /dev/mapper/main_crypt
vgcreate phanes-main /dev/mapper/main_crypt
lvcreate --name root --size 100G phanes-main
```

Create the file systems:
```
mkfs.ext4 /dev/md/boot
mkfs.ext4 /dev/phanes-main/root
```

Mount the file systems:
```
mount /dev/phanes-main/root /mnt
mkdir /mnt/boot
mount /dev/md/boot /mnt/boot
```

Install the necessary `debootstrap` package:
```
apt install debootstrap
```

### Installing the System

Bootstrap the basic system:
```
debootstrap focal /mnt
```

Bind-mount the necessary filesystems and `chroot` into the fresh system:
```
mount -o bind /dev /mnt/dev
mount -o bind /dev/pts /mnt/dev/pts
mount -o bind /proc /mnt/proc
mount -o bind /sys /mnt/sys
chroot /mnt
```

Activate more repos:
```
cat <<'EOF' >> /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu focal universe
deb http://archive.ubuntu.com/ubuntu focal-security main
deb http://archive.ubuntu.com/ubuntu focal-security universe
deb http://archive.ubuntu.com/ubuntu focal-updates main
deb http://archive.ubuntu.com/ubuntu focal-updates universe
EOF
apt update
```

Update stuff:
```
apt dist-upgrade
```

Install missing packages:
```
apt install \
  build-essential \
  clevis-initramfs \
  clevis-systemd \
  cpio \
  cryptsetup \
  curl \
  dnsutils \
  etckeeper \
  git \
  grml-rescueboot \
  grub-pc \
  linux-base \
  linux-firmware \
  linux-generic \
  linux-headers-generic \
  locate  \
  lrzip \
  lvm2 \
  man-db \
  mdadm \
  memtest86+ \
  openssh-server \
  parted \
  rsync \
  tang \
  ubuntu-server \
  vim \
  wget \
  zsh
```

Setup locale information:
```
sed -i "s/^# en_DK.UTF-8 UTF-8$/en_DK.UTF-8 UTF-8/" /etc/locale.gen
locale-gen
```

Set the host name:
```
echo phanes > /etc/hostname
```

Configure network stuff:
```
cat <<'EOF' > /etc/netplan/ethernet-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp9s0:
      accept-ra: yes
      addresses:
        - 192.168.1.1/24
        - 2a00:1ec0:102:470::100/64
      gateway4: 192.168.1.254
      gateway6: 2a00:1ec0:102:470::1
      nameservers:
        addresses: [192.168.1.253, 192.168.1.254, fd00:100::53, fd00:100::1]
EOF
```

Configure `fstab` and `crypttab`:
```
echo "# <file system> <mount point> <type> <options> <dump> <pass>" > /etc/fstab
echo "# / was on /dev/phanes-main/root during installation" >> /etc/fstab
echo "$(blkid /dev/phanes-main/root | cut -d' ' -f2) / ext4 errors=remount-ro 0 1" >> /etc/fstab
echo "# /boot was on /dev/md/boot during installation" >> /etc/fstab
echo "$(blkid /dev/md/boot | cut -d' ' -f2) /boot ext4 defaults 0 2" >> /etc/fstab
echo "main_crypt $(blkid /dev/md/main | cut -d' ' -f2) none luks" >> /etc/crypttab
```

Configure `clevis` to use all `tang` servers available:
```
for TANG in 1.2.3.4 5.6.7.8:8080; do clevis luks bind -d /dev/md/main tang '{"url":"http://'${TANG}'"}'; done
```

Add static IP for initrd to allow network-based unlocking of the LUKS
device:
```
cat <<'EOF' >> /etc/initramfs-tools/initramfs.conf

# Static IP for clevis
IP=192.168.1.1::192.168.1.254:255.255.255.0:phanos:
EOF
```

Download and configure `grml` ISO image:
```
wget https://download.grml.org/grml64-full_2020.06.iso -O /boot/grml/grml64-full_2020.06.iso
```
This needs some tweaking to work.  Haven't come around to that yet.

Fix `grub` config:
```
sed -i 's/^GRUB_TIMEOUT_STYLE=hidden/#GRUB_TIMEOUT_STYLE=hidden/' /etc/default/grub
sed -i 's/^GRUB_TIMEOUT=0/GRUB_TIMEOUT=5/' /etc/default/grub
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"/GRUB_CMDLINE_LINUX_DEFAULT=""/' /etc/default/grub
```

Fix keymap:
```
sed -i 's/^XKBVARIANT=""/XKBVARIANT="dvorak"/' /etc/default/keyboard
```

Refresh `grub` and `initramfs`:
```
update-grub
update-initramfs -uv
```

Install `grub` to the MBR of both drives:
```
grub-install /dev/sda
grub-install /dev/sdb
```

Make the `grub` partitions bootable:
```
parted /dev/sda toggle 1 boot
parted /dev/sdb toggle 1 boot
```

Create a user:
```
adduser <USER>
for GROUP in adm cdrom sudo dip plugdev lxd; do adduser <USER> ${GROUP}; done
```

Reboot.
