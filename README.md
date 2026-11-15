# Secure Jellyfin Home Lab

A self-hosted Jellyfin media server on a headless Debian VM, reachable only through a Tailscale private network. Built as a hands-on IT and security project, with every claim tested or marked as untested.

## Goals

- No service exposed to the public internet
- Access limited to my own devices, on the ports they need
- Document each change, test, and failure, including the ones I never fully explain

## Architecture

```
Windows host --> Tailscale tunnel --> ACL (tag:server) --> ufw --> Jellyfin (8096) / SSH (22)
```

The VM runs Debian 13 with no desktop, inside VirtualBox behind NAT. A second virtual disk holds the media and Jellyfin's data through bind mounts. Full diagram and storage layout: [Docs/Architecture.md](Docs/Architecture.md).

## What I built

- Headless Debian 13 install from the netinst image (SSH server and standard utilities only)
- Jellyfin installed from the official apt repository
- ufw with default-deny inbound, allowing ports 8096 and 22 only on the `tailscale0` interface
- A Tailscale ACL using a `tag:server` tag to limit which devices can reach the VM
- A separate ~196 GB data disk mounted by UUID, with bind mounts for `/var/lib/jellyfin`, `/var/cache/jellyfin`, and `/srv/media`

## Verification

Tested on 11/13/2026. Full commands, outputs, and caveats: [Docs/verification.md](Docs/Verification.md).

| Claim | Result |
|---|---|
| Jellyfin reachable over Tailscale (port 8096) | Verified |
| SSH reachable over Tailscale (port 22, intentionally allowed) | Verified |
| A port with no allow rule is blocked over Tailscale | Verified |
| Jellyfin data lives on the large disk | Verified (`df -h`) |
| Service healthy | Verified (`systemctl status`) |
| VM's regular (non-Tailscale) address does not accept 22 and 8096 | Configured, **not verified** |

I could not get a second VM onto a network that reached the Debian VM, so an external scan of its regular address would have returned "filtered" for the wrong reason. I did not count that result. The reasoning is in the verification doc.

## Threat model

I defend against internet scanners, other devices on the local network, unauthorized devices on my own tailnet, and my own configuration mistakes. Physical access, a compromised Tailscale account, and unpatched vulnerabilities are out of scope. Known gaps (SSH open to the tailnet, no automatic updates, no backups) and next steps are listed in [Docs/threat-model.md](Docs/threat-model.md).

## Troubleshooting highlights

Full log with symptoms, diagnosis, fixes, and lessons: [Docs/troubleshooting-log.md](Docs/troubleshooting-log.md).

- **A mount that looked successful was writing to the wrong disk.** `lsblk -f` showed the filesystem had no partition table, so the device I mounted did not exist.
- **The Jellyfin installer reported too little space** while `df` showed 163 GB free. I installed through apt instead and documented that I never found the cause.
- **The service refused to start** because `/var/cache/jellyfin` was about 0.5 GB short on a small partition. Moving one directory to the large disk was not enough; I had to check each directory Jellyfin writes to.
- **A scan that could not fail.** A Kali VM outside the tailnet reported every port filtered. I discarded it and retested from a device on the tailnet.

## Lessons learned

- Mount by UUID. Device letters like `sda` and `sdb` swapped between boots.
- Confirm a mount with `df -h <path>`; do not assume the command worked.
- Guided installers can split a small disk into partitions that run out of space, so check `lsblk` and `df -h` right after installing.
- A negative test result needs a positive control, or it proves nothing.
- Hypervisor logs and guest logs answer different questions.
- A server does not need a desktop environment.

## Limitations and next steps

- Move to key-only SSH, or close port 22 when not in use
- Serve Jellyfin over HTTPS with `tailscale serve`
- Finish the LAN-side scan with both VMs on a shared NAT Network and a positive control
- Back up Jellyfin's configuration directory

## Repo layout

```
README.md
LICENSE
configs/            sanitized config examples (ufw, fstab, ACL)
docs/
  Architecture.md
  threat-model.md
  verification.md
  troubleshooting-log.md
  screenshots/
```

## Disclaimer

No media files are included in this repository, and only content I own is used for testing. Jellyfin, Debian, Tailscale, and VirtualBox are the property of their respective owners. Configuration files are sanitized; UUIDs, addresses, and account details are replaced with placeholders or blocked out.

Released under the MIT License.
