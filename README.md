# VinylMaster on Linux with a USB Vinyl Cutter

Run **VinylMaster Pro** (Future Corporation) on Linux through Wine and cut on a USB vinyl cutter, with no Windows install, no VM, and no Windows drivers.

Tested on:
- **OS:** CachyOS (Arch-based), fish shell
- **Software:** VinylMaster Pro (registered), Wine
- **Cutter:** USCutter 24" (USB ID `7447:5419`)

Other cutters that show up as a USB *printer-class* device and accept HPGL (Hewlett-Packard Graphics Language) should work the same way. Pull requests with other tested models are welcome.

---

## How it works

```
VinylMaster (Wine) → Windows printer "USCutter" → CUPS queue → custom backend → /dev/usb/lp0 → cutter
```

Many cutters on cheap USB controller boards don't present as serial ports (`/dev/ttyUSB0`). Linux sees them as USB printers through the `usblp` kernel driver, so they appear as `/dev/usb/lp0`.

That rules out mapping a Wine COM port. Mapping an LPT port also fails: VinylMaster tries to open it as a named printer and throws **error 1801** (invalid printer name). The working path is:

1. A CUPS (Common Unix Printing System) queue that Wine automatically shows to VinylMaster as a normal Windows printer.
2. A tiny custom CUPS backend that copies each job straight to the device. CUPS's built-in `file://` output reported jobs as "completed" but never reached the cutter.

---

## 1. Check how Linux sees your cutter

Plug in the cutter and power it on:

```bash
lsusb
sudo dmesg | tail -20
ls -l /dev/usb/
```

You're looking for a line like this:

```
usblp 3-9:1.0: usblp0: USB Unidirectional printer dev 4 ... vid 0x7447 pid 0x5419
```

and a `/dev/usb/lp0` device. If you get `/dev/ttyUSB0` instead, your cutter is USB-serial and this guide isn't the right path.

Add yourself to the `lp` group (log out and back in afterward):

```bash
sudo usermod -aG lp $USER
```

**Raw test.** Load scrap vinyl first. This cuts a short line:

```bash
printf 'IN;PU0,0;PD1000,0;PU0,0;' > /dev/usb/lp0
```

If the head moves, the hardware side works.

---

## 2. Install Wine

Arch / CachyOS (multilib must be enabled, which it is by default on CachyOS):

```bash
sudo pacman -Syu
sudo pacman -S --needed wine wine-mono wine-gecko winetricks
winetricks corefonts vcrun2019
```

Then run the installer:

```bash
wine ~/Downloads/YourVinylMasterInstaller.exe
```

> **Installed the demo by accident?** Wipe the prefix and start clean, because leftover trial data can interfere with activation:
> ```bash
> wineserver -k
> rm -rf ~/.wine
> ```
> Then redo the `winetricks` line and the installer.

Skip Bottles or other Flatpak Wine managers for this. The sandbox makes device access harder.

---

## 3. Set up CUPS with the custom backend

Install CUPS and the backend script from this repo:

```bash
sudo pacman -S --needed cups
sudo systemctl enable --now cups

sudo install -m 700 -o root -g root uscutter /usr/lib/cups/backend/uscutter
sudo systemctl restart cups
```

The backend **must** be mode `700` and owned by root. That's what tells CUPS to run it as root, so device permissions can't get in the way.

Create the queue:

```bash
sudo lpadmin -p USCutter -E -v uscutter:/ -m raw
sudo lpadmin -p USCutter -o printer-error-policy=retry-job
```

The `retry-job` policy matters. Without it, CUPS pauses the queue whenever the cutter is off or disconnected (for example during a blade change), and every job after that silently piles up.

You'll see a warning that raw queues are deprecated. They still work in current CUPS 2.x.

**Test through CUPS:**

```bash
printf 'IN;PU0,0;PD1000,0;PU0,0;' | lp -d USCutter -o raw
```

---

## 4. Point VinylMaster at it

```bash
wineserver -k
```

Relaunch VinylMaster. In the cutting tool (VinylMaster Cut / the cut spooler):

1. Select your cutter model.
2. Set the port type to **Windows Port**.
3. Choose **Printer** (not Port), then select **USCutter**.

Send a small test shape on scrap.

---

## Troubleshooting

**Cutter stops responding after a power-cycle**

```bash
lpstat -p USCutter        # says "disabled"?
cancel -a USCutter        # clear stuck jobs first so they don't all fire at once
cupsenable USCutter
```

Setting `printer-error-policy=retry-job` (step 3) prevents this.

**Device moved to `/dev/usb/lp1`**

This can happen if another USB printer is connected. Replug the cutter, or edit the `DEVICE=` line in the backend. For a stable name, add a udev rule that creates a `/dev/uscutter` symlink (untested, so adjust the IDs for your cutter):

```bash
echo 'SUBSYSTEM=="usbmisc", KERNEL=="lp*", ATTRS{idVendor}=="7447", ATTRS{idProduct}=="5419", SYMLINK+="uscutter"' | sudo tee /etc/udev/rules.d/99-uscutter.rules
sudo udevadm control --reload-rules
```

Then set `DEVICE=/dev/uscutter` in the backend.

**Error 1801 in VinylMaster**

That's "invalid printer name." You picked the LPT1 port or a printer CUPS doesn't have. Use **Windows Port → Printer → USCutter**.

**Jobs "complete" in CUPS but nothing cuts**

You're probably using a `file:///dev/usb/lp0` queue. Switch to the custom backend in step 3.

**fish shell notes**

- fish has no heredocs (`<< EOF`). Run those snippets inside `bash` or use the files in this repo.
- `export WINEPREFIX=~/.wine` doesn't expand `~` in fish. Use `$HOME/.wine`, or leave `WINEPREFIX` unset to use the default prefix.

**Debug logging**

```bash
sudo cupsctl --debug-logging
sudo tail -40 /var/log/cups/error_log
sudo cupsctl --no-debug-logging    # turn it back off when done
```

---

## Other distros

Package names differ, but the CUPS backend and queue setup are the same everywhere:

- **Debian/Ubuntu:** `sudo apt install wine winetricks cups`. Backends live in `/usr/lib/cups/backend/`.
- **Fedora:** `sudo dnf install wine winetricks cups`. Same backend path.

---

## License

MIT. Do whatever you want with it.
