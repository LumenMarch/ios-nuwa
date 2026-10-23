# Working on ios-nuwa

- Read `docs/STATUS.md` first and `docs/PORTING.md` before touching code.
  `NOTES.md` is the inherited ios-sweet log (sweet / SM7150): use its method,
  never its board values.
- The target is `nuwa` (Xiaomi 13 Pro, SM8550). `fuxi` (Xiaomi 13) is a
  sibling, not the same board. Measure values from the phone's live FDT and
  record them in `docs/STATUS.md`; distinguish measured from guessed.
- Phone work uses an unlocked bootloader and temporary `fastboot boot` images.
  Flashing the iOS root to `userdata` erases Android. Back up `persist`,
  `modemst1`, `modemst2`, `fsg`, `fsc`, `boot`, `dtbo`, `vbmeta` first, and
  confirm each destructive step with the user before running it.
- `fastboot boot` after a large `userdata` flash often hangs in Xiaomi's ABL;
  re-enter fastboot by hand (Power + Volume down) instead of retrying commands.
- The user switches agents frequently. Update `docs/STATUS.md` and the handoff
  at the top of `NOTES.md` after each milestone and before stopping. Record
  commits, CI run IDs, exact build/boot results and unfinished work.
- Commit completed changes. Use `gh` for GitHub Actions kernel builds and logs.
  Local single-file cross-compiles are useful before spending a CI run.
- Keep device dumps, serial numbers, calibration data and downloaded Apple
  artifacts out of git.
- The inherited CI workflow builds the `sweet` configuration. Do not treat its
  artifacts as nuwa kernels until the workflow and machine config are
  retargeted.
