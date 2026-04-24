**Xbox One Wireless Adapter (Gen 1) Not Working on Linux — Fix Guide**

**Symptoms**
- Dongle recognized by `lsusb` (`045e:02e6`) but light never turns on on Linux, works fine on Windows same machine
- `xone-dongle` driver is loaded but nothing happens
- `dmesg` shows the device enumerating but no firmware upload, sometimes shows `Device is not authorized for usage`
- No `1-X:1.0` interface ever appears in `/sys/bus/usb/devices/`

---

**Root Cause**

A udev rule is explicitly blocking USB authorization of the dongle, preventing the kernel from setting its configuration and creating the interface that `xone-dongle` needs to bind to. No driver ever runs, no firmware gets uploaded, no light.

Check for it:
```
grep -r '045e\|02e6\|authorized' /etc/udev/rules.d/ /lib/udev/rules.d/ 2>/dev/null
```

If you see something like:
```
/etc/udev/rules.d/some-rule.rules:SUBSYSTEM=="usb", ATTR{idVendor}=="045e", ATTR{idProduct}=="02e6", ATTR{authorized}="0"
```
That file is your culprit. It sets `authorized=0` every time the dongle is plugged in. This rule is sometimes added intentionally (e.g. to prevent the dongle's 2.4GHz radio from interfering with other wireless devices) or left behind by a previous script or package.

---

**Fix**

**1.** Remove the blocking rule:
```
sudo rm /etc/udev/rules.d/<the-offending-file>.rules
```

**2.** Reload udev:
```
sudo udevadm control --reload-rules
```

**3.** Unplug and replug the dongle — it should light up and be ready to pair.

---

**How to diagnose this from scratch**

Check the driver is loaded:
```
lsmod | grep xone
```
Should show `xone_dongle`, `xone_gip`, `xone_gip_gamepad`.

Check the dongle is seen:
```
lsusb | grep 045e
```

Check dmesg for the authorization error:
```
sudo dmesg | grep -i '02e6\|xone\|authorized' | tail -20
```

Check for blocking udev rules:
```
grep -r 'authorized' /etc/udev/rules.d/ /lib/udev/rules.d/ 2>/dev/null
grep -r '02e6' /etc/udev/rules.d/ /lib/udev/rules.d/ 2>/dev/null
```

Confirm by manually authorizing (temporary test — find your device path with `lsusb -t`):
```
echo 1 | sudo tee /sys/bus/usb/devices/<device-path>/authorized
```
If the dongle lights up after this, the blocking udev rule is confirmed.

---

**xone driver setup (if you haven't done this yet)**

Fedora:
```
sudo dnf install dkms kernel-devel cabextract
git clone https://github.com/medusalix/xone
cd xone
sudo ./install.sh --release
sudo xone-get-firmware.sh --skip-disclaimer
```
For Debian/Ubuntu replace `dnf install dkms kernel-devel` with `apt install dkms linux-headers-$(uname -r)`.

Make sure `/lib/firmware/xow_dongle.bin` exists after the firmware script.

---

**Confirmed on**
> Fedora 43 — Kernel 6.19.12 — Intel Tiger Lake-H xHCI — Xbox One Wireless Adapter Gen 1 (`045e:02e6`) — xone driver
