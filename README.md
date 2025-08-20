# Disclaimer
This project was created as a **learning exercise**. It contains my current understanding of how to use OpenBSD 7.7 on a Raspberry Pi as a firewall/router.

Please note:
- I am **not** an OpenBSD expert, and this was made as I first begin to learn about this incredible operating system.
- I am **not** a security professional, and this configuration should **not** be used in production or anywhere that security/privacy are considered critical.
- The configuration and instructions herein are shared as-is with no guarantees of correctness, completeness, or security.
- Use at your own risk.

**Please refer to the official OpenBSD documentation as THE source of truth.**

If you see anywhere that I can make improvements, I would very much appreciate the learning opportunity. Thanks!
# Installation
## Creating Install Media
OpenBSD officially supports the hardware for the Raspberry Pi 4. this includes the 'closed but redistributable files on the system disk to load into the VC4 GPU'. Additionally, the Raspberry Pi 4 defaults to attempting to boot off the SD card. In an attempt to avoid having to create a Raspberry Pi OS SD to update the EEPROM to be able to boot off a USB this guide will solely use an SD card, and write the OpenBSD system in-place overtop the installer.

**NOTE: This installation was performed, and therefore verified, on OpenBSD 7.7. It makes no statements of applicability for any versions prior or since.**

1. Download the latest miniroot77.img from the [OpenBSD](https://www.openbsd.org/faq/faq4.html#Download) website.
2. In a console, navigate to the directory where you saved the miniroot77.img.
3. Insert the SD card into your computer, and using Linux determine the device name: `lsblk` (this guide will assume sda)
4. Use the `dd(1)` utility to flash the SD card: `dd if=miniroot77.img of=/dev/sda bs=1M`
5. Remove the SD card from the OpenBSD system, and insert it into the Raspberry Pi.
## Pre-Installation Setup
I completed the install via serial console over the Pi UART ports, and with a PoE hat to allow network and power to be one cable. If you have a keyboard, mouse, monitor, micro-HDMI, and power supply those are usable as well. This section will step-through the actions performed as I did them. Please modify accordingly.

**Required Hardware:**
- Adafruit [FT232H](https://www.adafruit.com/product/2264)
- Male to male jumper wires
- Raspberry Pi [PoE hat](https://www.raspberrypi.com/products/poe-hat/)
- Ethernet cord
- PoE capable switch

**Hardware Setup:**
1. Connect the PoE hat to the Raspberry Pi.
2. Using the jumper wires, connect the FT232H ports to the Raspberry Pi GPIO as:
- GPIO14 -> D1
- GPIO15 -> D2
- Ground -> Ground
3. Connect the USB port of the FT232H to your laptop, and start minicom to get a serial console: ` sudo minicom -D /dev/ttyUSB0 -b 115200`
4. Connect the ethernet cord to both the Raspberry Pi and the PoE switch. Verify you begin to see text in the minicom window.
## OpenBSD Installation
**NOTE: All following steps will be performed in the minicom session unless otherwise noted.**
1. After the installer finishes booting there sould be a prompt asking to `(I)nstsall, (U)pgrade, (A)utoinstall, or (S)hell`. Select: `(I)nstall`.
2. At this point the standard installation process will be followed. The selections made during the installation were as follows (all answers in square brackets were the default option):a
**NOTE: The password set for root and user during install are left off the answers. Please set accordingly.**
```sh
System hostname = pirewall
Network interface to configure = [bse0]
IPv4 address for bse0 = [autoconf]
IPv6 address for bse0 = [none]
Start sshd(8) by default = [yes]
Setup a user = user
Full name for user user = user
Allow root ssh login = [no]
Which disk is the root disk = [sd0]
Encrypt the root disk with a (p)assphrase or (k)eydisk = [no]
Use (W)hole disk or (E)dit the MBR = [whole]
Use (A)uto layout, (E)dit auto layout, or create (C)ustom layout = [a]
Location of sets = http
HTTP proxy URL = none
HTTP Server = ftp.eu.openbsd.org
Server directory = [pub/OpenBSD/7.7/arm64]
Set name(s) = -game*
Set name(s) = -x*
Set name(s) = -comp*
Set name(s) = done
Location of sets = done
```
3. The installer will then handle fetching/verifying the sets, performing a `fw_update`, and setting up the kernel for the new system. When complete you should be met with a prompt asking to `Exit to (S)hell, (H)alt or (R)eboot?`. Select: `(R)eboot`.
## Post-Installation Setup
1. After the Pirewall has finished rebooting, find it's IP address on your local network and attempt to SSH: `ssh user@XXX.XXX.XXX.XXX`
2. If successful, you may now disconnect the UART and FT232H for simplicity. If not, verify no LAN configuration conflicts would prevent SSH.
3. Attempt a simple `doas(1)` command to verify correct configuration: `doas ls -l`
4. If the `doas(1)` failed, then verify the `/etc/doas.conf` contains: `permit :wheel`. If not, then change to root and edit the file accordingly.
5. Verify the `/etc/installurl` exists and contains: `https://cdn.openbsd.org/pub/OpenBSD`
6. Follow the guidance in the `afterboot(8)` man page.
7. Apply any available binary patches: `syspatch`
8. Update all installed packages: `pkg_add -Uu`
9. When installing the Timezone should have been autodetected. However I had to manually fix this. If `date` shows an incorrect value then re-link the `/etc/localtime` with the `/usr/share/zoneinfo` applicable to you. For example: `doas ln -sf /usr/share/zoneinfo/US/Michigan /etc/localtime`.
10. If `doas rcctl ls on` shows `ntpd(8)` is running, then proceed to next step. Else:
```sh
doas rcctl enable ntpd
doas rcctl start ntpd
```
11. Verify no outstanding errors have been logged that need correcting: `tail -n 100 /var/log/messages`
12. Copy the files from this repo into the corresponding locations on the Pirewall.
13. Reboot the Pirewall once more.

# Usage
The workflow for using the Pirewall is as such:
1. Connect the unprotected device to the Ethernet port of the Pirewall
2. Verify DHCP gives an address of `172.16.0.2`. If the unprotected device is on OpenBSD this can be done via `ifconfig`, and if on Linux this can be done with either `ifconfig` or `ip a` depending on system configuration.
3. From the unprotected device, ssh into the Pirewall: `ssh user@172.16.0.1`
4. In the Pirewall, connect to a local WiFi SSID (this step will assume WPA encryption): `doas ifconfig bwfm0 nwid '{SSID}' wpakey '{Passphrase}'
5. If WiFi connection is successul, test Internet connection on the now protected device, close the SSH session, and close the terminal in which the session occurred.

**Congratulations!!! The Pirewall should now be protecting your traffic.**

# Known Issues
Please refer to the GitHub issues page as the source of truth. If known, all issues will be created with a temporary workaround whilst a permanent solution is being investigated.

# Future Work
This is a *functional* Pirewall, however it isn't necessarily an *effective* Pirewall (if such a thing exists). Future work includes:
- Auditing running services to shutdown anything not strictly needed for the function of the device (to include `sndiod(8)`)
- Attempting to squeeze more performance out of the networking capability (on a 1GbE WiFi only ~40Mbps was possible through the Pirewall)
- Learning what other hardening methods can be employed to make this an effective method of protecting a single Ethernet device (this is a learning exercise afterall)
