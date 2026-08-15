# librewolf-native

**Domain:** browser-media-pipeline
**Tags:** librewolf, ffmpeg, gstreamer, cachyos, arch, vpn-netns, flatpak, pacman, media-decode
**Date:** 2026-08-12
**Status:** working
**Type:** system-troubleshooting
**Purpose:** fix_media_decode_failure_after_system_update

---

## Known Issues Resolved

### 1. `NS_ERROR_DOM_MEDIA_METADATA_ERR` on all video sites in librewolf

- **Cause:** ffmpeg major version bump (ffmpeg-git removed, ffmpeg 2:9.0-5 installed 2026-08-12) broke ABI compat with librewolf-bin 1:153.0_3-1, compiled against older libavcodec.so/libavformat.so
- **Fix:**
  ```bash
  sudo pacman -S ffmpeg4.4
  ```
- **Verify:** play video in librewolf on any site, confirm no MediaDecoderStateMachine decode errors in terminal
- **Ruled out:**
  - hardware_acceleration (vainfo confirmed VA-API/NVDEC healthy, disabling hw accel in about:config did not fix)
  - browser_cache (`rm -rf ~/.cache/librewolf/` not attempted, ruled out via user report)
  - gstreamer_version (downgrade attempted, blocked by gst-plugin-va dependency chain, abandoned)
  - vpn/dns (resolvconf desync was separate unrelated issue, fixed independently via `sudo resolvconf -u`)
  - single_site_issue (confirmed multi-site, browser-wide)

### 2. `resolvconf: signature mismatch: /etc/resolv.conf`

- **Cause:** resolvconf tracking file out of sync, tun0 not present when checked (vpn disconnected)
- **Fix:**
  ```bash
  sudo resolvconf -u   # after reconnecting vpn (tun0 must exist)
  ```
- **Verify:**
  ```bash
  cat /etc/resolv.conf   # nameserver should be 10.98.0.1 after vpn connect
  ```

### 3. flatpak librewolf fails inside netns+sudo-u wrapper

- **Cause:** bubblewrap sandbox cannot create nested user namespace through `ip netns exec` + `sudo -u` chain (CanCreateUserNamespace EPERM), also drops DISPLAY env var
- **Fix:** reverted to native librewolf-bin in pvpn script, abandoned flatpak approach
- **Verify:**
  ```bash
  grep librewolf ~/.local/bin/pvpn   # should show bare 'librewolf', not flatpak run
  ```

---

## Pipeline

| Step | Command | Exec Mode | Interactive | Notes |
|---|---|---|---|---|
| identify_ffmpeg_abi_break | `grep -iE 'gst-plugins\|ffmpeg\|libva\|mesa\|nvidia' /var/log/pacman.log \| tail -30` | manual | false | — |
| install_compat_layer | `sudo pacman -S ffmpeg4.4` | manual | true | creates legacy ffmpeg4.4 shared libs alongside ffmpeg 9.0 |

---

## Teardown

```bash
sudo pacman -R ffmpeg4.4   # once librewolf-bin >=153.0.3_1-4 syncs to mirror with rebuilt ffmpeg-9 bindings
flatpak uninstall io.gitlab.librewolf-community   # unused, netns-incompatible
```

---

## State Assumptions

| Key | Value |
|---|---|
| vpn_script_path | `~/.local/bin/pvpn` |
| vpn_config_path | `~/Downloads/<protonvpn-config>.ovpn` |
| netns_name | `protected` |
| local_user | `<placeholder>` |
| vpn_launch_target | librewolf (native, reverted from `flatpak run io.gitlab.librewolf-community`) |
| os | CachyOS (Arch-based) |
| browser_pkg | librewolf-bin 1:153.0_3-1 (AUR/native, not flatpak) |
| shell | fish (interactive use), bash (pvpn script shebang) |

---

## Security Notes

- pvpn script stores ProtonVPN OpenVPN credentials in plaintext via `printf` into a `mktemp` file: `CRED_USER=<placeholder>`, `CRED_PASS=<placeholder>`
- **Action required:** rotate ProtonVPN OpenVPN credential pair in account dashboard if script contents were ever shared/logged outside local machine
- **Cred file handling:** `mktemp` + `chmod 600` + `rm` in trap cleanup, acceptable at rest but was pasted in plaintext during this session

---

## Open TODOs

- [ ] watch for librewolf-bin package version >=153.0.3_1-4 rebuilt against ffmpeg 9.0, then remove ffmpeg4.4 compat package
- [ ] confirm gst-plugin-va / gstreamer core stack remain stable at 1.28.6-1 (no downgrade was completed, reinstalled clean instead)
- [ ] rotate protonvpn ovpn credentials (plaintext exposed in chat)
