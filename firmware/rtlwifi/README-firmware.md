# RTL8192EU firmware (TL-WN821N v5, 2357:0107)

Place **`rtl8192eu_nic.bin`** here (32,286 B, md5 prefix `ed1fd62b…`).
It is embedded into the kernel image via CONFIG_EXTRA_FIRMWARE so the dongle
hot-plugs as `wlan1` with no userspace loader.

nRF52 / BLE (nice!nano, nRF52840, nRF52832) needs **no firmware blob** —
the BLE stack runs on-chip; the kernel only needs CONFIG_BT + CONFIG_BT_HCIBTUSB.
