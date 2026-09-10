# MacBookPro13,3 / 14,3 Rebuild Runbook

> Everything these machines need beyond a stock [Omarchy](https://omarchy.org/) install,
> in the order it has to be done. Four pieces of hardware do not work out of the box,
> and two of the fixes are order-sensitive — they will silently build against the wrong
> kernel if sequenced wrong.

**Canonical copy:** this file, in git. The Claude artifact is a rendered mirror and may lag.

## Machine facts

Two units. The left column is how the notes were first recorded (hostname `omarchy`); the
right is machine B, read off the hardware on 2026-09-10. Everything that matters for the
fixes is identical — same Baffin dGPU on the internal panel, same Wi-Fi, same audio codec.

| | first recorded (`omarchy`) | machine B (`headless`) |
|---|---|---|
| Model | MacBookPro14,3 (15", 2017, Touch Bar, T1) | **MacBookPro13,3** (15", 2016, Touch Bar, T1), board `Mac-A5C67F76ED83108C` |
| Kernel | `7.2.3-arch1-3` (was `7.1.9.arch1-2` until 2026-09-10) | `7.2.3-arch1-3` |
| systemd | `261.2-1` | `261.2-1` |
| Root | LUKS, btrfs `@` subvol | LUKS, btrfs `@` subvol; 2 GB vfat ESP at `/boot` |
| iGPU | Intel HD Graphics 530 — `i915`, `card0`, **no connected outputs** | same — `[8086:191b]`, all outputs disconnected |
| dGPU | AMD Radeon Pro 555 "Baffin / POLARIS11" `[1002:67ef]` — `amdgpu`, `card1`, **drives the internal panel (eDP-1)** | same `[1002:67ef]`, `card1-eDP-1: connected` |
| Wi-Fi (internal) | BCM43602 fullmac — `brcmfmac`, firmware `Nov 10 2015` | same `[14e4:43ba]`, firmware `Nov 10 2015 version 7.35.177.61` |
| Audio codec | Cirrus CS8409 + CS42L83 sub-codec | same — CS8409 on the Intel PCH (ALSA card `PCH`) |
| dGPU PCI subsystem | `0x106b3900` | `0x106b0166` |
| Network in service | — | USB CDC-NCM ethernet dongle `[0b95:1790]`, `cdc_ncm` |

> The two model / subsystem rows disagree, so one column is mislabelled — see the
> [README](README.md#which-unit-am-i-on). B is the verified one. It does not change any
> step below: every fix here is generation-wide, not model-specific.

---

## Step 0 — Before you wipe anything: back up the Apple EFI / Touch ID data

**If you ever want Touch ID working again (under macOS, or a future dual-boot), this must
happen before Linux touches the disk.** Touch Bar, the Escape key, and the camera do **not**
need any of this — only Touch ID / the Secure Enclave pairing does.

### What is at risk

The T1 chip's embedded OS and its **FDR (Full Disk Recovery) data** live on the EFI System
Partition under `EFI/APPLE/`. In particular `EFI/APPLE/EMBEDDEDOS/FDRData` is
cryptographically tied to *this specific T1* — another Mac's copy is useless. A Linux
installer that reformats or overwrites the ESP destroys it, and then Touch ID cannot be
re-paired even by reinstalling macOS.

> **Status (checked 2026-09-10, both units):** the NVMe is Linux-only — a 2 GB Linux ESP
> + a LUKS/btrfs root, no macOS partition, no `EFI/APPLE`. On B this was re-checked after
> its Omarchy reinstall and is still true: `/boot/EFI` holds only `BOOT`, `limine` and
> `Linux`. The Apple data is already gone, so the only way to get it back is a from-scratch
> macOS reinstall (below) — which has **not** happened on B.

### Regenerate it — reinstall macOS from scratch

1. **Internet Recovery:** power on and hold **Cmd + Option + R** at the chime until a
   spinning globe appears (this pulls the latest macOS the machine supports; there is no
   local recovery partition to use Cmd + R with).
2. Join Wi-Fi in the Recovery screen. The **internal Wi-Fi works fine under macOS** — the
   BCM43602 problem is Linux-only. (A USB ethernet dongle probably has no macOS driver.)
3. **Disk Utility** → View → Show All Devices → select the whole `APPLE SSD …` *device*
   (not a volume under it) → **Erase** → Format **APFS**, Scheme **GUID Partition Map**.
   This wipes the LUKS/btrfs root and the Linux ESP.
4. Quit Disk Utility → **Reinstall macOS** → follow it through.
5. **Complete Setup Assistant all the way to a real Finder desktop** — create a user, reach
   the desktop. The T1 writes fresh FDR data to the ESP during this first full boot;
   sitting in Recovery is not enough.
6. Optional sanity check: System Settings → Touch ID → enrol a fingerprint. If that works,
   the T1 pairing is healthy and worth backing up.

> **This doubles as a hardware test.** If this machine has been randomly rebooting under
> Linux, watch whether macOS stays up for an hour or two. macOS stable + Linux unstable
> isolates the fault to amdgpu; both unstable means hardware — run Apple Diagnostics
> (power on holding **D**) while you are already in Apple's world.

> Both MacBookPro13,3 and 14,3 are **T1** machines, not T2: there is no Startup Security
> Utility and no Secure Boot setting to turn off before re-installing Linux.

### Back it up

1. Terminal (from the full macOS session), mount the ESP:
   ```
   diskutil list                       # the EFI partition is disk0s1
   sudo diskutil mount disk0s1
   ls -la /Volumes/EFI/EFI/APPLE
   ```
2. Copy to external storage (not the internal disk). Take **all of `EFI/APPLE/`**, and if
   the USB has room, the whole `EFI/` folder as cheap insurance:
   ```
   cp -a "/Volumes/EFI/EFI/APPLE" /Volumes/<usb>/APPLE-backup-$(date +%F)
   cp -a "/Volumes/EFI/EFI"       /Volumes/<usb>/EFI-full-backup-$(date +%F)   # optional
   ```
3. Verify `EFI/APPLE/EMBEDDEDOS/FDRData` (or wherever the current bridgeOS puts the FDR
   blob) exists and is > 0 bytes. **Confirm the exact path against current t2linux.org /
   Arch-on-Mac docs** — the FDR blob location has moved between bridgeOS versions.
4. Photograph / save `system_profiler SPiBridgeDataType` (T1 info) for reference.
5. Store the USB backup somewhere safe and separate. It is the only copy tied to this T1.

### Restore it (after re-installing Linux)

**Prefer letting T1Bridge import the backup — do not write to the ESP unless you have to.**
T1Bridge ships a sandboxed importer that discovers EFI partitions, reads them *without
modifying them*, matches the data against the live sensor, and stores the selected record
under root-only `/var/lib/t1bridge/machine-data/`. Its docs list an explicit same-machine
EFI-tree/FDR backup as a working import source, so the USB backup is enough on its own.

1. Install T1Bridge. Touch ID needs all four packages — the Touch-Bar-only subset omits the
   fingerprint pair:
   ```
   omarchy update
   sudo pacman -S --needed linux-headers t1bridge t1bridge-dkms \
       libfprint-t1bridge fprintd-t1bridge
   ```
   `libfprint-t1bridge` and `fprintd-t1bridge` replace the distro's libfprint/fprintd
   system-wide and must stay a matched pair.

2. Attach the USB backup, then run the importer:
   ```
   sudo systemctl start t1bridge-import.service
   sudo systemctl status t1bridge-import.service
   ```
   Automatic discovery has a known open failure on some multi-ESP layouts; if it cannot
   find the data, point it at the explicit backup path per T1Bridge's setup docs.

3. Enrol:
   ```
   fprintd-enroll
   fprintd-verify
   ```

**Fallback — copying back onto the ESP.** Only if the importer cannot read the backup:

1. Mount the ESP: `sudo mount /dev/nvme0n1p1 /mnt/esp` (whichever partition is the vfat ESP)
2. `sudo cp -a /path/to/APPLE-backup-YYYY-MM-DD/. /mnt/esp/EFI/APPLE/`
3. `sudo sync && sudo umount /mnt/esp`, then reboot.

> **⚠ The ESP-write path is the part to validate.** Backing up `EFI/APPLE` before the wipe is
> well established. Copying it back onto a freshly-created ESP *should* restore the pairing,
> but confirm against current t2linux / Arch-on-Mac documentation at the time you do it —
> the T1 may need the files at specific paths, and a bridgeOS revive may still be required.
> The importer route above avoids this uncertainty entirely, which is why it is listed first.

### If you have **no** backup and macOS is already gone

The only route is:
1. On another Mac, install **Apple Configurator 2**.
2. Put this MacBook into DFU (T1: hold the right Option + right Ctrl + left Shift for ~3 s
   while connected, or follow Apple's current DFU key combo for MacBookPro14,3).
3. **Revive** (keeps data) or **Restore** (wipes) the bridgeOS from Configurator.
4. Install macOS and complete a full first boot — this regenerates FDR data.
5. Then back it up per the section above before re-installing Linux.

A revive reinstalls the T1 firmware but does **not** by itself restore the machine-specific
FDR data — macOS's first boot does. Verify the details before relying on this.

---

## What doesn't work out of the box, and why

All of these are hardware-support gaps in mainline Linux, not configuration mistakes.
Chasing settings on any of them wastes hours — each was measured to a definite cause.

### 1. Internal Wi-Fi cannot join modern APs — *worked around*

The BCM43602 is a **fullmac** card: association runs inside firmware dated `Nov 10 2015`,
the newest Broadcom ever shipped. Access points that beacon 802.11be **EHT** elements
(WiFi 6E / 7, e.g. UniFi U7) send association info this firmware cannot parse, so every
join aborts with `CTRL-EVENT-ASSOC-REJECT status_code=16`. Scanning works fine; only
association fails.

Ruled out by measurement: password, MAC filtering (a never-before-seen random MAC failed
identically), minimum RSSI / signal (failed across a 30 dB sweep, −79 → −49 dBm), PMF,
stale supplicant state, and a 2.4 GHz-only SSID — which still beaconed `EHT=1`. **There is
no client-side fix; use a USB adapter** (see step 7).

### 2. Internal microphone records digital silence — *fixed via DKMS*

The mic is on the Intel HDA codec (CS8409), **not** the T1. `Internal Mic` is pin `0x44`,
`Conn = Digital` — a digital mic behind a **CS42L83** sub-codec needing I²C init. Mainline
`snd-hda-codec-cs8409` contains zero Apple support: grepping the module for `cs42l83`,
`apple`, `106b` returns nothing, and the only variants implemented are BULLSEYE / CYBORG /
DOLPHIN / ODIN / WARLOCK — all Dell boards with CS42L42.

Without the driver, a capture is asymmetric garbage: a silent noise floor with recurring
spikes slamming to the negative rail (`peak=32768`, only-negative, `mean≈−6500`). With the
DKMS driver on the APPLE code path, it becomes a real signal. Fixed in steps 4–5.

### 3. No physical Escape key — *remapped*

The Touch Bar occupies the top row and sits behind the T1, so without a T1 bridge driver
there is no Escape at all — only `Apple SPI Keyboard` appears to the kernel. Remapped in
Hyprland (step 9). A T1 bridge would restore the real Touch Bar Escape at the cost of an
extra DKMS module and a documented lid/suspend regression — not worth it.

### 4. Random hard reboots + screen shake — *worked around with kernel params*

The discrete **Radeon Pro 555 (POLARIS11)** drives the internal panel (`eDP-1` is on
`card1` = amdgpu; `i915` has no connected outputs, so **amdgpu cannot be blacklisted** —
that would black-screen the laptop).

Two symptoms, same subsystem:

- **Random reboots**: boot logs show `amdgpu 0000:01:00.0: ring gfx timeout, but soft
  recovered` repeating every ~2 s with GPU coredumps piling up, then the machine hard-resets
  (journal just stops, no shutdown sequence). Other resets logged nothing at all — a harder
  hang flushes nothing, especially with `quiet loglevel=0` on the cmdline. No MCE, no
  thermal, no panic in any boot.
- **Screen shake**: slight vertical jitter of the whole image that worsens over time until a
  reboot is needed. Consistent with an eDP timing / PLL instability — commonly **Panel Self
  Refresh (PSR)** bugs, and/or DPM clock transitions glitching the panel PLL. The GPU was
  also observed pinned at max sclk (855 MHz, DPM level 7) while 0 % busy — DPM misbehaving.

**Fix** (step 8): kernel parameters `amdgpu.dpm=0 amdgpu.aspm=0 amdgpu.dcdebugmask=0x10`.

> All observed crashes were on kernel **7.1.9**. The 2026-09-10 Omarchy update moved this
> machine to **7.2.3**, which *may* have fixed the reboots on its own — not enough uptime
> yet to know. The kernel params are a cheap safety net on top.

---

## Rebuild sequence

Order matters. Steps 1–3 exist purely to clear out a module that cannot build and get the
running kernel and its headers matched before anything is compiled against them.

### 1. Install Omarchy, then update fully

Omarchy blocks a bare `pacman -Syu` to protect its snapshot / migration steps. Use its own
updater, and **reboot into the new kernel** before compiling anything.

```
omarchy update
# reboot into the new kernel before continuing
```

### 2. Remove the broken SPI DKMS module if present

**Do this before installing headers, not after.** Some earlier setups have
`macbook12-spi-driver-dkms` installed. On kernel 7.x it **fails to build**
(`struct acpi_driver has no member 'owner'`; `asm/unaligned.h` moved to
`linux/unaligned.h`) and clutters every `pacman` transaction. It is **not needed** — the
in-kernel `applespi` driver already provides `Apple SPI Keyboard` + `Apple SPI Touchpad`.

```
pacman -Q macbook12-spi-driver-dkms 2>/dev/null && \
  sudo pacman -R --noconfirm macbook12-spi-driver-dkms
```

Use `-R`, not `-Rns` — `-Rns` tries to take `dkms` itself, which step 4 needs.

> Removing it first means the `linux-headers` post-install DKMS pass has nothing broken to
> build, so step 3 runs clean instead of throwing a compile error you then have to read and
> dismiss. (The original order had headers first; that works, it is just noisier.)

### 3. Install headers matching the *running* kernel

```
uname -r
sudo pacman -S --needed linux-headers
pacman -Q linux linux-headers        # the two versions MUST match
```

If `linux-headers` comes in *ahead* of your running kernel, you did a partial upgrade —
`omarchy update` and reboot first, then install headers. (See gotchas.)

### 4. Build the Apple audio driver

Adds the Apple CS8409 + CS42L83 initialisation mainline lacks. The kernel will be tainted —
normal for an unsigned out-of-tree module.

```
yay -S snd-hda-macbookpro-dkms-git       # run in a real terminal; yay drops privileges
dkms status                              # want: snd-hda-macbookpro/0.1, <kernel>: installed
# reboot
```

Confirm it took the **Apple** path, not the Dell one:

```
journalctl -k -b | grep -i cs8409
# want: "Primary patch_cs8409 NOT FOUND trying APPLE"
```

### 5. Apply microphone gain

Even working, this mic is extremely quiet (~2 % of full scale). Without extra gain, Whisper
gets what looks like silence and returns an empty transcript. WirePlumber persists the
PipeWire volume across reboots; the ALSA `Internal Mic` / `Internal Mic Boost` controls are
already at max on a stock Omarchy install but set them explicitly to be sure.

```
pactl set-source-volume alsa_input.pci-0000_00_1f.3.analog-stereo 900%
amixer -c PCH sset 'Internal Mic' 100%
amixer -c PCH sset 'Internal Mic Boost' 100%
```

If speech comes out clipped/garbled once the hardware works, 900 % is too hot — try
400–500 %.

### 6. Enable voxtype's evdev hotkey

Without the `input` group the daemon logs *"No keyboard device found in /dev/input/"* and
**no key works at all** — the configured key name is a red herring.

```
sudo usermod -aG input $USER
voxtype config set hotkey.enabled true    # stock config had this false (Hyprland-bind mode)
voxtype config set hotkey.key CAPSLOCK
# REBOOT — a logout is not enough (see gotchas)
```

CapsLock is push-to-talk (hold to record, release to transcribe). If it toggles caps state
or feels wrong: `RIGHTCTRL`, `RIGHTALT`, `F13`, or `SCROLLLOCK` instead
(`voxtype config set hotkey.key <KEY>` + `systemctl --user restart voxtype`).

### 7. Plug in the USB Wi-Fi adapter

Both chipsets below are in-kernel and need no driver install. Verify the USB ID rather than
trusting a listing's chipset claim.

```
lsusb | grep -iE '0e8d|0846|3574|35bc'
# 0e8d:7612 -> MT7612U  (WiFi 5, mt76x2u)
# 0e8d:7961 -> MT7921AU (WiFi 6E, mt7921u)
```

Avoid Realtek RTL88xx and AICSemi AIC8800 adapters — both need out-of-tree drivers and
neither vendor supports Arch.

### 8. Tame the discrete GPU (random reboots + screen shake)

Kernel params for `amdgpu`. On Omarchy (limine + UKI) they go in `/etc/default/limine`,
which is user-owned and survives `omarchy update`.

Append to the `KERNEL_CMDLINE[default]+=` block a second line:

```sh
# /etc/default/limine
KERNEL_CMDLINE[default]+=" amdgpu.dpm=0 amdgpu.aspm=0 amdgpu.dcdebugmask=0x10"
```

| param | why |
|---|---|
| `amdgpu.dpm=0` | disable dynamic power management — the reboot trigger; also stops DPM clock transitions that glitch the eDP PLL |
| `amdgpu.aspm=0` | disable the GPU's PCIe Active State Power Management |
| `amdgpu.dcdebugmask=0x10` | disable Panel Self Refresh (PSR) — common cause of eDP flicker / shake |

Then rebuild the UKI and reboot:

```
sudo limine-update
# reboot
```

Verify after reboot:

```
cat /proc/cmdline                          # contains the three amdgpu.* params
cat /sys/module/amdgpu/parameters/dpm       # 0
cat /sys/module/amdgpu/parameters/dcdebugmask   # 16
```

Trade-off: GPU runs at fixed clocks — slightly more heat / battery. If the panel looks
*worse* (underruns, more flicker), drop `amdgpu.dpm=0` first and keep the other two.

**Live, no-reboot stopgap for the shake:** `echo high | sudo tee
/sys/class/drm/card1/device/power_dpm_force_performance_level` pins clocks (stops the
transitions). `auto` reverts.

### 9. Bind an Escape key

Append to `~/.config/hypr/bindings.lua`. The down/up split is Omarchy's own workaround,
copied from `default/hypr/bindings/clipboard.lua`: a single send can leave synthetic key
state stuck or repeating.

```lua
local function send_key_once(mods, key)
  return function()
    hl.dispatch(hl.dsp.send_key_state({ mods = mods, key = key, state = "down" }))
    hl.timer(function()
      hl.dispatch(hl.dsp.send_key_state({ mods = mods, key = key, state = "up" }))
    end, { timeout = 50, type = "oneshot" })
  end
end

o.bind("SUPER + TAB", "Escape (Touch Bar has no Esc)", send_key_once("", "Escape"))
```

```
hyprctl reload && hyprctl configerrors
```

---

## Headless duty — remote access

Machine B's panel is failing, so the machine has to outlive its display: reachable over
SSH, surviving a closed lid, and not depending on the Hyprland session being alive. `sshd`
is independent of the graphical session — a black screen does not take the box with it.

None of this needs a reboot. Done on B 2026-09-10.

### 1. Authorised key

Your GitHub public keys are a convenient, already-trusted source — no copying between
machines, nothing secret in transit:

```sh
mkdir -p ~/.ssh && chmod 700 ~/.ssh
gh api /users/<your-github-user>/keys --jq '.[] | .key' > ~/.ssh/authorized_keys
# or, without gh:  curl -fsSL https://github.com/<your-github-user>.keys > ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh-keygen -lf ~/.ssh/authorized_keys      # confirm the fingerprint is one you hold
```

Check the key type it hands you. An older `ssh-rsa` key still works with OpenSSH 10.x —
the client negotiates `rsa-sha2-*` — but if it is a 2048-bit RSA key, this is a good moment
to generate an `ed25519` and add it to GitHub.

### 2. sshd

`openssh` ships with Omarchy but the service is **disabled by default**. Host keys are
generated on first start.

```sh
sudo tee /etc/ssh/sshd_config.d/10-headless.conf >/dev/null <<'EOF'
PermitRootLogin no
PubkeyAuthentication yes
# password auth stays on until key login is confirmed working, then set to no
PasswordAuthentication yes
KbdInteractiveAuthentication no
ClientAliveInterval 60
ClientAliveCountMax 3
EOF
sudo systemctl enable --now sshd
```

**Leave `PasswordAuthentication yes` until you have logged in with the key from the other
machine.** Locking yourself out of a laptop whose screen is dying is a bad afternoon. Once
key login works, flip it to `no` and `sudo systemctl reload sshd`.

### 3. Open the firewall — this is the step that is easy to miss

Omarchy ships **`ufw` active with `deny (incoming)`**, so enabling `sshd` alone leaves the
port unreachable from anywhere but the machine itself. Scope the rule to the LAN rather
than opening it to the world:

```sh
sudo ufw allow from <lan-subnet>/24 to any port 22 proto tcp comment 'ssh from LAN'
sudo ufw status
```

Off-LAN access needs a VPN / mesh (Tailscale, WireGuard) — do not port-forward 22.

### 4. Stop the lid from suspending the machine

The one that silently breaks everything above. logind's default is `HandleLidSwitch=suspend`,
so a closed-lid headless box goes to sleep and drops off the network.

```sh
sudo tee /etc/systemd/logind.conf.d/20-headless-lid.conf >/dev/null <<'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF
sudo systemctl reload systemd-logind      # reload, NOT restart — restart can kill sessions
```

Verify it took effect live, without trusting the file:

```sh
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager HandleLidSwitch      # want: s "ignore"
```

**Idle is not a problem.** Omarchy's idle service (`~/.config/omarchy/shell.json`,
`idle.screensaver` / `idle.lock`) only blanks and locks the screen — it has no suspend
action at all, so an idle box stays reachable. Blanking is in fact desirable here: it is one
less thing asking the dying GPU to drive the panel.

### 5. Name resolution

Omarchy already has `avahi` running with `nss-mdns` wired into `/etc/nsswitch.conf`, so the
box answers to `<hostname>.local` from any other machine on the LAN — worth using instead
of the IP, which moves when the DHCP lease changes.

```sh
ssh <user>@<hostname>.local
```

### Verify

| command (from the *other* machine) | pass condition |
|---|---|
| `ssh -o BatchMode=yes <user>@<hostname>.local true` | succeeds silently — key auth works, no prompt |
| `ssh <user>@<hostname>.local 'systemctl is-active sshd'` | `active` |
| close the lid, then `ssh <user>@<hostname>.local uptime` | still answers |

On the machine itself, `ssh -o BatchMode=yes <user>@<its-own-ip> true` returning
`Permission denied (publickey,password)` proves sshd is listening and offering both methods
— it does **not** prove the `ufw` rule, since loopback traffic skips it. Only a connection
from another host tests the firewall.

---

## Gotchas that cost real time

Each of these presents as a different problem than it is.

**`pacman -S linux-headers` → version ahead of the running kernel.** It installs the
*current repo* version, not headers for your running kernel. That's a partial upgrade, and
DKMS then builds into a module tree the running kernel never loads. `omarchy update` and
reboot **first**, then install headers.

**`ERROR Hotkey listener error: No keyboard device found in /dev/input/`.** You are not in
the `input` group — and **logging out does not apply it**. `user@1000.service` survives as
long as any session remains, and the voxtype daemon inherits its stale credentials. Reboot.

**`CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16`.** Not a wrong password.
`brcmfmac` returns `WLAN_STATUS_AUTH_TIMEOUT` as a catch-all for any failed join because
association is offloaded to firmware. Re-entering the password will not help.

**`WARN Recording error: No audio was captured`** with the OSD still appearing. Stale audio
handle — something grabbed the source while voxtype had it suspended. Not a
reconfiguration: `systemctl --user restart voxtype`.

**Audio dies after a kernel update.** Check `dkms status` first — the out-of-tree audio
driver failing to rebuild is the likely cause. Roll back with
`sudo dkms remove snd-hda-macbookpro/0.1 --all` then
`sudo pacman -R snd-hda-macbookpro-dkms-git`, or reinstall headers and
`sudo dkms autoinstall`.

**Installing `linux-headers` triggers a `macbook12-spi-driver` build failure.** Expected if
that package is still installed — it does not build on kernel 7.x. Remove it (step 2); the
in-kernel `applespi` covers keyboard + touchpad.

**GPU stuck at max clock while idle / screen shaking.** amdgpu DPM misbehaving. Stopgap:
force `power_dpm_force_performance_level` to `high`. Durable: step 8 kernel params.

---

## Verification

Run these after a rebuild. Each has one unambiguous pass condition.

| command | pass condition |
|---|---|
| `id -nG` | includes `input` |
| `voxtype setup check` | Input section shows a green check, not a warning |
| `dkms status` | `snd-hda-macbookpro/0.1, <running kernel>: installed` |
| `journalctl -k -b \| grep -i cs8409` | reports `trying APPLE` |
| `iw dev <usb-iface> link` | associated on **5 GHz**, not 2.4 GHz |
| `cat /sys/module/amdgpu/parameters/dpm` | `0` |
| `hyprctl configerrors` | empty output |
| `systemctl is-active sshd` | `active` (headless units) |
| `busctl get-property org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager HandleLidSwitch` | `s "ignore"` (headless units) |

**Mic sanity test** — record and check the *level*, don't trust your ears; this hardware
fails by returning near-zero, indistinguishable from a muted app until measured:

```
parecord --device=alsa_input.pci-0000_00_1f.3.analog-stereo \
  --file-format=wav --rate=16000 --channels=1 /tmp/mic.wav
# peak near 1/32767 = dead;  peak in the thousands = working
```

---

## Package versions this was last verified against

kernel `7.2.3-arch1-3` · systemd `261.2` · voxtype `1.0.1` ·
`snd-hda-macbookpro-dkms-git 0.1-3` · `linux-headers 7.2.3.arch1-3`

Headless section (machine B, 2026-09-10): `openssh 10.5p1-1` · `ufw 0.36.2-7` ·
`avahi 1:0.9rc5-1` · `nss-mdns 0.15.1-2`

The Wi-Fi and audio findings are hardware-generation limits and will outlive these package
versions; the exact commands may not.
