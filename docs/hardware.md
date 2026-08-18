# Hardware Inventory

Captured from `full-inventory.txt` on 2026-08-17. Re-run the reference commands and update this table if hardware changes.

| Component | Spec | Notes |
|---|---|---|
| CPU | AMD Ryzen 7 3700X | 8 cores / 16 threads, AMD-V + AMD-Vi (IOMMU) enabled and active |
| GPU | NVIDIA GeForce RTX 3050 8GB (GA107, ZOTAC) | PCI `07:00.0` `[10de:2582]` + HD Audio function `07:00.1` `[10de:2291]`, both alone in **IOMMU Group 16** (clean split — no other devices to isolate) |
| RAM | 32GB (2x16GB G.Skill, 3200MT/s) | Only 2 of 4 DIMM slots populated (A2 + B2) — A1/B1 free, room to grow to 64GB later |
| Motherboard | ASUS TUF GAMING B550-PLUS (AM4) | BIOS/UEFI 3611, IOMMU confirmed active in `dmesg` |
| Storage — bulk | 3x 8TB Seagate BarraCuda `ST8000DM004-2U9188` | ZFS RAIDZ1 pool named `HDDs`, 21.8TB usable, ONLINE, no errors. ⚠️ SMR drives (confirmed via smartctl: "Model Family: Seagate BarraCuda 3.5 (SMR)") — see risk note below |
| Storage — extra | 1x 2TB Seagate BarraCuda `ST2000DM008-2FR102` | Currently NTFS-formatted, **not** in any ZFS pool — unassigned. Also an SMR drive. Role still TBD |
| Storage — fast | 2x 500GB KIOXIA-EXCERIA G2 NVMe | ZFS-mirrored as `rpool` (Proxmox boot pool, ~460GB usable), hosts both `local` and `local-zfs` Proxmox storage — i.e. all LXC/VM root disks land here. No capacity set aside separately as a ZFS special/cache device for the `HDDs` pool |
| Network | Realtek RTL8125 2.5GbE (`06:00.0`, `[10ec:8125]`), bridged as `vmbr0` | ⚠️ Link currently negotiating at only **1000Mb/s**, not the card's full 2.5Gb/s — check cabling/switch port if 2.5G throughput is wanted |

## ⚠️ SMR risk note

All four spinning drives (the three 8TB bulk drives and the 2TB extra drive) are SMR (Shingled Magnetic Recording), not CMR. SMR drives perform poorly on sustained/random writes and — more importantly — can make ZFS **resilvers** (rebuilding a RAIDZ1 vdev after a drive failure) dramatically slower, extending the window where a second drive failure would cause data loss.

The pool is already built as RAIDZ1 across 3 SMR drives — workable but non-ideal. For "important documents / irreplaceable photos," don't treat the RAIDZ1 pool alone as a backup — plan a second copy (e.g. Proxmox Backup Server on the spare 2TB drive, or an offsite/cloud copy of just the documents + Immich originals folders) rather than relying solely on RAIDZ1 redundancy.

## ZFS pool layout

```
HDDs        21.8T RAIDZ1 (3x 8TB SMR)     — bulk media/document storage
rpool         460G mirror (2x 500GB NVMe)  — Proxmox boot pool + all LXC/VM root disks
```

## Software stack

- **Hypervisor:** Proxmox VE 9.2.2 (kernel 7.0.2-6-pve, Debian 13 "trixie" base) — confirmed via `pveversion`
- **Existing state:** `CT100` — existing LXC, purpose not yet documented
- **Media server:** Plex (`CT101`) and Jellyfin (`CT103`), tested side by side
- **File sharing:** Samba (`CT102`)
- **Photo/doc storage:** Immich for photos; documents storage TBD (Nextcloud, Paperless-ngx, or plain file share)
- **Local LLM testing:** Ollama (or similar), GPU-accelerated, sharing the GPU with media/photo workloads
