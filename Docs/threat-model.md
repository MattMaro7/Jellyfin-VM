# Threat Model

A short, honest description of what this home lab protects, who it protects against, and what it does not cover.

## What is being protected

| Asset | Why it matters |
|---|---|
| Jellyfin admin account | Full control of the server and its library |
| The Debian VM | A foothold on my host machine's network if compromised |
| Media library | Personal content I own; not meant for public access |
| My Tailscale account | Controls which devices can reach the server |

## Setup in one paragraph

A headless Debian 13 VM runs in VirtualBox on a Windows 11 host. Jellyfin listens on port 8096. The VM joins a Tailscale network (tailnet) and is tagged `tag:server`. The host firewall (ufw) denies inbound traffic by default and allows specific ports only on the `tailscale0` interface. A Tailscale ACL limits which devices may reach the server and on which ports. Nothing is port-forwarded on the router.

## Who or what I am defending against

1. Internet scanners and attackers. Nothing is forwarded from the router, and the VM sits behind VirtualBox's NAT, so the service has no public exposure.
2. Other devices on the local network. ufw only accepts the allowed ports on `tailscale0`, so devices on the same LAN should not reach Jellyfin through the VM's regular address.
3. Devices on my own tailnet that should not have access. The ACL is meant to deny devices that are not members or do not match the allowed source rules.
4. Mistakes by me. Most real incidents in this project were misconfiguration (full disks, wrong mounts, default installer choices), not attackers. Verification steps and documentation are part of the defense.

## Controls in place

| Threat | Control | How I checked it |
|---|---|---|
| Public exposure of Jellyfin | No port forwarding; VM behind NAT; service reachable only via tailnet | See `verification.md` |
| LAN access to the VM | ufw default deny incoming; allow rules scoped to `tailscale0` | See `verification.md` |
| Unauthorized tailnet devices | Tailscale ACL with tag-based rules | See `verification.md` |
| Unauthenticated admin setup | Completed the Jellyfin first-run wizard immediately and set a strong admin password | Manual check |
| Unnecessary attack surface | Headless install, no desktop environment, SSH only where needed | `systemctl list-units`, `ss -tlnp` |
| Host exposure through VM integration | Shared folder used briefly for file export, then removed | VirtualBox settings |

## Assumptions

- My Tailscale account is protected (strong password, two-factor authentication) and has not been compromised.
- The host operating system and VirtualBox are trusted and kept updated.
- Devices I add to the tailnet are mine and reasonably secure.

## Out of scope

- Physical access to the host or the virtual disk files.
- Compromise of my Tailscale account or Tailscale's own infrastructure.
- Vulnerabilities in Jellyfin, Debian, or the Linux kernel that are not yet patched. I rely on updates for these.
- Denial of service, and attackers who already have code execution on the host.
- Protecting the media itself from copying by authorized users.

## Residual risks and known gaps

- SSH is open to the tailnet (when enabled). A compromised tailnet device that matches the ACL could attempt to log in. Planned hardening: key-only authentication, a non-root user, and closing port 22 when not needed.
- Traffic inside the tailnet is encrypted by Tailscale (WireGuard), but Jellyfin itself serves plain HTTP. Acceptable for a private network; the next step would be `tailscale serve` for HTTPS.
- No automatic patching configured yet. Updates are manual until I set up unattended upgrades.
- No intrusion monitoring. I would see problems through logs only if I looked.
- Backups are not in place. The VM and the data disk are single copies.

## What I would add next

1. Unattended security upgrades.
2. Key-only SSH with Tailscale SSH as an alternative.
3. HTTPS through `tailscale serve`.
4. A scheduled backup of Jellyfin's configuration directory.
5. Basic log review (journalctl, ufw logs) on a schedule.
