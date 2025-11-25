LibreSDR v0.38 with Timestamps - Working LTE Build

Second release with Phil Greenland's timestamp support - based on hz12's optimized v0.38 firmware (which itself is based on 0wl's LibreSDR port). Successfully tested with srsRAN_4G LTE at PRB 25!

What's Included

Ready-to-Use Images
- libresdr-v0.38-timestamps-525mhz.zip - Stable build (CPU: 750 MHz, DDR: 525 MHz)
- libresdr-v0.38-timestamps-750mhz.zip - Overclocked build (CPU: 750 MHz, DDR: 750 MHz)

Each zip contains:
- libre.dfu - DFU flashable image (recommended)
- libre.frm - Mass storage update file  
- build_sdimg/ - Complete bootable SD card files

Key Features

- Sample-aligned timestamps with CE-gated counter
- sdr_ip_gadget daemon for low-latency Ethernet transport (UDP 30432/30433)
- Hz12's LVDS optimizations for higher sample rates
- Compatible with SoapySDR and srsRAN_4G running on Linux host
- Built with Vivado 2022.2 for Zynq-7020 (xc7z020clg400-2)

Integration Notes

This build resolves a clock domain timing conflict between hz12's l_clk optimization and Phil's timestamp modules. Hz12's v0.38 uses axi_ad9361/l_clk for DMA clocks to achieve higher sample rates, but Phil's timestamp logic couldn't meet timing constraints in the faster l_clk domain. Solution was to revert the four DMA clock connections (adc_dma/fifo_wr_clk, dac_dma/m_axis_aclk, cpack_timestamp/dma_clk, upack_timestamp/dma_clk) back to sys_cpu_clk while keeping hz12's CPU/DDR overclocking and LVDS mode. This achieves stable timing closure (WNS +0.715ns) while maintaining performance for LTE-relevant sample rates.

Tested and Verified

Successfully tested with srsRAN_4G eNodeB on Linux host, LTE Band 4:
- PRB 15 (3 MHz): Stable
- PRB 25 (5 MHz): Stable - phones connect and reach authentication  
- PRB 50 (10 MHz): Not supported (AD9361 clock configuration limit)

Quick Install

# Flash via DFU (recommended)
sudo dfu-util -a firmware.dfu -D libre.dfu

See INSTALLATION.md for all installation methods.

Configuration Required

fw_setenv compatible ad9361
fw_setenv mode 2r2t
reboot

After boot, optimize for LTE operation:
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
echo 32768 > /sys/bus/iio/devices/iio:device3/buffer/length
echo 32768 > /sys/bus/iio/devices/iio:device2/buffer/length
renice -20 $(pidof sdr_ip_gadget)

srsRAN Configuration

For PRB 25:
n_prb = 25
device_args = driver=plutosdr,hostname=192.168.1.1,direct=1,timestamp_every=5760,loopback=0

Note: Use timestamp_every=5760 for PRB 25 (not the standard 7680)

Hardware Requirements

- LibreSDR with Ethernet
- Zynq-7020 / AD9361
- Stable 5V 3A power supply recommended for overclocked builds

Credits

- hz12 - LibreSDR v0.38 optimizations and overclocking (based on 0wl's port)
- 0wl - Original LibreSDR hardware port from PlutoSDR
- Phil Greenland - Timestamp functionality and LTE guides
- Analog Devices - PlutoSDR HDL framework
