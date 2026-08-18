# Media Storage + SMB Share — Decisions & Runbook

_Captured 2026-08-17._

## Decision locked in this session

- **Network share protocol: SMB**, not NFS. Main PC is Windows 11 Pro (NFS client exists there but is clunkier and Home-edition-incompatible in general; SMB has zero caveats regardless of client OS).
- **Architecture: dedicated LXC for Samba** (`CT102`, hostname `smb`), not installed on the Proxmox host directly and not folded into the Plex container. Matches the "one container, one job" pattern already used for Plex (`CT101`). Privileged container (`--unprivileged 0`) — same rationale as Plex: avoids unprivileged LXC UID/GID remapping pain when bind-mounting host ZFS paths.

See `architecture-decisions.md` for the consolidated decision log.

## Layout

- ZFS datasets on the `HDDs` pool: `HDDs/media/movies`, `HDDs/media/tv` (mirrors Plex's own recommended one-top-level-folder-per-library-type convention; add more, e.g. `HDDs/media/music`, the same way later).
- Host mounts these at `/HDDs/media/...` (ZFS default mountpoint = pool name).
- Bind-mounted via Proxmox `mp0` into **both** `CT102` (Samba) and `CT101` (Plex) at `/media` inside each container — same underlying dataset, visible to both.

## Runbook

```bash
# on the Proxmox host
zfs create HDDs/media
zfs create HDDs/media/movies
zfs create HDDs/media/tv

pveam update
pveam available --section system | grep debian-13
pveam download local <exact-filename>
pct create 102 local:vztmpl/<exact-filename> \
  --hostname smb \
  --cores 2 \
  --memory 1024 \
  --rootfs local-zfs:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 0
pct start 102
pct set 102 -mp0 /HDDs/media,mp=/media

pct exec 102 -- apt update
pct exec 102 -- apt install -y samba
```

Inside the container (`pct enter 102`), append to `/etc/samba/smb.conf`:

```
[Movies]
   path = /media/movies
   read only = no
   browsable = yes
   valid users = joao
   force user = joao
   force group = joao

[TV]
   path = /media/tv
   read only = no
   browsable = yes
   valid users = joao
   force user = joao
   force group = joao
```

`force user`/`force group` sidesteps ZFS/Samba permission mapping fuss — every file lands owned by the same account regardless of connection details. Then:

```bash
adduser --no-create-home --disabled-password joao
smbpasswd -a joao
systemctl restart smbd
```

From Windows: `\\<CT102-IP>\Movies` in File Explorer's address bar (get the IP via `pct exec 102 -- hostname -I` from the host), authenticate as `joao`.

Give Plex (`CT101`) the same media:

```bash
pct set 101 -mp0 /HDDs/media,mp=/media
```

Then add libraries in Plex's web UI pointed at `/media/movies` and `/media/tv`.

## Note for later

The RTL8125 NIC negotiating at 1Gb/s instead of its rated 2.5Gb/s (already on the master project instructions' open list) becomes actually relevant once real transfer volume starts flowing through this share — 1Gb/s caps around 110MB/s. Not a blocker, just the natural trigger to revisit that item if transfers feel slow.

## Still open

See `architecture-decisions.md` for the full, current list.
