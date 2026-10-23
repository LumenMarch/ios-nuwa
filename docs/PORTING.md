# Porting ios-sweet to Xiaomi 13 Pro (nuwa)

This is the working plan for turning the inherited ios-sweet tree into a nuwa
port. It is deliberately conservative: every board value must be measured on
the phone, and no value is copied from `sweet` or `fuxi`.

## Ground rules

1. **Codename is the key.** `nuwa` = Xiaomi 13 Pro. `fuxi` = Xiaomi 13.
   `sweet` = Redmi Note 10 Pro. Sibling boards do not share panel, touch,
   PMIC or memory values even when the SoC is the same.
2. **One stage at a time.** Do not flash a root image before the loader and
   kernel console work. Do not debug touch before the display works.
3. **`fastboot boot` only** for the shim and kernel. A root image goes to
   `userdata` and erases Android; back up `persist`, `modemst1`, `modemst2`,
   `fsg`, `fsc`, `boot`, `dtbo`, `vbmeta` first.
4. **No Apple binaries in git.** IPSW images, device trees, trust caches,
   firmware and dumps stay outside the repository (`.gitignore` covers
   `backup/`, `dtb/`, `recovery/`, `research/`).

## Measured values to collect first

From recovery or a rooted Android:

```sh
adb shell cat /sys/firmware/fdt > nuwa-live.dtb
adb shell cat /proc/iomem > nuwa-iomem.txt
adb shell cat /proc/partitions > nuwa-partitions.txt
adb shell ls -l /dev/block/by-name > nuwa-byname.txt
python3 tools/unpack_bootv2.py backup/boot.img dtb/boot_unpacked
```

Then extract from `nuwa-live.dtb`:

| What | Where in the FDT | Used by |
| --- | --- | --- |
| RAM banks and reserved-memory | `/memory`, `/reserved-memory/*` | ADT `dram-base`, `dram-size`, payload window |
| Splash framebuffer | `/reserved-memory/cont_splash_region` (or `splash_region`) | shim framebuffer, ADT Video |
| Panel geometry | panel node `qcom,mdss-dsi-panel-width/height` | framebuffer stride, iOS crop |
| GIC | `interrupt-controller@...` `reg` | ADT, kernel GIC |
| UFS | `ufshc@...` `reg` | ADT, UFS driver |
| Timer | `timer` `interrupts` | INTID |
| Touch | `fts@0` node, GPIOs, supplies | touch driver |
| Buttons | PMIC PON / GPIO keys | key driver |
| PSCI / restart | `/psci`, `restart@...`, `qcom,pshold` | loader, reboot path |

## Subsystem deltas

| Subsystem | ios-sweet (sweet) | nuwa target | Reference |
| --- | --- | --- | --- |
| SoC platform | SM7150, 2×A76 + 6×A55, GICv3, INTID 19 | SM8550, 1×X3 + 2×A715 + 2×A710 + 3×A510, GICv3, INTID 27 | fuxi (SM8550) |
| Machine config | `MASTER.arm64.sweet` | new `MASTER.arm64.nuwa` | fuxi's `MASTER.arm64.fuxi` |
| Memory | upper bank `0x100000000` + 0xfcc00000 | upper bank `0x8c0000000` + up to 9 GiB; start with 5 GiB (`0x140000000`) | fuxi (5 GiB bank at the same base) |
| Loader | `shim/` MMU/FDT/payload for sweet | new loader; SM8550 MMU map, PSCI, 1440×3200 framebuffer | `shim/fuxi-loader/` |
| UFS | `sweet_ufs.c`, UFS 2.1/3.1, 4 KiB | UFS 4.0 host/PHY, RPMh votes, 4 KiB | fuxi's UFS patches 0084–0088 |
| Display | `SweetFramebuffer.cpp`, 1080×2400, 1125×2436 logical | same services, 1440×3200 panel, 1125×2436 centered window; scaler optional | fuxi patches 0087/0093 |
| Touch | Goodix GT9896, I²C SE7, GT9896 protocol | FTS over SPI SE4, GPIO 25/24, L6M/L12B, x-max 14400 / y-max 32000; no firmware upload | fuxi `fuxi_touch.c` (patch 0091) |
| Keys | PM6150 PON + PM6150L GPIO2 | PM8550/PMK8550 PON + GPIO; read SIDs from FDT | fuxi patch 0092 |
| SMP | 6×A55 via PSCI | 8 cores; start with `cpus=1`, then 2–7 | fuxi patch 0090 |
| GPU | none | none for the first boot; Adreno 740 later | fuxi 0096/0098 (experimental) |
| Root | HFS+ in `userdata`, iPhone X 16.7.16 | same image and tools | ios-sweet `mkiosroot.sh` |

## Stage plan

### Stage 0 — inventory

Pass: a filled table in `docs/STATUS.md` with measured values and the exact
recovery/Android build the dump came from.

### Stage 1 — loader on the panel

Port the shim: MMU, live FDT, payload load, framebuffer, PSCI entry. The
loader must print its own diagnostics before XNU starts.

Pass: text appears on the 1440×3200 panel; the loader reports the FDT, the
splash region, RAM and the payload entry. If it reboots instantly, the memory
window overlaps a protected carve-out.

### Stage 2 — XNU entry

New machine config, GIC/timer, platform expert, early framebuffer console,
`cpus=1`.

Pass: kernel messages scroll; the timer ticks; no panic before the platform
expert. This is where most SM8550-specific kernel work lands.

### Stage 3 — storage and userland

UFS host driver, root device, HFS+ mount, launchd.

Pass: `launchd` runs, the shared cache maps, Setup or SpringBoard starts.
Use the ios-sweet root image; the device identity is still iPhone X.

### Stage 4 — display

IOSurface + IOMobileFramebuffer services; CPU copy into ABL's framebuffer;
present 1125×2436 inside the 1440×3200 panel.

Pass: backboardd swaps, SpringBoard draws, the lock screen is visible.

### Stage 5 — input

FTS touch and PM8550 keys; HID digitizer/consumer events.

Pass: swipe to unlock; volume keys work; the home gesture (if implemented)
returns to the springboard.

### Stage 6 — polish

CPU count, EPSS clock caps, panel refresh, persistent log, crash reports.

## Board-code source decision

Two honest options:

1. **Import the SM8550 layer from `ios-fuxi`** (MIT-compatible, same lineage,
   already boots on the Xiaomi 13). This is the shortest path: its loader,
   ADT builder, UFS, SMP, touch and key code are the right shape; nuwa needs
   value changes, not a rewrite. It has unrelated git history, so this is a
   deliberate import commit with attribution, not a merge.
2. **Write the SM8550 layer from scratch** in this repository. Cleaner
   history, months of work, and the same bugs will be rediscovered.

Record the decision here before starting stage 1.

## Open questions

- Does nuwa's ABL leave the panel on with continuous splash and autorefresh,
  as fuxi relies on?
- Exact splash base/size and stride at 1440×3200.
- UFS 4.0 gear/PHY values and whether the ABL-configured host can be reused.
- EPSS clock domain and safe caps for the X3/A715/A710 clusters.
- Touch controller firmware/protocol differences from fuxi (`st_fts_m3.ftb`
  vs `st_fts_l1.ftb`); ios-sweet's FTS driver does not upload firmware.
- PM8550/PMK8550 PON and GPIO SIDs for power/volume.
