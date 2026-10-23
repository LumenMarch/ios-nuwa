# ios-nuwa

**iOS 16 on the Xiaomi 13 Pro (`nuwa`, Snapdragon 8 Gen 2 / SM8550) — work in progress.**

This repository is a fork of [ios-sweet](https://github.com/dipa-studio/ios-sweet) (MIT),
retargeted from the Redmi Note 10 Pro (`sweet`, SM7150) to the Xiaomi 13 Pro
(`nuwa`, SM8550). The goal is the same: Apple's open-source XNU kernel
(xnu-8796, the iOS 16 generation) with the **unmodified iPhone X iOS 16.7.16
(20H392) userland** on top — launchd, the system daemons, backboardd,
SpringBoard and the stock apps — booted through a bare-metal shim and
`fastboot boot`, with the iOS root in the `userdata` partition.

> **Status 2026-10-23: it does not boot on a 13 Pro yet.**
> The tree is still ios-sweet's SM7150 code. There is no nuwa board support,
> loader/ADT port or nuwa driver in this repository. Do not flash images built
> from this tree to a Xiaomi 13 Pro. See [`docs/STATUS.md`](docs/STATUS.md)
> and [`docs/PORTING.md`](docs/PORTING.md).

## The device

| | Xiaomi 13 Pro (`nuwa`) | ios-sweet's target (`sweet`) |
| --- | --- | --- |
| SoC | SM8550-AB, 1×X3 + 2×A715 + 2×A710 + 3×A510, Adreno 740 | SM7150, 2×A76 + 6×A55, Adreno 618 |
| RAM | 8/12 GB LPDDR5X; upper bank at `0x8c0000000` | 6/8 GB; upper bank at `0x100000000` |
| Storage | UFS 4.0 | UFS 2.1/3.1 |
| Display | 6.73" 1440×3200 LTPO AMOLED, 120 Hz | 6.67" 1080×2400 AMOLED, 120 Hz |
| Touch | FocalTech FTS over SPI (`qupv3_se4_spi`), IRQ GPIO 25, reset GPIO 24, rails L6M/L12B | Goodix GT9896 over I²C (SE7) |
| PMIC | PM8550 / PM8550B / PMK8550 family | PM6150 / PM6150L |
| Boot | Unlocked ABL, `fastboot boot`, no UART | same |

The two boards share the ios-sweet architecture but almost none of the
board-specific values. Values from `sweet` and from the Xiaomi 13 (`fuxi`)
must not be copied into a nuwa build.

## What has to change

1. **Loader / shim** — new MMU map, live-FDT parsing, payload load addresses,
   framebuffer (`0xb8000000`, 1440×3200, stride 5760) and PSCI/SMP entry for
   SM8550.
2. **ADT** — memory window, GIC (`0x17100000` / `0x17180000`), UFS
   (`0x1d84000`), timer INTID 27, CPU MPIDR set, splash region, iPhone X
   identity, display geometry.
3. **Kernel platform** — SM8550 machine config, GICv3 Group-1 timer, PSCI
   CPU_ON, EPSS clock caps, watchdog stub.
4. **Storage** — UFS 4.0 host/PHY/gear and RPMh votes; HFS+ root in
   `userdata`.
5. **Display** — kernel IOSurface + IOMobileFramebuffer, CPU copy into ABL's
   splash buffer, presented to iOS as an iPhone X 1125×2436 screen inside the
   1440×3200 panel.
6. **Input** — FTS SPI touch (protocol, GPIOs, rails, `x-max=14400`,
   `y-max=32000`) and PM8550/PMK8550 power/volume keys.
7. **Root image** — same iPhone X 16.7.16 IPSW and `mkiosroot.sh` flow as
   ios-sweet; the bring-up tweaks are portable, the device identity is not.

## Plan

Staged, with a hard stop after each stage:

0. **Inventory** — dump the phone's live FDT, `/proc/iomem`, GPT and stock
   `boot.img`; fill `docs/STATUS.md` with measured values.
1. **Loader** — shim prints on the panel, finds the FDT and the splash buffer.
2. **Kernel entry** — XNU reaches the platform expert, timer and console.
3. **Storage** — UFS root device, HFS+ mount, launchd starts.
4. **Display** — IOSurface/IOMFB, first SpringBoard frame.
5. **Input** — touch and buttons, lock screen unlock.
6. **Polish** — CPU count, panel refresh, logs, stability.

The fastest honest route is to use **ios-fuxi** (the Xiaomi 13 / `fuxi`,
SM8550 port of ios-sweet) as the reference for the SM8550 board layer and
adapt its values to nuwa. It is a separate repository with unrelated history;
importing it is a deliberate step, not a drop-in merge.

## How it boots (inherited architecture)

```
ABL (stock Xiaomi bootloader, unlocked)
  -> fastboot boot shim-boot.img              (temporary; nothing is flashed to boot)
     -> shim: own MMU setup, reads the live FDT, loads the payload from the ramdisk
        -> XNU Mach-O + generated Apple Device Tree (ADT) + boot-args
           -> XNU -> UFS driver -> HFS+ root in the userdata partition
              -> dyld + shared cache -> launchd -> daemons -> backboardd + SpringBoard
```

There is no serial port on phones. The kernel draws its console into the
bootloader's framebuffer and keeps a persistent log ring at the end of
`userdata`; read it from recovery with `tools/sweet-log-pull.sh`.

## Safety

- Back up `persist`, `modemst1`, `modemst2`, `fsg`, `fsc`, `boot`, `dtbo`,
  `vbmeta` before anything else. They contain per-device calibration data.
- `fastboot boot` does not touch the boot partitions. Flashing the iOS root to
  `userdata` **erases Android's user data**.
- The resulting system has AMFI and sandbox restrictions switched off, a stub
  AppleKeyStore and no data protection. Do not put personal data on it.

## Read next

- [`docs/STATUS.md`](docs/STATUS.md) — what is done, what is measured, what is next.
- [`docs/PORTING.md`](docs/PORTING.md) — subsystem-by-subsystem port plan and values.
- [`docs/sweet-readme.md`](docs/sweet-readme.md) — the original ios-sweet README.
- [`NOTES.md`](NOTES.md) — inherited ios-sweet engineering log (sweet/SM7150; method, not values).

## Credits

Forked from [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet)
(MIT, Copyright © 2026 dipa-studio). Built on Apple's open-source releases
(xnu, IOHIDFamily, libpthread, hfs), [BearSSL](https://bearssl.org),
[blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build),
[ipsw](https://github.com/blacktop/ipsw), Qualcomm/Xiaomi's published Linux
kernel sources and [OrangeFox Recovery](https://orangefox.download).
iOS itself is Apple's and is not distributed here.

## License

MIT. See [`LICENSE`](LICENSE) and [`NOTICE.md`](NOTICE.md).
