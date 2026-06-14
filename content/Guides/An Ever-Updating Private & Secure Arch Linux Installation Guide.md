---
title: An Ever-Updating* Private & Secure Arch Linux Installation Guide
discription:
draft: false
---

Ok so this guide actually requires a lot of rework now that my setup has changed drastically and there's a need for a new blog explaining why I now use Secureblue instead of Arch Linux. Essentially, here's what has changed - systemd-boot instead of grub, not compromising on luks2 by using PBKDF2 instead of Argon2id, using apparmor.d for AppArmor profiles, giving up on bootable btrfs snapshots. I will update this guide soon. Bit of an ironical situation, innit?

### Introduction
This guide delves into the wilderness of installing Arch Linux while keeping privacy (which is already there unless you install something specifically privacy invasive) and security (which is NOT there unless you make an effort for it) in mind. We are going the Do-It-Yourself route, as God intended. I know that this is not going to be easy for many of you, especially for those of you who are not very tech-savvy, but trust me, it is going to be something you are really proud of once we are done with it. You are about to taste the joy of building something yourself and seeing it work like you wanted.

### What this guide IS NOT and What it IS?
This guide is NOT claiming to set up the _“perfect”_ privacy-respecting and security-oriented Arch Linux install. In fact, depending on your [threat model](https://www.privacyguides.org/en/basics/threat-modeling/), this might not even be sufficient for your needs. However, this guide is basically the setup I personally settled on the last time I used Arch Linux on my personal laptop. This guide will keep on updating as I keep on improving my own setup in the constant chase for _“_**_my_** _perfect”_ Arch Linux install with the below-mentioned goals in mind.

### Goals of this Setup and their current status
- Encryption with LUKS2 - due to the limitations of GRUB at the time of writing this post, we are restricted to using PBKDF2 instead of Argon2id
- Rollback support in case of a breakage caused by some update - achieved through BTRFS Snapshots via Snapper and GRUB's capability of booting into different snapshots
- Secureboot - via Microsoft keys
- TPM2 enrollment with PIN - grub-btrfs is incompatible with TPM2 at the time of writing this guide, however you can use it if you use systemd-boot instead, even better if you use Unified Kernel Images
- Kernel Hardening - Instead of using linux-hardened provided in the arch repo, we are manually hardening the kernel using kernel parameters used by [Secureblue](https://secureblue.dev/), this is due to the fact that if we use linux-hardened, we will need to use bubblewrap-suid for flatpaks and chromium sandbox will also run with suid, which is, as some of you can guess, really not a good idea
- AppArmor for Mandatory Access Control - more research is needed for profiles
- Flatpak for the vast majority of applications you will need with native packages from the official repos only acting as a base and only a handful of reputed packages from Arch User Repository(AUR)
- 'linux-lts' kernel as the fallback kernel in case the latest kernel breaks something in an update
- Some security suggestions as [mentioned on the ArchWiki](https://wiki.archlinux.org/title/Security)
- Miscellaneous things for Stability and Maintenance

### Why not just use the archinstall script?
It doesn't allow me to make all the changes I want, or at least not that I know of any way to make it do so. When it comes to a DIY distribution like Arch Linux, I prefer to know what I am doing and how exactly I am doing it. I don't get that feel while using a script, if you know what I mean. However, I must admit that it is really just personal preference without much thought-out reasoning behind it, and it doesn't really do anything that I can't manually do myself if I so wished to do something that it does, which is not mentioned at present in this guide.

### Pre-requisites
1. A relatively recent computer - as in, it has TPM2, Secureboot support, somewhat modern storage device(preferably SSD because BTRFS won't play good on a HDD)
2. An internet connection - good enough to download a few gigs with a decent speed
3. A USB pendrive - at least a 4 gigs one to be on the safer side
4. Patience
5. Some level of competence and the courage to overcome the fear of commands!

### Step-By-Step Process
#### Downloading the ISO and creating a bootable USB
1. Download the latest ISO from https://archlinux.org/download/. For direct download, choose a link from the list of mirrors based on countries, a mirror from your country or the country nearest to you. The best way would be to just download using the provided torrent at the top of the page. I recommend using [qBittorrent](https://www.qbittorrent.org/) to download torrent files, as it is well reputed and open-source.
2. Now you need to write this ISO file to a USB pendrive and turn it into a bootable pendrive as we need it to be able to boot into the ISO file we downloaded in step 1. To do so, there are several ways. I personally prefer using [Ventoy](https://www.ventoy.net/en/index.html). So, here, I will just go over that. Download the latest version of Ventoy from [this page](https://github.com/ventoy/Ventoy/releases/latest). Depending on your OS, you have 2 choices, either you pick the linux version or the windows version. The steps for both of them are the same, so download whichever suits you. [This Guide](https://www.ventoy.net/en/doc_start.html) on the official site of Ventoy goes over the installation instructions. Remember that, this will **format** your USB pendrive, which means **all your data on it will be completely DELETED**. After you have installed Ventoy on your USB pendrive, copy-paste the ISO file you previously downloaded to it.
3. Boot into the UEFI settings/firmware setup/etc of your PC/laptop. Depending on the model and manufacturer, the key to press during boot will defer. For me, it is the F2 key, for you, it might be DELETE key, the F10 key and so on. Google this up, and see what's the key for your computer.
	- Here, you need to disable Secureboot. Don't worry, I will cover how to setup secureboot in this guide later on.
	- Disable any of the device manufacturer's own security options like natural file guard and any other such setting.
	- Save the settings and poweroff the computer
4. Now, boot into the bootable pendrive we created in step 2. To do so, like in step 3, you need to press a specific key during boot to boot into the boot menu and select your bootable pendrive from there. A screen displaying Ventoy menu will appear and there, select the Arch Linux ISO. Once you select the Arch Linux ISO, Ventoy will boot into it and you will be greeted by a command line screen unlike traditional operating systems with graphical user interfaces for their install process.

This is the point where I will suggest you to read [the official installation guide](https://wiki.archlinux.org/title/Installation_guide) from the [ArchWiki](https://wiki.archlinux.org/title/Main_page). However, I do realize that it's going to be a tough and very confusing thing for a lot of you guys new to Arch Linux, Linux in general or for you really non-tech savvy folks. So, I am going to list down the way I do the install and we are going to follow the official installation guide for the most part. For a few things, however, I am going to refer to 3rd party sources, i.e. the installs of other people on the internet. 

#### Beginning the Installation
1. The first thing that I like to do is to change the font to make everything look a bit nicer and make it more visible,
   
	`setfont ter-v24n`
2. It's time to connect to the internet. If you are using an ethernet cable, you don't need to do anything other than plugging that in properly to connect to the internet. For those of you who are using a WiFi instead need to use the following commands one-by-one:
	
	`iwctl`
	
	`station <name_of_wireless_device_usually_wlan0> connect <SSID_or_WiFi_network_name>`
	
	if your WiFi is protected by a passphrase, which it should be, iwd will ask you to enter it and once you have entered it correctly, your PC should be connected to the WiFi in question.
3. To check the network, close iwd with `exit` command and try the following command:
	
	`ping archlinux.org` and hit `Ctrl + C` after a few seconds,
	
	if the output is something like, `64 bits from archlinux.org (<some_ip>) *blah blah* time=<something> ms`, then you are good to go, otherwise try troubleshooting your network, connecting to a different network or something
4. Update your system clock with:
	
	`timedatectl set-timezone <your>/<location>`
	
	a list of timezones can be obtained using `timedatectl list-timezones`

#### Partitioning and Encrypting the Disk
1. It's time to format the disk and set up a partition layout
   
	Depending on whether you are using a HDD or an SSD, the name of storage device could be `/dev/sda`, `/dev/nvme0n1` or something else, identify it using:
	
	`fdisk -l`
	
	for me, I am using an NVMe SSD, so my device name is `/dev/nvme0n1`
	
	Wipe any old existing partitions using the following commands one-by-one:
	
	`wipefs -af /dev/nvme0n1` (replace the device name with whatever it is for you)
	
	`sgdisk --zap-all --clear /dev/nvme0n1`
	
	`partprobe /dev/nvme0n1`
	
	Now create the partitions using `gdisk`:
	- `gdisk /dev/nvme0n1`
	- enter `o` and enter `Y` - this will create a new empty GUID partition table
	- enter `n` and leave the next 2 prompts empty,
	- on the 3rd prompt, enter `+1G` and then enter `ef00`, this will create a new EFI system partition, onto the next one,
	- enter `n` and leave the next 4 prompts empty, this will create a new Linux filesystem partition on the remaining disk space
	- enter `w`  and enter `Y` to write the changes
2. It's time to encrypt the disk and format the partitions,
   
	After the last step, when you enter `fdisk -l`, you will now see 2 new entries under `/dev/nvme0n1`, that are likely `/dev/nvme0n1p1` and `/dev/nvme0n1p2`, the naming will obviously vary depending on the name of your storage device.
	
	First, let's set up encryption for the Linux filesystem partiton, i.e. `/dev/nvme0n1p2` in my case:
	- enter `cryptsetup luksFormat --perf-no_read_workqueue --perf-no_write_workqueue --type luks2 --pbkdf pbkdf2 --hash sha3-512 /dev/nvme0n1p2`, this will prompt you for a password twice, create a strong password for encryption here and make sure you remember it
	- open this encrypted partition using
		`cryptsetup --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent open /dev/nvme0n1p2 main`
		and enter the password you setup previously.
	
	Format this partition using:
	
	`mkfs.btrfs /dev/mapper/main`
3. Mount `/dev/mapper/main` and create btrfs subvolumes, multiple of these are created to exclude them from the snapshots of root subvolume:
	
	`mount /dev/mapper/main /mnt`
	
	`btrfs sub create /mnt/@` - root subvolume
	
	`btrfs sub create /mnt/@home` - /home subvolume
	
	`btrfs sub create /mnt/@tmp` - contains temporary files
	
	`btrfs sub create /mnt/@abs` - stuff related to [Arch Build System](https://wiki.archlinux.org/title/Arch_build_system)? I create it because several installation guides on the internet also create it to exclude it from root snapshots
	
	`btrfs sub create /mnt/@cache` - contains caches
	
	`btrfs sub create /mnt/@log` - contains logs, temporary files etc.
	
	`btrfs sub create /mnt/@srv` - stores files that will be served to a network
	
	`btrfs sub create /mnt/@snapshots` - will store snapshots
	
	Now, unmount `/mnt` using `umount /mnt` and individually mount all the subvolumes using the following commands:
	
	`export mnt_opts="noatime,ssd,autodefrag,commit=120,compress=zstd:3,space_cache=v2,discard=async"`
	
	`mount -o ${mnt_opts},subvol=@ /dev/mapper/main /mnt`
	
	`mkdir -p /mnt{home,var/tmp,var/abs,var/cache,var/log,srv,.snapshots}`
	
	`mount -o ${mnt_opts},subvol=@home /dev/mapper/main /mnt/home`
	
	`mount -o ${mnt_opts},nodev,nosuid,noexec,subvol=@tmp /dev/mapper/main /mnt/var/tmp`
	
	`mount -o ${mnt_opts},nodev,nosuid,noexec,subvol=@abs /dev/mapper/main /mnt/var/abs`
	
	`mount -o ${mnt_opts},nodev,nosuid,noexec,subvol=@cache /dev/mapper/main /mnt/var/cache`
	
	`mount -o ${mnt_opts},nodev,nosuid,noexec,subvol=@log /dev/mapper/main /mnt/var/log`
	
	`mount -o ${mnt_opts,subvol=@srv /dev/mapper/main /mnt/srv`
	
	`mount -o ${mnt_opts},subvol=@snapshots /dev/mapper/main /mnt/.snapshots`
4. Now, format the first, EFI system, partition and mount it:
	
	`mkfs.fat -F32 /dev/nvme0n1p1` 
	
	`mkdir /mnt/boot`
	
	`mount -o nodev,nosuid,noexec /dev/nvme0n1p1 /mnt/boot`

#### Installing Base Packages and Generating fstab
1. *sigh* We are finally done with partitioning, formatting and encrypting the disk, now run `reflector` to select the fastest arch linux repo mirrors for you:
   
	`reflector -c <country> -a 12 --sort rate --save /etc/pacman.d/mirrorlist`
2. Install essential packages using the following command:
   
	`pacstrap -K /mnt base linux linux-headers linux-firmware sof-firmware btrfs-progs dosfstools vim man-db man-pages texinfo sudo networkmanager intel-ucode git` 
	
	(my PC has Intel CPU which is why I am installing `intel-ucode`, if you are using an AMD CPU, install `amd-ucode` instead) 
3. Generate the filesystem table using:
    
	`genfstab -U /mnt >> /mnt/etc/fstab` 

#### Chroot into the New system and Configure it
1. Set your timezone:
   
   `ln -sf /usr/share/zoneinfo/_Region_/_City_ /etc/localtime`, for example:
   
   `ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime` in my case.
2. Sync the hardware clock using:
   
   `hwclock --systohc`
3. Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other needed UTF-8 locales. Generate the locales by running:
   
   `locale-gen`
4. Set hostname to whatever you like, I, for example, set it to "arch" using:
   
   `echo "arch" >> /etc/hostname`
5.  Set root password using:
   
	`passwd`
6. Add a new user which you will be using by default:
   
	`useradd -m -g users -G wheel <username>`
	
	Set a password for it using:
	
	`passwd <username>`
	
	Add it to the sudoers list:
	
	`echo "<username> ALL=(ALL) ALL" >> /etc/sudoers.d/<username>`
7. Install additional packages and GNOME(you can install some other desktop environment if you so wish but I recommend GNOME because of [this](https://secureblue.dev/images#security-recommendation)):
   
	`pacman -S base-devel grub grub-btrfs efibootmgr network-manager-applet iptables-nft ipset firewalld acpid acpi_call reflector xdg-user-dirs xdg-utils gvfs gvfs-smb dnsutils bluez bluez-utils cups bash-completion rsync tuned tuned-ppd terminus-font pipewire pipewire-alsa pipewire-jack pipewire-pulse flatpak gdm gnome gnome-tweaks`
8. Enable some services to run on boot that we installed in the previous step:
   
	`systemctl enable NetworkManager`
	
	`systemctl enable bluetooth`
	
	`systemctl enable cups.service`
	
	`systemctl enable tuned`
	
	`systemctl enabel tuned-ppd`
	
	`systemctl enable reflector.timer`
	
	`systemctl enable fstrim.timer`
	
	`systemctl enable firewalld`
	
	`systemctl enable acpid`
	
	`systemctl enable gdm`

#### Update mkinitcpio.conf and setup Grub bootloader
1. Edit `mkinitcpio.conf` using `vim /etc/mkinitcpio.conf`
	- add `btrfs` to `MODULES`
	- add `encrypt` before `filesystems` in `HOOKS`
	- hit `Esc` key and enter `:wq` to save the changes
2. Generate new kernel image using:
   
	`mkinitcpio -P linux`
3. Install grub using:
   
	`grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
	
	generate GRUB configuration file using:
	
	`grub-mkconfig -o /boot/grub/grub.cfg`
4. Edit `/etc/default/grub`:
   
	`blkid /dev/nvme0n1p2 >> /etc/default/grub`, this will append the UUID of the encrypted partition to the grub configuration file at the end.
	
	it will look something like: `/dev/nvme0n1p2: UUID="<something_something" TYPE="crypto_luks" blah blah`, we just need the UUID part.
	
	`vim /etc/default/grub`
	
	cut the line we appended previously using `dd` and paste it with `p` under `GRUB_CMDLINE_LINUX_DEFFAULT` and edit the line like so:
	`GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=<UUID_identified_previously>:main root=/dev/mapper/main"` 
	
	save the file by hitting `Esc` key and entering `:wq`
	
	regenerate the GRUB configuration file using:
	
	`grub-mkconfig -o /boot/grub/grub.cfg`
	
	Now you can `exit` the chroot and `reboot` into your newly installed system.

#### Install firefox, paru, zram-generator, snapper and linux-lts
1. Open Console from the App menu and install a browser. You could install any one of your choice through flatpak, I prefer Firefox so I will install that using:
	
	`flatpak install flathub org.mozilla.firefox`
2. Install Paru, or any AUR helper of your choice, to manage packages from AUR:
	- `git clone https://aur.archlinux.org/paru.git`
	- `cd paru`
	- `makepkg -si`
3. Install `zram-generator` package using:
	- `sudo pacman -S zram-generator`
	- `sudo vim /etc/systemd/zram-generator.conf` and add the following lines:
	  
	  `[zram0]`
	  
	  `zram-size = min(ram / 2)`
	  
	  `compression-algorithm = zstd`
	  
	  press `Esc` key and enter `:wq` to save and exit
	- `sudo systemctl daemon-reload`
	- `sudo systemctl enable --now systemd-zram-setup@zram0.service`, this will create a zram0 device which will be visible when you enter `fdisk -l`, its size will be half the size of your PC's RAM
4. Install and setup snapper:
	- `sudo -s`
	- `cd /`
	- `paru -S snapper snap-pac`
	- `umount /.snapshots`
	- `rm -rf /.snapshots/`
	- `snapper -c root create-config /`
	- `btrfs subvolume delete /.snapshots`
	- `mkdir .snapshots`
	- `btrfs subvol list /` - here, grab the ID of the @ subvolume
	- `btrfs subvol set-default <ID_you_grabbed_previously> /`
	- `vim /etc/mkinitcpio.conf`
	- add `grub-btrfs-overlayfs` to the `HOOKS`, press `Esc` key and enter `:wq` to save and exit
	- `mkinitcpio -P linux`
	- Edit `/etc/snapper/configs/root` as well, and add `wheel` to `ALLOW_GROUPS=""`
	- `chown -R :wheel /.snapshots/`
	- `systemctl enable --now grub-btrfs.path`
	- `grub-mkconfig -o /boot/grub/grub.cfg`
	- `systemctl enable --now snapper-timeline.timer`
	- `systemctl enable --now snapper-cleanup.timer`
	- read https://wiki.archlinux.org/title/Snapper to configure snapper according to your needs
	- also consider watching this tutorial for snapper: https://youtu.be/_97JOyC1o2o
5. Install the Long-term Support(LTS) Kernel as a fallback kernel:
	- `sudo pacman -S linux-lts linux-lts-headers acpi_call-lts`
	- Edit grub configuration file using `sudo vim /etc/default/grub`,
		remove the `#` before the following:
		- `GRUB_DISABLE_SUBMENU=y`
		- `GRUB_SAVEDEFAULT=true`
		and set `GRUB_DEFAULT` to `saved`, now save and exit
	- regenerate grub configuration, `sudo grub-mkconfig -o /boot/grub/grub.cfg`


#### Kernel Hardening
Enter `sudo vim /etc/sysctl.d/99-custom.conf` to create and open a new configuration file in `/etc/sysctl.d` which will be used for loading the added kernel parameters on boot, and add the following lines to it:

	#Enable IP spoofing protection, turn on source route verification
	net.ipv4.conf.all.rp_filter = 1
	net.ipv4.conf.default.rp_filter = 1
	
	#Disable ICMP Redirect Acceptance
	net.ipv4.conf.*.send_redirects = 0
	net.ipv4.conf.*.accept_redirects = 0
	net.ipv6.conf.*.accept_redirects = 0
	
	#Enable ipv6 privacy extension
	net.ipv6.conf.all.use_tempaddr=2
	net.ipv6.conf.default.use_tempaddr=2
	
	#Enable Log Spoofed Packets, Source Routed Packets, Redirect Packets
	net.ipv4.conf.all.log_martians = 1
	net.ipv4.conf.default.log_martians = 1
	
	net.core.bpf_jit_harden = 2
	kernel.yama.ptrace_scope = 3
	kernel.unprivileged_bpf_disabled = 1
	kernel.sysrq = 0
	kernel.perf_event_paranoid = 3
	kernel.kptr_restrict = 2
	kernel.dmesg_restrict = 1
	fs.binfmt_misc.status = 0
	fs.suid_dumpable = 0
	fs.protected_regular = 2
	fs.protected_fifos = 2
	dev.tty.ldisc_autoload = 0
	
	#Restrict userfaultfd to CAP_SYS_PTRACE
	vm.unprivileged_userfaultfd = 0
	
	#Prevent kernel info leaks in console during boot.
	kernel.printk = 3 3 3 3
	
	#Disables kexec which can be used to replace the running kernel.
	kernel.kexec_load_disabled = 1
	
	#Disable core dump
	kernel.core_pattern = |/bin/false
	
	#Disable io_uring
	kernel.io_uring_disabled = 2
	
	#Improve ALSR effectiveness for mmap.
	vm.mmap_rnd_bits = 32
	vm.mmap_rnd_compat_bits = 16
These are taken from https://github.com/secureblue/secureblue/blob/live/files/system/etc/sysctl.d/60-hardening.conf

#### Security Suggestions from ArchWiki
1. For memory security, install `hardened_malloc` from AUR using `paru -S hardened_malloc`. Enabling this memory allocator globally wouldn't exactly play nice with your system and apps like Firefox and Thunderbird will even crash as they require to be rebuilt for making use of this memory allocator. Instead, use it on case-by-case basis, for example, with chromium-based browsers:
   
	`LD_PRELOAD="/usr/lib/libhardened_malloc.so" /usr/bin/chromium`
2. Install Linux Kernel Runtime Guard:
   
	`paru -S lkrg-dkms`
3. Enable `nftables.service` to load default ruleset on startup:
   
	`sudo systemctl enable nftables.service`
4. Setup dnscrypt-proxy:
	- `sudo pacman -S dnscrypt-proxy`
	- Edit its configuration file using `sudo vim /etc/dnscrypt-proxy/dnscrypt-proxy.toml`
	- set `listen_addresses` to `'127.0.0.1:53', '[::1]:53'`
	- set `server_names` to `'cloudflare-security', 'cloudflare-security-ipv6'` or the provider of your choice from [this list](https://download.dnscrypt.info/resolvers-list/v3/public-resolvers.md)
	- save and exit, then edit `resolv.conf` using `sudo vim /etc/resolv.conf` and add:
	  
		`nameserver ::1`
		
		`nameserver 127.0.0.1`
		
		`options edns0`
		
		and save and exit
	- `sudo systemctl enable --now dnscrypt-proxy.service`
5. Setup AppArmor for Mandatory Access Control:
	- `sudo pacman -S apparmor`
	- `sudo systemctl enable apparmor.service`
	- `sudo vim /etc/default/grub` and add the following line at the end of `GRUB_CMDLINE_LINUX_DEFAULT="blah blah <here>"`:
	  
		`lsm=landlock,lockdown,yama,integrity,apparmor,bpf`
		
		and save and exit. This line also enables 'Kernel Lockdown' in `integrity` mode.
	- `sudo grub-mkconfig -o /boot/grub/grub.cfg` and reboot

#### Secureboot
- `sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --modules="tpm" --disable-shim-lock`
- `sudo grub-mkconfig -o /boot/grub/grub.cfg`
- `sudo pacman -S sbctl`
- Reboot to firmware setup/UEFI Settings, set Secureboot mode to 'setup' mode and reboot
- `sudo sbctl create-keys`
- `sudo sbctl enroll-keys -m`
- `sudo sbctl verify` - this will list all the files that are needed to be signed
- sign all the listed files one-by-one, like so:
  
  	`sudo sbctl sign -s /efi/EFI/GRUB/grubx64.efi`
- If you get an error about certain file being immutable, use the following command to make it mutable and continue signing the necessary files:
  
	`sudo chattr -i /sys/firmware/efi/efivars/<filename>`
- Verify that all the necessary files are signed using `sudo sbctl verify`
- reboot to firmware setup/UEFI settings, set Secureboot to `enabled` or whatever is the option to enable it and reboot.
- verify that secureboot is enabled using `sudo sbctl status`

### Conclusion
After reading this entire guide, you can probably tell that it's more so a documentation of a _personal_ Arch Linux install rather than being an objectively secure install, which you would know if you know anything about what makes an operating system secure and have used actually security-oriented operating systems like Whonix, Kicksecure, GrapheneOS on Pixel phones, or my personal favorite, Secureblue. But the best and the worst security component is still the _human factor_, and this is totally not a statement I am making just to calm down my own paranoia. That said, this guide, as already mentioned before, will keep on updating and perhaps even overhauling depending on what helps me fulfill my above-mentioned goals completely and efficiently. Thank you for spending your precious time reading this little guide I wrote to kill time in the hopes that it will benefit somebody somewhere sometime, and if it doesn't, I still had fun.

### Commit History
(changes made to this guide are tracked in this section in the order of latest to the oldest)
- 28 February, 2025 - *"Initial Commit"*
  
	`- guide made public for the first time`

##### References
- https://wiki.archlinux.org/title/Installation_guide
- https://wiki.archlinux.org/title/Security
- https://wiki.archlinux.org/title/AppArmor
- https://gist.github.com/Th3Whit3Wolf/2f24b29183be7f8e9c0b05115aefb693
- https://github.com/secureblue/secureblue
- https://youtu.be/Qgg5oNDylG8
- https://youtu.be/_97JOyC1o2o
- https://youtu.be/-3W2f_JL3yw
- https://www.dwarmstrong.org/archlinux-install/
- https://gist.github.com/Th3Whit3Wolf/0150bd13f4b2667437c55b71bfb073e4
- https://www.arcolinuxd.com/installing-arch-linux-with-a-btrfs-filesystem/
- https://www.reddit.com/r/archlinux/comments/10pq74e/my_easy_method_for_setting_up_secure_boot_with/
- https://gitlab.com/eflinux/arch-basic
