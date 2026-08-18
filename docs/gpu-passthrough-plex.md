# GPU Passthrough + Plex Hardware Transcoding — Decisions & Runbook

_Captured 2026-08-17, updated same day as the host driver install was completed live. See `architecture-decisions.md` for the consolidated decision log._

## Decisions locked in this session

- **GPU passthrough strategy: LXC-based sharing**, not VM PCIe passthrough. NVIDIA driver installs on the Proxmox host; GPU device nodes get passed into individual LXCs via the modern `devN:` config syntax. This lets Plex/Jellyfin, Immich's ML, and Ollama all reach the GPU concurrently instead of locking it to one VM.
- **Plex install method: native Debian package inside its own LXC**, not via CasaOS's Docker app store. User runs CasaOS for managing other apps, but for the first GPU-transcoding pass, native install avoids the nested Docker + nvidia-container-toolkit layer (extra failure points: Compose `count: all` bug, `NVIDIA_DRIVER_CAPABILITIES` misconfig, device nodes missing after reboot). CasaOS can still run alongside for other services in the same or a different LXC; this decision only concerns Plex itself.
- **NVIDIA kernel module type: open-source (MIT/GPL)**, not proprietary. NVIDIA recommends the open kernel modules for Turing/Ampere/Ada/Hopper GPUs — the RTX 3050 (GA107, Ampere) qualifies. No functional difference for NVENC/CUDA; only the kernel-space shim differs, userspace libraries are identical either way.

## Status: host driver install — DONE ✅

As of 2026-08-17: `nvidia-smi` confirmed working on the Proxmox host. Driver `595.84`, CUDA `13.2`, GPU visible (`GeForce RTX 3050`, idle, `Persistence-M` on after running `nvidia-smi -pm 1`). Next step in progress: creating the dedicated Plex LXC (`CT101`) and passing the GPU into it.

### Gotchas hit during the host driver install (read before repeating this on another box)

1. **`nova_core` conflict.** Proxmox 9.2.2's kernel (7.0.2-6-pve) is new enough to include `nova_core`/`nova_drm` — the in-tree Rust NVIDIA driver landing in mainline Linux. It auto-bound to the GPU before the proprietary/open NVIDIA installer could claim it, causing the installer to silently fail (no `nvidia-smi` ever got installed) and showing up in `dmesg` as `NovaCore 0000:07:00.0: NVIDIA (Chipset: GA107...)` plus a GSP firmware load failure. **Fix:** blacklist it alongside nouveau:
   ```bash
   cat >>/etc/modprobe.d/blacklist-nouveau.conf <<'EOF'
   blacklist nova
   blacklist nova_core
   blacklist nova_drm
   options nova modeset=0
   EOF
   update-initramfs -u -k all
   reboot
   ```
   Confirm with `lsmod | grep -i nova` (should be empty) before re-running the NVIDIA installer.

2. **Enterprise repo 401 errors blocking `apt install build-essential`/`pve-headers`.** Fresh Proxmox installs without a paid subscription point at `enterprise.proxmox.com` by default, which returns `401 Unauthorized` and blocks `apt update` entirely — including for unrelated packages. Fix (one-time, do this on any fresh Proxmox box regardless of GPU work):
   ```bash
   # disable enterprise stanzas in /etc/apt/sources.list.d/pve-enterprise.sources
   # and /etc/apt/sources.list.d/ceph.sources (add `Enabled: no` to each)
   cat >/etc/apt/sources.list.d/proxmox.sources <<'EOF'
   Types: deb
   URIs: http://download.proxmox.com/debian/pve
   Suites: trixie
   Components: pve-no-subscription
   Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
   EOF
   apt update
   ```

3. **`cc`/`gcc` not found during DKMS build.** `build-essential` wasn't actually installed (blocked by gotcha #2 above at the time). Re-run `apt install -y build-essential dkms pve-headers-$(uname -r) proxmox-default-headers` after fixing the repos, confirm with `which cc gcc`.

