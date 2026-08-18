# Architecture Decisions

## Already made

| Decision | Choice | Why |
|---|---|---|
| ZFS pool layout for the 3x 8TB HDDs | RAIDZ1, pool `HDDs`, 21.8TB usable | Built and healthy — see SMR risk note in `hardware.md` |
| Role of the 2x 500GB NVMe pair | Mirrored as `rpool`; Proxmox boot pool *and* default LXC/VM root disk location | No portion reserved separately as ZFS special device/L2ARC for `HDDs` |
| GPU passthrough strategy | **LXC-based sharing**, not VM PCIe passthrough | Driver stays on the Proxmox host; GPU device nodes passed into individual LXCs via `devN:` syntax. Lets Plex/Jellyfin, Immich's ML, and Ollama all reach the GPU concurrently instead of locking it to one VM |
| Plex install method | **Native Debian package**, inside its own LXC (`CT101`) | Avoids nested Docker + nvidia-container-toolkit failure modes (Compose `count: all` bug, `NVIDIA_DRIVER_CAPABILITIES` misconfig, device nodes missing after reboot). CasaOS can still run alongside for other services |
| NVIDIA kernel module type | **Open-source (MIT/GPL)**, not proprietary | NVIDIA recommends open kernel modules for Turing/Ampere/Ada/Hopper — RTX 3050 (GA107, Ampere) qualifies. No functional difference for NVENC/CUDA, only the kernel-space shim differs |
| Network share protocol | **SMB**, not NFS | Main PC is Windows 11 Pro — SMB has zero client caveats regardless of OS; NFS client on Windows is clunkier and Home-incompatible |
| SMB architecture | Dedicated LXC (`CT102`, hostname `smb`) | Matches "one container, one job" pattern used for Plex. Privileged container, same rationale as Plex (avoids unprivileged UID/GID remap pain on ZFS bind mounts) |
| Jellyfin media access | Bind-mounts the **same** `HDDs/media` ZFS dataset directly at host level (identical to Plex) — no SMB round-trip, no duplicate media | SMB (`CT102`) is purely for getting files onto the pool from Windows, not for inter-container access |
| Jellyfin architecture | Dedicated LXC (`CT103`, hostname `jellyfin`), native Debian package install | Same "one container, one job" pattern and native-install reasoning as Plex |

### Host driver install status: ✅ DONE

As of 2026-08-17: `nvidia-smi` confirmed working on the Proxmox host. Driver `595.84`, CUDA `13.2`, GPU visible (idle, `Persistence-M` on). See `docs/gpu-passthrough-plex.md` for the full runbook and gotchas encountered.

## Still open

| Decision | Notes / candidates |
|---|---|
| Role of the spare 2TB drive (currently unused NTFS) | Candidates: Proxmox Backup Server datastore, `zfs send`/`recv` backup target for critical datasets, or scratch/downloads space. Not a good fit for joining the existing RAIDZ1 vdev (mismatched redundancy) |
| Final choice between Plex and Jellyfin | Both running side by side (`CT101` / `CT103`) sharing the same media dataset and GPU, for direct comparison |
| How documents (not photos) will be stored/served | Separate from movies/TV — Immich handles photos; this is still TBD |
| Whether to chase full 2.5Gb/s link speed on the RTL8125 NIC | Currently capped at 1Gb/s. Becomes actually relevant once real transfer volume flows through the SMB share (1Gb/s caps around 110MB/s) |
| Whether the shared 3-session NVENC cap becomes a real constraint | RTX 3050 driver-enforced limit of 3 simultaneous NVENC encode sessions is a shared pool across Plex + Jellyfin + any other GPU consumer. Workaround if needed: [keylase/nvidia-patch](https://github.com/keylase/nvidia-patch) (unofficial, against Nvidia's EULA, widely used) |
