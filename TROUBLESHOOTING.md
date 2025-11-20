# LibreSDR Timestamp Firmware - Troubleshooting Guide

## SD Card Boot Issues

### Problem: "not defined boot" Error

If you see this error when trying to boot from SD card:
```
0 " not definedboot LIBRESDR>
```

This means your u-boot doesn't have the SD boot commands configured.

#### Solution 1: Manual Boot (One-Time)

Connect to the debug USB port and manually boot:
```bash
# Connect via serial (115200 baud)
screen /dev/ttyUSB0 115200
# or
minicom -D /dev/ttyUSB0 -b 115200

# At the LIBRESDR> prompt, enter these commands:
fatload mmc 0 0x2000000 uImage
fatload mmc 0 0x2080000 devicetree.dtb  
fatload mmc 0 0x3000000 uramdisk.image.gz
bootm 0x2000000 0x3000000 0x2080000
```

#### Solution 2: Set Permanent SD Boot

At the LIBRESDR> u-boot prompt:
```bash
setenv sdboot 'echo Booting from SD...; fatload mmc 0 0x2000000 uImage && fatload mmc 0 0x2080000 devicetree.dtb && fatload mmc 0 0x3000000 uramdisk.image.gz && bootm 0x2000000 0x3000000 0x2080000'
setenv bootcmd 'run sdboot'
saveenv
reset
```

The device will now boot from SD card automatically.

#### Solution 3: DFU Flash Instead

If SD boot continues to have issues, use DFU method instead:
```bash
sudo dfu-util -a firmware.dfu -D libresdr-v0.37-timestamps-libre.dfu
```

---

## DFU Port Not Working

### Problem: Can't Enter DFU Mode

If `dfu-util` can't find the device or your USB DFU port is dead:

#### Solution 1: Force DFU via Debug Console
```bash
# Connect to debug USB port
screen /dev/ttyUSB0 115200

# Login (root/analog) and run:
pluto_reboot ram
# or
device_reboot ram
```

This forces a reboot into DFU mode.

#### Solution 2: Modify U-Boot to Auto-Enter DFU
```bash
# At LIBRESDR> u-boot prompt:
setenv dfu_alt_info ${dfu_alt_info_ram}
setenv bootcmd 'run dfu_ram'
saveenv
reset
```

Device will boot into DFU mode on next power cycle.

#### Solution 3: Use SD Card Method

Flash via SD card instead (see installation guide).

---

## No Signal / Phone Can't See Network

### Check 1: Verify sdr_ip_gadget is Running
```bash
ssh root@192.168.1.1
ps aux | grep sdr_ip_gadget
# Should show: /usr/sbin/sdr_ip_gadget
```

If not running:
```bash
/etc/init.d/S55sdr_ip_gadget start
```

### Check 2: Verify IIO Buffers
```bash
# RX buffer should be enabled
cat /sys/bus/iio/devices/iio:device3/buffer/enable
# Should show: 1

# Buffer size should match timestamp_every + 2
cat /sys/bus/iio/devices/iio:device3/buffer/length
# For 15 PRB: should show 3842 (3840 + 2)
# For 6 PRB: should show 1922 (1920 + 2)
```

### Check 3: Network Connectivity
```bash
# From PC, verify connection
ping 192.168.1.1
# Should have <1ms latency, 0% loss

# Check SoapySDR can find device
SoapySDRUtil --find="driver=plutosdr,hostname=192.168.1.1"
```

### Check 4: srsRAN Configuration

Verify enb.conf has correct settings:
```ini
[enb]
n_prb = 15  # or 6

[rf]
device_name = soapy
device_args = driver=plutosdr,hostname=192.168.1.1,direct=1,timestamp_every=3840,loopback=0
# timestamp_every for 6 PRB = 1920
# timestamp_every for 15 PRB = 3840
# timestamp_every for 25 PRB = 7680
```

---

## Signal Stuttering / Choppy at Higher Bandwidths

### Problem: 25 PRB (5 MHz) Shows Intermittent Signal

This indicates buffer underruns - the transport can't keep up with data rate.

#### Recommended Solution: Use Lower Bandwidth

**15 PRB (3 MHz) is the recommended maximum** for LibreSDR + sdr_ip_gadget over Ethernet.

#### If You Want to Try Higher Bandwidth:

1. **Enable both CPU cores:**
```bash
fw_printenv maxcpus
# If not "2":
fw_setenv maxcpus 2
reboot
```

2. **Increase network buffers:**
```bash
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
```

