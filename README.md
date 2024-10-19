# B760-DS3H-DDR4-unraid
B760 DS3H DDR4 unraid


powertop --auto-tune

setpci -s 02:00.0 0x80.B=0x42
setpci -s 00:1c.2 0x50.B=0x42

lspci -vv | awk '/ASPM/{print $0}' RS= | grep --color -P '(^[a-z0-9:.]+|ASPM )'
