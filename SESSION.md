# Session: mido 4.9 NetHunter Kernel Kit

## What We Built
A NetHunter-family kernel kit for the Redmi Note 4 (mido) that enables:
1. **nRF52 / nRF52840 nice!nano** — BLE support (on-chip, no firmware blob needed)
2. **RTL8192EU** (TP-Link TL-WN821N v5, 2357:0107) — USB WiFi dongle that hot-plugs as `wlan1`

Built entirely on GitHub Actions, delivered as an AnyKernel3 flashable zip (flashed in TWRP, NOT the NetHunter app).

## Kit Location
```
/home/surfer/Work/mido-nethunter/
├── .github/workflows/build-mido-kernel.yml   (workflow)
├── .gitignore
├── firmware/rtlwifi/rtl8192eu_nic.bin         (31,818 B) — RTL8192EU NIC firmware
├── firmware/rtlwifi/README-firmware.md
├── patch/mido-defconfig-additions.cfg
├── patch/mido-nethunter-defconfig-additions.cfg (duplicate)
└── README.md
```

## Kernel Base
- **Repo:** `KudProject/kernel_xiaomi_msm8953-4.9`
- **Branch:** `lineage-17.1`
- **Defconfig:** `mido_defconfig`
- **GitHub user repo:** `us01srv-surf/mido-kernel.git`

## Build History

| # | Run | Commit | Status | Duration | Error / Notes |
|---|-----|--------|--------|----------|---------------|
| 1-4 | various | various | FAILED | various | Runner image retired, ARM32 toolchain, BTFM_SLIM, USB-UART |
| 5 | `35412751071` | `8bef6ff` | FAILED | 6m32s | `TRACE_INCLUDE_PATH .` — define_trace.h re-include fatal under GCC |
| 6 | `35420168569` | `dda53d3` | FAILED | 7m33s | TRACE fix OK (70+ headers), then `kgsl_events.c:18 <kgsl_device.h>` angle-bracket |
| 7 | `35426297691` | `2d6f686` | FAILED | 10m24s | Angle-bracket fix OK, then `msm_cam-legacy.h` → `"msm_isp.h"` missing `-I.../isp` |
| 8 | `35434833778` | `b0be070` | FAILED | 10m34s | isp -I added, then `msm_sensor.h` → `"msm_camera_i2c.h"` missing `-I.../sensor/io` |
| 9 | `35436418916` | `2c63587` | FAILED | 0s | YAML syntax error (heredoc inside `run: |` block) |
| 10 | `35436700257` | `ddb77bb` | **IN PROGRESS** | ~15m | YAML fixed; comprehensive `subdir-ccflags-y` approach. Was still running when session paused. |

### Fix Commits (chronological)
```
0b147a5  build: fix runner image ubuntu-20.04 -> 22.04
bc14a04  build: add CROSS_COMPILE_ARM32 toolchain for compat vDSO
115536c  defconfig: disable BTFM_SLIM (clang-only include)
8bef6ff  defconfig: add USB-UART bridge drivers (CP210x/CH341/FTDI/PL2303)
dda53d3  build: rewrite TRACE_INCLUDE_PATH '.' -> ../../<dir> for GCC
2d6f686  build: quote same-dir angle-bracket includes for GCC (kgsl_events.c)
b0be070  build: add -I.../camera_v2-legacy/isp for cross-dir quoted includes
2c63587  build: comprehensive camera_v2-legacy subdir-ccflags-y for GCC
ddb77bb  build: fix YAML syntax error (heredoc inside run: | block)
```

## Workflow Steps (10 total)
1. Checkout kit
2. Checkout KudProject mido 4.9 @ lineage-17.1
3. Install GCC9 aarch64 cross-compiler
4. Apply defconfig additions (nRF52 BLE + RTL8192EU)
5. Stage RTL8192EU firmware blob
6. Fix TRACE_INCLUDE_PATH for GCC (rewrite `#define TRACE_INCLUDE_PATH .` → `../../<dir>`)
7. Fix local angle-bracket includes (convert `<local.h>` → `"local.h"` for same-dir headers)
8. **Fix camera_v2-legacy cross-directory includes** — comprehensive `subdir-ccflags-y` injection
9. Build kernel (Image.gz-dtb) + modules
10. Package AnyKernel3 zip + upload artifact

## Root Cause Analysis: GCC vs Legacy MSM Camera Driver Includes

**The core problem:** GCC does NOT search the includer's directory chain for quoted includes (`#include "foo.h"`). Only two places are searched:
1. The directory of the file containing the `#include` directive
2. The `-I` paths

The legacy camera driver tree (`camera_v2-legacy/`) was compiled with a different compiler or had broader `-I` paths. Under GCC 9 (Ubuntu 22.04), cross-directory quoted includes fail unless every needed dir is explicitly on `-I`.

