# LibreSDR Firmware with Timestamp Support

LibreSDR firmware (v0.37) with integrated timestamp functionality for LTE applications using srsRAN_4G.

## Overview

This project combines:
- **Hardware base**: [0wl/libresdr-fw](https://github.com/day0wl/libresdr-fw) - LibreSDR hardware support
- **Timestamp functionality**: [pgreenland/plutosdr-fw](https://github.com/pgreenland/plutosdr-fw) v0.37_timestamp - Phil Greenland's timestamp system
- **Target application**: LTE eNodeB using srsRAN_4G

## Key Features

- ✅ Sample-aligned timestamps with clock-enable gating (CE connection)
- ✅ Ethernet transport via `sdr_ip_gadget` daemon (UDP ports 30432/30433)
- ✅ Compatible with SoapySDR/srsRAN_4G
- ✅ Tested and working for LTE transmission on LibreSDR hardware
- ✅ Built with Vivado 2021.2

## Hardware Tested

- **LibreSDR** (HamGeek variant with Ethernet)
  - Zynq-7020 (xc7z020clg400-2)
  - AD9361 RF transceiver
  - Ethernet + USB connectivity

## What's Different from Stock LibreSDR

### FPGA/HDL Changes:
1. Integrated Phil Greenland's timestamp IP blocks:
   - `util_cpack2_timestamp` - RX path timestamping
   - `util_upack2_timestamp` - TX path timestamping
   - `c_counter_binary` with `CONFIG.CE true` - Sample-aligned counter

2. Modified `system_bd.tcl` for timestamp signal routing:
   - Counter clock-enable from `rx_fir_decimator/valid_out_0`
   - Timestamp insertion in RX/TX data paths
   - Clock domain crossing for DMA interface

### Firmware Changes:
1. Added `sdr_ip_gadget` daemon for low-latency Ethernet transport
2. Added dependencies: `libiio`, `libad9361`, `libusb`
3. Updated version string to `v0.37-timestamps-libre`

## Building from Source

### Prerequisites
```bash
sudo apt-get install git build-essential fakeroot libncurses5-dev libssl-dev ccache
sudo apt-get install dfu-util u-boot-tools device-tree-compiler mtools
sudo apt-get install bc python cpio zip unzip rsync file wget
```

### Vivado Installation

Requires Xilinx Vivado 2021.2. Set environment:
```bash
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2021.2/settings64.sh
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=$PATH:/opt/Xilinx/Vitis/2021.2/gnu/aarch32/lin/gcc-arm-linux-gnueabi/bin
export TARGET=libre
```

### Build
```bash
git clone --recurse-submodules https://github.com/pumatrax/libresdr-fw-timestamps.git
cd libresdr-fw-timestamps
make
```

Build artifacts in `build/`:
- `libre.dfu` - DFU image for flashing
- `libre.frm` - Firmware update file

## Installation

Flash via DFU:
```bash
sudo dfu-util -a firmware.dfu -D build/libre.dfu
```

Or copy `libre.frm` to LibreSDR mass storage device.

## Usage with srsRAN

### Install SoapySDR and Plugin

Follow Phil Greenland's guide: [Private LTE with PlutoPlus SDR](https://www.quantulum.co.uk/blog/private-lte-with-plutoplus-sdr/)

### Configure srsRAN

In `~/.config/srsran/enb.conf`:
```ini
[rf]
device_name = soapy
device_args = driver=plutosdr,hostname=192.168.1.1,direct=1,timestamp_every=1920,loopback=0
tx_gain = 89
rx_gain = 20
```

### Verify Connection
```bash
# Check if SoapySDR finds LibreSDR
SoapySDRUtil --find="driver=plutosdr,hostname=192.168.1.1"

# SSH to LibreSDR and verify sdr_ip_gadget is running
ssh root@192.168.1.1
ps aux | grep sdr_ip_gadget
```

## Testing

Successfully tested with:
- srsRAN_4G eNodeB
- Android UE device
- LTE Band 4 (AWS 1700/2100 MHz)
- 6 PRB configuration

Phone successfully completes:
- ✅ PSS/SSS detection
- ✅ MIB decode
- ✅ SIB1 acquisition  
- ✅ RRC connection
- ✅ Attach request

## Credits

- **0wl (day0wl)** - [LibreSDR firmware base](https://github.com/day0wl/libresdr-fw)
- **Phil Greenland (Quantulum)** - [Timestamp functionality](https://github.com/pgreenland/plutosdr-fw) and [blog guides](https://www.quantulum.co.uk/blog/)
- **Analog Devices** - [PlutoSDR HDL](https://github.com/analogdevicesinc/plutosdr-fw)

## License

Follows the same license structure as the upstream projects.

## Notes

- The CE (clock enable) connection on the timestamp counter is critical for LTE timing
- `SYNC_TRANSFER_START 1` configuration allows DMA self-synchronization
- Ethernet transport provides better performance than USB for LTE applications

## Support

For issues specific to this LibreSDR timestamp integration, please open an issue on this repository.

For general srsRAN or timestamp usage questions, refer to:
- [Phil's blog](https://www.quantulum.co.uk/blog/)
- [srsRAN documentation](https://docs.srsran.com/)
