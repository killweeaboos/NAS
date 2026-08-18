# Jellyfin — Shared Media + GPU Passthrough Runbook

_Captured 2026-08-17. Jellyfin runs alongside Plex (`CT101`) for a side-by-side trial — same host, same media, same GPU._

## Decisions locked in this session

- **No duplicate media, no SMB round-trip.** Jellyfin doesn't mount the Samba share — it bind-mounts the exact same `HDDs/media` ZFS dataset directly at the host level, identically to how Plex (`CT101`) already does. Same files on disk, visible to both containers simultaneously; SMB (`CT102`) exists purely for getting files onto the pool from the Windows PC, not for inter-container access.
- **Dedicated LXC (`CT103`, hostname `jellyfin`)** — same "one container, one job" pattern as `CT101`/`CT102`. Privileged, same rationale as before (avoids unprivileged UID/GID remap pain on ZFS bind mounts).
- **Native Debian package install**, not Docker — consistent with the Plex decision, same reasoning (avoids nested Docker + nvidia-container-toolkit failure modes).
- **Sized lighter than Plex**: 2 cores / 2GB RAM vs Plex's 4/4. NVENC does the transcode-heavy lifting either way; bump up if it feels starved in practice.

## Shared-GPU reality check

The RTX 3050 is now serving **two** concurrent transcode-capable services (Plex + Jellyfin), on top of the Immich/Ollama GPU use planned earlier. The driver-enforced cap of **3 simultaneous NVENC encode sessions** (see the Plex/GPU doc) is a shared pool across all of them, not per-service. Not a problem today, but if simultaneous streams start failing to hardware-transcode, this — or `keylase/nvidia-patch` — is where to look first.

## Runbook

```bash
# on the Proxmox host
pct create 103 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname jellyfin \
  --cores 2 \
  --memory 2048 \
  --rootfs local-zfs:16 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 0
pct start 103

# same media dataset Plex already uses — no copy, no SMB
pct set 103 -mp0 /HDDs/media,mp=/media
```

Pass the GPU in — identical device list to what's already working for Plex (`/dev/nvidia-modeset` deliberately excluded, same reasoning as the Plex doc: doesn't exist on this headless host, not needed for compute/transcode). Edit `/etc/pve/lxc/103.conf`:

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

```bash
pct reboot 103
```

Matching userspace NVIDIA driver inside the container — version must match the host's (currently `595.84`; check `nvidia-smi` on the host if it's been updated since):

```bash
pct enter 103
cd /tmp
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/595.84/NVIDIA-Linux-x86_64-595.84.run
chmod +x NVIDIA-Linux-x86_64-595.84.run
./NVIDIA-Linux-x86_64-595.84.run --no-kernel-module --silent
nvidia-smi
```

Install Jellyfin natively (still inside `CT103`):

```bash
apt install -y curl gnupg apt-transport-https
curl -fsSL https://repo.jellyfin.org/jellyfin_team.gpg.key | gpg --dearmor -o /usr/share/keyrings/jellyfin.gpg
echo "deb [signed-by=/usr/share/keyrings/jellyfin.gpg] https://repo.jellyfin.org/debian trixie main" > /etc/apt/sources.list.d/jellyfin.list
apt update
apt install -y jellyfin
```

**Known gotcha to watch for:** if `apt update` throws `Policy rejected non-revocation signature ... SHA1 is not considered secure`, that's the same Debian-13-wide `sqv`/Sequoia SHA1 cutoff (2026-02-01) that hit Plex's repo key — not Jellyfin-specific. Reuse the workaround from the Plex/GPU doc: extend the deadline in `/etc/crypto-policies/back-ends/apt-sequoia.config` inside this container too.

**Point Jellyfin at the shared library** — web UI at `http://<CT103-ip>:8096`, setup wizard or Dashboard → Libraries → Add Media Library, browse to `/media/movies` and `/media/tv` (the same folders Plex already uses).

**Enable hardware transcoding** — Dashboard → Playback → Hardware acceleration → **Nvidia NVENC**, select `/dev/dri/renderD128` as the render device, enable hardware decoding for H.264/HEVC at minimum.

## Still open

See `architecture-decisions.md` for the full, current list.