**Confirmed with local repro:** GCC 14 on host also fails the same way — when `b/main.c` includes `a/h1.h` which includes `"x.h"` where `x.h` lives in `b/`, GCC does NOT find `x.h` (only searches `a/` + `-I` paths).

**The class of failures seen:**
1. `include/trace/events/msm_cam-legacy.h` → `"msm_isp.h"` → needs `-I.../camera_v2-legacy/isp`
2. `camera_v2-legacy/sensor/msm_sensor.h` → `"msm_camera_i2c.h"` → needs `-I.../sensor/io`
3. Many more potential cross-directory chains exist in the legacy tree

**The comprehensive fix:** Inject `subdir-ccflags-y` into the top `camera_v2-legacy/Makefile` with the FULL union of all `-I` paths from every subdirectory Makefile. This propagates to ALL compilation units. Zero duplicate header basenames across the tree (verified) = no shadowing risk.

**The full include path union (15 paths):**
```makefile
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/sensor
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/sensor/io
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/sensor/cci
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/common
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/msm_vb2
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/camera
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/codecs
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/isp
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/pproc
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/pproc/cpp
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/msm_buf_mgr
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/fd
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/jpeg_10
subdir-ccflags-y += -Idrivers/media/platform/msm/camera_v2-legacy/jpeg_dma
```

Also fixed `isps` → `isp` typo in top Makefile.

## Defconfig Additions (`patch/mido-defconfig-additions.cfg`)
```ini
# nRF52840 nice!nano — BLE runs ON-CHIP, no firmware blob
CONFIG_BT=y
CONFIG_BT_LE=y
CONFIG_BT_HCIBTUSB=y
# nice!nano USB = UF2 bootloader (CDC-ACM) + ZMK serial console
CONFIG_USB_ACM=y

# RTL8192EU — NIC firmware embedded at build → dongle hot-plugs as wlan1
CONFIG_RTL8XXXU=y
CONFIG_RTL8XXXU_8192E=y
CONFIG_EXTRA_FIRMWARE="rtlwifi/rtl8192eu_nic.bin"
CONFIG_EXTRA_FIRMWARE_DIR="$(srctree)/firmware"
```

## Firmware Blob
- **File:** `rtl8192eu_nic.bin` (31,818 B)
- **Source:** `https://github.com/jackeyt/RTL-8XXX-Serial-Firmware/blob/master/rtl8192eu_nic.bin`
- **MD5:** `c8d25646cbc9efdcb818fc95ad5836a9`
- **Embedded at build time** via `CONFIG_EXTRA_FIRMWARE` — no userspace loader needed

## Artifact
- **Artifact name:** `mido-nethunter-kernel`
- **Zip name:** `Black-Hydra-Serp3n7-mido-4.9-nrf52-rtl8192eu.zip`

## Key Decisions Made
1. **Kernel base:** KudProject 4.9 mido tree @ lineage-17.1
2. **No out-of-tree rtl8192eu driver** — only in-tree `rtlwifi/rtl8xxxu`
3. **Firmware embedded at build** — `CONFIG_EXTRA_FIRMWARE` bakes the blob into kernel
4. **nRF52 needs NO firmware blob** — BLE runs on-chip
5. **Flash via TWRP** — AnyKernel3 zip, NOT the NetHunter app
6. **Comprehensive subdir-ccflags-y** over whack-a-mole per-directory fixes

## User's Device State
- **Model:** mido (Redmi Note 4/4X)
- **Running kernel:** 4.9.337 Black-Hydra-Serp3n7
- **adb:** wireless debugging at 192.168.55.74:5555
- **RTL8192EU dongle:** detected but probe fails (no firmware in old kernel)

## Next Steps When Continuing
1. **Check build #10** (run `35436700257`, commit `ddb77bb`) — should already be done
   - `gh run view 35436700257 --repo us01srv-surf/mido-kernel --json status,conclusion`
2. **If FAILED:** get fatal error with `gh run view <run> --log 2>&1 | grep "fatal error"`, patch, push
3. **If SUCCESS:** download artifact: `gh run download 35436700257 --repo us01srv-surf/mido-kernel --name mido-nethunter-kernel --dir /tmp/mido-kernel-zip`
4. **Flash on device:** `adb push /tmp/Black-Hydra-Serp3n7-mido-4.9-nrf52-rtl8192eu.zip /sdcard/` → TWRP → Install → reboot
5. **Verify:** `ip link show wlan1` + `dmesg | grep rtl8192` + `ip addr show wlan0` (regression check)

## Post-Flash Verification Commands
```bash
adb shell ip link show wlan1                    # should show wlan1 UP
adb shell dmesg | grep -i rtl8192              # should show fw loaded, no errors
adb shell ip addr show wlan0                    # regression check — wlan0/prima should still work
```
