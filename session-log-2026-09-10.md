# Session log — 2026-09-10

Work done on `omarchy` (MacBookPro14,3) with Claude Code. Chronological, with the
measurements that drove each decision. Feeds into `rebuild-runbook.md`.

## Goal

Start: "get voxtype set up with the capslock key." Ended up also fixing a dead internal
mic, removing a broken DKMS module, and diagnosing + mitigating random hard reboots and a
screen shake.

## 1. voxtype hotkey → CapsLock

Starting state:
- voxtype `1.0.1`, service running.
- `~/.config/voxtype/config.toml` had `[hotkey] enabled = false` (comment: "configured in
  Hyprland") and `hotkey.key = SCROLLLOCK`.
- Only Hyprland reference was a **commented-out** line: `-- o.bind("SUPER + H", nil,
  "voxtype record toggle")` — so nothing was actually bound.
- User **not** in the `input` group (`id -nG` → `$USER wheel`).

Changes:
- `voxtype config set hotkey.enabled true`
- `voxtype config set hotkey.key CAPSLOCK`
- Tidied the now-stale comment in `config.toml`.
- `pkexec usermod -aG input $USER` → `getent group input` shows `$USER` (session still
  needs reboot to pick it up).
- Reboot. CapsLock now triggers voxtype.

## 2. Internal mic was dead

After reboot, voxtype "couldn't hear" anything. Diagnosis:

| check | result |
|---|---|
| `dkms status` | empty |
| `journalctl -k -b \| grep cs8409` | stock driver, **no** "trying APPLE" |
| `pacman -Q linux-headers` | not installed |
| 4 s capture from `alsa_input.pci-0000_00_1f.3.analog-stereo` | `peak=32768` (railed), `mean=-6457`, only-negative signal, 5101/32000 samples at the negative rail, rest a −1..−3 noise floor |
| ALSA mixer (`Internal Mic`, `Internal Mic Boost`) | already maxed (+12 dB / +20 dB, on) |

So: not gain, not mute — the CS42L83 sub-codec was never initialised. Runbook defect #2.

## 3. Install headers — collateral: broken SPI DKMS module

- Repo `linux-headers` = `7.2.3.arch1-3`, exactly matching the running kernel and installed
  `linux` — safe to install now, gotcha #1 did not apply.
- `pkexec pacman -S --noconfirm --needed linux-headers` — also pulled `pahole`.
- The post-install DKMS rebuild hit **`macbook12-spi-driver/0+git.315`**, which failed:
  ```
  apple-ibridge.c:901: error: 'struct acpi_driver' has no member named 'owner'
  applespi.c:64: fatal error: asm/unaligned.h: No such file or directory
  ```
  Both are kernel-7.x API changes (`.owner` removed from `acpi_driver`; `asm/unaligned.h` →
  `linux/unaligned.h`). Module state was only `added`, never `installed` — it has never
  built on this kernel.
- Not needed: in-kernel `applespi` is loaded and provides `Apple SPI Keyboard` +
  `Apple SPI Touchpad`. Runbook deliberately avoids the SPI/iBridge route.
- Removed: `pkexec pacman -R --noconfirm macbook12-spi-driver-dkms` (plain `-R`; `-Rns`
  wanted to take `dkms` too). UKI rebuilt by the limine hook. Keyboard/touchpad unaffected.

## 4. Apple audio driver

- `yay -S snd-hda-macbookpro-dkms-git` (user ran it in a real terminal).
- `dkms status` → `snd-hda-macbookpro/0.1, 7.2.3-arch1-3, x86_64: installed (Original
  modules exist)`.
- Reboot. `journalctl -k -b | grep cs8409`:
  ```
  snd_hda_intel: Primary patch_cs8409 NOT FOUND trying APPLE
  ```
  Module loads from `updates/dkms/snd-hda-codec-cs8409.ko.zst`. Kernel tainted (expected,
  unsigned).

## 5. Mic gain + verification

- `pactl set-source-volume alsa_input.pci-0000_00_1f.3.analog-stereo 900%`
- `amixer -c PCH sset 'Internal Mic' 100%` / `'Internal Mic Boost' 100%` (already were).

Re-measured 4 s capture:

| | before driver | after driver + gain |
|---|---|---|
| peak | 32768 (railed) | 17358 |
| railed samples | 5101, negative only | 0 |
| mean (DC bias) | −6457 | 1040 |
| shape | silent floor + rail glitches | real varying waveform |

voxtype now transcribes. Done.

## 6. Random hard reboots — diagnosis

User reported the machine randomly reboots, "numerous times," possibly not since the
Omarchy update (which bumped the kernel).

`journalctl --list-boots` — 12 boots. Classified by whether the boot ended with a clean
shutdown sequence (`Unmounted /home`, "Reached target Shutdown", etc.):

| boot | length | ending |
|---|---|---|
| −11, −9, −5, −3 | mins–hrs | clean (normal reboots, incl. today's controlled ones) |
| −10 | 76 s | crash — journal stops mid wifi-scan |
| −8 | 5 s | crash — stops as the user session starts |
| −7 | 5 min | crash — **amdgpu ring gfx timeout** (see below) |
| −6 | 57 s | crash — stops the instant Chromium launches |

Boot −7 (2026-09-09 ~10:09), last ~15 s of log:
```
amdgpu 0000:01:00.0: ring gfx timeout, but soft recovered
amdgpu 0000:01:00.0: [drm] AMDGPU device coredump file has been created
  ... x9, every ~2 s ...
```
then the log stops with no shutdown → hard reset.

- No MCE, no thermal-critical, no kernel panic in any of the 12 boots.
- `quiet loglevel=0 splash` on the cmdline means the unlogged instant resets (−6/−8/−10)
  leave no trace — consistent with a harder GPU hang but not proven.
- GPUs: `card0` = i915 (HD 530), **all outputs disconnected**. `card1` = amdgpu
  (POLARIS11, Radeon Pro 555), `card1-eDP-1: connected` → **the internal panel is on the
  AMD GPU**. Cannot blacklist amdgpu.
- Timeline: **every crash was on kernel 7.1.9.** Today's `omarchy update` (`linux
  7.1.9.arch1-2 → 7.2.3.arch1-3` at 08:22) may have fixed it; not enough uptime to know.

## 7. Screen shake (reported mid-session)

User: the display started shaking / vertical jitter, slight; has previously worsened to
unusable, needing a reboot.

Live checks:
- `power_dpm_force_performance_level = auto`, but `pp_dpm_sclk` pinned at level 7
  (855 MHz `*`) with `gpu_busy_percent = 0` — DPM sitting at max clock while idle.
- `amdgpu.dcdebugmask = 0` (PSR **not** disabled), `dc = -1`, `dpm = -1` (all default).
- eDP-1 mode `2880x1800@60.00100` (Hyprland). No new kernel errors during the shake.
- Assessment: eDP timing / PLL instability — most likely **PSR** and/or DPM clock
  transitions. Same subsystem as the reboots.

Stopgap applied (reversible, no reboot):
`pkexec sh -c 'echo high > /sys/class/drm/card1/device/power_dpm_force_performance_level'`
→ pins clocks so DPM stops transitioning. **User to confirm whether the shake stopped.**

## 8. amdgpu kernel params (durable fix, pending reboot)

Edited `/etc/default/limine` (backup at `/etc/default/limine.bak.<epoch>`), added:
```sh
KERNEL_CMDLINE[default]+=" amdgpu.dpm=0 amdgpu.aspm=0 amdgpu.dcdebugmask=0x10"
```
- `amdgpu.dpm=0` — reboot trigger + PLL-glitch source
- `amdgpu.aspm=0` — GPU PCIe ASPM
- `amdgpu.dcdebugmask=0x10` — disable PSR (the shake)

`pkexec limine-update` → UKI rebuilt, `/boot/EFI/Linux/omarchy_linux.efi`. Confirmed the
embedded `.cmdline` section now begins with the three params.

**Not yet active — needs a reboot.**

## 9. Blank-screen episode (boot -1, 2026-09-10 09:04–09:40)

Machine left locked/idle; user returned to a blank screen that would not wake (backlight
lit on keypress, no image), force-powered-off after ~2 min of power-button presses.

Boot -1 log analysis:
- System was **fully alive** throughout — `man-db` cron ran and finished at 09:37;
  `systemd-logind` logged every `Power key pressed short` (09:38:54 → 09:40:35).
- Timeline: lock 09:30 → screensaver 09:33 → lock-timeout 09:35 → **idle-monitor: active
  09:38:46** (user back) → blank → power presses → off.
- **Zero amdgpu / drm / i915 errors in the entire boot.** No devcoredump, no pstore.
- Not the ring-gfx-timeout hang. This is amdgpu failing to re-drive the eDP panel coming
  out of the screensaver/DPMS blank — a display-wake failure, machine underneath was fine.
- Boot -1 ran the **stock cmdline** — the `dcdebugmask` edit landed later in the session.
  Only `power_dpm_force_performance_level=high` (manual, non-persistent) was active, and it
  did not help.

Current boot (0) has all three params active (`dpm=0 aspm=0 dcdebugmask=16`); amdgpu init
clean ("Display Core v3.2.384 initialized on DCE 11.2", no errors).

**Decision:** leave idle/blanking enabled as-is and test whether the blank-on-wake recurs
now that PSR is disabled. Do NOT pre-emptively disable the screensaver — that only masks
the trigger. If it recurs, disabling screen-off in `~/.config/omarchy/shell.json`
(`screensaver: 150`, `lock: 300`) is the fallback.

If it recurs: try `Ctrl+Alt+F2` for a text console before power-cycling; then after reboot
`journalctl -k -b -1 | grep -iE 'amdgpu|drm|dc_|atom|link train|dpms' | tail -40`.

## Open items

- [x] amdgpu params active as of boot 0 (2026-09-10 09:41): `dpm=0 aspm=0 dcdebugmask=16`.
- [ ] Watch for recurrence of the blank-on-wake and/or the shake. Success = 1–2 weeks of
      normal idle/wake cycles with neither. Capture `journalctl -k -b -1` if it recurs.
- [ ] If stable for weeks, optionally pull `amdgpu.dpm=0` back out to learn whether kernel
      7.2.3 alone is enough (keep `dcdebugmask=0x10`).
- [ ] **Apple EFI / Touch ID:** disk confirmed Linux-only on 2026-09-10 (2 GB Linux ESP + LUKS root; no macOS, no `EFI/APPLE`). User is
      going to reinstall macOS from scratch (Internet Recovery) to regenerate the FDR data,
      back up `EFI/APPLE`, then re-install Omarchy from the runbook. This Linux install and
      all live diagnostic access end at that wipe — everything learned is in this repo.
- [ ] voxtype minor: "voxtype" self-transcribes as "box type" / "Apache" — consider a
      `[text] replacements` entry in `config.toml` if it matters.
