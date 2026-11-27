# nikkmk2: Komplete Kontrol S MK2 screens under Wine

`nikkmk2.sys` is a Wine stand-in for Native Instruments' `nikksm2usb.sys`, the
USB driver for the two screens of the Komplete Kontrol S49 / S61 / S88 MK2
(USB 17cc:1610 / 1620 / 1630).

NI's driver is a KMDF driver: it imports `WDFLDR.SYS`, and Wine cannot load
KMDF drivers. So under Wine NIHardwareService never finds the screens. The
Light Guide and the buttons still work, because they are plain HID. With
`nikkmk2.sys` NI's own software draws both screens.

It is a port of `nimk3.sys`, the Maschine MK3 driver of ni-wine by Selim
Bucher and contributors: https://github.com/selimbucher/native-instruments.
All the credit for the design goes there. The rest of this repository is
ni-wine as upstream ships it; its README is [README.ni-wine.md](README.ni-wine.md).

This repository does not contain or distribute any software from Native
Instruments.

## What changed from nimk3.sys

The source is [hardware/nikkmk2.c](hardware/nikkmk2.c), with
[hardware/komplete-kontrol-mk2.inf](hardware/komplete-kontrol-mk2.inf).

NI's `nikksm2usb.sys` (5.0.0.57) and the MK3's `nimc3usb.sys` belong to the
same driver family:
- they handle the same IOCTLs;
- both report kind 3 for the vendor interface (class 0xff, subclass 0xbd);
- both allow bulk writes only on that interface.

The port changes:
- the device interface GUID to `{C404974A-C9C1-4477-A9E1-9F7B8E0E4BC9}`, the
  one NIHardwareService looks for an S MK2 under;
- the interfaces: MI_03 is the display (bulk OUT 0x03, kind 3) and MI_00 is
  audio (kind 0). AddDevice refuses every other interface, such as MI_04,
  the firmware-update one;
- GET_VERSION to `{2,0,5,0,0,0,0,0}`, which is what `nikksm2usb.sys` reports.

## Tested

- Hardware: Komplete Kontrol S88 MK2.
- Wine: 11.17-staging, through vstenv 0.1.168.
- NI software: NIHardwareService 2.8.2.
- Result: both screens and the Light Guide work in Maschine 2.

S49 and S61 MK2 have the same interfaces and are in the INF, but they are
untested.

## Build

You need a 64-bit mingw-w64 cross compiler (Arch: `mingw-w64-gcc`).

    cd hardware
    make nikkmk2.sys
    x86_64-w64-mingw32-objdump -p nikkmk2.sys | grep 'DLL Name'   # ntoskrnl.exe and HAL.dll only

Or in a container, without installing anything on the host:

    podman run --rm -v "$PWD/hardware:/src:Z" -w /src docker.io/library/archlinux:latest \
        sh -c 'pacman -Syu --noconfirm --needed mingw-w64-gcc make && make nikkmk2.sys'

## Install

Your prefix needs:
- NI's Komplete Kontrol MK2 driver installed;
- the keyboard's USB and hidraw nodes writable by your user (for example a
  udev rule with `TAG+="uaccess"` for 17cc:1610/1620/1630).

The simplest way puts the build in place of NI's file. The device bindings
that NI's installer made then load it:

    P=/path/to/wineprefix
    D="$P/drive_c/windows/system32/drivers"
    mkdir -p "$P/nikkmk2-backup"
    cp -p "$D/nikksm2usb.sys" "$P"/system.reg "$P"/user.reg "$P/nikkmk2-backup/"
    install -m 0644 hardware/nikkmk2.sys "$D/nikksm2usb.sys"

Then:
1. Quit the NI programs.
2. Unplug and replug the keyboard, in the same USB port.
3. Wait for the NI services to restart. ni-wine's `nirescan` service does that.
4. Open Maschine or Komplete Kontrol.

If the screens still show nothing, stop Wine (`wineserver -k`) and start again.

Another way is to bind the interfaces with `komplete-kontrol-mk2.inf` as a
separate `nikkmk2` service, the way ni-wine installs `maschine-mk3.inf`.

## Undo

    cp -p "$P/nikkmk2-backup/nikksm2usb.sys" "$D/nikksm2usb.sys"

Then replug the keyboard. Restore the `.reg` backups only if the prefix is
broken, and only with Wine stopped.

## Debugging

    WINEDEBUG=warn+debugstr    # the driver's nikkmk2: lines

## License

GPL-3.0, the same as upstream ni-wine (see [LICENSE](LICENSE)).