4. **Persistence daemon: went with `nvidia-smi -pm 1` instead of the `nvidia-persistenced` systemd service.** The `.run` installer doesn't register a `nvidia-persistenced.service` unit on Debian automatically (it only ships a template you'd have to manually install from `/usr/share/doc/NVIDIA_GLX-1.0/sample/nvidia-persistenced-init.tar.bz2`). Simpler path taken: a custom oneshot unit (`nvidia-boot.service`) that runs `nvidia-smi` at boot (before `pve-guests.service`) to force device-node creation, plus `nvidia-smi -pm 1` for persistence mode. Good enough for a single-host homelab; revisit with the real daemon only if this proves flaky.

5. **NVIDIA driver `.run` installer prompts** — for reference, answered: "Multiple kernel module types" → **NVIDIA Proprietary or MIT/GPL** → chose **MIT/GPL** (see decision above). "Register with DKMS?" → **Yes** (survives future kernel updates — this whole gotcha list started because a driver *wasn't* DKMS-managed). "Run nvidia-xconfig to update X config?" → **No** (headless host, no X server ever runs here).

6. **`/dev/nvidia-modeset` doesn't exist on this host, and that's fine.** That device node is only created when the `nvidia-drm` module has KMS modesetting enabled (`nvidia-drm.modeset=1`), which happens for GPUs actually driving a display. This is a headless server — nothing ever asks the GPU to drive a monitor — so the node never gets created, and attempting to pass it into an LXC (`devN: /dev/nvidia-modeset`) fails Proxmox's container start with `Device /dev/nvidia-modeset does not exist`. It's not required for NVENC transcoding, CUDA, or any compute/encode workload (Plex, Immich ML, Ollama) — just leave it out of the `devN:` list. (If some future need for actual display output on this GPU comes up, the fix would be `nvidia-modprobe -m` to force-create it, not something to set up preemptively.)

## Runbook: NVIDIA driver on host + Plex LXC with HW transcoding

### 1. Install NVIDIA driver on the Proxmox host — ✅ done, see gotchas above

Proxmox 9.2.2 runs kernel 7.0.2-6-pve — driver series 550.x fails to compile against this kernel (VMA locking API changes). Use driver 580.x production branch or newer (595.x/610.x also fine). To always get a current version rather than a stale hardcoded one:

```bash
cd /tmp
VER=$(curl -s https://download.nvidia.com/XFree86/Linux-x86_64/latest.txt | awk '{print $1}')
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/${VER}/NVIDIA-Linux-x86_64-${VER}.run
chmod +x NVIDIA-Linux-x86_64-${VER}.run
./NVIDIA-Linux-x86_64-${VER}.run --dkms
reboot
```

After reboot: `nvidia-smi` on the host should show the RTX 3050. (Confirmed working — see Status above.)

Boot-time device node + persistence handling actually used (see gotcha #4 for why this differs from the originally planned `nvidia-persistenced` service):

```bash
cat >/etc/systemd/system/nvidia-boot.service <<'EOF'
[Unit]
Description=Touch NVIDIA GPU at boot to create device nodes
Before=pve-guests.service
DefaultDependencies=no

[Service]
Type=oneshot
ExecStart=/usr/bin/nvidia-smi
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now nvidia-boot.service
nvidia-smi -pm 1
```

### 2. Create a dedicated Plex LXC — in progress

Template filenames/versions drift — always check the current one rather than hardcoding:

```bash
pveam update
pveam available --section system | grep debian-13
pveam download local <exact-filename-from-above>
```

```bash
pct create 101 local:vztmpl/<exact-filename> \
  --hostname plex \
  --cores 4 \
  --memory 4096 \
  --rootfs local-zfs:16 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 0 \
  --features nesting=0
pct start 101
```

(`--unprivileged 0` = privileged container, simplest for direct `/dev/nvidia*` device access. Sized 4 cores / 4GB RAM — adjust against whatever else lands on this box; Plex itself is light except during transcodes.)

### 3. Pass the GPU into the LXC

Edit `/etc/pve/lxc/101.conf` and add:

```
dev0: /dev/nvidia0
dev1: /dev/nvidiactl
dev2: /dev/nvidia-uvm
dev3: /dev/nvidia-uvm-tools
dev4: /dev/dri/card0
dev5: /dev/dri/renderD128
dev6: /dev/nvidia-caps/nvidia-cap1
dev7: /dev/nvidia-caps/nvidia-cap2
```

Deliberately **not** included: `/dev/nvidia-modeset` — doesn't exist on this headless host and isn't needed for compute/transcode workloads (see gotcha #6 above). If you ever add it back after forcing its creation, it's harmless to include.

Note on the caps devices: `nvidia-cap1`/`nvidia-cap2` are MIG (Multi-Instance GPU) control nodes (`mig-config`/`mig-monitor` capabilities) — a datacenter-GPU feature the RTX 3050 doesn't support. The driver creates these nodes on this host regardless (confirmed via `ls /dev/nvidia-caps/` — they exist even though MIG itself is a no-op on this card), so there's no reason to leave them out; they're just permanently inert here.

Reboot the container to apply the new device config: `pct reboot 101`. (Note: `pct restart` is **not** a valid subcommand — Proxmox's `pct` uses `reboot`/`start`/`stop`/`shutdown`, not `restart`.)

### 4. Install matching userspace driver inside the LXC

Version **must match** the host driver exactly (currently `595.84`) — don't use the "grab latest" one-liner here, latest may have moved past what the host has installed by the time you do this.

The LXC is a separate filesystem from the host, so get the installer into it one of two ways:

**Option A — download fresh, directly inside the container (simplest):**
```bash
pct enter 101
```
This drops you into a root shell *inside* the container. From there:
```bash
cd /tmp
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/595.84/NVIDIA-Linux-x86_64-595.84.run
chmod +x NVIDIA-Linux-x86_64-595.84.run
./NVIDIA-Linux-x86_64-595.84.run --no-kernel-module --silent
nvidia-smi
```
`exit` to return to the Proxmox host shell when done.

**Option B — push the file you already downloaded on the host, without entering the container:**
```bash
pct push 101 /tmp/NVIDIA-Linux-x86_64-595.84.run /tmp/NVIDIA-Linux-x86_64-595.84.run
pct exec 101 -- chmod +x /tmp/NVIDIA-Linux-x86_64-595.84.run
pct exec 101 -- /tmp/NVIDIA-Linux-x86_64-595.84.run --no-kernel-module --silent
pct exec 101 -- nvidia-smi
```
(Only works if that file's still sitting in `/tmp` on the host — if it's gone, Option A is easier than re-fetching it on the host just to push it over.)

`--no-kernel-module` skips building/loading a kernel module inside the container (the container shares the host's kernel — that module is already loaded on the host side); this install just drops in the matching userspace libraries (`nvidia-smi`, NVENC/CUDA libs) so they line up version-for-version with what's running on the host.

### 5. Install Plex natively

```bash
curl https://downloads.plex.tv/plex-keys/PlexSign.key | gpg --dearmor | tee /usr/share/keyrings/plex-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/plex-archive-keyring.gpg] https://downloads.plex.tv/repo/deb public main" > /etc/apt/sources.list.d/plexmediaserver.list
apt update
apt install -y plexmediaserver
```

Point Plex's transcode temp directory at the NVMe `rpool` (fast, not the SMR `HDDs` pool) — Settings → Transcoder → Transcoder temporary directory, e.g. `/var/lib/plexmediaserver/transcode` on the container's rootfs, which already lives on `rpool` via `local-zfs`.

### 6. Enable hardware transcoding in Plex

Settings → Transcoder:
- "Use hardware acceleration when available" — **requires an active Plex Pass subscription**, this is a Plex-side licensing gate, not a technical one.
- "Use hardware-accelerated video encoding"

Test with a stream that forces transcoding and watch `nvidia-smi` on the host for a `plex` process using the encoder.

### Known gotcha: NVENC session limit

Consumer NVIDIA GPUs (including the RTX 3050) are limited by the driver to **3 simultaneous NVENC encode sessions**, regardless of Plex Pass. The homelab community workaround is [keylase/nvidia-patch](https://github.com/keylase/nvidia-patch), applied on the host or inside the LXC against the installed driver — unofficial and technically against Nvidia's driver EULA, but widely used. Worth knowing about before assuming 4+ concurrent transcodes will just work.

## Still open

See `architecture-decisions.md` for the full, current list.
