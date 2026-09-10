# macbookpro14-3-linux

Notes for running Linux ([Omarchy](https://omarchy.org/) / Arch + Hyprland) on
**MacBookPro14,3** (15", 2017, Touch Bar, T1). Three pieces of hardware don't work out of
the box — Wi-Fi, the internal microphone, and the Escape key — and each fails in a way that
looks like something else. This is what they actually are, and the order to fix them in.

Written from two units of the same model, so some of it is panel-and-GPU lottery rather
than universal. Machine-specific identifiers are redacted to placeholders.

## Two machines, A and B

These notes come from two units of the same model, distinguished by condition rather than
by any identifier — identifiers change every reinstall, symptoms don't:

- **Machine A** — healthy. Daily driver. Screen and GPU stable.
- **Machine B** — failing display / discrete GPU: random hard reboots, vertical screen
  shake, and a blank-screen episode. The same fault appears during a macOS install, so it
  is **hardware**, not a driver bug. Retired to headless duty.

If you are unsure which you are on, ask the journal:

```sh
journalctl -k --no-pager | grep -c 'ring gfx timeout'   # 0 on a healthy unit
```

**Scope of the findings:** the Wi-Fi, microphone, Escape-key and Touch ID gaps are
**model-wide** — they apply to any MacBookPro14,3. The amdgpu instability and its kernel
params are **B-only**, one unit's dying GPU; don't apply them to a healthy machine, where
they cost fixed GPU clocks and extra heat for nothing.

## Status at a glance

| Thing | State | Fix |
|---|---|---|
| Internal Wi-Fi (BCM43602) | ✗ can't associate with WiFi 6E/7 APs (2015 firmware) | USB adapter (MT7612U / MT7921AU) |
| Internal mic (CS8409 / CS42L83) | ✓ fixed | `snd-hda-macbookpro-dkms-git` + gain |
| Escape key | ✓ remapped | Hyprland `SUPER+TAB` → Escape |
| Speakers | ✓ work via generic HDA | — |
| Keyboard / touchpad | ✓ in-kernel `applespi` | (do **not** install `macbook12-spi-driver-dkms` — breaks on kernel 7.x) |
| Random hard reboots + screen shake | ⚠ **machine B only** — mitigated, observing | `amdgpu.dpm=0 amdgpu.aspm=0 amdgpu.dcdebugmask=0x10` |
| Touch ID | ✗ needs original `EFI/APPLE` FDR data | back it up from macOS **before** wiping (runbook Step 0) |

## Conventions

- Canonical copy of these docs is this git repo. A rendered mirror may exist as a Claude
  artifact; treat it as read-only / possibly stale.
- Privileged commands in the runbook use `sudo` (assumes a terminal). In an agent context
  without a TTY, `pkexec <cmd>` substitutes.
