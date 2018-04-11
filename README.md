*  Use of cgdisk / cfdisk / whatever to partition GPT disk as follows:    (https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LUKS_on_LVM)
```
  - 1MB or so of free space (irrelevant really - more for disk alignment than anything else)
  - 100MB of EFI system for /boot/efi (mkfs.vfat -F32 /dev/sda1) (hex ef00)
  - 250MB of ext4 for /boot
  - Remaining space for LVM group which will contain swap and crypto root
```

*  Create your LVM    (https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LUKS_on_LVM)
```bash
lvm pvcreate /dev/sda3
lvm vgcreate GROUPNAME /dev/sda3
lvm lvcreate -L 500M -n swap GROUPNAME
lvm lvcreate -l 100%FREE -n root GROUPNAME
```

*  Set up crypt on root now, and give it an fs
```
cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/mapper/GROUPNAME-root
crypsetup open --type luks /dev/mapper/GROUPNAME-root root
mkfs -t ext4 /dev/mapper/root
mount /dev/mapper/root /mnt
```

*  Make a /mnt/boot folder now, and mount /dev/sda2 to it (with an ext4 partition)
*  Make a /mnt/boot/efi folder, and mount /dev/sda1 to it (with a vfat partition as specified earlier)
*  Follow standard arch install up to the "mkinitcpio" section (I suggest lowering Australia's score in the /etc/pacman.d/mirrorlist to 0.1 - so that it uses our servers)  (https://wiki.archlinux.org/index.php/Installation_guide#Installation)
*  pacstrap with base, base-devel, grub-efi-x86_64, efibootmgr, zsh, vim, git, and anything else you can think of now
*  Once installation is up to mkinitcpio, edit hooks for mkinitcpio in /etc/mkinitcpio.conf (HOOKS section) - after "block", and "encrypt" and "lvm2" - in that order. At the end of it (before quotation closes though), append **shutdown** (which allows encrypted swap stuff to dismount/clean up properly at the end)   (https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio_3)
*  Edit /etc/default/grub, and to the GRUB_CMDLINE_LINUX_DEFAULT, add: cryptdevice=/dev/mapper/GROUPNAME-root:root root=/dev/mapper/root    (where :root is the name it'll be mounted as after decryption)    (https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration#Boot_loader)
```bash
mkinitcpio -p linux       (https://wiki.archlinux.org/index.php/Installation_guide#Configure_the_system)
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub    (https://wiki.archlinux.org/index.php/GRUB#Installation_2)
grub-mkconfig -o /boot/grub/grub.cfg  ("Failed to connect" warnings are fine here)  (https://wiki.archlinux.org/index.php/GRUB#Generate_the_main_configuration_file)
```
*  Encrypted swap is pretty simple. This will regenerate the encryption key for swap as entirely random each time. Edit your /etc/crypttab file to refer to swap by disk-id, or you'll potentially kill your root:
```bash
> ls /dev/disk/by-id/* | grep swap    # mine showed as /dev/disk/by-id/dm-name-GROUPNAME-swap
# Add to crypttab as follows:
swap    /dev/disk/by-id/dm-name-GROUPNAME-swap  /dev/urandom    swap,cipher=aes-cbc-essiv:sha256,size=256
```
*  Next, edit your fstab to reflect this:
```bash
/dev/mapper/swap    none    swap    sw  0   0
```

*  Unmount and reboot

# FDE with YubiKey 2FA

```bash
yaourt -S mkinitcpio-ykfde
echo "root /dev/cryptoroot/root -" > /etc/crypttab.initramfs
```

Edit /etc/ykfde.conf:
```bash
[general]
yk slot = 2
device name = root
second factor = yes
[350350]
luks slot = 7
```

In yubikey-personalization-gui, edit the second slot, and make it HMAC-SHA1
challenge response. Then, run: sudo ykfde   -   it will ask for a current valid
passphrase in order to install itself. Once done, create a new passphrase:

ykfde --ask-new-2nd-factor

Generate the CPIO challenge archive with: sudo ykfde-cpio

Enable systemd: sudo systemctl enable ykfde.service

Edit /etc/mkinitcpio.conf, and make the HOOKS look like:
```
HOOKS="base udev autodetect systemd sd-encrypt modconf ykfde block encrypt sd-lvm2 filesystems keyboard fsck shutdown"
```

Generate the new CPIO: sudo mkinitcpio -p linux

Generate the new GRUB: sudo grub-mkconfig -o /boot/grub/grub.cfg

# 2FA with YubiKey using HMAC-SHA1 Challenges
This is used to do offline 2FA for your user account. This makes use of the same HMAC-SHA1 second factor we're using for FDE, so no extra YubiKey programming is required.

## For standard authentication and sudo
Start off by installing `yubico-pam`. Then, open up `/etc/pam.d/system-auth` and add the following at the top:

```
auth [success=1 default=ignore] pam_succeed_if.so quiet user notingroup yubikey
auth		required	pam_yubico.so mode=challenge-response
```

Create a group for those who are to auth using their YubiKeys, and add yourself to it:
```
sudo groupadd yubikey
sudo usermod -aG yubikey username
```

If you skipped the FDE step above, use the following to program slot 2 of your YubiKey for HMAC-SHA1 auth:
```
sudo ykpersonalize -2 -ochal-resp -ochal-hmac -ohmac-lt64 -oserial-api-visible
```

Insert your YubiKey, and run this command to create a "challenge" file in your home directory for that key:
```
ykpamcfg -2 -v
```

You are now set up for standard user auth with YubiKey.

## For i3lock

Create the file `/etc/udev/rules.d/90-yubikey.rules` with the content:

```
SUBSYSTEMS=="usb", ATTR{product}=="Yubico Yubikey II", MODE="0660", GROUP="yubikey"
```

Then run `sudo udevadm control --reload` to reload rules. Lock your screen, and you should then be able to unlock with only your YubiKey.

# Customising post-install
```bash
#systemctl enable netctl-ifplugd@enp2s0
#systemctl enable netctl-auto@wlp3s0  #for wifi, run Wi-Fi-menu to set up for first time
pacman -S dialog xorg-server xorg-xinit dmenu xorg i3 wpa_actiond ifplugd wpa_supplicant sudo zsh git tmux zathura ranger vlc weechat ttf-dejavu ttf-inconsolata lxappearance ntfs-3g udevil cifs-utils zenity curlftpfs scrot nfs-utils sshfs vim-systemd
useradd -m -G wheel -s /usr/bin/zsh USERNAME
visudo # uncomment the %wheel line to allow wheel group full sudo access
pacman -S sddm
sddm --example-config > /etc/sddm.conf
pacman -S rxvt-unicode rxvt-unicode-terminfo
systemctl enable sddm.service
yaourt -S xfce-theme-greybird
lxappearance   #set Greybird
pacman -S arandr   (http://tutos.readthedocs.org/en/latest/source/Arch.html)

```

*  Edit ~/.Xdefaults:

```conf
! urxvt

URxvt*geometry:                115x40
!URxvt*font: xft:Liberation Mono:pixelsize=14:antialias=false:hinting=true
!URxvt*font: xft:Inconsolata:pixelsize=17:antialias=true:hinting=true
!URxvt*boldFont: xft:Inconsolata:bold:pixelsize=17:antialias=false:hinting=true
!URxvt*boldFont: xft:Liberation Mono:bold:pixelsize=14:antialias=false:hinting=true
URxvt*depth:                24
URxvt*borderless: 1
URxvt*scrollBar:            false
URxvt*saveLines:  2000
URxvt.transparent:      true
URxvt*.shading: 10

! Meta modifier for keybindings
!URxvt.modifier: super

!! perl extensions
URxvt.perl-ext:             default,url-select,clipboard

! url-select (part of urxvt-perls package)
URxvt.keysym.M-u:           perl:url-select:select_next
URxvt.url-select.autocopy:  true
URxvt.url-select.button:    2
URxvt.url-select.launcher:  chromium
URxvt.url-select.underline: true

! Nastavuje kopirovani
URxvt.keysym.Shift-Control-V: perl:clipboard:paste
URxvt.keysym.Shift-Control-C:   perl:clipboard:copy

! disable the stupid ctrl+shift 'feature'
URxvt.iso14755: false
URxvt.iso14755_52: false

!urxvt color scheme:

URxvt*background: #2B2B2B
URxvt*foreground: #DEDEDE

URxvt*colorUL: #86a2b0

! black
URxvt*color0  : #2E3436
URxvt*color8  : #555753
! red
URxvt*color1  : #CC0000
URxvt*color9  : #EF2929
! green
URxvt*color2  : #4E9A06
URxvt*color10 : #8AE234
! yellow
URxvt*color3  : #C4A000
URxvt*color11 : #FCE94F
! blue
URxvt*color4  : #3465A4
URxvt*color12 : #729FCF
! magenta
URxvt*color5  : #75507B
URxvt*color13 : #AD7FA8
! cyan
URxvt*color6  : #06989A
URxvt*color14 : #34E2E2
! white
URxvt*color7  : #D3D7CF
URxvt*color15 : #EEEEEC
```

# Network Manager
```bash
pacman -S networkmanager network-manager-applet networkmanager-{openconnect,openvpn,pptp,vpnc} dnsmasq modemmanager dhclient
pacman -S gnome-keyring
systemctl enable NetworkManager.service
```

# Expanding your Disk
If you need to expand your disk later on, there's a few steps required.

## Extend your Physical Partition
First, you'll need to grow the physical partition your LV sits on. Use whatever tools you'd normally use; `gparted` is a nice easy one for this.

After resizing, confirm with:
```bash
sudo pvs
```

You should see some free space listed here.

## Extend your LV
Exten your Logical Volume, keeping some free space for snapshots. 

```bash
sudo lvextend -l+75%FREE /dev/cryptoroot/root
```

## Extend your Crypt and Filesystem
Inside of the LVM, you'll need to expand the LUKS crypt, and then the filesystem.

```bash
sudo cryptsetup luksOpen /dev/cryptoroot/root root
sudo cryptsetup resize root
sudo e2fsck -f /dev/mapper/root
sudo resize2fs -p /dev/mapper/root
```

# Creating Snapshots
The following are a series of commands to display snapshot creation and management of your LV. Keep in mind, as this is LUKS inside the LV, it's a snapshot of a LUKS volume (rather than LV in LUKS, where the snapshot is of your files).

```bash
# Create a 1GB snapshot
sudo lvcreate  -L 1GB -s -n my-snapshot /dev/cryptoroot/root
# Make sure it displays in a list of volumes
sudo lvs
# See details info on it
sudo lvdisplay my-snapshot
# Extend it to accomodate more than 1GB of changes
sudo lvextend -L +1GB /dev/cryptoroot/my-snapshot
```

## Restoring and Merging

To restore or merge, unmount any mounted snapshots, and close your LUKS volumes off.

```bash
# Merges snapshot over the top of current
sudo lvconvert --merge /dev/cryptoroot/my-snapshot
# If you tried to merge, and it was busy, deactivate and reactivate the LV after unmounting etc
sudo lvchange -an /dev/cryptoroot/root
sudo lvchange -ay /dev/cryptoroot/root
```

