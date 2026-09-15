# Linux on a 2017 MacBook Pro 13" with Touch Bar (MacBookPro14,2 / A1706)

Notes from getting Arch (EndeavourOS) fully working on a T1 MacBook Pro. Everything here
was tested on the actual machine.

This is about the T1 chip in the 2016 and 2017 Touch Bar models. It is not about the T2 in
2018 and later machines. Almost all Linux-on-Mac documentation you'll find targets T2, and
a lot of it will send you in the wrong direction. If you have a 2016 model (MacBookPro13,2)
things should be close enough to follow along.

## What I'm running

MacBookPro14,2, Kaby Lake i5, Iris Plus 650, 8 GB soldered RAM. EndeavourOS with KDE on
Wayland, `linux-lts-t2` 6.18.x from the arch-mact2 repo, dracut rather than mkinitcpio,
GRUB chainloaded from rEFInd. WiFi is a Broadcom BCM43602 (14e4:43ba), audio is a Cirrus
CS8409 with a CS42L83, subsystem id 0x106b3600.

The arch-mact2 repo is the t2linux community Arch repo. It's built for T2 Macs. It works
fine as a base for a T1 but some of its assumptions don't hold, which comes up below.

## Status

Working: Touch Bar display and keys, keyboard backlight, keyboard and trackpad, WiFi
including 5 GHz, internal speakers, headphone jack with auto-switching, webcam, Bluetooth,
suspend and resume, hardware video decode, fans, battery, graphics.

Not working: hibernation, Touch ID.

## Four things that will waste your time

I lost hours to each of these, so they go first.

After you edit anything in `/etc/modprobe.d/`, rebuild the initramfs. On dracut that's
`sudo dracut-rebuild`, on mkinitcpio it's `sudo mkinitcpio -P`. The modules load out of the
initramfs, so until you rebuild, your config file is completely ignored. The confusing part
is that `cat /sys/module/<mod>/parameters/<x>` will happily show you the old value while the
file on disk says something else, so it looks like the parameter isn't working rather than
not being applied. I tested five different `fnmode` values this way and got identical
results from all of them, because none of them were ever loaded.

Don't run `modprobe -r apple-touchbar`. Its remove path leaves a dangling reference, the
module wedges at use count -1 (lsmod will literally print `-1`), anything that then touches
input devices hangs, and only a reboot clears it.

Don't play audio to `hw:0,0` directly. It bypasses volume control entirely. I did this once
at what I thought was 40% volume and it was loud enough to be genuinely unpleasant.

Your macOS/EFI partition holds the T1 firmware. `lsusb` needs to show
`05ac:8600 Apple, Inc. iBridge`. If you see `05ac:1281 ... [Recovery Mode]` instead, the T1
has no firmware loaded and the Touch Bar is impossible until you reinstall macOS, boot it
all the way to the desktop once, and then reinstall Linux without reformatting the ESP.

## Touch Bar

The T1 works completely differently from the T2. On a T2 the Touch Bar is a dumb display and
the host renders every pixel, which is what `tiny-dfr` and `appletbdrm` exist for. On a T1
the firmware draws the strip itself and sends key events back. Two consequences: anything
mentioning `tiny-dfr`, `appletbdrm`, `hid-appletb-kbd` or `hid-appletb-bl` is T2-only and
won't help you, and you can't put custom graphics on a T1 Touch Bar. You get Apple's layouts
and that's it.

### The driver