3. **Kill unnecessary services:**
```bash
killall httpd
/etc/init.d/S46iiod stop
```

4. **Check PC-side network:**
```bash
ping -f 192.168.1.1
# Should show 0% loss
```

Even with optimizations, 25 PRB may not be stable enough for reliable connections.

---

## Authentication Failures

### Problem: Phone Attaches But Auth Fails
```
UL NAS: Authentication Failure
MAC code failure
```

This is **not a firmware issue** - it's EPC configuration.

#### Solution: Fix user_db.csv
```bash
nano ~/.config/srsran/user_db.csv
```

Make sure:
- IMSI matches your SIM card
- K (authentication key) is correct
- OP/OPc key is correct

The firmware and timestamps are working correctly if you see:
- S1 Setup Request
- Initial UE message
- Attach Request with IMSI

---

## Version Verification

After flashing, verify correct version:
```bash
ssh root@192.168.1.1
cat /etc/os-release
```

Should show:
```
VERSION_ID="v0.37-timestamps-libre"
```

Check timestamp daemon:
```bash
ps aux | grep sdr_ip_gadget
ls -la /usr/sbin/sdr_ip_gadget
```

---

## Build Issues

### Problem: Vivado Build Fails with CE Error

If you see:
```
ERROR: [BD 41-85] Exec TCL - Illegal Name: The name 'counter_timestamp/CE' contains illegal characters
```

**Solution:** Make sure you have this line in `hdl/projects/libre/system_bd.tcl`:
```tcl
ad_ip_parameter counter_timestamp CONFIG.CE true
```

This enables the clock-enable pin on the counter, required for Vivado 2021.2.

### Problem: Submodule Errors During Build
```bash
git submodule update --init --recursive
```

---

## Performance Issues

### Check CPU Load
```bash
ssh root@192.168.1.1
top
```

Normal operation at 15 PRB:
- sdr_ip_gadget: 30-40% CPU
- System should be responsive

If sdr_ip_gadget is >80% CPU constantly, you may be pushing bandwidth limits.

### Check Network Stats
```bash
ifconfig eth0
```

Look for:
- **RX/TX errors** - should be 0
- **Dropped packets** - should be 0

If you see drops/errors, check:
- Ethernet cable quality
- Switch/router performance
- PC network card

---

## Hardware Compatibility

### Tested Hardware

This firmware has been tested on:
- **LibreSDR (HamGeek variant)** - Full functionality
- Zynq-7020 (xc7z020clg400-2)
- AD9361 RF transceiver

### Known Variants

Different LibreSDR variants may have:
- Different u-boot configurations
- Different package sizes (clg400 vs clg225)
- Different memory configurations

If your variant has issues, please open a GitHub issue with:
- Board markings/version
- Debug console output
- Error messages

---

## Getting More Help

1. **Check existing GitHub issues:**
   https://github.com/pumatrax/libresdr-fw-timestamps/issues

2. **Review Phil's documentation:**
   https://www.quantulum.co.uk/blog/

3. **srsRAN documentation:**
   https://docs.srsran.com/

4. **Open a new issue with:**
   - LibreSDR variant/version
   - Complete error messages
   - Debug console output
   - Steps to reproduce

---

## Debug Console Access

The debug USB port provides serial console access for troubleshooting.

### Connect to Debug Port
```bash
# Linux
screen /dev/ttyUSB0 115200
# or
minicom -D /dev/ttyUSB0 -b 115200

# Windows: Use PuTTY
# - Serial line: COM port
# - Speed: 115200
```

### Useful Debug Commands
```bash
# View boot messages
dmesg

# Check running services
ps aux

# Network configuration
ifconfig

# View system info
cat /proc/cpuinfo
cat /proc/meminfo
free

# U-boot environment
fw_printenv
```

---

## Success Indicators

Your system is working correctly when you see:

**On LibreSDR:**
```bash
ps aux | grep sdr_ip_gadget  # Running
cat /sys/bus/iio/devices/iio:device3/buffer/enable  # Shows: 1
```

**On srsRAN:**
```
S1 Setup Request
Initial UE message: ATTACH_REQUEST
Attach request -- IMSI: [your IMSI]
```

**On Spectrum Analyzer / SDR:**
- Clean, continuous LTE signal
- Bandwidth matches n_prb setting
- No stuttering or gaps

**On Phone:**
- Cell appears in network scan
- Can initiate connection attempt
- Gets to authentication stage (even if fails due to wrong keys)
