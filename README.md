Downloads are on the [latest release](https://github.com/Bradyok/novaos-debian13/releases/latest).

```text
NovaOS Debian 13 for the Rigol DHO800 / DHO900 and MHO900
========================================================

NO OSCILLOSCOPE APPLICATION. This is an operating system to build on, and the scope
does not acquire under it: nothing here starts the FPGA.

Not made, endorsed or supported by Rigol. Rigol is named only to say which scopes
this fits, and nothing of Rigol's is in it. Your stock firmware stays on the card.
Writing to the card is at your own risk, with no warranty: back it up first.

What was tested: a DHO924S with this image in recovery, restarted, its card booting our
own first stage instead of Rigol's loader; the same U-Boot chain booting this Debian from
RAM on that DHO's earlier build and on an MHO934, whose card was not written. Rigol's
loader starting this exact image has not been run here (it starts the same r115 U-Boot
chain in a forum user's own build). apt was tested on a PC under QEMU, not on a scope.
Nobody has used the touchscreen by hand in these tests, only seen X detect it.

What this is
------------
Plain Debian 13 ("trixie", arm64) with the Xfce desktop, running on the scope
itself. It is ordinary Debian: install anything with apt, build your own
programs on it (build-essential, git, gdb and pkg-config are already there).

What it carries beside Debian's own packages is only what the scope needs to
boot and drive its hardware:
  - the scope's Linux kernel and its modules, and the board's device trees;
  - the boot files: a small start-up script in the boot image that finds this
    system's file on the card and starts it;
  - configuration: the GPU on the open-source Panfrost driver, the panel for X,
    the login screen, and two small systemd units that call Debian's own tools
    (resize2fs grows the system on its first start; busybox devmem tells the
    boot chain that a boot stayed up).
None of the scope's programs are in it: no oscilloscope application, no
NovaOS service, no keys of anyone's.

One set of files works on both families: DHO800/DHO900 and MHO900 have the same
RK3399 platform, kernel, 7" 1024x600 touch panel and touch controller.

This build: novaos-debian13-67b643dd0268 (kernel 6.12.115+, release build)

The files
---------
  novaos-debian13-root.ext4.gz the Debian system: unpacked, rigol-debian/root.ext4
                               in the stock firmware's data area (/data)
  novaos-debian13-dho900-boot.img       the boot image for a DHO800/DHO900 (kernel, device
  novaos-debian13-mho900-boot.img       tree, start-up script), and for an MHO900
  novaos-debian13-<board>-recovery-chain.img (board: dho900 or mho900)
                               the same boot image inside the mainline U-Boot
                               chain image (U-Boot proper as published, release
                               r115): for a scope whose recovery partition already
                               holds that chain
  SHA256SUMS                   every file

Installing
----------
Debian lives on the scope's internal card as one file in the stock firmware's
data area; the card's recovery partition holds its boot image.

With the boot chain you already run (U-Boot proper in recovery, booting a FIT):
  1. Copy root.ext4 to /data/rigol-debian/root.ext4 on the scope (as root on the
     stock firmware, or on any Linux running there). The data area needs about
     2 GB free for it, and keeps 2 GB for the stock firmware.
  2. Write novaos-debian13-<board>-recovery-chain.img to the recovery partition, or
     pack novaos-debian13-<board>-boot.img into your own chain image the way this release's
     source bundle's install/INSTALLING.txt describes for a kernel you built.
     From a PC with the card in a reader, or a shell of your own. NovaOS and this
     Debian keep the card's other partitions read-only while they run: there,
     run blockdev --setrw on the recovery partition first.
  3. Restart. The boot chain boots Debian from recovery.

A scope still on stock firmware: the stock U-Boot starts recovery only when the first
sector of the boot partition is zero. Save boot and recovery
first, because writing them back is how you return to stock. On the card itself, boot
starts at byte 0x3C00000 (32 MiB long) and recovery at 0x5C00000 (64 MiB long). Check
with lsblk that the device you write to really is the card.

Use the image for your board. Never write the dho900 image to an MHO: GPIO1_C2 is the
DHO's backlight enable and the MHO's clock chip select.

Keep your own copy of the recovery partition before you write it: writing that
copy back restores what was there.

Logging in
----------
The panel boots straight into the desktop as the user nova, with no password.
sudo asks for none; root has no password (use sudo). The screen never locks.

SSH lets nobody in until you choose how: run sudo passwd nova on the scope, or
put your key in ~/.ssh/authorized_keys, then log in as nova. Anyone at the panel
is an administrator, so if that matters where the scope sits, set a password and
remove the two autologin lines from /etc/lightdm/lightdm.conf.d/50-rigol-debian.conf.

The touchscreen works in the desktop; there is an on-screen keyboard ("Onboard") in the Applications menu,
and a USB keyboard and mouse work too.

The network
-----------
Ethernet works by DHCP (NetworkManager; nmcli or the panel's network icon).
Wi-Fi needs a USB Wi-Fi adapter and its firmware package from Debian
(non-free-firmware is in the apt sources). SSH runs on port 22 (Debian's own
OpenSSH, password or key). The kernel has no fixed MAC for the Ethernet port
here, so NetworkManager keeps one of its own per system: the scope keeps the
same DHCP address across restarts, but it is not the address the stock firmware
gets.

What to expect
--------------
  - Space: the system's file starts at about 2 GB and grows to 8 GB on its first
    start, as long as the stock firmware's data area keeps 2 GB free (with less
    room it grows less). df -h / shows it.
  - The GPU (Mali-T860) runs on Panfrost (Mesa); X draws with it (modesetting
    with glamor).
  - Power off: Log Out > Shut Down. The boot files' start-up script closes the
    system's file and switches the scope off through its front-panel controller.
    The power key itself does nothing under Debian; hold it, or pull the power,
    if the scope ever hangs.
  - The MHO900's fan runs at full speed (the kernel's pwm-fan driver; nothing in
    this system slows it). The DHO has no speed-controlled fan.
  - No timer writes to the card in the background: Debian's apt-daily,
    dpkg-db-backup, e2scrub and fstrim timers are off. Run apt by hand, or turn
    one on with: sudo systemctl enable --now apt-daily.timer
  - Documentation, man pages and translations are left out to keep the system
    small, and apt keeps leaving them out. To get them, remove
    /etc/dpkg/dpkg.cfg.d/99mmdebstrap and reinstall the packages you want them for.
  - The kernel is the scope kernel's own build, with its modules. A Debian kernel
    package will not boot the scope; do not install linux-image-*.
  - apt sources: trixie, trixie-updates, trixie-security (main, non-free-firmware).
  - The scope's internal card is hidden from the desktop: it holds Rigol's factory
    calibration and the stock firmware. Please leave it alone.
  - The serial console is on ttyS2 at 115200 baud, with a login.

Removing Debian
---------------
Write your copy of the recovery partition back (from this Debian, sudo blockdev
--setrw on that partition first), and delete /data/rigol-debian.

Source code
-----------
Linux (GPL-2.0: the kernel in the boot files and its modules in
/usr/lib/modules), BusyBox (GPL-2.0: Debian's busybox-static 1.37.0-6+b9, in the
boot files' initramfs) and Debian's packages in the system are free software;
/usr/share/doc/*/copyright in the system names each package's license. Their
complete corresponding source for this build is its GPL source bundle,
novaos-debian13-67b643dd0268-gpl-source.tar.xz (its sha256 in
novaos-debian13-67b643dd0268-gpl-source.tar.xz.sha256), which comes with this
download: the kernel's public base tree, our patches to it, the complete .config
and both board device trees, with a script that rebuilds this kernel and its
modules byte for byte; the Debian source package of that BusyBox; and the list
of the Debian source packages of every package in the system, by name and
version (debian-sources.txt); their source is Debian's own, kept by version at
snapshot.debian.org, and the written offer below covers it.

The kernel is, byte for byte, the one of release r136, whose kernel source is
also published at
https://github.com/Bradyok/novaos-kernel-source/releases/tag/r136

WRITTEN OFFER. For three years from the date you received this release, Olympian LED, Inc. will give anyone
a copy of this source on a physical medium for no more than the cost of making and sending it.
Write to Olympian LED, Inc., brady@aioscilloscope.com, naming the release (novaos-debian13-67b643dd0268).

U-Boot in chain/ (GPL-2.0+) is release r115's U-Boot proper, byte for byte the
published build (build id 1dd98c584b86); its complete source, with the scripts
that build it, is at
  https://github.com/Bradyok/novaos-boot-source/releases/tag/r115
and the written offer above covers it as well.
```
