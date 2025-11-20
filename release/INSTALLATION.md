# LibreSDR v0.37 Timestamps - Installation Guide

## Installation Methods

### Method 1: DFU Update (Recommended for existing LibreSDR)
```bash
# Flash the DFU file
sudo dfu-util -a firmware.dfu -D libresdr-v0.37-timestamps-libre.dfu

# Reboot LibreSDR
```

### Method 2: Mass Storage Update

1. Connect LibreSDR via USB
2. Copy `libresdr-v0.37-timestamps-libre.frm` to the LibreSDR mass storage device
3. Eject and reboot

### Method 3: SD Card Boot (For recovery or fresh install)

1. Format SD card as FAT32
2. Extract files from the zip to find `sd-card-image/` folder
3. Copy all files from `sd-card-image/` to SD card root
4. Insert SD card into LibreSDR
5. Power on - it will boot from SD card

## Building from Source

If you want to rebuild from source:
```bash
git clone --recurse-submodules https://github.com/pumatrax/libresdr-fw-timestamps.git
cd libresdr-fw-timestamps

export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2021.2/settings64.sh
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=$PATH:/opt/Xilinx/Vitis/2021.2/gnu/aarch32/lin/gcc-arm-linux-gnueabi/bin
export TARGET=libre

make
make sdimg
```

Build artifacts in `build/`:
- `libre.dfu` - DFU image for flashing
- `libre.frm` - Firmware update file

SD card boot files in `build_sdimg/`:
- `BOOT.bin` - Boot image
- `uImage` - Kernel
- `devicetree.dtb` - Device tree
- `uramdisk.image.gz` - Root filesystem
- All other boot files

## Verification

After flashing, SSH to LibreSDR:
```bash
ssh root@192.168.1.1
# Check version
cat /etc/os-release
# Should show: v0.37-timestamps-libre

# Verify sdr_ip_gadget is running
ps aux | grep sdr_ip_gadget
# Should see: /usr/sbin/sdr_ip_gadget

# Check IIO buffers
cat /sys/bus/iio/devices/iio:device3/buffer/length
# Should show: 1922 (for timestamp_every=1920)
```

## What's Included in the ZIP

- DFU files (.dfu, .frm) - Ready to flash
- sd-card-image/ folder - Complete SD card boot files
- Complete build artifacts
- JTAG bootstrap files (separate zip)

## Checksums

See `SHA256SUMS.txt` for file integrity verification.

## Troubleshooting

If LibreSDR doesn't boot:
1. Use SD card method to recover
2. Or use JTAG bootstrap files with Vivado

For srsRAN setup, see main README.md
