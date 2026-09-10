# Session log — 2026-09-10 (machine B, `headless`)

Second session of the day, on the **other** unit: a fresh Omarchy install being set up for
headless duty with Claude Code. Where the [earlier log](session-log-2026-09-10.md) was
diagnosis on a running system, this is a rebuild from the runbook — plus the remote-access
work the failing panel forced to the front.

## Starting state

Fresh Omarchy install, ~6 minutes of uptime, nothing from the runbook applied:

| check | result |
|---|---|
| `cat /sys/class/dmi/id/product_name` | **`MacBookPro13,3`** — not the 14,3 the repo is named for |
| `hostnamectl --static` | `headless` |
| `uname -r` / `pacman -Q linux` | `7.2.3-arch1-3` / `7.2.3.arch1-3` — matched, `checkupdates` empty |
| `journalctl -k \| grep -c 'ring gfx timeout'` | **0** — no crash history on this install |
| `pacman -Q macbook12-spi-driver-dkms` | installed (the broken one) |
| `pacman -Q linux-headers` | not installed |
| `dkms status` | empty — no audio driver, mic dead |
| `/proc/cmdline` | stock; no `amdgpu.*` params |
| `id -nG` | `lecstor wheel` — no `input` group |
| network | USB CDC-NCM ethernet dongle `[0b95:1790]` up; internal `wlp3s0` down, as expected |
| `sudo` | needs a password; `pkexec` works without prompting (polkit + wheel) |

Two things worth recording about identification:

- **The DMI model contradicts the repo.** `MacBookPro13,3`, board `Mac-A5C67F76ED83108C`,
  dGPU PCI subsystem `0x106b0166` — against the `MacBookPro14,3` / `0x106b3900` in the
  original notes. Same hardware generation either way (Baffin `[1002:67ef]` on `eDP-1`,
  BCM43602, CS8409), so no fix changes. Left flagged rather than silently relabelled.
- **The "which unit am I on" heuristic in the README does not survive a reinstall.** The
  `ring gfx timeout` count is 0 here, which would call this a healthy unit. The journal only
  knows about the current install. DMI is the durable answer.

## 1. Remote access — pulled forward

