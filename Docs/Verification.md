# Verification

## Environment

| Item | Value |
|---|---|
| Host | Windows 11, VirtualBox |
| Guest | Debian 13 (headless), Jellyfin from the official apt repository |
| Network | VirtualBox NAT; Tailscale on host and VM |
| VM tag | `tag:server` |
| Test date | 11/13/2026 |

## Summary

| # | Claim | Test | Run from | Result |
|---|---|---|---|---|
| 1 | Jellyfin reachable over Tailscale | `Test-NetConnection <vm-tailscale-ip> -Port 8096` | Windows host (on tailnet) | True |
| 2 | Allowed SSH reachable over Tailscale | `Test-NetConnection <vm-tailscale-ip> -Port 22` | Windows host | True (intentionally allowed) |
| 3 | Other ports blocked over Tailscale | `Test-NetConnection <vm-tailscale-ip> -Port 80` | Windows host | False(blocked) |
| 4 | Not reachable on the VM's regular address | Not verified (see "Not verified" below) | n/a | Configured, not tested |
| 5 | Data lives on the large disk | `df -h /var/lib/jellyfin /var/cache/jellyfin /srv/media` | VM | /dev/sdb 196GB |
| 6 | Service is healthy | `systemctl status jellyfin` | VM | active running |

## Test details

### 1. Jellyfin reachable over Tailscale

```powershell
Test-NetConnection <vm-tailscale-ip> -Port 8096
```

Expected: `TcpTestSucceeded : True`. I also opened `http://<vm-tailscale-ip>:8096` in a browser and confirmed the Jellyfin page loaded.

### 2 and 3. Allowed versus blocked ports

Ports 8096 and 22 are intentionally allowed in both ufw and the Tailscale ACL. Ports without an allow rule (80, 443, 3389) act as negative controls.

```powershell
Test-NetConnection <vm-tailscale-ip> -Port 80
Test-NetConnection <vm-tailscale-ip> -Port 443
Test-NetConnection <vm-tailscale-ip> -Port 3389
```

### 4. Not reachable on the regular address

On the VM: `ip a` to find the non-`100.x` address. From a machine on the same virtual network:

```bash
nmap -Pn -p 22,8096 <vm-regular-ip>
```

Expected: both ports `filtered`, because ufw only allows them on the `tailscale0` interface.

This test only counts if the scanning machine can actually reach the VM's regular address. If the two machines are on separate NATs, the scan is filtered for the wrong reason.

### 5. Storage layout

```bash
lsblk -f
df -h /var/lib/jellyfin /var/cache/jellyfin /srv/media
```

The large disk is mounted at `/mnt/mediadisk` by UUID in `/etc/fstab`, and bind mounts point Jellyfin's data, cache, and media directories at it.


### . Service health

```bash
systemctl status jellyfin
```

## Invalid test I ran, and why it was invalid

An early scan from a Kali VM against the VM's Tailscale address returned every port as `filtered`. Kali was not logged in to the tailnet, so its packets never reached the `100.x` address, and the result would have looked identical with no firewall or ACL at all.

I repeated the tests from the Windows host, which is on the tailnet, so that the ACL and ufw rules were actually being evaluated.

## Closed versus filtered

During an earlier install, scans of unused ports reported `closed` (the host replied with a reset), while later results on the rebuilt VM showed `filtered` (packets dropped). Both mean the port cannot be used, but `filtered` indicates the packets were dropped by a rule instead of reaching a host with nothing listening. I treat "allowed port works and the LAN address is filtered" as the stronger evidence than any single closed-port result.

## Not verified

**Claim 4: the VM's regular (non-Tailscale) address does not accept ports 22 and 8096.**

I configured ufw to allow these ports only on the `tailscale0` interface
(`ufw status verbose`), and `ss -tlnp` shows the services listen on all
interfaces, so ufw is the intended control. I could not confirm this from
a second machine. My Kali VM and the Debian VM did not share a network I
could get working, so scans from Kali could not reach the Debian VM, and a
"filtered" result would have meant "no route" instead of "blocked by ufw."
I did not record that result as evidence.

What I can support instead:
- The ufw rules, as configured (`ufw status verbose`).
- VirtualBox's NAT has no port-forwarding rules for this VM, so it is not
  directly reachable from outside the host.

To finish this test, I would put both VMs on the same NAT Network, confirm
Kali can reach the Debian VM (for example, with a temporary open test
port), and then rerun the scan.

## Limitations

- These checks show reachability from the devices I tested, not every possible path.
- I did not test for vulnerabilities in Jellyfin or the operating system.
- Results reflect the configuration on the test date above.