Mainline has nothing for the T1 iBridge, so you need an out-of-tree driver. For kernel 6.x
use [parport0/mbp-t1-touchbar-driver](https://github.com/parport0/mbp-t1-touchbar-driver).
For 7.x use [AJ-dev-i60/t1-touchbar](https://github.com/AJ-dev-i60/t1-touchbar), which is a
port of the same thing.

Be aware the module names differ between them. parport0 calls them `apple-ibridge` and
`apple_ib_tb`. The AJ fork calls them `apple-ibridge` and `apple_touchbar`. Older guides all
say `apple_ib_tb`, so translate as you read.

DKMS install of the AJ fork on Arch:

```bash
sudo pacman -S --needed dkms git linux-lts-t2-headers
git clone https://github.com/AJ-dev-i60/t1-touchbar
sudo cp -r t1-touchbar/apple-ib-drv /usr/src/apple-ib-drv-0.1
sudo dkms install apple-ib-drv/0.1
```

Version 0.1, matching its `dkms.conf`. Skip the repo's `install.sh` on Arch, it's apt-only
and will make a mess. Don't delete the cloned directory afterwards, DKMS symlinks into it.

### hid-sensor-hub steals the device

This is the one that isn't documented anywhere and it's why a lot of people give up.

Since kernel 6.3, commit e04955db6a7c ("HID: Recognize sensors with application collections")
reclassifies the iBridge's second HID interface into `HID_GROUP_SENSOR_HUB`. Generic
`hid-sensor-hub` matches it and claims it before `apple-ibridge` ever gets a probe. The
result is that the driver binds, dmesg is completely clean, and the Touch Bar stays black.
No errors, nothing to search for.

Check it:

```bash
readlink /sys/bus/hid/devices/0003:05AC:8600.000*/driver
```

If either of those points at `hid-sensor-hub`, that's the problem.

```bash
sudo tee /etc/modprobe.d/blacklist-hid-sensor-ibridge.conf <<'EOF'
blacklist hid_sensor_hub
blacklist hid_sensor_als
blacklist hid_sensor_trigger
blacklist hid_sensor_iio_common
EOF
sudo dracut-rebuild
```

When it's right, `lsusb -t` shows four virtual sub-devices (1d6b:0301 twice, 1d6b:0302
twice). If you only see two, something is still holding interface 2.

### Freeze guard

Loading the driver hard-locks some T1 machines through an unconditional `ASOC.SOCW(1)` ACPI
power-on call at probe time. The AJ fork adds a parameter to skip it:

```bash
echo 'options apple_ibridge skip_acpi_power=1' | sudo tee /etc/modprobe.d/apple-ibridge.conf
```

If a boot ever hangs, hit `e` at the GRUB menu and append
`modprobe.blacklist=apple_ibridge,apple_touchbar`.

### Module parameters

```bash
echo 'options apple_touchbar fnmode=2 idle_timeout=-1 dim_timeout=-1' \
  | sudo tee /etc/modprobe.d/apple-touchbar.conf
sudo dracut-rebuild
```

`idle_timeout=0` does not mean "never blank". It means "turn touch bar display off, and
input does not turn it on again". Setting it to 0 kills key translation and presents as a
driver bug. The value you want is -1, which is "on, does not turn off automatically". Same
idea for `dim_timeout`: -1 never dims, -2 derives from the idle timeout and is the default.

fnmode is 0 for function keys only, 1 for Fn switching special to function keys (the
default), 2 for the inverse of that, 3 for special keys only, 4 for escape only.

Autoload both modules:

```bash
printf 'apple-ibridge\napple-touchbar\n' | sudo tee /etc/modules-load.d/apple-t1-touchbar.conf
```

### Making it survive suspend

The iBridge doesn't resume cleanly. You'll see
`usb 1-3: PM: failed to resume async: error -107` in dmesg.

The fix is to unbind the USB device before suspend and rebind after. Unbinding only on
resume doesn't work: the driver's input devices from before the suspend are still
registered, so you get `tb: Duplicate connect to tbkbd input device` and
`input: failed to attach handler appletb to device inputN, error: -17`, and the bar lights
up but no keys do anything. That one took me a while, because visually it looks like it's
working.

Find your port, usually 1-3:

```bash
for d in /sys/bus/usb/devices/*/; do
  [ -f "$d/idVendor" ] || continue
  printf '%s %s:%s\n' "$(basename $d)" "$(cat $d/idVendor)" "$(cat $d/idProduct)"
done | grep 05ac:8600
```

```ini
# /etc/systemd/system/touchbar-resume.service
[Unit]
Description=Unbind iBridge before suspend, rebind after resume
Before=sleep.target
StopWhenUnneeded=yes

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c 'echo 1-3 > /sys/bus/usb/drivers/usb/unbind'
ExecStop=/bin/sh -c 'sleep 2; echo 1-3 > /sys/bus/usb/drivers/usb/bind'

[Install]
WantedBy=sleep.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable touchbar-resume.service
```

I didn't need this, but if usbhid grabs the device on your machine, this is reported to work
instead of the unbind hook:

```
options usbhid ignore_special_drivers=1 quirks=0x05ac:0x8600:0x4
```

## WiFi

`brcmfmac` binds on its own and the firmware ships in linux-firmware, so you'll get 2.4 GHz
working immediately. 5 GHz needs an NVRAM file, and which one you use matters enormously.

The version that circulates most widely, with `boardtype=0x61b`, `boardrev=0x1421`,
`ccode=NL` and `regrev=245`, does not work on this board. 5 GHz networks show up in scans
and association fails every single time. I spent a long time convinced this was a driver
problem.

What works is the Apple Boot Camp lineage: `boardtype=0x073e`, `boardrev=0x1101`, and
importantly `ccode=0` with `regrev=1`. The chip's country detection is broken and ccode=0
sidesteps it rather than feeding it a country string it mishandles.

```bash
cd /tmp
curl -fLO https://gist.githubusercontent.com/cristianmiranda/ba9d64b4324f0803d9422d765de62252/raw/brcmfmac43602-pcie.txt
sed -i "s/^macaddr=.*/macaddr=<your MAC>/" brcmfmac43602-pcie.txt
sudo cp brcmfmac43602-pcie.txt /lib/firmware/brcm/
sudo systemctl stop NetworkManager
sudo modprobe -r brcmfmac_wcc brcmfmac
sudo modprobe brcmfmac
sudo systemctl start NetworkManager
```

Get your MAC from `ip -br link`, and make sure it's the hardware one. If the second hex
digit is 2, 6, A or E it's a NetworkManager randomised address and baking that into firmware
config is not what you want. The real one on this card is in Broadcom's 00:90:4c range.

This file isn't shipped by linux-firmware, so keep a backup somewhere outside `/lib`.

### Use wpa_supplicant, not iwd

Several guides recommend iwd for this chip. In my testing iwd could not associate at all.
NetworkManager's wpa_supplicant backend works:

```bash
sudo pacman -S --needed wpa_supplicant
sudo tee /etc/NetworkManager/conf.d/wifi_backend.conf <<'EOF'
[device]
wifi.backend=wpa_supplicant
EOF
sudo systemctl unmask wpa_supplicant
sudo systemctl disable --now iwd
sudo systemctl enable --now wpa_supplicant
sudo systemctl restart NetworkManager
```

The t2 ISO ships with wpa_supplicant.service masked, hence the unmask.

### Regulatory domain

If your AP is on a DFS channel (52 through 144, and plenty of consumer routers pick these) a
regdomain reset to WORLD will break association until something sets it back. Mine did this
after resumes. Pin it at module level:

```bash
sudo pacman -S iw wireless-regdb
echo 'options cfg80211 ieee80211_regdom=NL' | sudo tee /etc/modprobe.d/cfg80211-regdom.conf
sudo dracut-rebuild
```

Substitute your own country. Also uncomment yours in `/etc/conf.d/wireless-regdom`. Arch
doesn't use `/etc/default/crda`, which several guides will tell you to edit.

### Debugging notes

`status_code=16` from brcmfmac tells you nothing. `brcmf_bss_connect_done()` reports every
single failure as `WLAN_STATUS_AUTH_TIMEOUT (16)` no matter what actually went wrong, and
NetworkManager turns that into "no secrets" or a wrong password prompt. See kernel bug
193121. I was shown the wrong-password dialog dozens of times with a completely correct
passphrase.

An all-zero BSSID in `CTRL-EVENT-ASSOC-REJECT` means the station never associated, which
means it never reached the 4-way handshake, which means it definitively isn't a passphrase
problem.

`no clm_blob available (err=-2)` is harmless. For linux-firmware's brcmfmac builds the
regulatory data is embedded in the .bin file; the separate blob only exists for
Cypress-sourced firmware. I nearly reinstalled macOS to extract a blob I didn't need.

Avoid `brcmfmac.feature_disable=0x82000`. It's recommended in several places for this chip
and it disables the entire 5 GHz band. Symptom is that association starts working and every
5 GHz network vanishes from scans.

## Audio

Mainline's `snd_hda_codec_cs8409` knows about the codec but not the amplifier behind it, so
no speaker output path is ever created. What you get is audio that "plays" with working
volume controls and no sound. The tell is `amixer -c0 scontrols` showing only PCM, Mic,
IEC958 and Internal Mic, with no Master or Speaker anywhere.

Patched driver first:

```bash
git clone https://github.com/davidjo/snd_hda_macbookpro
cd snd_hda_macbookpro
sudo ./dkms.sh
```

`./dkms.sh -r` uninstalls and restores the stock module. Recent versions have DKMS support
with AUTOINSTALL, so it rebuilds on kernel updates. As with the Touch Bar driver, leave the
clone where it is.

That alone still gave me silence, because the hardware only accepts 44.1 kHz with S24_LE or
S32_LE, and PipeWire offers 48 kHz S16_LE, which gets rejected. So:

```
# ~/.config/wireplumber/wireplumber.conf.d/51-macbook-audio.conf
monitor.alsa.rules = [
  {
    matches = [ { node.name = "~alsa_output.*" } ]
    actions = {
      update-props = {
        audio.format = "S32LE"
        audio.rate = 44100
        audio.channels = 2
        api.alsa.period-size = 1024
      }
    }
  }
]
```

```bash
systemctl --user restart wireplumber pipewire
```

`audio.channels = 2` is important. The hardware is four drivers, left and right tweeters
plus left and right woofers. Set it to 4 and PipeWire treats it as surround and sends stereo
to the front pair only, so sound comes out of one side. At 2 the driver handles duplicating
across all four itself.

Restarting those services resets the sink to muted, so unmute afterwards or you'll think it
didn't work.

## Suspend

Thunderbolt is the problem. The controllers come back from D3cold inaccessible, you get
`tb_cfg_read: -108` and `xhci_hcd ... pci_pm_resume returns -19`, and the kernel spends
about a minute timing out on them and takes the USB controllers down with it.

```bash
echo 'blacklist thunderbolt' | sudo tee /etc/modprobe.d/blacklist-thunderbolt.conf
sudo dracut-rebuild
```

USB and USB-C keep working without the module since they're on xHCI, and external displays
over DisplayPort alt-mode are fine. What you lose is Thunderbolt docks and eGPUs.

You also need s2idle. On `deep`, neither WiFi nor the Touch Bar survive a resume for me.

```
nowatchdog nvme_load=YES intel_iommu=on iommu=pt pcie_ports=compat loglevel=3 mem_sleep_default=s2idle
```

The cost of s2idle is real: roughly 10 to 12 percent of battery per hour while suspended,
and the laptop stays warm in a bag. `deep` fixes that and breaks resume. I haven't found a
configuration that gives both, and I tried fairly hard.

Waking needs the power/TouchID button. The SPI keyboard and trackpad aren't wake sources.

### What didn't help

There's a [well-known guide](https://takachin.github.io/mbp2017-linux-note/en/suspend-resume.html)
for the 2017 13" recommending a system-sleep hook that sets `d3cold_allowed=0` on every PCI
device. On the non-Touch-Bar model it apparently works well. On mine it made things worse:
with the hook installed, WiFi and the Touch Bar stopped surviving even s2idle resumes. I
confirmed from the hook's own log that it ran and set 32 devices, watched resume break, and
removing it fixed resume immediately. Worth knowing before you copy it.

### Hibernation

Doesn't work. Swap file in place, `resume=` and `resume_offset=` correct and visible in
`/proc/cmdline`, `systemd-hibernate-resume` present in the initramfs and running. What
happens is that `PM: hibernation: hibernation entry` gets logged, and then the machine powers
off before any image is written. No "Creating image", nothing. Next boot reports
`PM: Image not found (code -22)`, correctly, because there is no image. I don't know why.

## Trackpad

`applespi` handles keyboard and trackpad and it's in mainline, so both work out of the box.
What doesn't work is palm detection. libinput has none on this device by default, because
applespi exposes `ABS_MT_TOUCH_MAJOR` and `MINOR` but no `ABS_MT_PRESSURE`, and size-based
palm detection only kicks in when a quirks file supplies thresholds. There's no setting for
this in KDE, it just silently isn't running.

```
# /etc/libinput/local-overrides.quirks
[Apple SPI Touchpad]
MatchName=Apple SPI Touchpad
MatchUdevType=touchpad
AttrTouchSizeRange=350:175
AttrPalmSizeThreshold=1200
```

Log out and back in. To tune it, run `sudo evtest /dev/input/event5` and watch
`ABS_MT_TOUCH_MAJOR` while touching the pad normally and then resting a palm on it. Set the
first number just below your normal touch. The second is hysteresis and wants to be about
half the first.

It won't match macOS. macOS reads the actual pressure sensors, libinput is guessing from
contact area. Mine is usable now but still catches a palm occasionally.

## Smaller things

Bluetooth works, `bluetooth.service` just ships disabled. `sudo systemctl enable --now bluetooth`
and you're done. The missing `BCM20703A2-*.hcd` patch file produces alarming log lines about
firmware not being found and doesn't actually stop anything working.

The webcam is part of the iBridge and works through uvcvideo once apple-ibridge is loaded.
If you want it off, `blacklist uvcvideo` does it without breaking the Touch Bar, despite both
being the same physical device.

Hardware video decode needs `libva-utils intel-media-driver`. Firefox may still blocklist it
with `FEATURE_FAILURE_VIDEO_DECODING_TEST_FAILED`, which is fixed by putting
`LIBVA_DRIVER_NAME=iHD` in `/etc/environment`. Check `about:support` afterwards.

Install `intel-ucode` and regenerate GRUB. Check with `sudo dmesg | grep -i microcode`.

zram is worth setting up on 8 GB:

```bash
sudo pacman -S --needed zram-generator
sudo tee /etc/systemd/zram-generator.conf <<'EOF'
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
EOF
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service
```

Fans work through applesmc with no configuration at all. Idle sits around 1250 and 1350 RPM
and they ramp properly under load.

## Booting straight into Linux

By default you hold Option at every boot, which gets old. If OpenCore is installed and
defaulting to macOS, the relevant thing is its `HideAuxiliary=true` setting, which hides
non-macOS entries from its picker. I tried fixing it there and couldn't get OpenCore to
chainload GRUB successfully.

rEFInd turned out to be much simpler, and it's designed for this:

```bash
sudo efibootmgr -v
sudo efibootmgr -o 0002,0001,0080,0000
```

Use whatever number rEFInd actually is on your machine. Then:

```
# /boot/efi/EFI/refind/refind.conf
timeout 5
default_selection "endeavouros"
```

`default_selection` matches a substring of the title as rEFInd displays it, so check what
yours says in the menu before setting it. Mine was
`Boot EFI\endeavouros\grubx64.efi from EFI`.

One thing that confused me: if GRUB then shows a coloured rectangle and no menu text,
comment out `GRUB_BACKGROUND` in `/etc/default/grub` and regenerate. The splash image
renders and the menu text doesn't draw on top of it, so it looks like GRUB is skipping the
menu entirely.

## Kernel choice

`linux-t2` tracks mainline aggressively and was on 7.x when I set this up. The out-of-tree
drivers are tested well behind that. davidjo targets 6.8, and the Touch Bar driver needed
explicit patches just to build against 7.x. Everything did work on 7.2.4, but it felt like
borrowed time.

`linux-lts-t2` (6.18.x) is the more sensible default. Both DKMS modules build against it
cleanly and everything in this document works. Install it alongside, verify, then make it the
GRUB default and keep the other as a fallback entry:

```bash
sudo pacman -S linux-lts-t2 linux-lts-t2-headers
sudo dkms autoinstall -k <new-kernel-version>
dkms status
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Get in the habit of checking `dkms status` before rebooting after a kernel update. Both
modules should say `installed` for the new version. A silent build failure means no Touch Bar
or no audio next boot, and you won't find out until you're already there.

## Every file this touches

```
/etc/modprobe.d/blacklist-hid-sensor-ibridge.conf   keeps hid-sensor-hub off the iBridge
/etc/modprobe.d/apple-ibridge.conf                  skip_acpi_power=1
/etc/modprobe.d/apple-touchbar.conf                 fnmode and timeouts
/etc/modprobe.d/blacklist-thunderbolt.conf          required for usable suspend
/etc/modprobe.d/blacklist-webcam.conf               optional
/etc/modprobe.d/cfg80211-regdom.conf                pin the regulatory domain
/etc/modules-load.d/apple-t1-touchbar.conf          autoload the touchbar modules
/etc/systemd/system/touchbar-resume.service         unbind before sleep, rebind after
/etc/NetworkManager/conf.d/wifi_backend.conf        wpa_supplicant instead of iwd
/etc/libinput/local-overrides.quirks                trackpad palm detection
/lib/firmware/brcm/brcmfmac43602-pcie.txt           NVRAM, back this up
/etc/default/grub                                   mem_sleep_default=s2idle
~/.config/wireplumber/wireplumber.conf.d/51-macbook-audio.conf
```

DKMS sources, don't move or delete these:

```
~/t1-touchbar            apple-ib-drv/0.1
~/snd_hda_macbookpro     snd_hda_macbookpro/0.1
```

## Credits

Essentially none of the underlying work is mine, this is an integration guide.

Ronald Tschalär wrote the original applespi and iBridge drivers.
[parport0](https://github.com/parport0/mbp-t1-touchbar-driver) maintains the T1 Touch Bar
driver and [AJ-dev-i60](https://github.com/AJ-dev-i60/t1-touchbar) ported it to kernel 7.
[davidjo](https://github.com/davidjo/snd_hda_macbookpro) reverse-engineered the CS8409 and
CS42L83 audio path from macOS. [t2linux](https://wiki.t2linux.org/) maintain the Arch repo
and kernel builds that all of this sits on.
[xtocdra/macbookpro13-2](https://github.com/xtocdra/macbookpro13-2) is the other T1 write-up
and was useful, and [takachin](https://takachin.github.io/mbp2017-linux-note/) did the 2017
MBP suspend investigation.


Written up with AI assistance, tested myself on my own macbook.
