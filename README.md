# Homelab-Self-Hosted-Cloud-Storage-Project-Immich-Zer0-Drive-Cloudreve-
A secure, multi-user self-hosted media and file storage solution built on Docker, TrueNAS Scale, NFS persistence, Traefik reverse proxy, and layered security (CrowdSec + Authentik planned).

## Project Goals & Learning Outcomes

- Practice least-privilege principles (dedicated users/groups, non-root containers where possible)
- Master Docker Compose modularity and networking segmentation
- Implement secure NFS storage with UID/GID matching
- Configure reverse proxy with automatic HTTPS
- Multi-tenant setup with per-user storage quotas
- Debug common permission and thumbnail generation issues
- Custom branding and basic frontend overrides

## Architecture Overview

- Host: Ubuntu Server VM
- Storage Backend: TrueNAS Scale (ZFS datasets + NFS shares)
- Containers: Docker Compose (separate stacks for isolation)
- Reverse Proxy: Traefik (HTTPS via Let's Encrypt)
- Networking: Shared proxy-net for web-facing services; internal networks for DB/Redis
- Security Layers: CrowdSec (IPS) + Authentik (OIDC/forward auth)

TrueNAS Scale
   ├── Dataset: immich/photos (8TB quota, owner: immich UID )
   └── Dataset: cloudreve/uploads (4TB quota, owner: cloudreve UID )
         ↓ NFS shares (mapall to dedicated users)
Ubuntu Docker Host
   ├── /mnt/truenas/immich → Immich library & thumbnails
   ├── /mnt/truenas/cloudreve → Cloudreve user uploads
   ├── Separate local volumes for DBs, config, avatars
   └── Docker stacks:
        ├── immich/ (server, microservices, ML, postgres, valkey)
        └── cloudreve/ (cloudreve, postgres, redis) → branded "Zer0 Drive"
Traefik → Subdomains

## Setup Summary
# 1. TrueNAS Scale Configuration

- Created groups/users: immich and cloudreve
- Datasets: datapool/immich/photos and datapool/cloudreve/uploads with quotas
- NFS shares with mapall to dedicated users (secure ownership mapping)
- Network restricted to Ubuntu subnet

# 2. Ubuntu NFS Mounts

/mnt/truenas/immich   → nfs truenas-ip:/mnt/datapool/immich/photos
/mnt/truenas/cloudreve → nfs truenas-ip:/mnt/datapool/cloudreve/uploads
chown immich:user /mnt/truenas/immich
chown cloudreve:user /mnt/truenas/cloudreve

# 3. Immich Stack (~/docker/immich)
- Official docker-compose.yml + .env
# Key customizations:
- UPLOAD_LOCATION=/mnt/truenas/immich
- user: "immich:user" on server & microservices
- Dedicated Postgres (pgvecto.rs) + Valkey

- Networking: Only immich-server on proxy-net
- Traefik labels created
- Fixed thumbnail generation:
- Manual job trigger in Admin > Jobs
- Ensured writable /thumbs subdir
- Mobile app: Enable "Prefer remote images"
- Multi-User: 6 users, each with ~1TB quota (set in Admin > Users)

# 4. Zer0 Drive (Cloudreve) Stack (~/docker/cloudreve)

- Customized official compose
- Volumes:
  - /mnt/truenas/cloudreve:/cloudreve/uploads (big files)
  - ./data:/cloudreve/data (config, logs, conf.ini)
  - ./avatars:/cloudreve/avatar

- Networking: Only main service on proxy-net
- Traefik labels for files.example.com
- Custom branding:
- Overrode /cloudreve/statics/index.html with custom HTML landing page titled "Zer0 Drive"
- Cyber-themed design (black/green matrix style)

- Permission fixes:
- Initial run as root to create config
- Then chown host dirs and optional non-root user
- Storage policy: Local path /cloudreve/uploads
- Multi-User: 6 users in custom group, each with 500GB quota

# 5. Security & Future Enhancements
- Network segmentation (internal DBs unreachable)
- UID/GID matching for NFS security

# Planned:
- Authentik: Native OIDC for Immich, forward auth for Cloudreve

# Challenges Overcome
- NFS + Docker permission conflicts → solved with mapall + matching UIDs
- Immich thumbnails/black mobile images → manual job queue + preference setting
- Cloudreve non-root permission errors → dedicated config volume + staged chown
- Cloudreve default landing → custom static override

# Outcome
- Fully functional private Google Photos (Immich) + personal Dropbox (Zer0 Drive) with per-user quotas, secure external access, and custom branding.
- Skills Demonstrated
- Docker orchestration & networking
- Storage management (ZFS, NFS, quotas)
- Reverse proxy configuration
- Least-privilege hardening
- Troubleshooting container permissions and background jobs
- Basic frontend customization
