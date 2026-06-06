# Dev VMs — OPNsense + OpenWRT plugin rigs

Two VMs on Proxmox (`pve.bodaay.org`, .91) for developing the OPNsense plugin and the OpenWRT package without touching production gateways. Both bootstrapped fully hands-off (no console clicks) and reachable from the Mac with SSH key auth.

| VM  | Host             | OS                          | IP            | CPU/RAM/Disk       |
|-----|------------------|-----------------------------|---------------|--------------------|
| 113 | `opnsense-dev`   | OPNsense 25.7 (FreeBSD 14.3)| 192.168.0.224 | 4 / 4 GB / 32 GB   |
| 114 | `openwrt-dev`    | OpenWrt 24.10.0 x86-64      | 192.168.0.217 | 1 / 512 MB / 2 GB  |

SSH key auth: `~/.ssh/id_ed25519` (khalefa) and the homelab agent key both installed. Root passwords stored in 1Password Agent vault items **OPNsense Dev VM (113)** and **OpenWRT Dev VM (114)** — fields kept in sync with the actual VMs.

## OPNsense 113 — bootstrap method

OPNsense has no cloud-init. The serial-amd64 preinstalled image was imported, then first-boot config was driven over the qemu serial unix socket from Proxmox:

1. Image: `OPNsense-25.7-serial-amd64.img.bz2`, decompressed + imported to `local-lvm`, disk resized to 32 GB.
2. VM created with `q35`, `serial0 socket`, `vga serial0` — gives a unix-domain serial socket at `/var/run/qemu-server/113.serial0`.
3. After boot, a small Python driver on Proxmox `connect()`s that socket and runs an expect-style loop:
   - Auto-login as `root` / `opnsense` (default).
   - Select menu option **8** (drop to shell — note this is `tcsh`, NOT bash).
   - Stream a first-boot shell script in as 70-char base64 chunks via `printf "%s" "chunk" >> /tmp/blob.b64`, then `base64 -d ... | /bin/sh`.
   - The script: writes both SSH pubkeys to `/root/.ssh/authorized_keys`, sets root password via `pw usermod -H 0`, patches `/conf/config.xml` (Python `ElementTree`) to enable SSH + permit root login + set LAN (`vtnet0`) to DHCP, then `configctl interface reload all`.
4. LAN gets DHCP from the upstream OPNsense, sshd starts on :22, key auth works from the Mac.

**Gotchas that bit us during bring-up:**
- OPNsense's root shell is `tcsh`. `2>&1` errors with "Ambiguous output redirect" and heredocs follow csh rules. **Always `/bin/sh` before doing anything non-trivial**, or wrap commands as `sh -c '...'`.
- Don't paste long lines over the serial console — they wrap, interleave with the next command, and leave csh stuck on the `?` continuation prompt. Chunk under 80 chars, or use the base64-blob trick above.
- `service sshd onestart` fails on OPNsense (`sshd does not exist in /etc/rc.d`). SSH is managed by configd — enable it by writing `<ssh><enabled>enabled</enabled></ssh>` into `config.xml` and calling `configctl interface reload all`.
- LAN-only single-NIC setup: `vtnet0` ships as static `192.168.1.1/24` (LAN). Change to DHCP (clear `<ipaddr>` + set to `dhcp`, remove `<dhcpd/lan/enable>`) so it gets a real IP on `vmbr0`.

The full bootstrap driver was a throwaway — credentials staged at `/tmp/pharos-vm-creds.sh` on Proxmox, deleted after the vault was updated.

## OpenWRT 114 — bootstrap method

Cloud-init equivalent on OpenWRT is **uci-defaults**: any script in `/etc/uci-defaults/` runs once on first boot, then deletes itself. The OpenWRT x86-64 ext4-combined image is a raw disk file, so the script was injected directly into the image before first boot:

1. Image: `openwrt-24.10.0-x86-64-generic-ext4-combined.img.gz`, gunzipped to raw.
2. Mounted the second partition (rootfs) via `losetup -P`, dropped `/etc/uci-defaults/99-pharos-firstboot` into it. The script:
   - Sets root password from a pre-computed SHA-512 hash.
   - Switches LAN (`br-lan`) from static `192.168.1.1` to DHCP client.
   - Disables the LAN DHCP server (we're a client, not a router here).
   - Installs both SSH pubkeys to `/etc/dropbear/authorized_keys`, disables password auth.
   - Sets hostname `openwrt-dev`.
3. Image imported with `qm importdisk 114 ...`, VM booted. DHCP'd within 7 s. SSH key works.

**Gotchas:**
- OpenWRT default shell is `ash` (BusyBox), not bash. `hostname` is also missing as a separate binary (use `uci get system.@system[0].hostname`).
- The `gunzip` step prints "trailing garbage ignored" — harmless, the image is fine.

## Quick reconnect cheatsheet

```sh
# From Mac (key auth):
ssh root@192.168.0.224     # OPNsense  — drop into sh: ssh root@... 'sh -c "..."'
ssh root@192.168.0.217     # OpenWRT

# Console fallback (from Proxmox):
ssh root@pve.bodaay.org 'socat - UNIX-CONNECT:/var/run/qemu-server/113.serial0'   # OPNsense serial
ssh root@pve.bodaay.org 'qm terminal 114'                                          # OpenWRT console
```

If you ever need to redo first-boot, the cleanest reset is `qm stop <vmid>; qm rollback <vmid> first-boot` (if a snapshot exists) or just rebuild from the image.
