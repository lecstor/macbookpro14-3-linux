# macbookpro14-3-linux

Notes for running Linux ([Omarchy](https://omarchy.org/) / Arch + Hyprland) on the 15"
Touch Bar MacBook Pro — **MacBookPro13,3** (2016) and **MacBookPro14,3** (2017), both T1.
Three pieces of hardware don't work out of the box — Wi-Fi, the internal microphone, and
the Escape key — and each fails in a way that looks like something else. This is what they
actually are, and the order to fix them in.

Written from two units, so some of it is panel-and-GPU lottery rather than universal. The
two share a hardware generation — Baffin/POLARIS11 dGPU `[1002:67ef]` driving the internal
panel, BCM43602 Wi-Fi, Cirrus CS8409 audio — so the findings carry across both models.
Machine-specific identifiers (UUIDs, IP addresses, keys) are redacted to placeholders —
**this repo is public.**

> The repo is named for the 14,3 because that is the unit it started on. It covers both.

## Two machines, A and B

- **Machine A** — healthy. Daily driver, hostname **`linmac`** (was `omarchy`; wiped and
  rebuilt 2026-09-10 after a macOS restore). Screen and GPU stable.
- **Machine B** — **MacBookPro13,3**, hostname `headless`. Failing display / discrete GPU:
  random hard reboots, vertical screen shake, horizontal tear lines through text, and
  repeated blank-screen episodes. The same fault appears during a macOS install, so it is
  **hardware**, not a driver bug.
  **As of 2026-09-10 the internal panel is considered dead** and B runs with no compositor
  at all — `multi-user.target`, `sddm` disabled, reached over SSH. See
  [Headless duty](rebuild-runbook.md#headless-duty--remote-access). Every software
  mitigation was tried and none worked; the four USB-C DisplayPort outputs are untested but
  present, so an external monitor is the remaining option if B ever needs a screen.

### Which unit am I on?

Ask DMI. It is the only answer that survives a reinstall:

```sh
cat /sys/class/dmi/id/product_name        # MacBookPro14,3 = A / MacBookPro13,3 = B
hostnamectl --static                      # linmac = A / headless = B
```

DMI is the reliable half. The hostname is not: A answered to `omarchy` until it was
rebuilt on 2026-09-10 and came back as `linmac`, so a hostname only tells you what the
current install was named.

Do **not** use the crash history to decide:

```sh
journalctl -k --no-pager | grep -c 'ring gfx timeout'   # 0 on a healthy unit
```

That returns `0` on a freshly reinstalled B as well — the journal only knows about the
current install. It tells you whether *this install* has crashed, not which unit you are on.

> **Resolved (2026-09-10, read off A itself):** A is `MacBookPro14,3`, board
> `Mac-551B86E5744E2388`, iGPU **HD Graphics 630** (Kaby Lake, `[8086:591b]`), dGPU
> subsystem **`0x106b0179`**. B is `MacBookPro13,3` with subsystem `0x106b0166`. The two
> are different models, as the runbook says. Two of the old A figures were wrong and are
> now corrected: the iGPU was recorded as HD 530 (that is B's Skylake part) and the dGPU
> subsystem as `0x106b3900` (matches neither unit).

**Scope of the findings:** the Wi-Fi, microphone, Escape-key and Touch ID gaps are
**generation-wide** — they apply to both models. The amdgpu instability and its kernel
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
| Random hard reboots | ⚠ **machine B only** — no recurrence since kernel 7.2.3, unproven | `amdgpu.dpm=0 amdgpu.aspm=0` |
| Screen shake / tear lines / blank-on-wake | ✗ **machine B only** — **unfixable in software**, panel abandoned | run headless (`sddm` disabled) |
| Touch ID / Touch Bar | ✗ T1 stuck in recovery (`05ac:1281`) on both units — **root cause: neither ever completed a macOS first boot**, so the T1 never activated and the backed-up FDR data is a never-activated snapshot | a completed macOS first boot regenerates real FDR data; failing that, the Linux-only USB activation (runbook route 3, Pass A) |
| T1Bridge stack | ✓ installed & verified on A (`0.1.7-1`) | signed `[standardagents]` repo; waiting on an activated T1 to do anything |
| Lid close suspends a headless box | ✓ fixed | `logind.conf.d` `HandleLidSwitch=ignore` |
| Reaching B once its panel fails | ✓ fixed | SSH + key + LAN-scoped `ufw` rule (runbook, Headless duty) |

## Conventions

- Canonical copy of these docs is this git repo. A rendered mirror may exist as a Claude
  artifact; treat it as read-only / possibly stale.
- Privileged commands in the runbook use `sudo` (assumes a terminal). In an agent context
  without a TTY, `pkexec <cmd>` substitutes.
