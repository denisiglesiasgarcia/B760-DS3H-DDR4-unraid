# B760-DS3H-DDR4-unraid
B760 DS3H DDR4 unraid

Tested on 7.0.0 beta3

## Powertop

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

## Realtek ethernet

Disable Realtek ethernet in BIOS or use this to activate ASPM

https://forums.unraid.net/topic/156160-gigabyte-b760m-ds3h-ddr4-verschiedene-messungen-werte/#comment-1392769

```bash
setpci -s 03:00.0 0x80.B=0x42
setpci -s 00:1c.2 0x50.B=0x42
```

Check ASPM status
```bash
lspci -vv | awk '/ASPM/{print $0}' RS= | grep --color -P '(^[a-z0-9:.]+|ASPM )'
```
