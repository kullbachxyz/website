---
title: "Flashing Coreboot on the ThinkPad T480"
date: 2026-02-20
draft: false
tags: ["coreboot", "thinkpad", "t480", "firmware", "edk2", "uefi", "linux"]
description: "A practical guide on how I flashed coreboot with the EDK2 UEFI payload on my Lenovo ThinkPad T480, using a Raspberry Pi 4 and an Arch Linux build machine."
---

## Introduction

After successfully corebooting my ThinkPad X220 a while back, I decided it was time to do the same with my ThinkPad T480. The T480 is a significantly more complex target — it has Intel Boot Guard enabled, which means you can't just dump and flash like on the older ThinkPads. Thanks to [Máté Kukri's deguard utility](https://doc.coreboot.org/soc/intel/deguard.html), which exploits a bug in the Intel Management Engine to bypass Boot Guard, this is now possible.

My goal was to run **coreboot with EDK2** as the payload, giving me a proper UEFI environment on open source firmware.

The main resource I followed was the [official coreboot documentation for the Skylake/Kabylake ThinkPads](https://doc.coreboot.org/mainboard/lenovo/skylake.html).

## My Setup

- **ThinkPad T480** (i5-8250U, 16GB RAM)
- **Raspberry Pi 4** for external SPI flashing
- **SOIC-8 clip** for attaching to the flash chip
- **Arch Linux desktop** for compiling coreboot
- A Phillips screwdriver and a spudger

## Update Firmware

Before flashing coreboot, it's important to get the stock firmware to the right versions. The coreboot docs require the EC firmware to be at version **1.22 (N24HT37W)**, and any BIOS version from 1.39 to 1.54 is acceptable for EC UART support.

### Update the Thunderbolt Firmware

The T480 shipped with a buggy Thunderbolt 3 controller firmware that writes logging data to its own flash chip. When that chip fills up, Thunderbolt and fast charging stop working. I updated mine when I first got the device. If you haven't done this yet, grab the update from [Lenovo's support page](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/thinkpad-t-series-laptops/thinkpad-t480-type-20l5-20l6/downloads/ds502613) and apply it while you're still on stock firmware. After this update, Thunderbolt and USB over USB-C work flawlessly, even after flashing coreboot.

### (Optional) Update the BIOS and EC

If your EC isn't at version 1.22 yet, you'll need to update. Download the BIOS update ISO from [Lenovo's support page](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/thinkpad-t-series-laptops/thinkpad-t480-type-20l5-20l6/downloads/ds502355) and extract the El Torito boot image to a USB stick:

```bash
curl -fL -o geteltorito https://raw.githubusercontent.com/rainer042/geteltorito/refs/heads/main/geteltorito.pl
chmod +x geteltorito
./geteltorito -o t480_bios_update.img /path/to/downloaded.iso
sudo dd if=t480_bios_update.img of=/dev/sdX bs=4M conv=fsync status=progress
```

Before booting the updater, go into your BIOS setup and:

- Enable **"Flash BIOS Updating by End Users"**
- Disable **"Secure Rollback Prevention"**
- Enable **Legacy/CSM boot** (the updater boots in BIOS mode)

Boot from the USB (F12 at startup), select option 2, and follow the instructions. Make sure you have a charged battery **and** AC power connected — the updater won't proceed without both.

Note that updating the BIOS also updates the EC firmware. The BIOS itself will be replaced by coreboot, but the EC firmware persists on a separate chip and will remain at the version you flashed.

## Disassembly

Taking the T480 apart is straightforward:

1. Disconnect AC power and remove the external battery.
2. Remove the captive Phillips screws on the bottom cover.
3. Gently pry off the back cover starting from the edges
4. Lift up the clips under the battery compartment.
5. Disconnect the internal battery, as well as the CMOS battery.


{{< figure src="/images/t480-coreboot/back-panel-marked.jpg" caption="The captive Phillips screws on the bottom of the T480" >}}


## Dumping the Stock BIOS

The T480 has a single **16MB SOIC-8 flash chip** (mine was a Winbond W25Q128.V), located roughly in the center of the board near the RAM.

### Wiring the Raspberry Pi 4

I used the SPI pins on the Pi's GPIO header. Here are the pinouts:

#### SOIC-8 Flash Chip Pinout

```bash
                    ______                   
           CS  1 --|      |-- 8  VCC (3.3 V)
         MISO  2 --| BIOS |-- 7  No Connection
No Connection  3 --|      |-- 6  CLK
          GND  4 --|______|-- 5  MOSI
```

#### Raspberry Pi 4 GPIO Pinout

```bash
                        CS   GND
  2                     |     |         40
+-----------------------v-----v-----------+
| x x x x x x x x x x x x x x x x x x x x |
| x x x x x x x x x x x x x x x x x x x x |
+-----------------^-^-^-^-----------------+
  1               | | | |               39
                VCC | | CLK
               MOSI/   \MISO
```

#### Wiring Table

| Chip Pin | Function | RPi GPIO Pin            |
|:--------:|:--------:|:------------------------|
| 1        | CS       | GPIO 8 (CE0) — Pin 24   |
| 2        | MISO     | GPIO 9 (MISO) — Pin 21  |
| 3        | WP       | No connection           |
| 4        | GND      | GND — Pin 25            |
| 5        | MOSI     | GPIO 10 (MOSI) — Pin 19 |
| 6        | CLK      | GPIO 11 (SCLK) — Pin 23 |
| 7        | HOLD     | No connection           |
| 8        | VCC      | 3.3V — Pin 17           |


{{< figure src="/images/t480-coreboot/chip-closeup.jpg" caption="SOIC-8 clip attached to the flash chip" >}}


{{< figure src="/images/t480-coreboot/rpi-connected.jpg" caption="Chip wired to the Raspberry Pi" >}}

Make sure SPI is enabled on the Pi (`sudo raspi-config` → Interface Options → SPI → Enable).

### Reading the Flash

I took three dumps and verified they all matched:

```bash
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r t480_dump1.bin
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r t480_dump2.bin
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r t480_dump3.bin
sha256sum t480_dump*.bin
```

All three checksums matched — good to go. **Keep a backup of this dump somewhere safe!** It's your way back to stock if anything goes wrong.

## Building Coreboot (on Arch Linux)

The rest of the process happened on my Arch Linux desktop.

### Prerequisites

Install the required packages. Notably, you need `gcc-ada` (GNAT) for libgfxinit support (Intel graphics init), `nasm` for EDK2, and `imagemagick`:

```bash
sudo pacman -S gcc-ada nasm imagemagick base-devel git python
```

### Clone the Repos

```bash
mkdir ~/t480 && cd ~/t480
git clone https://review.coreboot.org/coreboot.git
cd coreboot
git submodule update --init --checkout
cd ..
git clone https://review.coreboot.org/deguard.git
```

### Build the Coreboot Toolchain

```bash
cd ~/t480/coreboot
make crossgcc-i386 CPUS=$(nproc)
```

This takes a while. Make sure you have `gcc-ada` installed *before* running this, or libgfxinit won't be available and you won't have display output.

### Extract Blobs from the Stock Dump

```bash
cd util/ifdtool
make
./ifdtool -x -p sklkbl /path/to/t480_dump1.bin
mkdir -p ../../binaries
mv flashregion_0_flashdescriptor.bin ../../binaries/ifd.bin
mv flashregion_3_gbe.bin ../../binaries/gbe.bin
cd ../..
```

You'll see some "Error while writing: Bad address" messages for unused regions (10–15) — these are harmless.

### Prepare the ME with Deguard

The T480 has Intel Boot Guard, which needs to be bypassed using the deguard utility. This requires a "donor" ME image from a Dell system:

```bash
cd ~/t480

# Download the Dell firmware updater
wget "https://web.archive.org/web/20241110222323/https://dl.dell.com/FOLDER04573471M/1/Inspiron_5468_1.3.0.exe"

# Extract it
git clone https://github.com/vuquangtrong/Dell-PFS-BIOS-Assembler.git
python Dell-PFS-BIOS-Assembler/Dell_PFS_Extract.py Inspiron_5468_1.3.0.exe

# Find and copy the ME binary (should be ~2031616 bytes)
find . -size 2031616c
# Output: ./Inspiron_5468_1.3.0.exe_extracted/1 -- 3 Intel Management Engine (Non-VPro) Update v11.6.0.1126.bin

cp "./Inspiron_5468_1.3.0.exe_extracted/1 -- 3 Intel Management Engine (Non-VPro) Update v11.6.0.1126.bin" \
  coreboot/binaries/me_donor.bin

# Verify the hash
sha256sum coreboot/binaries/me_donor.bin
# Expected: 912271bb3ff2cf0e2e27ccfb94337baaca027e6c90b4245f9807a592c8a652e1
```

Now run deguard:

```bash
cd ~/t480/deguard
./finalimage.py \
  --delta data/delta/thinkpad_t480 \
  --version 11.6.0.1126 \
  --pch LP \
  --sku 2M \
  --fake-fpfs data/fpfs/zero \
  --input ../coreboot/binaries/me_donor.bin \
  --output ../coreboot/binaries/me_deguarded.bin
```

### Set the HAP Bit on the IFD

```bash
cd ~/t480/coreboot
util/ifdtool/ifdtool -p sklkbl -M 1 binaries/ifd.bin
mv binaries/ifd.bin binaries/ifd.bin.orig
mv binaries/ifd.bin.new binaries/ifd.bin
```

### Configure and Build

Create `configs/defconfig`:

```bash
CONFIG_VENDOR_LENOVO=y
CONFIG_BOARD_LENOVO_T480=y
CONFIG_IFD_BIN_PATH="binaries/ifd.bin"
CONFIG_ME_BIN_PATH="binaries/me_deguarded.bin"
CONFIG_GBE_BIN_PATH="binaries/gbe.bin"
CONFIG_HAVE_IFD_BIN=y
CONFIG_HAVE_ME_BIN=y
CONFIG_HAVE_GBE_BIN=y
CONFIG_PAYLOAD_EDK2=y
```

Then build:

```bash
make defconfig
make -j$(nproc)
```

### A Note on GCC Compatibility

If you're on a recent version of Arch (or any distro with GCC 14+), you may hit `-Werror=discarded-qualifiers` errors in both vboot's `cbfstool.c` and EDK2's BaseTools. I fixed this with:

```bash
# Fix vboot cbfstool
find . -path "*/host/lib/cbfstool.c" -exec sed -i \
  's/char \*end = strchr(start/const char *end = strchr(start/' {} \;

# Fix EDK2 BaseTools (nuclear option — disables -Werror)
sed -i 's/-Werror/-Wno-error/' \
  payloads/external/edk2/workspace/mrchromebox/BaseTools/Source/C/Makefiles/header.makefile
```

After these patches, the build completed successfully.

### Inject the Original MAC Address

The GBE region in the built ROM will have a generic MAC address. Since I extracted the original GBE from my stock dump, I just injected it back in to preserve my real MAC address:

```bash
cd util/ifdtool && make && cd ../..
util/ifdtool/ifdtool -p sklkbl -i GBE:binaries/gbe.bin build/coreboot.rom
mv build/coreboot.rom build/coreboot.rom.old
mv build/coreboot.rom.new build/coreboot.rom
```

## Flashing

Copy the ROM to the Raspberry Pi and flash:

```bash
scp build/coreboot.rom ph@raspberrypi:~/t480/
```

On the Pi, with the SOIC-8 clip firmly seated on the chip:

```bash
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -w coreboot.rom
```

```bash
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on linux_spi.
Reading old flash chip contents... done.
Erase/write done from 0 to ffffff
Verifying flash... VERIFIED.
```

**VERIFIED.** The most beautiful word in the flashing world.

## The Result

Reassembled the laptop, popped the battery in, hit the power button and...

{{< figure src="/images/t480-coreboot/coreboot-splash.jpg" caption="Coreboot boot screen" >}}

Coreboot with EDK2 UEFI payload, running on my ThinkPad T480.

## Post-Flash Notes

- **Internal microphone**: Does not work under coreboot — this is a known issue. Honestly, the stock mic sounded terrible anyway, so not much of a loss. An external USB or Bluetooth microphone is a better option regardless.
- **Thunderbolt & USB-C**: Works perfectly — both Thunderbolt 3 data and USB over USB-C. Make sure you've updated the Thunderbolt firmware beforehand (see Step 1).
- **Fn keys**: Some Fn+F1–F12 combos aren't handled correctly yet.
- **thinkpad_acpi**: Add `options thinkpad_acpi force_load=1` to a file in `/etc/modprobe.d/` for fan control and thermal monitoring to work.
- **Future updates**: Once coreboot is installed with an unlocked IFD, you can flash internally without the clip:
  ```bash
  sudo flashrom -p internal -w coreboot.rom --ifd -i bios -N
  ```

## Conclusion

Thats it! Coming from the X220, the biggest difference is the deguard step required to bypass Boot Guard - everything else is similar. Having a full UEFI environment through EDK2 makes this feel like a proper modern firmware replacement.

Massive thanks to [Máté Kukri](https://github.com/kukrimate) for the coreboot port and the deguard utility, and to the coreboot community for the excellent documentation.
