# NAS / Server config tracker and guide

Single physical server combining three roles:

1. **NAS** — storage for personal data: photos/images (via Immich) and important documents.
2. **Media vault** — movies, TV shows, and music, served by Plex and/or Jellyfin (evaluating both side by side).
3. **Homelab / AI sandbox** — Proxmox VE hypervisor running assorted services in LXCs/VMs, plus local LLM experimentation (Ollama), sharing GPU/VRAM and RAM with the other workloads.

## Docs

- [`docs/hardware.md`](docs/hardware.md) — hardware inventory, ZFS pool layout, network
- [`docs/architecture-decisions.md`](docs/architecture-decisions.md) — decisions made so far, and what's still open
- [`docs/gpu-passthrough-plex.md`](docs/gpu-passthrough-plex.md) — NVIDIA driver install on host + Plex LXC with HW transcoding
- [`docs/media-storage-smb.md`](docs/media-storage-smb.md) — ZFS media datasets + Samba share for movies/TV
- [`docs/jellyfin.md`](docs/jellyfin.md) — Jellyfin LXC, shared media + GPU, running alongside Plex for comparison

## Stack summary

- **Hypervisor:** Proxmox VE 9.2.2 (kernel 7.0.2-6-pve, Debian 13 "trixie" base)
- **Media servers:** Plex (`CT101`) and Jellyfin (`CT103`), tested side by side, both natively installed (no Docker), sharing the GPU and the same ZFS media dataset
- **File sharing:** Samba (`CT102`) for getting files onto the pool from the Windows PC
- **Photo storage:** Immich (planned)
- **Local LLM:** Ollama or similar, GPU-accelerated (planned)

See `docs/architecture-decisions.md` for the full list of what's settled vs. still open.
