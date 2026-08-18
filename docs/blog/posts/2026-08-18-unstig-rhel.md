---
date:
  created: 2026-08-18
comments: true
authors:
  - connor
categories:
  - Infrastructure
  - Security
tags:
  - STIG
  - RHEL
  - Linux
---

# UNSTIG: Undoing STIG Lockdowns on RHEL

If you've ever been handed a RHEL box that's been hardened against the Security Technical Implementation Guide (STIG), you know the pain. Things that should just work — mounting a USB drive, running a script you just downloaded, `sudo` not asking for your password every five seconds — suddenly don't, and the error messages rarely point you at the actual cause.

This post is a running list of STIG-enforced settings I've run into and how to undo them. This is meant for personal labs, dev boxes, or environments where you have the authority to make this call — not for production systems still under a compliance mandate. Always check with whoever owns the STIG policy before changing any of this on a system you don't fully control.

<!-- more -->

## Unblock USB Storage

STIG disables USB mass storage by blacklisting the kernel module. Comment out the relevant lines in the modprobe config and reboot:

```bash
sudo vim /etc/modprobe.d/usb-storage.conf
# comment out the blacklist/install lines
sudo reboot
```

## Unblock USB Peripherals

`usbguard` blocks USB peripherals (keyboards, mice, etc.) from being used unless explicitly allowed:

```bash
sudo systemctl disable --now usbguard
```

## Unblock Bluetooth

Same idea as USB storage — the Bluetooth kernel modules are blacklisted:

```bash
sudo vim /etc/modprobe.d/bluetooth.conf
# comment out the blacklist lines
sudo reboot
```

## Unblock Executing Files

STIG mounts `/home`, `/var`, and `/var/tmp` with `noexec`, which blocks running scripts or binaries from those locations. To fix it for the current session:

```bash
sudo mount -o remount,exec /home
sudo mount -o remount,exec /var/tmp
sudo mount -o remount,exec /var
```

To make it stick across reboots, edit `/etc/fstab` and change `noexec` to `exec` on the relevant lines.

## Unblock Sudo Timeout

By default STIG sets `sudo` to re-prompt for a password on every single command. Add this to `/etc/sudoers` (via `visudo`) to restore a normal timeout:

```
Defaults timestamp_timeout=0
```

Set it to whatever timeout (in minutes) works for you — `0` means the password is asked every time by design, so bump it up (e.g. `15`) if you want `sudo` to remember your password for a while instead.

## Unblock Executing Unsigned Applications

`fapolicyd` enforces an application allowlist, which blocks running anything that isn't explicitly trusted:

```bash
sudo systemctl disable fapolicyd
```

## Unblock SELinux

To drop SELinux into permissive mode for the current session:

```bash
sudo setenforce 0
```

To make it permanent, edit `/etc/selinux/config` and set `SELINUX=permissive` (or `SELINUX=disabled`), then reboot.

---

I'll keep adding to this list as I run into more STIG headaches. If you've hit something not covered here, let me know in the comments.
