# Odroid HC4 (meson64, Mali G31, AML video decoding)

This is a note and reminder to myself about state of the support and can be
hard to read for people who don't understand Linux system well but contains
useful hints.

Tested with recent images in early March 2021.  Legacy kernel is 4.9 and 5.10
for `meson64`, 5.11 for `odroid`.

## Overview

So far the performance is good but it has a terrible support, as has
pretty much always been the case in the ARM world.  Shame.

Instead I'd recommend Intel mini ITX (passive cooling) or maybe SBC if
you can match this board with 2 native SATA ports.  The Intel boards
are cheaper but memory and case will put you over the price of this
board.  But it will save you a headache, not to mention your valuable
time.

Mainline kernel is an option if you don't want to use it as a media
center or you're ok with software decoding (suboptimal solution at
least but can work for some uses).

If you decide to stick with old kernel you will need to forgo the idea
to use `arm64` for everything as you won't get deconding without using
`armhf` build.  There are also issues with some drivers.

Sticking with old kernel and not having X or Wayland is limiting.  I'd
personally much rather run X and have web browser readily available
along with Kodi.  If this was my main use case I'd rather sell it and
stick with a platform that is supported well.

## Android

There is no Android HC4 image and 64-bit Pie image for C4 did not quite work.
The hardware is probably different enough to not work, at no point TV worked
and after reboot it repeated these errors on serial console:

```
init: Could not find service hosting interface vendor.amlogic.hardware.hdmicec@1.0::IDroidHdmiCEC/default
init: Could not find service hosting interface vendor.amlogic.hardware.systemcontrol@1.0::ISystemControl/default
init: Could not find service hosting interface vendor.amlogic.hardware.systemcontrol@1.1::ISystemControl/default
```

It did boot off of SD card that caused I/O errors with Linux images (see SD
card issues section below).  It might mean that it's Linux driver issue
although I'm not sure if this image is large enough to trigger this.

USB flash drives and SD readers did not behave the way they did on Linux but
without root I could only see them being recognized and partition table
enumerated, but did not actually test them.

## Pre-requisites

Having `u-boot-tools` and `abootimg` might come in handy.

## Installing Linux

It has Petitboot.  Boot files (`boot.ini` or `boot.scr`) always had to be
placed in `/` otherwise they were not detected.

I don't know much about booting ARM so Petitboot was nice especially for
experimenting and booting from SATA drives.  It seems that it can be bypassed
with a button to boot directly from SD or removing altogether.  I tried it and
flashed back with `spiupdate_odroidhc4_20201222.img.xz` from
http://ppa.linuxfactory.or.kr/images/petitboot/odroidhc4/.

### Netboot
Armbian installed over netboot:

- in Petitboot drop to shell and run `netboot_default`
- go back and select Debian 11 `bullseye`, finish installation

This will boot.  It had 5.11 `odroid` kernel.

### Image

Beware:

- Armbian image does not boot, it has /boot instead of separate partition.
  You need to rename /boot and place boot files in `/` or use separate boot
  partition.

- Also it had 5.10 `meson64` kernel - I was not able to boot it (using another
  set of scripts either), I did not try 4.9 kernel

- It uses `boot.scr` which can't be edited by hand.  I first copied to
  `boot.ini` removed first line - binary header and recompiled with `mkimage
  -A arm64 -T script -O linux -d boot.ini boot.scr` but removing `boot.scr`
  and only having `boot.ini` should be enough.

- `vmlinuz-*` is not gzipped, this can be fixed by `gzip -n` but I'm not sure
  if it's needed since I was not able to boot it anyway (even using entirely
  different `boot.ini`)

I think it works when booting from SD directly.  I think it also has uImage so
maybe that's why.  I was not successful with Petitboot and didn't have any
reason to continue trying.

## CoreELEC kernel

I'm writing this from memory so you may need to correct any mistakes along the
way.

Reuse CoreELEC boot files, customize as needed.  Then basically just copy
`/lib/modules/4.9.113`, maybe even `/lib/firmware`.  It's advisable to copy
`amlvideodri.conf` and `media_modules.conf` in `/etc/modules-load.d` to
automatically load drivers for decoding.

You need to modify `initrd` to mount different `/sysroot`, otherwise it tries
to load `SYSTEM` squashfs with CoreELEC.  It may trigger 60s timeout if there
is no `/usr/lib/kernel-overlays`.  If `splash-image` is running it obscures
the messages.

See `repack` script on how to modify ramdisk image.

Run this script as root (needs permision to extract device file in
`dev/`) and it copies modified files from `files/`.  Example init file
is taken from CoreELEC 16.  Mainly it replaces mounting `/systroot`
but also adds `root=` option so you don't have to hardcode device
name, UUID or label into the script.

## Accelerated decoding

After all of my experiences I am not confident that building this will work on
*any* Linux distribution and hardware.  Maybe it will work better for you and
with different library.  I can't put my finger on the differences between
`arm64`, `armhf` and official CoreELEC 16 builds.

