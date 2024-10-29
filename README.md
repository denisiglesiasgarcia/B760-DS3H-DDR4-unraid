# B760-DS3H-DDR4-unraid

This is my notes to (try to) make an efficient unraid server using Gigabyte B760 DS3H DDR4.

## Table of Contents

- [TODO list](#todo-list)
- [Setup](#setup)
  - [Bios settings](#bios-settings)
  - [Mover settings](#mover-settings)
- [What i tried and it did not work as expected](#what-i-tried-and-it-did-not-work-as-expected)
  - [IBM ServeRAID M1015](#ibm-serveraid-m1015)
  - [Solarflare 10GB SF329-9021-R7.4](#solarflare-10gb-sf329-9021-r74)
  - [Kingston NV2 4TB M.2 SSD](#kingston-nv2-4tb-m2-ssd)
  - [Still doesn't work as expected](#still-doesnt-work-as-expected)
- [How to activate power savings](#how-to-activate-power-savings)
  - [Powertop usage](#powertop-usage)
  - [Realtek ethernet](#realtek-ethernet)
  - [ASPM](#aspm)
- [Estimation of power consumption and C-states](#estimation-of-power-consumption-and-c-states)
- [Sources](#sources)

## TODO list

- ~~Add BIOS configuration~~
- ~~Redo tests~~
- ~~Debug X710-DA2 C6 to achieve C10~~
- Debug pcie 16x power saving problems
- ~~Add new samsung SSD M2 cache~~
- Add list of apps

## Setup

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

### Bios settings

<details>

<summary>Bios settings</summary>

> [!NOTE]  
> You can disable the realtek NIC if you use the X710-DA2.

![04CBD157-37C9-400B-85CC-EC78CE7B10FD_1_105_c](https://github.com/user-attachments/assets/4e4424c2-e512-4b68-b93c-9b26badd64a1)

![980CB055-2729-4316-A1F9-384FB5775C30_1_105_c](https://github.com/user-attachments/assets/f7c61b01-d39c-4824-9345-30951f8f0af0)

![327AE0C8-8D36-42CE-872C-D1C19FBF4699_1_105_c](https://github.com/user-attachments/assets/4700b1cb-6ce2-4527-89b0-96c12224d314)

![3DB7597F-2DA1-4A03-913C-F409EDA961F0_1_105_c](https://github.com/user-attachments/assets/24bd94f1-96c6-40d8-a855-9799b360f6ec)

![675DF690-384E-4971-AA91-D2FD62F5647B_1_105_c](https://github.com/user-attachments/assets/5c7cb6be-ea5d-48e1-a206-b2186bf9eb02)

</details>

### Mover settings

I did not like the way mover was being scheduled. I wanted:

- If mover is not already running and cache usage is over 80%:
  - Stop all the active docker containers and remember which one was active
  - Set tunable `md_write_method` to `reconstruct write` (sometimes called `Turbo write`). This will spin up all the disks and improve mover speed.
  - Start mover
  - When mover is done
    - Set tunable `reconstruct write` to `auto` so disks can spin down again
    - Start all the docker containers that were active before
- If mover is running do nothing
- If cache usage is below 80% do nothing

> [!WARNING]  
> This works fine in 6.12.13, but in the 7.0.0-beta3 *enabling Turbo Write* does not work. When i update to the newer version i will fix this script.

I run hourly this script using the plugin `User scripts`. There is 2 things to adjust:

- The path to the cache disk in function `get_cache_usage`. Set to `/mnt/cache`
- The threshold of % usage of cache to activate the script. The variable is `CACHE_THRESHOLD` in the main function. Set to `80` so it is 80%

<details>

<summary>Click to see script</summary>

```bash
#!/bin/bash

# Function to get cache usage percentage. Change cache path if necessary.
get_cache_usage() {
    df -h /mnt/cache | awk 'NR==2 {print $5}' | tr -d '%'
}

# Function to check if mover is already running
is_mover_running() {
    pgrep -f "/usr/local/sbin/mover" > /dev/null
}

# Function to enable Turbo Write
enable_turbo_write() {
    echo "Enabling Turbo Write"
    /usr/local/sbin/mdcmd set md_write_method 1
}

# Function to disable Turbo Write
disable_turbo_write() {
    echo "Disabling Turbo Write"
    /usr/local/sbin/mdcmd set md_write_method 0
}

# Function to stop Docker containers and save their state
stop_docker_containers() {
    echo "Stopping active Docker containers"
    docker ps -q > /tmp/active_containers.txt
    docker stop --time=60 $(docker ps -q)
}

# Function to start previously active Docker containers
start_docker_containers() {
    echo "Starting previously active Docker containers"
    if [ -f /tmp/active_containers.txt ]; then
        xargs docker start < /tmp/active_containers.txt
        rm /tmp/active_containers.txt
        echo "Successfully started previously active Docker containers"
    else
        echo "No record of previously active containers found"
    fi
}

# Main script
CACHE_THRESHOLD=80
CACHE_USAGE=$(get_cache_usage)
echo "Cache usage is at ${CACHE_USAGE}%."

if is_mover_running; then
    echo "Mover is already running. Exiting script."
    exit 0
fi

if [ "$CACHE_USAGE" -gt "$CACHE_THRESHOLD" ]; then
    echo "Cache usage is above ${CACHE_THRESHOLD}%. Starting mover process."
    
    stop_docker_containers
    
    enable_turbo_write
    
    echo "Executing mover"
    /usr/local/sbin/mover start
    
    # Wait for mover to complete
    while is_mover_running; do
        echo "Mover is still running. Waiting..."
        sleep 60
    done
    
    echo "Mover completed."
    
    disable_turbo_write
    
    start_docker_containers
else
    echo "Cache usage is below ${CACHE_THRESHOLD}%. Skipping mover execution."
fi
```

</details>

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

#### Still doesn't work as expected

1) Using the 16x pcie port does seem to block power saving. It is now empty and i use other pcie ports for the add-on cards. Upgrading BIOS to F11 has not solved this.
2) ~~Intel X710-DA2 was Dell branded when i bought it. I flashed it to OEM. It should achieve C10 but it is stuck at C6 and limits everything else.~~ I upgraded the MB BIOS to F11 and it works as expected!

## How to activate power savings

### Powertop usage

Install powertop using `nerd tools` in Apps

![image](https://github.com/user-attachments/assets/c36a4c92-bbbf-4adb-a283-46248e2f9f56)

Activate auto-tune

```bash
powertop --auto-tune
```

Or use the recommendations of the forum user mgutt instead of `powertop --auto-tune` [Source](https://forums.unraid.net/topic/98070-reduce-power-consumption-with-powertop/)

<details>

<summary>Click to see script</summary>

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

</details>

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

<details>

<summary>Image 1</summary>

![0492BF14-92CB-4E26-82C3-6EDCBEE78EEB_1_105_c](https://github.com/user-attachments/assets/80977512-18f0-4b40-86d6-d01c2717760f)

</details>

<details>

<summary>Image 2</summary>

![0CB6FA03-20C7-40FC-89C1-0304568932CD_1_105_c](https://github.com/user-attachments/assets/c67ce8fd-43a8-4bad-9d0b-f2cdf0b379e3)

</details>

<details>

<summary>Image 3</summary>

![87D65579-3ABC-4E11-9886-CDD22ABE4607_1_105_c](https://github.com/user-attachments/assets/b069d77c-e16c-432f-aebc-d23dad7f0cb0)

</details>

<details>

<summary>Image 4</summary>

![2B6D7CFD-B57A-4C78-915D-A504D0597B88_1_105_c](https://github.com/user-attachments/assets/8d1b9031-6f17-4996-9076-c1342c7ff243)

</details>

<details>

<summary>Image 5</summary>

![D8B1C282-D188-4D8A-ABAC-63E6542C0CEE_1_105_c](https://github.com/user-attachments/assets/4f508f8a-b7d7-4bee-885c-4ba229e03f50)

</details>

<details>

<summary>Image 6</summary>

![A4E7ECE9-3E60-4405-A1A9-63000C0493DA_1_105_c](https://github.com/user-attachments/assets/5d79f41b-01d4-46d6-97d5-85a9b3d5ec0a)

</details>

<details>

<summary>Image 7</summary>

![E67CD7AB-A4BF-41D9-9622-B8F3795384C9_1_105_c](https://github.com/user-attachments/assets/ed35b30e-cdcf-4fa6-997e-626d2e0a5cee)

</details>

<details>

<summary>Image 8</summary>

![IMG_6369](https://github.com/user-attachments/assets/214bc226-e0cd-4be0-a031-c47d95a971fc)

</details>

## Sources

I want to thank every person that shared his/her knowledge with the rest of us. This is the sources i used:

- https://forums.unraid.net/topic/98070-reduce-power-consumption-with-powertop/
- https://forums.unraid.net/topic/156160-gigabyte-b760m-ds3h-ddr4-verschiedene-messungen-werte/
- https://forums.unraid.net/topic/141770-asm1166-flashen-mit-der-firmware-der-silverstone-ecs06-karte-sata-kontroller/
- https://forums.unraid.net/topic/156160-gigabyte-b760m-ds3h-ddr4-verschiedene-messungen-werte/#comment-1392769
- https://docs.phil-barker.com/posts/upgrading-ASM1166-firmware-for-unraid/
- https://gist.github.com/mietzen/736583d37a1d370273c0775aaaa57aa5
