# LibreSDR v0.37 Timestamps - Installation Guide

## Installation Methods

### Method 1: DFU Update (Recommended for existing LibreSDR)
```bash
# Flash the DFU file
sudo dfu-util -a firmware.dfu -D dfu-update/libresdr-v0.37-timestamps-libre.dfu

# Reboot LibreSDR
```

### Method 2: Mass Storage Update

1. Connect LibreSDR via USB
2. Copy `dfu-update/libresdr-v0.37-timestamps-libre.frm` to the LibreSDR mass storage device
3. Eject and reboot

### Method 3: SD Card Boot (For recovery or fresh install)

1. Format SD card as FAT32
2. Copy all files from `sd-card-image/` to SD card root
3. Insert SD card into LibreSDR
4. Power on - it will boot from SD card

### Method 4: Complete Firmware Package

Extract `libresdr-fw-v0.37-timestamps-libre.zip` for all build artifacts.

## Verification

After flashing, SSH to LibreSDR:
```bash
ssh root@192.168.1.1
# Check version
cat /etc/os-release
# Verify sdr_ip_gadget is running
ps aux | grep sdr_ip_gadget
```

## What's Included

- **dfu-update/** - Ready-to-flash DFU files
- **sd-card-image/** - Bootable SD card files
- **libresdr-fw-v0.37-timestamps-libre.zip** - Complete firmware package
- **libresdr-jtag-bootstrap-v0.37-timestamps-libre.zip** - JTAG recovery files

## Checksums

See `SHA256SUMS.txt` for file integrity verification.
