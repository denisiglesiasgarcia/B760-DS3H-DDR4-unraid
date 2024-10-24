# B760-DS3H-DDR4-unraid

This is my notes to (try to) make an efficient unraid server using Gigabyte B760 DS3H DDR4.

## Table of Contents

- [TODO list](#todo-list)
- [Setup](#setup)
  - [Now](#now)
  - [Bios settings](#bios-settings)
- [What i tried and it did not work as expected](#what-i-tried-and-it-did-not-work-as-expected)
  - [IBM ServeRAID M1015](#ibm-serveraid-m1015)
  - [Solarflare 10GB SF329-9021-R7.4](#solarflare-10gb-sf329-9021-r74)
  - [Kingston NV2 4TB M.2 SSD](#kingston-nv2-4tb-m2-ssd)
- [Estimation of power consumption and C-states](#estimation-of-power-consumption-and-c-states)
- [Things that don't work as expected right now](#things-that-dont-work-as-expected-right-now)
- [Powertop usage](#powertop-usage)
- [Realtek ethernet](#realtek-ethernet)
- [ASPM](#aspm)
- [Sources](#sources)

## TODO list

- ~~Add BIOS configuration~~
- ~~Redo tests~~
- ~~Debug X710-DA2 C6 to achieve C10~~
- Debug pcie 16x power saving problems
- ~~Add new samsung SSD M2 cache~~

## Setup

### Now

Power meter

- Steffen [link](https://www.digitec.ch/fr/s1/product/steffen-appareil-de-mesure-denergie-numerique-ip20-compteur-de-courant-5807865)

Software:

- Unraid 6.12.13

Hardware:

- Gigabyte B760 DS3H DDR4 BIOS ~~F10d~~ *F11* [link](https://www.gigabyte.com/Motherboard/B760-DS3H-DDR4-rev-10#kf)
- i5-13500
- 32Gb DDR4 3200
- Add-on cards
  - Intel Ethernet CNA X710-DA2 brand Dell
    - Used this [tutorial](https://gist.github.com/mietzen/736583d37a1d370273c0775aaaa57aa5) to flash it and activate all the functionalities
    - [ebay](https://www.ebay.fr/itm/296716909983)
  - ASM1166 6x SATA PCIe 4x
    - Used this [tutorial](https://docs.phil-barker.com/posts/upgrading-ASM1166-firmware-for-unraid/) to activate all power saving functionalities in that card
    - [ebay](https://www.ebay.fr/itm/355366723386)
- Storage
  - Samsung SSD 970 EVO Plus 500GB M.2 as cache for appdata/system
  - Samsung SSD 990 Pro 2TB M.2 as cache for the rest
  - 2x Seagate Exos X16 14TB as parity
  - 4x Toshiba N300 6TB
  - 2x Seagate Desktop 6TB
- Ventilation
  - 3x Fractal design 140mm 3-pin
  - 4x Be Quiet Pure Wings 2 4-pin PWM (one for the CPU)
  - Everything except the CPU is connected to a board that is integrated in the computer case [Fractal Design Meshify 2 XL](https://www.fractal-design.com/products/cases/meshify/meshify-2-xl-dark-tempered-glass/light-tempered-glass/). The main advantage is that it regulates the voltage of the 3-pin and 4-pin fans.
- PSU
  - Seasonic Focus PX (750 W)

#### Bios settings

![04CBD157-37C9-400B-85CC-EC78CE7B10FD_1_105_c](https://github.com/user-attachments/assets/4e4424c2-e512-4b68-b93c-9b26badd64a1)

![980CB055-2729-4316-A1F9-384FB5775C30_1_105_c](https://github.com/user-attachments/assets/f7c61b01-d39c-4824-9345-30951f8f0af0)

![327AE0C8-8D36-42CE-872C-D1C19FBF4699_1_105_c](https://github.com/user-attachments/assets/4700b1cb-6ce2-4527-89b0-96c12224d314)

![3DB7597F-2DA1-4A03-913C-F409EDA961F0_1_105_c](https://github.com/user-attachments/assets/24bd94f1-96c6-40d8-a855-9799b360f6ec)

![675DF690-384E-4971-AA91-D2FD62F5647B_1_105_c](https://github.com/user-attachments/assets/5c7cb6be-ea5d-48e1-a206-b2186bf9eb02)

### What i tried and it did not work as expected

#### IBM ServeRAID M1015

This card was flashed to LSI 9211-8i IT mode using a tutorial similar to [this](https://www.servethehome.com/ibm-serveraid-m1015-part-4/) 10 years ago. This card is great but it is not oriented to power savings, i don't remember in which C-state it blocks. Expected power usage is around 10W plus 1-2W per disk. It runs hot.

Replaced by the ASM1166.

#### Solarflare 10GB SF329-9021-R7.4

This [card](https://www.ebay.fr/itm/193363150794) is an excellent card and works out of the box in everything i tried. Not power savings friendly. Runs very hot. Power usage estimation is around 10-20W. It blocks the C-state of the rest, so it increments significatively power usage of all the system.

Replaced by the Intel X710-DA2

#### Kingston NV2 4TB M.2 SSD

Very cheap but does not have any power savings activated and runs as hot as a volcano. Not recommended. Was used as cache for everything. Limited to C2.

Replaced by Samsung 990 Pro 2TB

## Estimation of power consumption and C-states

base = PSU + Gigabyte B760 DS3H DDR4 BIOS + 1x Be Quiet Pure Wings 2 4-pin PWM

| Test number | Software | Hardware  | max C-state | Idle power usage estimation (W) | Image | Comments |
|:-----------:|:--------:|-----------|:-----------:|:-------------------------------:|:-----:|----------|
|A| Unraid 6.12.13 | base + realtek 1G activated | C3 | 17-18 | 1  | Realtek NIC seems to have a bug in Unraid and limits ASPM |
|B| Unraid 6.12.13 | base + realtek 1G script| C10 | 12-13 | 2  | Use this [commands](#realtek-ethernet) to activate C10 |
|C| Unraid 6.12.13 | base + realtek 1G disabled in bios + X710-DA2 (pcie1x) | C10 | 15 | 3  | pcie1x for X710-DA2 works fine |
|D| Unraid 6.12.13 | base + realtek 1G disabled in bios | - | 11-12 | - | This test is to have an idea of the impact of the realtek NIC, it seems around 1W |
|E| Unraid 6.12.13 | base + realtek 1G disabled in bios + X710-DA2 (pcie16x) | C2 | 23-24 | 4 | The placement in the pcie 16x breaks C10, this is the pcie slot fo the CPU. Avoid this slot if you want achieve C10 |
|F| Unraid 6.12.13 | base + realtek 1G disabled in bios + X710-DA2 (pcie1x) + ASM1166(pcie1x) | C10 | 15-16 | 5 | The ASM1166 does not increment significantly power usage, maybe around 1-2W |
|G| Unraid 6.12.13 | base + realtek 1G disabled in bios + X710-DA2 (pcie1x) + ASM1166(pcie1x) + SSD M.2 Samsung 500G (slot chipset) | C10 | 16 | 6 |  |
|H| Unraid 6.12.13 | base + realtek 1G disabled in bios + X710-DA2 (pcie1x) + ASM1166(pcie1x) + SSD M.2 Samsung 500GB (slot chipset) + SSD M.2 Samsung 2TB (slot CPU) | C10 | 16 | 7 | Adding SSD M.2 has negligable impact on power consumption. Slot does not make a difference. |
|I| Unraid 6.12.13 | base + realtek 1G disabled in bios + X710-DA2 (pcie1x) + ASM1166(pcie1x) + SSD M.2 Samsung 500GB (slot chipset) + SSD M.2 Samsung 2TB (slot CPU) + 4 HD to SATA MB + 4 HD to ASM1166 | C10 | 22-24 | 8 |  |

Image 1

![0492BF14-92CB-4E26-82C3-6EDCBEE78EEB_1_105_c](https://github.com/user-attachments/assets/80977512-18f0-4b40-86d6-d01c2717760f)

Image 2

![0CB6FA03-20C7-40FC-89C1-0304568932CD_1_105_c](https://github.com/user-attachments/assets/c67ce8fd-43a8-4bad-9d0b-f2cdf0b379e3)

Image 3

![87D65579-3ABC-4E11-9886-CDD22ABE4607_1_105_c](https://github.com/user-attachments/assets/b069d77c-e16c-432f-aebc-d23dad7f0cb0)

Image 4

![2B6D7CFD-B57A-4C78-915D-A504D0597B88_1_105_c](https://github.com/user-attachments/assets/8d1b9031-6f17-4996-9076-c1342c7ff243)

Image 5

![D8B1C282-D188-4D8A-ABAC-63E6542C0CEE_1_105_c](https://github.com/user-attachments/assets/4f508f8a-b7d7-4bee-885c-4ba229e03f50)

Image 6

![A4E7ECE9-3E60-4405-A1A9-63000C0493DA_1_105_c](https://github.com/user-attachments/assets/5d79f41b-01d4-46d6-97d5-85a9b3d5ec0a)

Image 7

![E67CD7AB-A4BF-41D9-9622-B8F3795384C9_1_105_c](https://github.com/user-attachments/assets/ed35b30e-cdcf-4fa6-997e-626d2e0a5cee)

Image 8

![IMG_6369](https://github.com/user-attachments/assets/214bc226-e0cd-4be0-a031-c47d95a971fc)

### Things that don't work as expected right now

1) Using the 16x pcie port does seem to block power saving. It is now empty and i use other pcie ports for the add-on cards. Upgrading BIOS to F11 has not solved this.
2) ~~Intel X710-DA2 was Dell branded when i bought it. I flashed it to OEM. It should achieve C10 but it is stuck at C6 and limits everything else.~~ I upgraded the MB BIOS to F11 and it works as expected!

## Powertop usage

Install powertop using `nerd tools` in Apps

![image](https://github.com/user-attachments/assets/c36a4c92-bbbf-4adb-a283-46248e2f9f56)

Activate auto-tune

```bash
powertop --auto-tune
```

Or use the recommendations of the forum user mgutt instead of `powertop --auto-tune` [Source](https://forums.unraid.net/topic/98070-reduce-power-consumption-with-powertop/)

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

Use the plugin `User scripts` to execute one of the 2 scripts every time unraid starts

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

## Sources

I want to thank every person that shared his/her knowledge with the rest of us. This is the sources i used:

- https://forums.unraid.net/topic/98070-reduce-power-consumption-with-powertop/
- https://forums.unraid.net/topic/156160-gigabyte-b760m-ds3h-ddr4-verschiedene-messungen-werte/
- https://forums.unraid.net/topic/141770-asm1166-flashen-mit-der-firmware-der-silverstone-ecs06-karte-sata-kontroller/
- https://forums.unraid.net/topic/156160-gigabyte-b760m-ds3h-ddr4-verschiedene-messungen-werte/#comment-1392769
- https://docs.phil-barker.com/posts/upgrading-ASM1166-firmware-for-unraid/
- https://gist.github.com/mietzen/736583d37a1d370273c0775aaaa57aa5
