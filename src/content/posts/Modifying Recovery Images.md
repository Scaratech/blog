---
title: Modifying Recovery Images
published: 2024-09-04
description: 'How to modify ChromeOS Recovery Images'
image: ''
tags: ['ChromeOS']
category: 'Programming'
draft: false 
language: 'English'
---

If you are reading this you are probably familiar with "Recovery Images" for ChromeOS.\
They allow you to boot a USB and install a new fresh version of ChromeOS if you manage to break something.

## How do they work?
Before we talk about modifying them I wanted to mention how they work.

### Partitions
There are 12 partitions on the Recovery Image. Only three are useful.
- `ROOT-A`/`RootFS`
    - This is the partition that contains all the programs, scripts, etc. that can be executed.
- `KERN-A`
    - This is the kernel for the Recovery Image.
- `stateful`
    - This partition contains heavily encrypted user data. Basically everything in `/home/*`

### DM-Verity
All of the partitions except `ROOT-A` are checked by something called `dm-verity`.\
Basically `dm-verity` is an algorithm that verifies the integrity of files to make sure they have been modified.\
`ROOT-A` is actually checked by `dm-verity` in verified mode, however if you boot the USB from developer mode it won't be checked.

### Developer Mode vs Verified 
You probably know what devmode is so I won't explain it in detail however I did want to clarify some things.

1. Your data will be wiped when transitioning from Developer Mode to Verified, but how?\
Simple! All of your user data is stored on that `stateful` partition and it will just wipe that partition.

2. As a mentioned already, you can only modify the `ROOT-A` partition in devmode, not verified. This is because verified boot will check `ROOT-A` with `dm-verity`

### How does the system actually recover?
There are a couple of scripts stored in the `ROOT-A` found in `/usr/sbin/`.
- `chromeos-install`
    - This is the actual script that will install a fresh version of ChromeOS
- `chromeos-recovery`
    - This is the script that invokes `chromeos-install` as it takes some arguments
- `chromeos-postint`
    - After the install is complete (or an update happens) this script will be executed

## How do I modify a Recovery Image?
First you need to be in some sort of Linux environment. While you could try WSL I'm not sure if it works. I recommend doing it in a liveboot or running a VM (or even better just daily drive Linux!)

1. First obtain a Recovery Image. To find them visit https://cros.tech and search your Chromebook model
2. Once the image is finished downloading extract the `.zip` file, then you should have a `.bin` file
3. Create a directory and move the recovery image there, then download and place [this](https://github.com/MercuryWorkshop/RecoMod/blob/main/lib/ssd_util.sh) script in that directory.
4. Then run these commands in your terminal:
```sh
loopdev=$(losetup -f)
losetup -P "$loopdev" "./PATH_TO_RECOVERY_IMAGE.bin"

chmod +x ssd_util.sh
./ssd_util.sh --remove_rootfs_verification --no_resign_kernel -i "$loopdev" --partitions 2

sync

ROOT=$(mktemp -d)
mount "${loopdev}p3" "$ROOT"
```

The RootFS of the Recovery Image has been mounted and you can start modifying it!\
However, we are not quite done yet, the `chromeos-recovery` script is run inside of a `chroot` jail" and is very sandboxed. Lucky [Mercury Workshop](https://mercurywork.shop) made a [script](https://github.com/MercuryWorkshop/RecoMod/blob/main/utils/chromeos-recovery.sh) that performs a `chroot` escape as well as gains `PID1` code execution!

To get started open `chromeos-recovery` with your favorite text editor and replace it with:
```sh
USB_MNT=/usb
set +x

# Chroot Escape
echo 0 >/proc/sys/kernel/yama/ptrace_scope
if ! [ "$(cat /proc/sys/kernel/yama/ptrace_scope)" = "0" ]; then
    echo "failed to enable ptrace"
    sleep 1d
fi

sleep 1

# PID1 Code Execution
clamide -p 1 --syscall execve "str:$USB_MNT/usr/sbin/bootstrap-shell"

spinner=$(pgrep sh | tail -1)
kill -9 $spinner
pkill -f frecon
pkill -f tail
```

Then make a new file called `bootstrap-shell` under `/usr/sbin`.\
Whatever code you place in this file will be executed freely!

However here are some notes you probably want:
1. The code is executed as `bushboxsh`, not `bash`
2. It runs as `PID1` (`initramfs`), meaning if it dies the kernel will panic
3. You can access `freecon`! If you are unaware, `freencon` is Google's replacement for a kernel console and can do some extra graphical stuff.

### Finishing up your modifications
Once you have made your modifications there is still some clean up to do, open your terminal and run these commands:
```sh
curl -sL "https://github.com/CoolElectronics/clamide/releases/latest/download/clamide" -o "clamide"
cp clamide "$ROOT/usr/sbin/clamide"
chmod +x "$ROOT/usr/sbin/clamide"

chmod +x "$ROOT/usr/sbin/chromeos-recovery"
chmod +x "$ROOT/usr/sbin/bootstrap-shell"

sync

umount "$ROOT"
losetup -d "$loopdev"
rm -rf "$ROOT"
```

And there you have it, your own modified ChromeOS Recovery Image!

## But how do I use it?
Simple!

1. Take the `.bin` file and flash it to a USB using a tool like `dd`, the ChromeOS Recovery Utility, etc.
2. Boot the USB like you would boot sh1mmer
    - `esc + refresh + power`
    - `ctrl + d`
    - `enter`
    - `esc + refresh + power`
    - Insert USB

## Credits
- [Recomod](https://github.com/mercuryworkshop/recmod)
    - [ssd_util](https://github.com/MercuryWorkshop/RecoMod/blob/main/lib/ssd_util.sh)
    - [Chroot Escape & PID1 Code Execution](https://github.com/MercuryWorkshop/RecoMod/blob/main/utils/chromeos-recovery.sh)
    - [Commands for Modifying Recovery Images](https://github.com/MercuryWorkshop/RecoMod/blob/main/recomod.sh)

