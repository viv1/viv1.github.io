---
layout: post
title: My Raspberry Pi failed to show the Graphical desktop interface on startup, and how I fixed it
date: 2026-01-01 14:30
category: Debugging
tags: ["raspberry-pi", "debugging", "lightdm", "storage", "tty", "vncserver", "gui"]
description: My Raspberry stopped loading the GUI view. This is a walkthrough of steps used for debugging, and what worked.
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [What was Happening ?](#what-was-happening-)
- [Initial Symptoms](#initial-symptoms)
- [Troubleshooting Steps I Took](#troubleshooting-steps-i-took)
   * [Configure Boot to Desktop](#configure-boot-to-desktop)
   * [Verify Desktop Environment Installation](#verify-desktop-environment-installation)
   * [Check Display Manager Status](#check-display-manager-status)
   * [Check Detailed Logs](#check-detailed-logs)
   * [Tried to Manual X Server Start](#tried-to-manual-x-server-start)
- [The Root Cause: Full Disk Storage](#the-root-cause-full-disk-storage)
- [Obvious Solution: Free Up Disk Space](#obvious-solution-free-up-disk-space)
   * [Identify Large Files](#identify-large-files)
   * [Clean Up Commands](#clean-up-commands)
   * [Verify and Reboot](#verify-and-reboot)
- [Result](#result)
- [How can I prevent this in future ?](#how-can-i-prevent-this-in-future-)
- [Conclusion](#conclusion)

<!-- TOC end -->

## Introduction

I occasionally use my raspberry pi to learn, practice and experiment with new concepts. It usually just works for me, until it doesn't one day.

It's always connected to my home network, to which I connect via SSH from any of my other personal machines. If I need to use GUI, I simply start a `vncserver` and connect to it using any VNC Viewer.

Recently, I needed to view the GUI, but my vncserver kept crashing. 

This article is simply the set of steps to try, in case this happens again.

## What was Happening ?

I tried connecting My Raspberry Pi directly to a monitor, but it was booting directly to a text terminal (tty4) instead of showing the graphical desktop interface, even though I had a monitor connected.

## Initial Symptoms

- System boots to terminal prompt instead of desktop
- Shows `tty4` or `tty1` at login
- Monitor is properly connected but no GUI appears

## Troubleshooting Steps I Took

### Configure Boot to Desktop

First, I tried configuring the system to boot into graphical mode:

```sh
sudo raspi-config
```

Navigation:
1. System Options → Boot / Auto Login
2. Selected "Desktop Autologin"
3. Rebooted

Or, I could have done:

```sh
sudo systemctl set-default graphical.target
```

After reboot, check :

```sh
systemctl get-default
# output should be : "graphical.target"
```

After this change, the terminal switched from `tty4` to `tty1`, but still no desktop appeared.

### Verify Desktop Environment Installation

I checked if the desktop environment was properly installed:

```sh
dpkg -l | grep lxde
```

My output:

```sh
pi@pi:~ $ dpkg -l | grep lxde
ii  lxde                                  10                                       all          metapackage for LXDE
ii  lxde-common                           0.99.2-3                                 all          LXDE common configuration files
ii  lxde-core                             10                                       all          metapackage for the LXDE core
ii  lxde-icon-theme                       0.5.1-2                                  all          LXDE standard icon theme
ii  openbox-lxde-session                  0.99.2-3                                 all          LXDE session manager and configuration files
```

The output confirmed LXDE was installed correctly with all necessary packages.

### Check Display Manager Status

I investigated the lightdm display manager:

```sh
systemctl status lightdm
```

This revealed the real problem: lightdm was constantly crashing and restarting with `status=1/FAILURE`.

But it did not show exactly why.

### Check Detailed Logs

Looked deeper into the logs:

```sh
journalctl -u lightdm -b
```

The logs showed repeated crashes:
```
lightdm.service: Main process exited, code=exited, status=1/FAILURE
lightdm.service: Failed with result 'exit-code'
```

### Tried to Manual X Server Start

Attempting to start X manually revealed the root cause:

```sh
sudo startx
```

Error messages appeared:
```sh
xauth: unable to write authority file /tmp/serverauth.xxxxx-n
Fatal server error:
Could not write pid to lock file in /tmp/.tX0-lock
```

## The Root Cause: Full Disk Storage

Finally, I checked disk space:

```bash
df -h
```

**The problem:**
```
/dev/root    12G   12G     0 100% /
```

The root filesystem was completely full! X server couldn't create necessary temporary files and lock files, causing `lightdm` to crash repeatedly.

## Obvious Solution: Free Up Disk Space

### Identify Large Files

```sh
sudo find / -type f -size +100M 2>/dev/null | head -20
```

Removed all unnecessary large files.

### Clean Up Commands

```sh
# Standard cleanup
sudo apt clean
sudo apt autoremove
sudo journalctl --vacuum-size=50M
```

### Verify and Reboot

```sh
df -h  # Should show available space now
sudo reboot
```

## Result

After freeing up disk space and rebooting, the Raspberry Pi successfully booted into the graphical desktop environment!

## How can I prevent this in future ?

- Increase Storage , I have been fine with it so far but will need to be done.
- Warn on reaching 90% storage. Probably should have a simple program to warn in such cases.
- Delete unnecessary project files and all cache cleanups.

## Conclusion

In hindsight, storage problems impacting display is expected, however, it is not something that comes to mind right away. Ensuring proper available storage is a must.
