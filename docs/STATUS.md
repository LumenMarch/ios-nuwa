# STATUS

Updated: 2026-10-23. This file is the single source of truth for the port;
`NOTES.md` is the inherited ios-sweet log and is historical.

## Current state

- Repository: `LumenMarch/ios-nuwa`, a fork of `dipa-studio/ios-sweet` (MIT).
- Code: **unchanged ios-sweet tree** (sweet / SM7150). Only documentation and
  naming have been changed so far.
- Device: no nuwa build, no nuwa loader, no nuwa ADT, no boot test.
- Phone: not touched by this repository. Bootloader is unlocked.

**It does not boot on a Xiaomi 13 Pro.** The next milestone is not a boot; it
is a measured board inventory (stage 0).

## Verified facts (with source)

| Item | Value | Source |
| --- | --- | --- |
| Codename / SoC | `nuwa` / SM8550-AB, Adreno 740 | LineageOS wiki `devices/nuwa` |
| Display | 1440×3200 LTPO AMOLED, 120 Hz | LineageOS wiki, GSMArena |
| Touch | FocalTech FTS, `compatible = "st,spi"`, on `qupv3_se4_spi`; IRQ `tlmm 25`, reset `tlmm 24`; `avdd = L6M`, `vdd = L12B`; `x-max = 14400`, `y-max = 32000` | LineageOS `android_kernel_xiaomi_sm8550-devicetrees`, `qcom/nuwa-sm8550.dtsi` |
| PMIC family | PM8550 / PM8550B / PMK8550 | same devicetree (PM8550B GPIOs, PMK8550 VADC) |
| RAM (upper bank) | `0x8c0000000` + ~9216 MiB, 14 segments; low DRAM starts at `0x811d0000` | `wh1stl3/UniXNU-nuwa`, `docs/inherited/nuwa-observed.json`, `iomem.txt` |
| UEFI base | Mu-Silicium has `Platforms/Xiaomi/nuwaPkg`; Display, Internal Storage, Side Buttons, USB device, Mass Storage and Linux boot are marked working; project state `Inactive` | `Project-Silicium/Mu-Silicium`, `Status.md` |
| Sibling port | Xiaomi 13 (`fuxi`, same SM8550) boots iPhone X 16.7.16 to the SpringBoard home screen with FTS touch and PMK8550 keys | `bangprovn/ios-fuxi` |
| XNU baseline | xnu-8796.141.3, 16 KiB pages, iPhone X `iPhone10,3` / `D22AP` userland 16.7.16 (20H392) | ios-sweet tree |

## Expected but unverified

These are plausible from the SM8550 family and the Xiaomi 13 sibling; they
must be read from the live device, not assumed:

- splash framebuffer at `0xb8000000` (size and stride unknown);
- GIC `0x17100000` / `0x17180000`, timer INTID 27;
- UFS host at `0x1d84000`, UFS 4.0 gear/PHY values;
- button PON/GPIO SIDs and the volume-up GPIO;
- EPSS clock domain limits;
- whether ABL keeps the 1440×3200 panel alive with continuous splash and
  autorefresh (ios-fuxi relies on this on fuxi).

## Blockers

1. No nuwa board code in this tree. The SM8550 platform, UFS 4.0, FTS SPI
   touch, PM8550 keys and 1440×3200 display path all need to be written or
   imported.
2. No live board inventory yet (FDT, iomem, GPT, stock boot.img).
3. The inherited CI workflow builds `sweet`; it has not been retargeted.

## Next actions

1. Boot a recovery on the phone and collect:

   ```sh
   adb shell cat /sys/firmware/fdt > nuwa-live.dtb
   adb shell cat /proc/iomem > nuwa-iomem.txt
   adb shell cat /proc/partitions > nuwa-partitions.txt
   adb shell ls -l /dev/block/by-name > nuwa-byname.txt
   ```

   Unpack the stock `boot.img` for its DTB (`tools/unpack_bootv2.py`).
2. Run the FDT inspector and record the real values in this file:
   `tools/sweet-inspect-dtb.py` (will be forked to `tools/nuwa-inspect-dtb.py`).
3. Decide the board-code source: import the SM8550 layer from `ios-fuxi`
   (recommended) or write it from scratch. Record the decision and its
   license/attribution consequences.
4. Only then start stage 1 (loader on the panel).

## Non-goals for now

- Wi-Fi, Bluetooth, cellular, audio, camera, sensors, GPU acceleration.
- A daily-driver phone. Even ios-sweet on its own device is a bring-up demo.
