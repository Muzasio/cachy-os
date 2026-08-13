component: librewolf-native
domain: browser-media-pipeline
tags: [librewolf, ffmpeg, gstreamer, cachyos, arch, vpn-netns, flatpak, pacman, media-decode]
date: 2026-08-12
status: working
type: system-troubleshooting
purpose: fix_media_decode_failure_after_system_update

known_issues_resolved:
  - issue: NS_ERROR_DOM_MEDIA_METADATA_ERR on all video sites in librewolf
    cause: ffmpeg major version bump (ffmpeg-git removed, ffmpeg 2:9.0-5 installed 2026-08-12) broke ABI compat with librewolf-bin 1:153.0_3-1, compiled against older libavcodec.so/libavformat.so
    fix: "sudo pacman -S ffmpeg4.4"
    verify_cmd: play video in librewolf on any site, confirm no MediaDecoderStateMachine decode errors in terminal
    ruled_out:
      - hardware_acceleration (vainfo confirmed VA-API/NVDEC healthy, disabling hw accel in about:config did not fix)
      - browser_cache (rm -rf ~/.cache/librewolf/ not attempted, ruled out via user report)
      - gstreamer_version (downgrade attempted, blocked by gst-plugin-va dependency chain, abandoned)
      - vpn/dns (resolvconf desync was separate unrelated issue, fixed independently via sudo resolvconf -u)
      - single_site_issue (confirmed multi-site, browser-wide)

  - issue: "resolvconf: signature mismatch: /etc/resolv.conf"
    cause: resolvconf tracking file out of sync, tun0 not present when checked (vpn disconnected)
    fix: "sudo resolvconf -u after reconnecting vpn (tun0 must exist)"
    verify_cmd: "cat /etc/resolv.conf; nameserver should be 10.98.0.1 after vpn connect"

  - issue: flatpak librewolf fails inside netns+sudo-u wrapper
    cause: "bubblewrap sandbox cannot create nested user namespace through ip netns exec + sudo -u chain (CanCreateUserNamespace EPERM), also drops DISPLAY env var"
    fix: reverted to native librewolf-bin in pvpn script, abandoned flatpak approach
    verify_cmd: "grep librewolf ~/.local/bin/pvpn should show bare 'librewolf', not flatpak run"

pipeline:
  - step: identify_ffmpeg_abi_break
    cmd: "grep -iE 'gst-plugins|ffmpeg|libva|mesa|nvidia' /var/log/pacman.log | tail -30"
    exec_mode: manual
    interactive: false
  - step: install_compat_layer
    cmd: "sudo pacman -S ffmpeg4.4"
    exec_mode: manual
    interactive: true
    creates: legacy ffmpeg4.4 shared libs alongside ffmpeg 9.0

teardown:
  - "sudo pacman -R ffmpeg4.4   # once librewolf-bin >=153.0.3_1-4 syncs to mirror with rebuilt ffmpeg-9 bindings"
  - "flatpak uninstall io.gitlab.librewolf-community   # unused, netns-incompatible"

state_assumptions:
  vpn_script_path: ~/.local/bin/pvpn
  vpn_config_path: ~/Downloads/<protonvpn-config>.ovpn
  netns_name: protected
  local_user: <placeholder>
  vpn_launch_target: librewolf (native, reverted from flatpak run io.gitlab.librewolf-community)
  os: CachyOS (Arch-based)
  browser_pkg: librewolf-bin 1:153.0_3-1 (AUR/native, not flatpak)
  shell: fish (interactive use), bash (pvpn script shebang)

security_notes:
  - "pvpn script stores ProtonVPN OpenVPN credentials in plaintext via printf into a mktemp file: CRED_USER=<placeholder>, CRED_PASS=<placeholder>"
  - action_required: rotate ProtonVPN OpenVPN credential pair in account dashboard if script contents were ever shared/logged outside local machine
  - cred_file_handling: mktemp + chmod 600 + rm in trap cleanup, acceptable at rest but was pasted in plaintext during this session

open_todos:
  - watch for librewolf-bin package version >=153.0.3_1-4 rebuilt against ffmpeg 9.0, then remove ffmpeg4.4 compat package
  - confirm gst-plugin-va / gstreamer core stack remain stable at 1.28.6-1 (no downgrade was completed, reinstalled clean instead)
  - rotate protonvpn ovpn credentials (plaintext exposed in chat)