Confirmed working (built 19.1 kodi from CoreELEC
https://github.com/CoreELEC/xbmc/tree/aml-4.9-19.1 last change published on
2021-03-03
https://github.com/CoreELEC/xbmc/commit/e9df02b84573eacd7ce8a260b809e0c378c77465):

- Armbian bullseye (build made on bullseye did not work on buster)
- CoreELEC kernel 4.9.113
- only with oldest `armhf` `libamcodec` https://sources.coreelec.org/libamcodec-5faa7d4d554640dd6343900e24f43be327cea314.tar.xz
- Kodi has to be custom built for `armhf` and some code changed/removed

DietPi for HC4 does decoding but it is unusable.  As are some other
combinations.  It has newer 4.9 kernel and I was able to run this `kodi-aml`
build.

### kplayer

Package `aml-libs_20200921-8b4ccad-12_arm64.deb` from
http://deb.odroid.in/c4/pool/main/a/aml-libs/ is notable because it contains
`c4_amllibs` with `kplayer`.

This is useful for decoding on `arm64` because it works.  It is also
useful for testing between any failed decoding attempts as sometimes
it can break decoding and require a reboot.

There are others like `amplayer`/`smplayer`, but I have not tried them.

### Kodi

CoreELEC has state of the art support for AML systems.  Key information is
that you need to use `chroot` to run Kodi and libraries are available for
`armhf`.

Currently CoreELEC uses `libamcodec-31cd6eceaa1402b9f4ff5cc349e53899860fe9b9`
and provides 4 different `armhf` `libamcodec` libraries.  At least some of
them work.

What is notable is that for some reason
`5faa7d4d554640dd6343900e24f43be327cea314` tends to work outside of `chroot`
but causes `vfree` errors on `vdec_release`.  Inside of `chroot` there are no
`vdec_release`.

If you don't want to build your own Kodi take whole CoreELEC and put it in
`chroot`.  I've built it and it does work but whatever you do, always use
`chroot`.  Otherwise it does not work and nobody is going to tell you this.

```
CHROOT=/path/to/elec

mount -t proc none $CHROOT/proc
mount -o bind /dev $CHROOT/dev
mount -o bind /run $CHROOT/run
mount -o bind /sys $CHROOT/sys

# to access any local media
mkdir -p $CHROOT/mnt/host
mount -o bind / $CHROOT/mnt/host

chroot $CHROOT /usr/lib/kodi/kodi.bin
```

Of course you will still need to run your system on CoreELEC kernel.

#### CoreELEC quirks

- Remove `/usr/share/kodi/system/settings/appliance.g12x.xml` for turning off display mode switching
- `echo "1360x768p60hz\n720p60hz\n1080p60hz" > /storage/.kodi/userdata/disp_cap`
- `<showexitbutton>true</showexitbutton>` in `/usr/share/kodi/system/advancedsettings.xml`

#### Building Kodi

Beware: Don't even attempt this, there lies insanity.  Building is not that
hard but it does not run as expected!

This is where I started:
https://forum.odroid.com/viewtopic.php?p=251673&sid=76187422e888b7822d2564cae74d6899#p251673

This is still WIP but key issues are these:

- get `libamcodec` from https://sources.coreelec.org/, you want
  `31cd6eceaa1402b9f4ff5cc349e53899860fe9b9` at the time of writing

- `-DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc
  -DCMAKE_CXX_COMPILER=arm-linux-gnueabihf-g++`

- need to add `--cpu=cortex-a73.cortex-a53 --arch=arm` to `CMakeLists.txt`

- (because of `ENABLE_INTERNAL_FMT=ON`) `libfmt.a` is also built as 64-bit, I
  had to set `CMAKE_CXX_COMPILER:FILEPATH=/usr/bin/arm-linux-gnueabihf-g++`
  manually in `build/fmt/src/fmt-build/CMakeCache.txt` and let it build again
  (you might need to delete some build stage tracking files)

- remove `CAESinkPULSE::Register();` in
  `xbmc/windowing/amlogic/WinSystemAmlogic.cpp` if you don't build with
  PulseAudio support

I also did these while running Kodi outside of `chroot` but it's pointless:

- remove `multi_vdec` (I think this prevents invisible video)

- prevent division by 0 (happened only on `armhf`) in
  `xbmc/cores/VideoPlayer/VideoPlayer.cpp`: `if (hint.fpsscale > 0 &&
  hint.fpsrate / hint.fpsscale > 200) {` and maybe fallback to values 60000
  and 1001 like the original `if`
  
You need to install `armhf` dependencies for build.

If Kodi is built with `libsmbclient` then `samba-libs:armhf` will require
`python3:armhf`.

##### `arm64` build notes

List taken from Linux build readme file in repository.

(This includes `libsmbclient`!) Dependencies I used for `armhf` build
(including `libsmbclient:armhf`), I had to do a fair share of fixing
dependencies after doing `arm64` build beforehand - I recommend clean system:

```
autoconf automake autopoint gettext autotools-dev cmake curl default-jre
gawk flatbuffers-compiler gdc gperf libasound2-dev:armhf libass-dev:armhf
libavahi-client-dev:armhf libavahi-common-dev:armhf libbluetooth-dev:armhf
libbluray-dev:armhf libbz2-dev:armhf libcdio-dev:armhf libcec-dev:armhf
libp8-platform-dev:armhf libcrossguid-dev:armhf libcurl4-gnutls-dev:armhf
libcwiid-dev:armhf libdbus-1-dev:armhf libegl1-mesa-dev:armhf
libenca-dev:armhf libflac-dev:armhf libfontconfig-dev:armhf libfmt-dev:armhf
libfreetype6-dev:armhf libfribidi-dev:armhf libfstrcmp-dev:armhf
libgcrypt-dev:armhf libgif-dev:armhf libgles2-mesa-dev:armhf libgl-dev:armhf
libglew-dev:armhf libglu-dev:armhf libgnutls28-dev:armhf
libgpg-error-dev:armhf libgtest-dev:armhf libiso9660-dev:armhf
libjpeg-dev:armhf liblcms2-dev:armhf liblirc-dev:armhf libltdl-dev:armhf
liblzo2-dev:armhf libmicrohttpd-dev:armhf libmariadb-dev-compat:armhf
libnfs-dev:armhf libogg-dev:armhf libomxil-bellagio-dev:armhf
libpcre3-dev:armhf libplist-dev:armhf libpng-dev:armhf libpulse-dev:armhf
libshairplay-dev:armhf libsmbclient-dev:armhf libspdlog-dev:armhf
libsqlite3-dev:armhf libssl-dev:armhf libtag1-dev:armhf libtiff5-dev:armhf
libtinyxml-dev:armhf libtool libudev-dev:armhf libva-dev:armhf
libvdpau-dev:armhf libvorbis-dev:armhf libxkbcommon-dev:armhf libxmu-dev:armhf
libxrandr-dev:armhf libxslt1-dev:armhf libxt-dev:armhf waylandpp-dev:armhf
netcat wayland-protocols wipe lsb-release meson nasm ninja-build python3-dev
python3-pil python3-minimal rapidjson-dev:armhf swig unzip uuid-dev:armhf yasm
zip zlib1g-dev:armhf libcap2-dev:armhf ccache libinput-dev:armhf
libdav1d-dev:armhf clang-format libunistring-dev:armhf libpython3.9-dev:armhf
libiso9660++-dev:armhf ```

(This includes `libsmbclient`!) Dependencies I used for `arm64` build
(depending on build options some may be unused):

```
autoconf automake autopoint gettext autotools-dev cmake curl default-jre
gawk flatbuffers-compiler gdc gperf libasound2-dev libass-dev
libavahi-client-dev libavahi-common-dev libbluetooth-dev libbluray-dev
libbz2-dev libcdio-dev libcec-dev libp8-platform-dev libcrossguid-dev
libcurl4-gnutls-dev libcwiid-dev libdbus-1-dev libegl1-mesa-dev libenca-dev
libflac-dev libfontconfig-dev libfmt-dev libfreetype6-dev libfribidi-dev
libfstrcmp-dev libgcrypt-dev libgif-dev libgles2-mesa-dev libgl-dev
libglew-dev libglu-dev libgnutls28-dev libgpg-error-dev libgtest-dev
libiso9660-dev libjpeg-dev liblcms2-dev liblirc-dev libltdl-dev liblzo2-dev
libmicrohttpd-dev libmariadb-dev-compat libnfs-dev libogg-dev
libomxil-bellagio-dev libpcre3-dev libplist-dev libpng-dev libpulse-dev
libshairplay-dev libsmbclient-dev libspdlog-dev libsqlite3-dev libssl-dev
libtag1-dev libtiff5-dev libtinyxml-dev libtool libudev-dev libva-dev
libvdpau-dev libvorbis-dev libxkbcommon-dev libxmu-dev libxrandr-dev
libxslt1-dev libxt-dev waylandpp-dev netcat wayland-protocols wipe lsb-release
meson nasm ninja-build python3-dev python3-pil python3-minimal rapidjson-dev
swig unzip uuid-dev yasm zip zlib1g-dev libcap2-dev ccache libinput-dev
libdav1d-dev clang-format libunistring-dev ```

```
apt install `cat DEPS.install`
mkdir $HOME/kodi-build
cd $HOME/kodi-build
## cmake ../kodi -DCMAKE_INSTALL_PREFIX=/usr/local -DCORE_PLATFORM_NAME=aml -DAPP_RENDER_SYSTEM=gles
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local -DCORE_PLATFORM_NAME=aml \
    -DENABLE_INTERNAL_FLATBUFFERS=ON \
    -DENABLE_ALSA=ON -DENABLE_AIRTUNES=ON -DENABLE_UPNP=ON \
    -DENABLE_INTERNAL_FMT=ON -DENABLE_INTERNAL_RapidJSON=ON \
    -DENABLE_OPENGLES=ON -DENABLE_OPENGL=OFF -DENABLE_X=OFF \
    -DVERBOSE=OFF -DENABLE_NEON=ON \
    -DWITH_CPU=cortex-a73.cortex-a53 -DWITH_ARCH=aarch64 \
    -DENABLE_PULSEAUDIO=OFF -DENABLE_CCACHE=ON \
    -DENABLE_INTERNAL_FFMPEG=ON \
    -DENABLE_APP_AUTONAME=OFF -DENABLE_DVDCSS=OFF -DENABLE_INTERNAL_CROSSGUID=OFF -DENABLE_OPTICAL=OFF \
    -DENABLE_EVENTCLIENTS=ON -DENABLE_CAP=ON \
    -DENABLE_VDPAU=OFF \
-DAML_INCLUDE_DIR=AML/c4_amllibs.arm64/usr/include \
-DAPP_RENDER_SYSTEM=gles \ 
    ../xbmc

# other possible changes
# -DENABLE_SMBCLIENT=OFF \
# -DENABLE_AIRTUNES=OFF -DENABLE_UPNP=OFF \
# -DENABLE_INTERNAL_RapidJSON=OFF \
# -DENABLE_MDNS=OFF \
```

Then based on this
https://github.com/AlexELEC/AE-AML/blob/master/projects/S812/packages/libamcodec/package.mk
I had to use `libamcodec` headers from
https://github.com/surkovalex/libamcodec/tarball/bb19db7 and do some updates -
this patch
https://github.com/lippone/EmuELEC/blob/EmuELEC/packages/multimedia/libamcodec/patches/codec_ctrl-added-codec_get_vdec_info.patch
and `struct vdec_info` in `amstream.h` from
https://github.com/CoreELEC/linux-amlogic/blob/4a29617122b6bbc234cb7e2c70627f1c54f71f67/include/linux/amlogic/media/utils/amstream.h
to finish build.

Unfortunately it was unstable, especially playing H265 and seeking.  H264 was
fairly stable but I didn't test it extensively.  I guess the resulting library
is not compatible enough.  I later tried `chroot` and revisited header changes
but it did not seem to make any difference.

I tried to build several other `libamcodec` but I also put back `multi_vdec`
whcih proved so far to add problems (`vfm/map` probably did not set up
properly and video was not visible).

So far my build from `surkovalex` and `c2_aml_libs` worked best (the latter
flickers UI but in the former video image disappeared between experiments).
`codesnake` `5e23a8181` build prints errors like `send control
failed,handle=54,cmd=80085308,paramter=6ccb5650, t=ffffffff errno=25`.

#### `armhf` build notes

This is what I ended up with for `armhf` (at least keep out `libsmbclient`
because it requires `armhf` `python3`):

```
cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=/usr/local \
 -DCORE_PLATFORM_NAME=aml -DENABLE_INTERNAL_FLATBUFFERS=ON \
 -DENABLE_ALSA=ON -DENABLE_AIRTUNES=ON -DENABLE_UPNP=ON \
 -DENABLE_INTERNAL_FMT=ON -DENABLE_INTERNAL_RapidJSON=ON \
 -DENABLE_OPENGLES=ON -DENABLE_OPENGL=OFF -DENABLE_X=OFF -DVERBOSE=OFF \
 -DENABLE_NEON=ON -DWITH_CPU=cortex-a73.cortex-a53 -DWITH_ARCH=arm \
 -DENABLE_PULSEAUDIO=OFF -DENABLE_CCACHE=ON -DENABLE_INTERNAL_FFMPEG=ON \
 -DENABLE_APP_AUTONAME=OFF -DENABLE_DVDCSS=OFF \
 -DENABLE_INTERNAL_CROSSGUID=OFF -DENABLE_OPTICAL=OFF \
 -DENABLE_EVENTCLIENTS=ON -DENABLE_CAP=ON -DENABLE_VDPAU=OFF \
 -DAML_INCLUDE_DIR=AML/libamcodec/usr/include/ -DAPP_RENDER_SYSTEM=gles \
 -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc \
 -DCMAKE_CXX_COMPILER=arm-linux-gnueabihf-g++ \
-DENABLE_MDNS=OFF \
-DENABLE_SMBCLIENT=OFF \
 ../xbmc

# other possible changes
# -DENABLE_AIRTUNES=OFF -DENABLE_UPNP=OFF \
# -DENABLE_INTERNAL_RapidJSON=OFF \
```

Outside of `chroot` this was only stable with
`5faa7d4d554640dd6343900e24f43be327cea314` version of `libamcodec`, my build
of `libamcodec` was fairly stable without `multi_vdec` but double seek could
cause hang and reboot.

## IR wakeup

Only works with CoreELEC kernel.  Other kernels (at least 5.11) don't seem to
be able to set IR code to trigger this (this doesn't work in either suspend or
poweroff).

## WoL wakeup

Works with CoreELEC, not exactly sure about others.  Pre-requisite is that
interface goes down on poweroff otherwise it doesn't work.

Make sure you have interface defined in `/etc/network/interfaces` so it can be
shut down.  Not sure if `ifupdown-extra` package was needed too.

What is notable is that `ethtool` may report `d` but it will actually work.
During boot/bootloader on serial console some messages regarding setting
wake-on-lan to 0 or 1 can be seen.  So some driver support might be involved
if it still doesn't work.  It doesn't seem to work with 5.11 though leaving no
way to wake up from suspend.

Related explanations:

- https://askubuntu.com/a/962364
- https://bugs.launchpad.net/ubuntu/+source/ifupdown/+bug/981461

## Console and/or Kodi UI going black/invisible

Boot parameters from CoreELEC work well but had to add `consoleblank=0`.  This
prevents any image beside video going away after ~5 minutes, even though it's
set to 600 by default.

## X11/Wayland

It kind of worked but I have my doubts about performance (Kodi UI) or video
smoothness (software decoding).  Wayland had weird crashes on start with
`enlightenment`.

On Debian `bullseye` 5.10 `meson64` kernel I managed to install Mesa Panfrost
driver with `panfrost.ko` module for Mali G31.  X11 and Wayland works okay.
I'm not sure if `PAN_MESA_DEBUG="bifrost"` was needed:
https://wiki.debian.org/PanfrostLima#Enable_support_for_Mali_.22Bifrost.22_GPUs

If you see `llvmpipe` as driver then it does not work.

Seeing `[drm] Initialized panfrost` is good indication that it will work but
X11 tries to use `fbdev` driver because Debian has config to use it, I tried
this instead (it might work automatically if not forced to `fbdev`):

```
Section "ServerFlags"
        Option  "AutoAddGPU" "off"
EndSection

Section "OutputClass"
        Identifier "Meson"
        MatchDriver "meson"
        Driver "modesetting"
        Option "PrimaryGPU" "true"
EndSection
```

I also tried `libva-v4l2-request` `release-2019.03` with some patch I found to
fix build issues.  `vainfo` fails to init.  I still tried to use v4l2 decoding
via `mpv` and `ffplay` but only got H264 to play and image severely was
broken.  I did not

Related commands:
```
# drm context did not work, had to remove it
LIBVA_DRIVER_NAME=v4l2-request vainfo
LIBVA_DRIVER_NAME=v4l2-request vainfo --display drm --device /dev/dri/renderD128
mpv --drm-connector=HDMI-A-1 --vo=gpu --gpu-context=drm --hwdec=auto --hwdec-codecs=all
ffplay -vcodec h264_v4l2m2m
v4l2-ctl  --list-devices
# only showed something about VP9
v4l2-ctl -d /dev/video0 --all
```

I'm not sure what to make of this.  There may be issues in both drivers and
clients for `v4l2` decoding.

## Kodi systemd service

Just basic service file to autostart Kodi:

```
cat >/etc/systemd/system/kodi-chroot.service <<END
[Unit]
Description=Kodi chroot mounts

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/coreelec/kodi-prepare
ExecStop=/coreelec/kodi-teardown
User=root
StandardOutput=journal

[Install]
WantedBy=kodi.service
END

cat >/etc/systemd/system/kodi.service <<END
[Unit]
Description=Kodi

[Service]
# work around black screen, change to mode that we don't use
# and kodi will change mode back fixing black screen in the process
# ...if black screen appears just exit kodi and let it reset
ExecStartPre=/bin/sh -c 'echo 720p60hz > /sys/class/display/mode'
ExecStart=/kodi
User=root
Restart=always
StandardOutput=journal
RootDirectory=/coreelec
MountAPIVFS=true

[Install]
WantedBy=multi-user.target
END

systemctl daemon-reload
systemctl enable kodi
systemctl enable kodi-chroot
```

## Unresolved issues

### RAID

With old 4.9.113 kernel `dm-raid` 1.9.1 seems to prevent suspend.
Sometimes even assembled array seems to prevent successful suspend but
mounted `raid` seems to break it always.  Newer 4.9 with 1.9.1 did not
have this issue so it might be fixable.

### TV compatibility

On one TV everything works ok and on another it's black screen (and no sound)
until I switch to another HDMI output and back.  Everything works with Intel
graphics, no switching needed.

Couldn't confirm whether Android fixes this as C4 image was unable to output
anything.

### CEC wakeup

I think this only works on CoreELEC or maybe it requires CoreELEC Kodi but it
caused a reboot.  No further testing.

### SATA suspend

During experiments I encontered SATA drives being disconnected after suspend.
I'm not sure which distribution and kernel version it was.  Seems to work at
least with CoreELEC kernel.  No further testing.

### Storage compatibility

My limited experiments on Android suggest that each of these issues is
fixable (and some of them even fixed) in Linux drivers.  Unfortunately some of
these issues are present in Petitboot drivers too.

**UPDATE**: After more testing, USB drive and SD reader issues seem to be
fixed on 5.11 `odroid` kernel from netboot.  Internal reader appears to be
working too although I would need to boot off it to confirm.  It is possible
that installer is running old kernel and had problems with internal reader and
it could be installed via working USB reader.  There is still possibility that
Petitboot wouldn't be able to boot off it via internal reader anyway since it
has the same issues (it uses 4.9 kernel) and needs to be able to read boot
files (having small boot partition at the start work around that).

#### SD card compatibility

I was able to use older 2GB SD fine and for small images like CoreELEC my 8GB
Class 10 cards worked too.  IIRC it didn't matter if it booted 4.9 or 5.10
kernel and netboot installer failed to create partitions on 8GB card.

SanDisk A1 C10 V30 UHS-I U3 32GB cards are recommended by Armbian.

#### USB storage compatibility

For 4.9 kernel a bunch of I/O errors usually, `cant read partition table`,
`sector 0` and so on:

- Error: USB flash drives - 8GB, 16GB
- Error: SDXC card reader
- Works: USB-SATA (at least this one)
- Works: SD card reader (older one) that acts like hub
- Works: microSD card reader (old and tiny)

Common denominator seems to be some kind of faster flash memory (SDXC?) and
faster SD card readers.  It doesn't quite make sense to me.

I managed to boot off USB SD reader with 8GB SD card.  On second attempt for
some reason but it's a valid workaround for internal reader.

I haven't tested as much hardware on 5.10 kernel but broken SD reader worked
(Petitboot can't scan and boot off it though).  And it is possibly fixed in 4.10:

- http://lists.infradead.org/pipermail/linux-amlogic/2016-November/001687.html

- https://marc.info/?l=linux-usb&m=147938307209685&w=2

### Kodi

My build outside of `chroot` plays videos with `5faa7d4d554640dd6343900e24f43be327cea314` and so far stable but on each seek and stopped playback I get this:

```
[ 3033.565114] vdec_release instance ffffff8013396000, total 1
[ 3033.565121] Trying to vfree() bad address (ffffffbf026b95c0)
[ 3033.565129] ------------[ cut here ]------------
[ 3033.565140] WARNING: CPU: 1 PID: 7003 at __vunmap+0xac/0xd0
[ 3033.565142] Modules linked in: lz4hc lz4hc_compress joydev zram mali_kbase(O) meson_ir meson_remote rc_core amvdec_vp9(O) amvdec_vc1(O) amvdec_real(O) amvdec_ports(O) v4l2_commo
n videobuf2_dma_contig videobuf2_memops v4l2_mem2mem videobuf2_v4l2 videobuf2_core amvdec_mpeg4(O) amvdec_mpeg12(O) amvdec_mmpeg4(O) amvdec_mmpeg12(O) amvdec_mmjpeg(O) amvdec_mjpeg
(O) amvdec_mh264(O) amvdec_h265(O) amvdec_h264mvc(O) amvdec_h264(O) amvdec_mavs(O) amvdec_avs(O) amvdec_avs2(O) stream_input(O) decoder_common(O) firmware(O) media_clock(O) amlvide
odri videobuf_res videobuf_core videodev media fuse ip_tables x_tables fbcon bitblit softcursor font

[ 3033.565207] CPU: 1 PID: 7003 Comm: VideoPlayer Tainted: G        W  O    4.9.113 #1
[ 3033.565209] Hardware name: Hardkernel ODROID-HC4 (DT)
[ 3033.565211] task: ffffffc0a304d400 task.stack: ffffffc09d7c4000
[ 3033.565215] PC is at __vunmap+0xac/0xd0
[ 3033.565217] LR is at __vunmap+0xac/0xd0
[ 3033.565220] pc : [<ffffff800920894c>] lr : [<ffffff800920894c>] pstate: 40400149
[ 3033.565221] sp : ffffffc09d7c7bf0
[ 3033.565222] x29: ffffffc09d7c7bf0 x28: ffffffc0c7baa010 
[ 3033.565226] x27: ffffff8009d05000 x26: 0000000000000006 
[ 3033.565229] x25: ffffff800aca1000 x24: 0000000000000000 
[ 3033.565232] x23: 0000000000000000 x22: ffffffbf026b95c0 
[ 3033.565235] x21: 0000000000000000 x20: ffffffc0c806cc10 
[ 3033.565238] x19: ffffffbf026b95c0 x18: 0000000000000000 
[ 3033.565240] x17: 0000007f891d6360 x16: ffffff8009286400 
[ 3033.565243] x15: 0000000000000000 x14: 0000000000000001 
[ 3033.565246] x13: 000000000000104a x12: 0000000000000001 
[ 3033.565249] x11: 0000000000000002 x10: ffffff800aca9910 
[ 3033.565252] x9 : 0000000000000000 x8 : ffffff800aca6000 
[ 3033.565255] x7 : 0000000000000000 x6 : 000000000004cb14 
[ 3033.565257] x5 : ffffffc0cd358b50 x4 : 0000000000000001 
[ 3033.565260] x3 : 0000000000000006 x2 : 0000000000000007 
[ 3033.565263] x1 : ffffffc0a304d400 x0 : 0000000000000030 
[ 3033.565267] 
               SP: 0xffffffc09d7c7b70:
[ 3033.565268] 7b70  026b95c0 ffffffbf 00000000 00000000 00000000 00000000 0aca1000 ffffff80
[ 3033.565278] 7b90  00000006 00000000 09d05000 ffffff80 c7baa010 ffffffc0 9d7c7bf0 ffffffc0
[ 3033.565287] 7bb0  0920894c ffffff80 9d7c7bf0 ffffffc0 0920894c ffffff80 40400149 00000000
[ 3033.565296] 7bd0  00000000 00000000 0909cb60 ffffff80 ffffffff 0000007f 00000000 00000000
[ 3033.565304] 7bf0  9d7c7c20 ffffffc0 09208ac0 ffffff80 026b95c0 ffffffbf c806cc10 ffffffc0
[ 3033.565313] 7c10  9ae57000 00000000 0909cbf8 ffffff80 9d7c7c40 ffffffc0 0909cc08 ffffff80
[ 3033.565322] 7c30  00001000 00000000 ffffffc8 00000000 9d7c7c90 ffffffc0 01e2ba0c ffffff80
[ 3033.565331] 7c50  13396118 ffffff80 c806cc10 ffffffc0 0aad3fa0 ffffff80 026b95c0 ffffffbf
[ 3033.565341] 
               X1: 0xffffffc0a304d380:
[ 3033.565341] d380  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565350] d3a0  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565358] d3c0  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565367] d3e0  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565375] d400  00400008 00000000 ffffffff ffffffff 00000000 00000000 00000000 00000000
[ 3033.565384] d420  9d7c4000 ffffffc0 00000002 00400040 00000000 00000000 00000000 00000000
[ 3033.565392] d440  00000001 00000001 00000009 00000000 000a6d4c 00000001 c88f4600 ffffffc0
[ 3033.565401] d460  00000001 00000001 00000078 00000078 00000078 00000000 09d094b8 ffffff80
[ 3033.565411]
               X5: 0xffffffc0cd358ad0:
[ 3033.565412] 8ad0  0000000f 00000000 00000000 00000000 0000000d 00000000 00000000 00000000
[ 3033.565420] 8af0  00000000 00000000 00000000 00000000 090f7350 ffffff80 00000000 00000000
[ 3033.565429] 8b10  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565437] 8b30  1270befb 00000005 8c78cc2f 00000000 cd35e618 ffffffc0 00000001 00000000
[ 3033.565446] 8b50  00000007 00000000 00000000 00000000 0910f474 ffffff80 00000000 00000000
[ 3033.565454] 8b70  00000000 01400000 00000000 0003cd89 00000000 00000000 00000000 00000000
[ 3033.565463] 8b90  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565471] 8bb0  00000000 00000000 00000000 00000000 f248f248 00000000 00000000 00000000
[ 3033.565484] 
               X20: 0xffffffc0c806cb90:
[ 3033.565484] cb90  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565493] cbb0  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565502] cbd0  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565510] cbf0  00000000 00000000 00000000 00000000 c8bde4c0 ffffffc0 ffffffff 00000000
[ 3033.565518] cc10  0ab73fb0 ffffff80 c8095840 ffffffc0 c8bde4c0 ffffffc0 c806d028 ffffffc0
[ 3033.565527] cc30  c806c818 ffffffc0 0ab73fc0 ffffff80 c8889f80 ffffffc0 0ab73ce0 ffffff80
[ 3033.565536] cc50  c80b5d20 ffffffc0 00000003 00000007 00000000 00000000 00000000 00000000
[ 3033.565544] cc70  00000001 00000000 c806cc78 ffffffc0 c806cc78 ffffffc0 00000000 00000000
[ 3033.565555] 
               X28: 0xffffffc0c7ba9f90:
[ 3033.565556] 9f90  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565564] 9fb0  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565573] 9fd0  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565581] 9ff0  00000000 00000000 00000000 00000000 c8123690 ffffffc0 c8123690 ffffffc0
[ 3033.565590] a010  095d3380 ffffff80 c7baa018 ffffffc0 c7baa018 ffffffc0 406e406e 05ba05ba
[ 3033.565598] a030  00000000 404f404f 00000000 00000000 9ad18000 ffffffc0 00000001 00000000
[ 3033.565607] a050  00780078 00000006 be0a6a70 ffffffc0 00000000 00000000 00000001 00000000
[ 3033.565615] a070  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[ 3033.565625] 
               X29: 0xffffffc09d7c7b70:
[ 3033.565625] 7b70  026b95c0 ffffffbf 00000000 00000000 00000000 00000000 0aca1000 ffffff80
[ 3033.565634] 7b90  00000006 00000000 09d05000 ffffff80 c7baa010 ffffffc0 9d7c7bf0 ffffffc0
[ 3033.565643] 7bb0  0920894c ffffff80 9d7c7bf0 ffffffc0 0920894c ffffff80 40400149 00000000
[ 3033.565652] 7bd0  00000000 00000000 0909cb60 ffffff80 ffffffff 0000007f 00000000 00000000
[ 3033.565660] 7bf0  9d7c7c20 ffffffc0 09208ac0 ffffff80 026b95c0 ffffffbf c806cc10 ffffffc0
[ 3033.565669] 7c10  9ae57000 00000000 0909cbf8 ffffff80 9d7c7c40 ffffffc0 0909cc08 ffffff80
[ 3033.565678] 7c30  00001000 00000000 ffffffc8 00000000 9d7c7c90 ffffffc0 01e2ba0c ffffff80
[ 3033.565686] 7c50  13396118 ffffff80 c806cc10 ffffffc0 0aad3fa0 ffffff80 026b95c0 ffffffbf

[ 3033.565696] ---[ end trace 82984be4259c6ec9 ]---
[ 3033.565698] Call trace:
[ 3033.565702] Exception stack(0xffffffc09d7c7a20 to 0xffffffc09d7c7b50)
[ 3033.565705] 7a20: ffffffbf026b95c0 0000007fffffffff ffffffc09d7c7bf0 ffffff800920894c
[ 3033.565707] 7a40: ffffffc09d7c7a60 ffffff800aac7eb8 0000000100000001 ffffff800aaea358
[ 3033.565710] 7a60: ffffffc09d7c7af0 ffffff800910e90c ffffffc09d7c7b50 ffffff800a1b02a8
[ 3033.565713] 7a80: 0000000000000000 ffffffbf026b95c0 0000000000000000 0000000000000000
[ 3033.565715] 7aa0: ffffff800aca1000 0000000000000006 ffffff8009d05000 ffffffc0c7baa010
[ 3033.565718] 7ac0: 0000000000000030 ffffffc0a304d400 0000000000000007 0000000000000006
[ 3033.565720] 7ae0: 0000000000000001 ffffffc0cd358b50 000000000004cb14 0000000000000000
[ 3033.565723] 7b00: ffffff800aca6000 0000000000000000 ffffff800aca9910 0000000000000002
[ 3033.565725] 7b20: 0000000000000001 000000000000104a 0000000000000001 0000000000000000
[ 3033.565727] 7b40: ffffff8009286400 0000007f891d6360
[ 3033.565732] [<ffffff800920894c>] __vunmap+0xac/0xd0
[ 3033.565736] [<ffffff8009208ac0>] vunmap+0x50/0x60
[ 3033.565740] [<ffffff800909cc08>] __dma_free+0xa8/0xe0
[ 3033.565783] [<ffffff8001e2ba0c>] vdec_input_release+0x1cc/0x240 [decoder_common]
[ 3033.565826] [<ffffff8001e242d4>] vdec_destroy+0x24/0x90 [decoder_common]
[ 3033.565868] [<ffffff8001e28794>] vdec_release+0x1b4/0x510 [decoder_common]
[ 3033.565901] [<ffffff8001e65eac>] video_port_release+0x6c/0xc0 [stream_input]
[ 3033.565931] [<ffffff8001e67b3c>] amstream_release+0x1ec/0x310 [stream_input]
[ 3033.565937] [<ffffff8009233b3c>] __fput+0xac/0x1f4
[ 3033.565940] [<ffffff8009233d04>] ____fput+0x24/0x30
[ 3033.565945] [<ffffff80090ca1f0>] task_work_run+0xdc/0x10c
[ 3033.565950] [<ffffff800908a284>] do_notify_resume+0xb4/0xc0
[ 3033.565953] [<ffffff8009082e1c>] work_pending+0x8/0x10
[ 3033.565953] [<ffffff8009082e1c>] work_pending+0x8/0x10
[ 3033.566402] the vdec            clock off, ref cnt: 0
[ 3033.566409] the parser_top      clock off, ref cnt: 0
[ 3033.566412] the demux           clock off, ref cnt: 0
[ 3033.566571] VID: VD1 off
```

## Other useful files

```
# if display is black switching to another mode and back helps
# this breaks image on my build of kodi and needs to be restarted
# or perform mode switching from the settings which does the same
echo 720p60hz > /sys/class/display/mode
echo 1080p60hz > /sys/class/display/mode
echo 1360x768p60hz > /sys/class/display/mode
# show decoded video
echo 0 > /sys/class/video/disable_video
# video processing
echo "rm default" > /sys/class/vfm/map
echo "add default decoder amvideo" > /sys/class/vfm/map
# minimum requirement for visible video
echo "add default decoder amlvideo amvideo" > /sys/class/vfm/map
# what kodi uses
echo "add default decoder ppmgr amlvideo deinterlace amvideo" > /sys/class/vfm/map
# not sure what is this but kodi uses it 0/1
cat /sys/class/video/blackout_policy
cat /sys/class/video/freerun_mode
```
 