Mid-session the user reported the screen "seems to be getting worse", making SSH the
priority: the machine has to outlive its panel. Full detail is now in the runbook's
[Headless duty](rebuild-runbook.md#headless-duty--remote-access) section.

- `~/.ssh/authorized_keys` seeded from the user's GitHub public keys (`gh api
  /users/<user>/keys`). It is a 2048-bit `ssh-rsa` key — fine with OpenSSH 10.5, which
  negotiates `rsa-sha2-*`, but an `ed25519` key would be the better long-term answer.
- `sshd` enabled and started (`openssh 10.5p1-1` was installed but the unit was
  **disabled**; host keys did not exist and were generated on first start).
- Drop-in `/etc/ssh/sshd_config.d/10-headless.conf`: `PermitRootLogin no`, keepalives
  60s x 3. **Password auth deliberately left on** until key login is confirmed from the
  other machine — no lockout on a laptop with a dying screen.
- `ufw` was already **active, deny incoming** — enabling `sshd` alone would have left the
  port unreachable. Added a LAN-scoped rule for 22/tcp, not a world-open one.
- Lid: logind's default `HandleLidSwitch=suspend` would take a closed-lid headless box off
  the network. Drop-in sets all three lid options to `ignore`;
  `systemctl reload systemd-logind` applied it **live**, confirmed over D-Bus
  (`HandleLidSwitch` -> `s "ignore"`). Reload, not restart — restart can kill sessions.
- Checked whether idle was also a risk: it is not. Omarchy's idle service
  (`shell/plugins/services/idle/Service.qml`) implements only screensaver and lock timers,
  with no suspend action, so an idle box stays reachable.
- `avahi` + `nss-mdns` were already active, so `<hostname>.local` resolves on the LAN —
  better than the DHCP address.

Local check `ssh -o BatchMode=yes <user>@<own-ip> true` returned
`Permission denied (publickey,password)`: sshd listening, both methods offered. That does
**not** exercise the `ufw` rule — loopback skips it — so the real test is the first
connection from the other machine.

## 2. Runbook, stability + cleanup scope

Scope chosen for a headless box: skip audio, voxtype and the Escape binding (all
display/desk conveniences), do the stability work.

- **Removed `macbook12-spi-driver-dkms` first, then installed `linux-headers`** — the
  reverse of the original step order. Doing it this way, the headers' post-install DKMS
  pass had nothing broken to compile and ran clean; no `acpi_driver has no member 'owner'`
  failure to read and dismiss. Runbook reordered to match.
- `linux-headers 7.2.3.arch1-3` — matches `linux` and `uname -r` exactly; the partial-upgrade
  gotcha did not apply. `dkms status` still empty afterwards (nothing left to build).
- Live stopgap for the shake: `power_dpm_force_performance_level` -> `high`, pinning GPU
  clocks so DPM stops transitioning. No reboot, reversible with `auto`.
- Durable fix: appended `amdgpu.dpm=0 amdgpu.aspm=0 amdgpu.dcdebugmask=0x10` to
  `/etc/default/limine` (backup at `/etc/default/limine.bak.<epoch>`), ran `limine-update`,
  and confirmed the params are embedded in `/boot/EFI/Linux/omarchy_linux.efi` by reading
  the string back out of the UKI. **Pending reboot.**

Note the difference from the earlier session: there, the params were a response to logged
`ring gfx timeout` crashes. Here they are **prophylactic** — this install has no crash
history at all, and the justification is the user's report of the shake plus the unit's
known history.

## 3. Touch ID / EFI status on B

Re-checked after the Omarchy reinstall: `/boot` is a 2 GB vfat ESP holding only `BOOT`,
`limine` and `Linux` — **no `EFI/APPLE`**.

> **Corrected later the same day.** I read the empty ESP as "the macOS restore never
> happened here". It did: B was restored, its `EFI/APPLE` was backed up to a USB key, and
> the Omarchy install afterwards rewrote the ESP. The USB key holds the only copy of B's
> FDR data. The lesson for the runbook: an empty ESP dates the last thing that wrote to the
> disk, and says nothing about whether a restore preceded it.

## 4. Post-reboot verification

Rebooted the same day. Everything took, and everything persisted:

| check | result |
|---|---|
| `/proc/cmdline` | starts with the three `amdgpu.*` params |
| `/sys/module/amdgpu/parameters/{dpm,aspm,dcdebugmask}` | `0` / `0` / `16` |
| amdgpu init | clean — `Display Core v3.2.384 initialized on DCE 11.2`, no errors |
| `ring gfx timeout` this boot | 0 |
| `card1-eDP-1` | still `connected` — panel driven normally with PSR off |
| `systemctl is-active sshd` | `active`, listening on 22 (v4 + v6) |
| `HandleLidSwitch` over D-Bus | `s "ignore"` |
| `ufw status` | LAN rule for 22/tcp intact |

The kernel log also confirms B's dGPU identity independently of DMI:
`initializing kernel modesetting (POLARIS11 0x1002:0x67EF 0x106B:0x0166 0xC7)` — subsystem
`0x106b0166`, as recorded in the machine-facts table.

**Discovery: `amdgpu.dpm=0` removes the DPM sysfs interface entirely.** After the reboot,
`power_dpm_force_performance_level`, `pp_dpm_sclk` and `gpu_busy_percent` no longer exist
under `/sys/class/drm/card1/device/` (only `power` and `power_state` remain). The
`powerplay` IP block is still detected at init, but the knobs are gone. So the runbook's
live stopgap only works *before* rebooting into these params, and any future attempt to
watch clocks or GPU busy-ness for diagnosis has to drop `dpm=0` first. Runbook updated with
the note and a matching gotcha entry.

The ACPI errors in the boot log (`AE_ALREADY_EXISTS` on SSDT loads, `\_SB.OSCP` not found)
are ordinary Apple firmware noise on this generation, unrelated to the GPU work.

## 5. The panel died (later the same day)

About 15 minutes into the post-reboot session the screen blanked again and did not come
back — the same failure as the earlier unit's boot -1 episode, but this time **with all
three amdgpu params active**. The difference: SSH was up, so instead of a hard power-off,
the machine was diagnosed live from machine A.

First: SSH worked. `SSH_CONNECTION=<machine-a-ip> ... 22`, session resumed, machine up 15
minutes with `ring gfx timeout` still at 0 — so the box was completely healthy underneath a
dead display. That is precisely what the morning's work was for.

**Auth landed on the wrong method.** `journalctl -u sshd` shows
`Accepted password for lecstor` — not publickey. The GitHub-sourced key is not the one
machine A holds, so the password fallback (deliberately left enabled) carried the login.

### What the software reported while the screen showed nothing

| layer | reported |
|---|---|
| Hyprland | `dpmsStatus: 1`, `disabled: false`, `2880x1800@60` active |
| DRM | `card1-eDP-1`: `connected`, `enabled`, dpms `On` |
| Backlight | `gmux_backlight: 253` — lit, matching "backlight on, no image" |
| Kernel | zero amdgpu/drm messages in the preceding 15 minutes |

### Recovery attempts, all failed

- `hl.dsp.dpms("off")` then `("on")` — accepted, no effect, **amdgpu logged nothing**.
- Full modeset: native -> `1280x800` -> native via `monitors.lua` + `hyprctl reload`.
  Hyprland performed both changes (confirmed in `hyprctl monitors`); the panel stayed dark
  and amdgpu again logged nothing.
- Earlier the same session: `1920x1200` instead of native, to test whether the tear lines
  were an eDP bandwidth problem. No change, so it is not bandwidth. Reverted.

### Findings

1. **PSR is exonerated.** The blank-on-wake recurred with `dcdebugmask=0x10` active. The
   runbook's leading suspect for this exact symptom is wrong.
2. **The kernel params fix no display symptom.** Shake unchanged, tear lines unchanged
   (and they predate the params — the user confirmed the lines were the original fault, not
   a regression from `dpm=0`), blank-on-wake recurred.
3. **The driver has no idea anything is wrong**, which is why no software lever moves it.
   Hardware, as the README always said.
4. **The USB-C DisplayPort outputs enumerate normally** (`card1-DP-1` .. `DP-4`, all
   `disconnected`). Untested, but the likely route to a screen if one is ever needed.

## 6. Converted to a true headless box

Decision: stop starting a graphical session at all.

Discovered in the process that **Omarchy autologins through SDDM**, not a getty — PID 895
`/usr/bin/sddm` -> `sddm-helper ... --autologin`. So `set-default multi-user.target` alone
would not have stopped it, and terminating the session would just have triggered another
autologin.

```sh
sudo systemctl set-default multi-user.target
sudo systemctl disable --now sddm
```

Verified after: default target `multi-user.target`, `sddm` `disabled`/`inactive`, the tty1
session gone from `loginctl list-sessions`, and the orphaned Claude process (PID 2498,
stranded in a `foot` terminal on the dead screen) gone with it. `fbcon` remains bound to
the panel — harmless.

**`pkexec` stopped working at this point** and this is worth knowing in advance: polkit's
authentication agent belonged to the graphical session. Over SSH, with no controlling
terminal, `pkexec` fails outright and an agent session has *no* route to root — the `!`
shell prefix fails the same way. Privileged work now needs
`ssh -t <host> 'sudo ...'` from machine A. Both commands above were run by the user that
way.

## Open items

- [x] Reboot B — done, params active (see above).
- [x] Watched for the shake and blank-on-wake: **both recurred**, params notwithstanding.
      Panel abandoned; B is headless.
- [ ] **Key auth.** Login is currently by password. Run `ssh-copy-id lecstor@<B>` from A,
      confirm `Accepted publickey` in `journalctl -u sshd`, then set
      `PasswordAuthentication no` in `/etc/ssh/sshd_config.d/10-headless.conf` and
      `sudo systemctl reload sshd` (needs `ssh -t` — see the pkexec note above).
- [ ] Consider an `ed25519` key for B rather than the existing 2048-bit RSA one.
- [ ] Off-LAN access (Tailscale / WireGuard) if B needs to be reachable from outside the
      LAN. Not set up; the `ufw` rule is LAN-scoped on purpose.
- [ ] **Reconsider the amdgpu params.** They fix no display symptom, and the display is now
      unused. `dpm=0` and `aspm=0` are still plausibly holding off the random reboots, which
      is the one symptom that would take the whole machine down — but that is unproven on
      kernel 7.2.3. `dcdebugmask=0x10` (PSR) is pointless on a panel nobody looks at.
- [ ] Test an external monitor on a USB-C DP output, if B ever needs a screen.
- [ ] **Settle the A/B model question:** run `cat /sys/class/dmi/id/product_name` and
      `hostnamectl --static` on A and label the runbook's machine-facts columns for certain.
