# B760-DS3H-DDR4-unraid
B760 DS3H DDR4 unraid

## Setup

### Now
Software:
- Unraid 7.0.0 beta3

Hardware:
- Gigabyte B760 DS3H DDR4 BIOS F10d https://www.gigabyte.com/Motherboard/B760-DS3H-DDR4-rev-10#kf
- i5-13500
- 64Gb DDR4 3600
- Add-on cards
  - Intel Ethernet CNA X710-DA2 brand Dell
    - Used this [tutorial](https://gist.github.com/mietzen/736583d37a1d370273c0775aaaa57aa5) to flash it and activate all the functionalities
    - [ebay](https://www.ebay.fr/itm/296716909983)
  - ASM1166 6x SATA PCIe 4x
    - Used this [tutorial](https://docs.phil-barker.com/posts/upgrading-ASM1166-firmware-for-unraid/) to activate all power saving functionalities in that card
    - [ebay](https://www.ebay.fr/itm/355366723386)
- Storage
  - Samsung SSD 970 EVO Plus 500GB M.2 as cache for appdata/system
  - SSD SATA 512GB as cache for rest of data
  - 2x Seagate Exos X16 14TB as parity
  - 4x Toshiba N300 6TB
  - 2x Seagate Desktop 6TB
- Ventilation
  - 3x Fractal design 140mm 3-pin
  - 4x Be Quiet Pure Wings 2 4-pin PWM (one for the CPU)
  - Everything except the CPU is connected to a board that is integrated in the computer case [Fractal Design Meshify 2 XL](https://www.fractal-design.com/products/cases/meshify/meshify-2-xl-dark-tempered-glass/light-tempered-glass/). The main advantage is that it regulates the voltage of the 3-pin and 4-pin fans.
- PSU
  - Seasonic Focus PX (750 W)

### What i tried and it did not work as expected

#### IBM ServeRAID M1015

This card was flashed to LSI 9211-8i IT mode using a tutorial similar to [this](https://www.servethehome.com/ibm-serveraid-m1015-part-4/) 10 years ago. This card is great but it is not oriented to power savings, i don't remember in which C-state it blocks. Expected power usage is around 10W plus 1-2W per disk. It runs hot.

Replaced by the ASM1166.

#### Solarflare 10GB SF329-9021-R7.4

This [card](https://www.ebay.fr/itm/193363150794) is an excellent card and works out of the box in everything i tried. Not power savings friendly. Runs very hot. Power usage estimation is around 10-20W. It blocks the C-state of the rest, so it increments significatively power usage.

Replaced by the Intel X710-DA2

#### Kingston NV2 4TB M.2 SSD

Very cheap but does not have any power savings activated and runs as hot as a volcano. Not recommended. Was used as cache for everything.

Will be replaced by 2x Samsung 990 EVO 2TB

## Estimation of power consumption

> **_NOTE:_** I will do more tests when i add the Samsung SSD 2TB M2

base = PSU + Gigabyte B760 DS3H DDR4 BIOS + 1x Be Quiet Pure Wings 2 4-pin PWM 

| Software | Hardware  | Power usage estimation (W) | max C-state  | Comments |
|----------|-----------|-----------------|--------------|----------|
| Unraid 7.0.0 beta3 | base | 10-17 | C10 |   |
| Unraid 7.0.0 beta3 | base + LSI 9211-8i IT |  30 | 0 | |
| Unraid 7.0.0 beta3 | base + Solarflare 10GB | 30 | 0 | |
| Unraid 7.0.0 beta3 | base + LSI 9211-8i IT + Solarflare 10GB | 30-40 | 0 | |
| Unraid 7.0.0 beta3 | base + LSI 9211-8i IT + Solarflare 10GB + Kingston NV2 4TB M.2 SSD | more than 40 | 0 | Kingston NV2 4TB does not support ASPM. Runs very hot. |
| Unraid 7.0.0 beta3 | base + ASM1166 | 17 | - |  C8 or C10 i don't remember. ASM1166 should add 2W-4W to the base consumption. |
| Unraid 7.0.0 beta3 | base + Samsung SSD 970 EVO Plus 500GB M.2 | 10-17 | C10 | Samsung SSD 970 EVO Plus 500GB M.2 does not increment the power consumption at idle by more than 1W and the difference between M.2 slots is negligable |
| Unraid 7.0.0 beta3 | base + Intel X710-DA2 | - | C6 | Runs cool. |
| Unraid 7.0.0 beta3 | base + ASM1166 + Intel X710-DA2 | - | C6 |  |
| Unraid 7.0.0 beta3 | base + ASM1166 + Intel X710-DA2 + Samsung SSD 970 EVO Plus 500GB M.2 | 27 | C6 |  |

With everything connected (base + ASM1166 + Intel X710-DA2 + Samsung SSD 970 EVO Plus 500GB M.2 + SSD cache + all ventilation + all HD spinned down) it idles around 30W and reaches C6. It needs more consistent tests because some power consumption values are probably not right.

### Things that don't work as expected right now

This motherboard is very efficient but there are some things that i need to debug.

1) Using the 16x pcie port does seem to block power saving. It is now empty and i use other pcie ports for the add-on cards.
2) Intel X710-DA2 was Dell branded when i bought it. Iflashed it to OEM. It should achieve C10 but it is stuck at C6 and limits everything else.


## Powertop usage

Install powertop using `nerd tools` in Apps

![image](https://github.com/user-attachments/assets/c36a4c92-bbbf-4adb-a283-46248e2f9f56)

Activate auto-tune

```bash
powertop --auto-tune
```

Or use the recommendations of the forum user mgutt instead of `powertop --auto-tune` https://forums.unraid.net/topic/98070-reduce-power-consumption-with-powertop/

```bash
# -------------------------------------------------
# Set power-efficient CPU governor
# -------------------------------------------------
/etc/rc.d/rc.cpufreq powersave

# -------------------------------------------------
# Disable CPU Turbo
# -------------------------------------------------
[[ -f /sys/devices/system/cpu/intel_pstate/no_turbo ]] && echo "1" > /sys/devices/system/cpu/intel_pstate/no_turbo
[[ -f /sys/devices/system/cpu/cpufreq/boost ]] && echo "0" > /sys/devices/system/cpu/cpufreq/boost

# -------------------------------------------------
# Enable power-efficient ethernet
# -------------------------------------------------

# enable IEEE 802.3az (Energy Efficient Ethernet): Could be incompatible to LACP bonds!
for i in /sys/class/net/eth?; do dev=$(basename $i); [[ $(echo $(ethtool --show-eee $dev 2> /dev/null) | grep -c "Supported EEE link modes: 1") -eq 1 ]] && ethtool --set-eee $dev eee on; done

# Disable wake on lan
for i in /sys/class/net/eth?; do ethtool -s  $(basename $i) wol d; done

# -------------------------------------------------
# powertop tweaks
# -------------------------------------------------

# Enable SATA link power management
echo med_power_with_dipm | tee /sys/class/scsi_host/host*/link_power_management_policy

# Runtime PM for I2C Adapter (i915 gmbus dpb)
echo auto | tee /sys/bus/i2c/devices/i2c-*/device/power/control

# Autosuspend for USB device
echo auto | tee /sys/bus/usb/devices/*/power/control

# Runtime PM for disk
echo auto | tee /sys/block/sd*/device/power/control

# Runtime PM for PCI devices
echo auto | tee /sys/bus/pci/devices/????:??:??.?/power/control

# Runtime PM for ATA devices
echo auto | tee /sys/bus/pci/devices/????:??:??.?/ata*/power/control
```
Use the plugin `User scripts` to execute this every time unraid starts

![image](https://github.com/user-attachments/assets/321f397b-471b-4743-a1f8-f660d4526ae7)

### Realtek ethernet

Disable Realtek ethernet in BIOS or use this to activate ASPM. [Source](https://forums.unraid.net/topic/156160-gigabyte-b760m-ds3h-ddr4-verschiedene-messungen-werte/#comment-1392769)

```bash
setpci -s 03:00.0 0x80.B=0x42
setpci -s 00:1c.2 0x50.B=0x42
```

### ASPM

Check ASPM status
```bash
lspci -vv | awk '/ASPM/{print $0}' RS= | grep --color -P '(^[a-z0-9:.]+|ASPM )'
```
