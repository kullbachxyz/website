---
title: "Corebooting the ThinkPad T480"
date: 2026-02-20
draft: false
description: "A practical guide on how I flashed coreboot with the EDK2 UEFI payload on my Lenovo ThinkPad T480, using a Raspberry Pi 4 and an Arch Linux build machine."
---

After successfully corebooting my ThinkPad X220 a while back, I decided to try the same with my ThinkPad T480. Unlike older ThinkPads, the T480 ships with Intel Boot Guard enabled, which means you can’t simply dump and flash the firmware as you could on earlier models.

The [deguard utility](https://doc.coreboot.org/soc/intel/deguard.html) from coreboot exploits a vulnerability in the Intel Management Engine to bypass Boot Guard. For the payload, I chose [EDK2](https://github.com/tianocore/edk2), giving me a full UEFI implementation.

I mostly followed the official [coreboot documentation](https://doc.coreboot.org/mainboard/lenovo/skylake.html) for the Skylake/Kabylake ThinkPads.

## The Setup

- **ThinkPad T480** (i5-8250U, 16GB RAM)
- **Raspberry Pi** for firmware extraction and flashing
- **SOIC-8 clip** for attaching to the BIOS chip
- **6 F/F jumper wires** for attaching the SOIC clip to the Pi
- **Another Computer** for compiling coreboot
- A Phillips screwdriver and a spudger

## Update Firmware

Before flashing coreboot, you need to get the stock firmware to the right versions. The coreboot docs require the EC firmware to be at version **1.22 (N24HT37W)**, and any BIOS version from **1.39 to 1.54** works.

### Update the Thunderbolt Firmware

The ThinkPad T480 originally shipped with a buggy Thunderbolt 3 controller firmware that repeatedly wrote diagnostic data to its own SPI flash. Over time, this excessive logging could corrupt the controller and permanently break Thunderbolt functionality. I updated mine as soon as I got the machine.

If you haven't done this yet, grab the update from [Lenovo's support page](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/thinkpad-t-series-laptops/thinkpad-t480-type-20l5-20l6/downloads/ds502613) and apply it while you're still on stock firmware.

### Update the BIOS and EC

If your EC isn't at version 1.22 yet, you'll need to update that too. Download the BIOS update ISO from [Lenovo's support page](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/thinkpad-t-series-laptops/thinkpad-t480-type-20l5-20l6/downloads/ds502355) and extract the El Torito boot image to a USB stick:

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

Boot from the USB (F12 at startup), select option 2, and follow the instructions. 

## Disassembly

Taking the T480 apart is straightforward:

1. Disconnect AC power and remove the external battery.
2. Remove the captive Phillips screws on the bottom cover, as shown in the picture below.
3. Disconnect the internal battery, as well as the CMOS battery.
4. Pry off the back cover, making sure to lift up the clips near the battery compartment.


{{< figure src="/images/t480-coreboot/back-panel-marked.jpg" caption="The captive Phillips screws on the bottom of the T480" >}}


## Dumping the Stock BIOS

The T480 has a single **16MB SOIC-8 flash chip** (mine was a Winbond W25Q128.V), located roughly in the center of the board near the RAM.

{{< figure src="/images/t480-coreboot/bios-chip-location.jpg" caption="BIOS chip location" >}}


### Wiring the Raspberry Pi

I used the SPI pins on the Pi's GPIO header. Here are the pinouts:

#### BIOS Flash Chip Pinout

```bash
                    ______                   
           CS  1 --|o     |-- 8  VCC (3.3 V)
         MISO  2 --|      |-- 7  No Connection
No Connection  3 --|      |-- 6  CLK
          GND  4 --|______|-- 5  MOSI
```

#### Raspberry Pi GPIO Pinout

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
|:--------:|:--------:|:-----------------------:|
| 1        | CS       | GPIO 8 (CE0) - Pin 24   |
| 2        | MISO     | GPIO 9 (MISO) - Pin 21  |
| 3        | WP       | No connection           |
| 4        | GND      | GND - Pin 25            |
| 5        | MOSI     | GPIO 10 (MOSI) - Pin 19 |
| 6        | CLK      | GPIO 11 (SCLK) - Pin 23 |
| 7        | HOLD     | No connection           |
| 8        | VCC      | 3.3V - Pin 17           |


{{< figure src="/images/t480-coreboot/chip-closeup.jpg" caption="SOIC-8 clip attached to the flash chip" >}}


{{< figure src="/images/t480-coreboot/rpi-connected.jpg" caption="Chip wired to the Raspberry Pi" >}}

Make sure SPI is enabled on the Pi (`sudo raspi-config` -> Interface Options -> SPI -> Enable).

### Installing flashrom

Install the required build dependencies on the Pi first:
```bash
sudo apt install git libpci-dev libusb-1.0 libusb-dev
```

Clone the flashrom repo, compile and install the binary:
```bash
git clone https://github.com/flashrom/flashrom.git
cd flashrom
make
sudo make install
```

### Reading the Flash

I took three dumps and verified they all matched:

```bash
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r t480_dump1.bin
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r t480_dump2.bin
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r t480_dump3.bin
sha256sum t480_dump*.bin
```

## Building Coreboot

The rest of the process happened on my desktop computer.

### Prerequisites

Install the required packages. You'll need `gcc-ada` (GNAT) for libgfxinit support (Intel graphics init), `nasm` for EDK2, and `imagemagick`:

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

This takes a while.

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

You'll probably see some "Error while writing: Bad address" messages for unused regions (10–15) - these are harmless.

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

The HAP bit is a special flag inside the Intel firmware configuration that tells the Intel Management Engine to disable most of its functionality after early initialization. To set this in the IFD:

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

If you're on a distro with GCC 14+, you may hit `-Werror=discarded-qualifiers` errors in both vboot's `cbfstool.c` and EDK2's BaseTools. I fixed this with:

```bash
# Fix vboot cbfstool
find . -path "*/host/lib/cbfstool.c" -exec sed -i \
  's/char \*end = strchr(start/const char *end = strchr(start/' {} \;

# Fix EDK2 BaseTools (nuclear option - disables -Werror)
sed -i 's/-Werror/-Wno-error/' \
  payloads/external/edk2/workspace/mrchromebox/BaseTools/Source/C/Makefiles/header.makefile
```

After these patches, the build completed successfully.

### Inject the Original MAC Address

Since I extracted the original GBE from my stock dump, I injected it back in to preserve my real MAC address:

```bash
cd util/ifdtool && make && cd ../..
util/ifdtool/ifdtool -p sklkbl -i GBE:binaries/gbe.bin build/coreboot.rom
mv build/coreboot.rom build/coreboot.rom.old
mv build/coreboot.rom.new build/coreboot.rom
```

## Flashing

Copy the ROM to the Raspberry Pi and flash:

```bash
scp build/coreboot.rom pi@raspberrypi:~/t480/
```

Flash the image, making sure the clip still has a good connection:

```bash
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -w coreboot.rom
```

Expected Output:
```bash
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on linux_spi.
Reading old flash chip contents... done.
Erase/write done from 0 to ffffff
Verifying flash... VERIFIED.
```

## The Result

Reassembled the laptop, popped the battery in, hit the power button and... SUCCESS!!!

{{< figure src="/images/t480-coreboot/coreboot-splash.jpg" caption="Coreboot with EDK2 UEFI payload, running on my ThinkPad T480" >}}

## Post-Flash Notes

- **Internal microphone**: Does not work under coreboot - this is a known issue.
- **Thunderbolt & USB-C**: As of March 2025, Thunderbolt data transfer is not supported upstream by [coreboot](https://review.coreboot.org/c/coreboot/+/83274). Charging, USB-C data transfer and video output through the Thunderbolt port work fine tough.
- **Fn keys**: Some Fn+F1–F12 combos aren't handled correctly yet.
- **thinkpad_acpi**: Add `options thinkpad_acpi force_load=1` to a file in `/etc/modprobe.d/` for fan control and thermal monitoring to work reliably.
- **Future updates**: Once coreboot is installed, you can flash internally without the clip:
  ```bash
  sudo flashrom -p internal -w coreboot.rom --ifd -i bios -N
  ```

## Conclusion

Thats it! Coming from the X220, the biggest difference is the deguard step required to bypass Boot Guard - everything else is similar.

Massive thanks to [Máté Kukri](https://github.com/kukrimate) for the coreboot port and the deguard utility, and to the coreboot community for the excellent documentation.
