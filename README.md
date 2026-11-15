# Secure Jellyfin Home Lab

Self-hosted Jellyfin media server on a headless Debian VM (VirtualBox),
reachable only through a Tailscale private network.

## Goals
- No services exposed to the public internet
- Least-privilege access between devices
- Document and verify every claim

## Architecture
(short paragraph + link to docs/architecture.md)

## What I built
- Headless Debian 13 install (no desktop environment)
- Jellyfin installed from the official repo
- ufw: default deny inbound, allow 8096 (and 22) only on tailscale0
- Tailscale ACL: tagged server reachable only on required ports
- Separate data disk mounted via UUID in fstab, with bind mounts for
  Jellyfin's data, cache, and media directories

## Verification
[Docs/Verification.md](https://github.com/MattMaro7/Jellyfin-VM/blob/36cc2188e39d5748d7f3a8b385ce327e4aab83ee/Docs/Verification.md)

## Troubleshooting highlights
[Docs/troubleshooting-log.md](https://github.com/MattMaro7/Jellyfin-VM/blob/36cc2188e39d5748d7f3a8b385ce327e4aab83ee/Docs/troubleshooting-log.md) 

## Disclaimer
No media is included. Only content I own is used for testing.
