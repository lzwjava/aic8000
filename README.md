# AIC8800 Linux Driver (ported to kernel 7.0)

AIC (Aic8800 / 8800DC) dual-mode **Wi-Fi + Bluetooth** driver working over **USB** and
**SDIO** interfaces, plus the `aicrf_test` host-side RF/smoke test tool.

This tree is a working drop of the original
`aic8800fdrvpackage_amd64_2023_0807.deb` driver sources (2021–2022 vintage), patched
so that the modules **build, install and run on Linux kernel 7.0** (Ubuntu 26.04,
`7.0.0-34-generic`, x86_64).

Verified live on `lzw@192.168.1.133`.

---

## Repository layout

```
├── kernel-7.0.patch          # unified diff of every change made for kernel 7.0
├── release_note.txt          # vendor release notes (2022_1219_1126 and older)
├── aic.rules                 # udev rule: eject USB MSC dongle (idVendor a69c)
├── drivers/
│   └── aic8800/
│       ├── Makefile          # top-level kbuild Makefile (platform switch)
│       ├── aic_load_fw/      # module: firmware/Bluetooth loader  → aic_load_fw.ko
│       └── aic8800_fdrv/     # module: Wi-Fi driver               → aic8800_fdrv.ko
├── fw/
│   └── aic8800DC/            # 8800DC firmware + radio patch blobs
└── aicrf_test/               # host-side test tools (wifi_test, bt_test)
```

## Why the port was needed

The stock `aic8800fdrvpackage_amd64_2023_0807.deb` **would not compile** against
kernel 7.0: the 2021–2022 driver source relies on kernel APIs that were removed or
renamed in the 6.x→7.0 window. The tree was brought up to date through **4 rounds of
kernel-API patches**, then built, installed and loaded.

## Patches applied (all under `drivers/aic8800/aic8800_fdrv/`)

| File | Change |
|---|---|
| `rwnx_rx.c` | `del_timer`/`del_timer_sync` → `timer_delete`/`timer_delete_sync`; `from_timer()` → `container_of()`; `in_irq()` → `in_hardirq()`; added missing `mesh_control` arg to `ieee80211_amsdu_to_8023s`; added `link_id` to `cfg80211_rx_spurious_frame` / `cfg80211_rx_unexpected_4addr_frame` |
| `aicwf_sdio.c` | timer API + `from_timer` fixes (same API churn) |
| `rwnx_main.c` | `wdev->mtx` → `wiphy_lock()` / `wiphy_unlock()`; updated 5 cfg80211 op signatures: `change_beacon` (→ `cfg80211_ap_update`), `set_monitor_channel`, `set_wiphy_params`, `set_tx_power`, `start_radar_detection` (all gained new params in 7.0) |
| `rwnx_mod_params.c` / `rwnx_compat.h` | `REGULATORY_IGNORE_STALE_KICKOFF` removed in 7.0 → compat `#define 0` for kernels ≥ 6.9 |
| `rwnx_radar.c` | `cfg80211_cac_event` gained `link_id` arg |

The complete unified diff is in [`kernel-7.0.patch`](kernel-7.0.patch).

## Result

- `aic_load_fw.ko` and `aic8800_fdrv.ko` compile cleanly and were installed to
  `/lib/modules/7.0.0-34-generic/kernel/drivers/net/wireless/aic8800/` via
  `make install` + `depmod`.
- Firmware (`aic8800DC` blobs) lives under `/lib/firmware/`; the udev rule
  (`aic.rules`) ejects the USB MSC dongle on insertion so the wireless device takes
  over. The USB device (vendor `a69c`) is handled by the AIC driver, interface comes
  up as `wlx…`.

## Build & install (native Ubuntu / Debian)

```bash
# kernel headers for the running kernel are required
sudo apt install linux-headers-$(uname -r) build-essential

cd drivers/aic8800
make                          # builds both kernel modules
sudo make install             # installs .ko, runs depmod
```

Load the modules and verify:

```bash
sudo modprobe aic_load_fw
sudo modprobe aic8800_fdrv
ip link show | grep wlx       # confirm the wlx… interface came up
```

## Firmware + udev setup

```bash
# udev rule — eject the USB MSC (mass-storage) dongle so the Wi-Fi device is exposed
sudo install -m 644 aic.rules /etc/udev/rules.d/90-aic.rules
sudo udevadm control --reload-rules && sudo udevadm trigger

# firmware — copy the 8800DC blobs where the driver loads them from
sudo mkdir -p /lib/firmware/aic8800DC
sudo install -m 644 fw/aic8800DC/* /lib/firmware/aic8800DC/
```

## Test tools

```bash
cd aicrf_test
make            # produces wifi_test, bt_test (optionally cross-compiled)
```

## Notes / caveats

- Kernel modules are **per-kernel**: after a kernel upgrade you must rebuild and
  reinstall (`make && sudo make install`) with the source and, if needed,
  re-apply `kernel-7.0.patch`. The live changes are **not** repackaged into a `.deb`.
- Vendor `Makefile` also has Rockchip/Allwinner/Amlogic Android platform presets
  (arm/arm64 cross-compile); Ubuntu/x86_64 is selected by `CONFIG_PLATFORM_UBUNTU := y`.
- `REGULATORY_IGNORE_STALE_KICKOFF` (`rwnx_compat.h`) is only a compile-time compat
  shim for kernels ≥ 6.9; ignore the actual flags-scan message about removal.