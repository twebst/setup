# Arch install guide (my setup)

## Downloading
1. Download from one of the mirrors [here](https://archlinux.org/download/) (iso, sig, checksums, and import signing key).
2. Verify the checksums, and the sig file (make sure import the public signing key).
3. Write to external thumbdrive (e.g. use `dd`).

## Base install
1. Setup up an internet connection with iwctl (assuming wireless).
2. Sync the system clock with `timedatectl status`
3. Use cfdisk to partition the disks. Currently boot is unencrypted (512 MB, efi boot), and the rest is for encrypted lvm (the rest, linux file system).
4. On the non-boot partition, run `cryptsetup luksFormat /dev/<partition>`.
5. Open with `cryptsetup open /dev/<partition> main-luks`. This will map the cryptdevice to /dev/mapper/main-luks.
6. `pvcreate /dev/mapper/main-luks` to initialize the physical volume.
7. `vgcreate main-vg /dev/mapper/main-luks` to create a new volume group with main-luks as the (only) physical volume.
8. Create the following logical volumes on main-vg:
  - `lvcreate --size 512M --name swap main-vg`
  - `lvcreate --size 30G --name root main-vg`
  - `lvcreate --extents 100%FREE --name home main-vg` (fills the rest of the space).
9. `mkfs.ext4 /dev/main-vg/root`
10. `mkfs.ext4 /dev/main-vg/home`
11. `mkswap/dev/main-vg/swap`
12. `mkfs.fat -F 32 /dev/efi_system_partition` (boot partition).
13. `mount /dev/main-vg/root /mnt`
14. `mount --mkdir /dev/main-vg/home /mnt/home`.
15. `swapon /dev/main-vg/swap` 
16. `mount --mkdir /dev/vg/efi_system_partition /mnt/boot`.
17. (Optional) Edit the install mirrors in `/etc/pacman.d/mirrorlist`, moving the geographically closest mirrors to the top.
18. `pacstrap -K /mnt base base-devel linux linux-firmware <packages>`. `base` and `base-devel` are for the core and develop meta packages, and [`<packages>`](#packages) is the list of all the extra packages needed later (see below).
19. `genfstab -U /mnt >> /mnt/etc/fstab` to setup the fstab to match the current setup in `/mnt` (`-U` for uuids instead of `-L` for labels).
20. `arch-chroot /mnt`
21. `ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime` to setup the time zone.
22. Sync the hardware clock with the ztstem clock `hwclock --systohc` (generates `
/etc/adjtime`).
23. Uncomment the needed locales in `en_US.UTF-8 UTF-8`, and then generate them with `locale-gen`.
24. Create `/etc/locale.conf` with `LANG=<desired-locale>`.
25. Create `/etc/hostname` with `<hostname>`
26. Update the mkinitcpio hooks in `/etc/mkinitcpio.conf` to include `encrypt` and `lvm2` before `filesystem`, looking something like this `HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)`.
27. `mkinitcpio -P`, creating the initial ram disk.
28. `passwd` to set up a root password.
29. `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=arch-GRUB` to install the grub efi application and its x86_64 modules.
30. Edit `GRUB_CMDLINE_LINUX` and `GRUB_PRELOAD_MODULES` in `/etc/default/grub` so the encrypted root is specified, and lvm module is loaded. Example:

```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<uuid>:<name> root=/dev/main-vg/root"

# Preload both GPT and MBR modules so that they are not missed
GRUB_PRELOAD_MODULES="part_gpt part_msdos lvm"
```

`<uuid>` is the block id of the encrypted partition (read into vim with `:r !blkid`, cut out the correct uuid), and `<name>` is what the cryptdevice will be opened as (e.g. `main-luks`).

31. Generate the grub config in the boot directory `grub-mkconfig -o /boot/grub/grub.cfg`.
32. Exit the chroot environment (`exit` or `Ctrl+d`).
33. `umount -R /mnt`
34. `cryptsetup close /dev/mapper/main-luks`
35. `<reboot>`, and then continue to [post install](#post-install).

## Post Install


## Packages
- neovim (editor)
- intel-ucode (processor microcode)
- grub (bootloader)
- efibootmgr (for modifying the EFI Boot Manager)
- nvidia (proprietary nvidia drivers, requires restart)
- cryptsetup (managing the luks device(s))
- iw (wireless utility)
- networkmanager (wireless utility, includes nmcli and nmtui)
- lvm2 (needed for managing physical volumes, volume groups, and logical volumes).
- man (read man pages)
- zsh (alternative shell)
- git
- wget
- gufw (simple firewall management with GUI)
- gnome (desktop environment, without the extra apps included in gnome-extra), pick noto-ttf-fonts
- gnome-shell-extensions (extensions for GNOME shell)
- gnome-tweaks (editing extra GNOME settings graphically, normally hidden and edited with dconf-editor)
- firefox (firefox web browser)
- wl-clipboard (includes wl-copy and wl-paste, cmd line wayland clipboard utils)

### Aur
These packages are only available through the aur, and need to be installed (compiled) with a separate program (e.g. yay). Install yay by following [these instructions](https://github.com/Jguer/yay#installation).

- gnome-browser-connector (system side util for gnome extension browser integration)
- brave-bin (brave web browser)
