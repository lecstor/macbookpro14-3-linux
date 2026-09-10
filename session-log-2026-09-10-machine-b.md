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
`limine` and `Linux` — **no `EFI/APPLE`**. The macOS-reinstall route to regenerate FDR data
has not been taken on this machine. Runbook Step 0 still applies unchanged.

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

## Open items

- [x] Reboot B — done, params active (see above).
- [ ] **Watch for the shake and the blank-on-wake.** With the params now live, this is the
      real test; success = 1–2 weeks of normal idle/wake cycles with neither. Note the
      pre-reboot `power_dpm_force_performance_level=high` stopgap can no longer be used as
      a comparison — `dpm=0` removed the knob.
- [ ] Confirm key-based SSH from machine A, then set `PasswordAuthentication no` in
      `/etc/ssh/sshd_config.d/10-headless.conf` and `systemctl reload sshd`.
- [ ] Consider an `ed25519` key for B rather than the existing 2048-bit RSA one.
- [ ] Off-LAN access (Tailscale / WireGuard) if B needs to be reachable from outside the
      LAN. Not set up; the `ufw` rule is LAN-scoped on purpose.
- [ ] **Settle the A/B model question:** run `cat /sys/class/dmi/id/product_name` and
      `hostnamectl --static` on A and label the runbook's machine-facts columns for certain.
