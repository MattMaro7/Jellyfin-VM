# Troubleshooting Log

Format for each entry: Symptom > Diagnosis > Fix > Lesson. 

---

## 1. `scp` fails with "Connection refused"

- Symptom: `scp` to the VM over Tailscale returned `connect to host ... port 22: Connection refused`.
- Diagnosis: "Refused" (an immediate reply) is different from a timeout (dropped packets). Something answered and said no, which pointed to the SSH service rather than the firewall or ACL.
- Fix: The minimal Debian image had no SSH server. Installed `openssh-server`, enabled it, and confirmed with `ss -tlnp | grep :22`.
- Lesson: Refused versus timed out tells you where to look first.

---

## 2. Disk filled up during the media transfer

- Symptom: The VM ran out of space after copying a video file.
- Diagnosis: The 20 GB OS disk was too small for the library.
- Fix: Added a second virtual disk (about 200 GB), formatted it as ext4, mounted it by UUID, and moved data with `rsync`.
- Lesson: Size the data disk before loading data. A full disk during a copy can also leave a truncated file, so I verify file size after transfers.

---

## 3. Mount appeared to work but nothing was on the new disk

- Symptom: `blkid /dev/sda1` printed nothing, and a bind mount I created seemed to succeed.
- Diagnosis: `lsblk -f` showed the filesystem sat directly on `/dev/sda` with no partition table, so `/dev/sda1` did not exist. My `mount /dev/sda1` had failed silently, leaving an empty folder on the OS disk, and the bind mount pointed at that folder. The giveaway was `/` and `/var/lib/jellyfin` sharing one device in `lsblk` output.
- Fix: Unmounted the bad bind mount, mounted the whole device by UUID, then recreated the bind mount.
- Lesson: Confirm a mount with `df -h <path>` instead of assuming the command worked.

---

## 4. Device names changed between boots

- Symptom: The large disk showed up as `sda` in one session and `sdb` in another.
- Diagnosis: Linux assigns device letters in detection order, which can change.
- Fix: Used UUIDs in `/etc/fstab` instead of device names.
- Lesson: Mount by UUID.

---

## 5. VM appeared to shut itself down during boot

- Symptom: After adding a disk and running an update, the VM showed progress bars and then powered off, every boot.
- Diagnosis:
  - The VirtualBox log (hypervisor level) showed a guest-unresponsive warning of about 222 seconds, followed by a suspend and then a power-off request issued through the VirtualBox window. The guest had not shut itself down; the VM was powered off from the GUI after it appeared frozen.
  - The guest journal (`journalctl -b -1 -p err`) showed GNOME and virtual-graphics-driver errors, which meant the install included a full desktop environment I did not need on a server.
- Fix: Rebuilt the VM from the Debian netinst ISO, unchecked every desktop option, and kept only the SSH server and standard utilities. The data disk was left untouched during the reinstall.
- Root cause: Not fully confirmed. The slow boot and graphics errors were consistent with a desktop environment struggling in a VM, but I did not isolate a single cause.
- Lesson: A hypervisor log and a guest log answer different questions. A headless server does not need a desktop.

---

## 6. Jellyfin installer reported insufficient space despite 163 GB free

- Symptom: `install-jellyfin.sh` printed `Insufficient free space for /var/lib/jellyfin: ~1.6 GB found, 2 GB required`, even though `df -h /var/lib/jellyfin` showed 163 GB free on the bind-mounted disk.
- Diagnosis: The guided installer had split the 20 GB OS disk into small partitions, leaving `/var` at about 2 GB. The "found" figure matched `/var`'s free space, not the bind-mounted path, which suggested the script's check did not account for the bind mount.
- Fix: Skipped the wrapper script and installed through apt by adding the Jellyfin repository manually.
- Root cause: I did not read the script's check logic, so the exact reason is unconfirmed.
- Lesson: When a tool's output contradicts `df`, trust the system command and find another route, but note that the mismatch was never explained.

---

## 7. Jellyfin service failed to start (`signal=ABRT`)

- Symptom: After installing, `systemctl status jellyfin` showed the service failing repeatedly and stopping with "start request repeated too quickly."
- Diagnosis: The journal reported that `/var/cache/jellyfin` was about 0.5 GB short of the free space the service wanted. I believe recent Jellyfin versions check free space on each data directory at startup, though I have not confirmed the exact threshold.
- Fix: Created a folder on the large disk and bind-mounted it onto `/var/cache/jellyfin`, added it to `/etc/fstab`, set `jellyfin` ownership, then reset the failed state and restarted.
- Lesson: Moving one directory was not enough. List every directory a service writes to and check each one's free space.

---

## 8. `/srv` reached 100% on the OS disk

- Symptom: No room for new media. `lsblk -f` showed `/srv` on a small partition of the OS disk.
- Diagnosis: Guided partitioning had created a separate 12 GB `/srv` partition, and the media directory was on it.
- Fix: Stopped Jellyfin, copied `/srv/media` to the large disk with `rsync`, verified the copy, bind-mounted the new location onto `/srv/media`, and made it permanent in `/etc/fstab`.
- Lesson: Check `lsblk` and `df -h` right after an install, before loading data.

---

## 9. A scan I should not have trusted

- Symptom: Scanning the VM's Tailscale address from Kali showed every port as filtered.
- Diagnosis: Kali was not on the tailnet, so the packets never reached the address. The result would have been identical with no firewall.
- Fix: Retested from the Windows host, which is on the tailnet. See `verification.md`.
- Lesson: A test that cannot fail is not evidence. Run a positive control (an allowed port that should succeed) alongside the negative one.

