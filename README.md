# mido NetHunter 4.9 kernel kit — nRF52 BLE + RTL8192EU hotplug

GitHub Actions–driven rebuild of the **Redmi Note 4/4X (mido)** 4.9 kernel
with NetHunter-friendly additions:

1. **nRF52 BLE** — nRF52840 nice!nano / nRF52832 dev boards: generic Bluetooth
   HCI-over-USB. BLE stack runs on-chip (SoftDevice), so **no firmware blob**
   is needed — just `CONFIG_BT` + `CONFIG_BT_HCIBTUSB` (+ `CONFIG_USB_ACM`
   for the nice!nano UF2/serial bootloader).
2. **RTL8192EU** — TP-Link TL-WN821N v5 (2357:0107): NIC firmware embedded
   into the kernel image (`CONFIG_EXTRA_FIRMWARE`) so the dongle hot-plugs as
   `wlan1` with no userspace firmware loading.

Base tree: **KudProject/kernel_xiaomi_msm8953-4.9 @ lineage-17.1**
(Same msm8953 4.9 family Kali NetHunter mido kernels are built from.)

## Build (GitHub Actions)
1. Push this repo (or use the existing us01srv-surf/mido-kernel.git).
2. Actions → **Build mido 4.9 NetHunter kernel** → Run workflow
   (optionally pin a KudProject commit).
3. ~40–60 min later: artifact `mido-nethunter-kernel` →
   `Black-Hydra-Serp3n7-mido-4.9-nrf52-rtl8192eu.zip`.

## Flash
**The NetHunter app installs the Kali chroot + support apps — it does NOT
flash kernels.** Flash the AnyKernel3 zip in **TWRP**, exactly like the
Black-Hydra zip you already used:

1. Reboot to TWRP
2. Install → `Black-Hydra-Serp3n7-mido-4.9-nrf52-rtl8192eu.zip`
3. Reboot → nRF52 BLE live; `wlan1` hot-plugs with the RTL8192EU dongle.

## Firmware (optional but recommended)
Drop `firmware/rtlwifi/rtl8192eu_nic.bin` (32,286 B) into the kit (or the
`firmware/` dir of your Actions repo) to embed RTL8192EU NIC firmware.
nRF52/nice!nano needs none.
