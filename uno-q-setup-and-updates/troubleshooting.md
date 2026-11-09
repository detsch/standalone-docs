---
title: Troubleshoot the UNO Q Walkthrough
description: Checks for build, console, enrollment, networking, and application update failures.
type: page
doc-category: informational
authors: David Griego, Codex:GPT-6, Andre Detsch, Claude:Opus-5.5
last-edited: 2026-11-09
license: MIT
access: public
references:
  - https://github.com/foundriesio/meta-foundries/tree/ee25a99b3f2ef66af36a51da583f8223bcaf58d1
  - https://github.com/foundriesio/update-server/blob/e307a38b702da479b971b8286d534f0545956dec/docs/quick-start.md
  - https://github.com/foundriesio/update-server/blob/e307a38b702da479b971b8286d534f0545956dec/docs/updates.md
  - https://github.com/foundriesio/composeapp/tree/07a5b14b2e55f6882f7323ad6a32e62438aa5098
  - https://github.com/arduino/docs-content/tree/dab66ecbd6ad52cd742da6c19c5b6330a7f2caae
relations:
  parent: README.md
---

# Troubleshoot the UNO Q Walkthrough

[Guide overview](README.md)

## Troubleshoot a Failed Checkpoint

| Symptom | Check |
| --- | --- |
| Resolved commits differ from the [pinned table](build.md#what-is-pinned) | Move any `kas/uno-q-wrynose.lock.yml` from an earlier guide revision out of `kas/`; kas applies it automatically. Build with this revision's `kas/uno-q-wrynose.yml`, which pins meta-foundries and the two Qualcomm layers that the lockfiles depend on. |
| kas reports `Branch "..." in repository "..." does not contain commit "..."` | A configuration from an earlier guide revision named a branch that did not contain its pinned commit. Use this revision's files: every repository in the [pinned table](build.md#what-is-pinned) is on a branch that contains its commit. |
| kas checkout fails to apply a patch | The `meta-lts-mixins` `linux-firmware` patches come from `meta-qcom` at its pinned commit and fit only the pinned revisions. Check the resolved commits. |
| BitBake stops at parsing, reports `Nothing PROVIDES`, or fails a `license-format` QA check | These follow from mixing revisions, for example development branches with these Wrynose layers. Check the resolved commits and remove any old lockfile. |
| Layer compatibility error | Check the resolved commits and the optional M-05 layer, copied from this guide revision. Do not bypass `LAYERSERIES_COMPAT`. |
| A pinned revision fails to fetch or build | Keep the pins together: the layers were validated as a set. Keep the error output and `workspace/os-v1-resolved.yml`, and report the failure to meta-foundries. |
| OS update fails to finalize after reboot | Confirm meta-foundries `ee25a99` in the resolved configuration (`kas-container dump --resolve-refs kas/uno-q-wrynose.yml`). On the UNO Q, confirm `aktualizr-lite --version` prints `57e4f7f`. Then inspect the booted deployment and updater logs. The guide relies on the client's OS-name derivation with `pacman.os` unset. |
| You need a RAM dump after a kernel crash | Rebuild with the optional diagnostic overlay `kas/diag-download-mode.yml` appended to `GUIDE_KAS`. It sets `qcom_scm.download_mode=1`: a crash then leaves the board in Qualcomm RAM-dump mode until power-cycled, instead of resetting. meta-qcom's CI configuration sets it by default; the guide's configuration replaces that entry. Leave it out for normal use. |
| Synchronous external abort during deployment on an affected 4 GB board | Check the recorded firmware and [M-05 selection](build.md#select-the-boards-memory-configuration). Preserve logs and confirm the reservation in the booted image before retrying. |
| M-05 overlay reports a dangling append or failed patch | Stop and check the selected `linux-qcom_6.18` recipe and that the copied layer matches this guide revision. Do not disable the error or widen the append to other versions. |
| Registration rejects `--factory` | Use the documented `DEVICE_FACTORY=unoq-lab` environment assignment; this build has no `--factory` option. |
| `pytactl list` finds no Bughopper | Check its USB data cable and connection; confirm the installed `pytactl` version and inspect the host's USB enumeration. |
| `pytactl` reports access denied or a missing native library | Complete [Bughopper tool installation](flash.md#install-bughopper-tools), including native libraries and device permissions; reconnect Bughopper after installing the rules. |
| `qdl` waits for a device | Complete [Emergency Download (EDL) entry](flash.md#enter-edl-mode) using Bughopper or a jumper, and allow up to 15 seconds for the board to appear. Verify `05c6:9008`, the UNO Q USB-C data cable, and host USB permissions. |
| Nothing appears on the computer when the board is connected, not even in `journalctl -k` | The USB-C cable carries power only. Use a data cable; without one the computer logs no USB event in any mode. Connecting the board directly to the computer, rather than through a dock, also rules out the dock. |
| `qdl` stops with `ERR: failed to setup programming` | Add `--storage emmc`. `qdl` defaults to UFS storage, and the UNO Q uses eMMC. |
| `qdl --version` prints an older version | Another `qdl` comes first on `PATH`. Repeat the [`qdl` installation](flash.md#prepare-the-equipment-and-flashing-tools) `export PATH` line in this shell. |
| Board stays in EDL after flashing with Bughopper | After `qdl` succeeds, run `pytactl oneshot reset --serial "$BUGHOPPER_SERIAL"` on the host to clear EDL and boot normally. |
| No Android Debug Bridge (ADB) device after boot | Check that the flashed image contains the enabled ADB configuration; verify USB permissions and normal boot rather than EDL. On the board, `journalctl -u android-tools-adbd` reporting `Module libcomposite not found` means the image lacks the USB gadget modules; rebuild with the guide's `kas/uno-q-wrynose.yml`. |
| Serial output is unreadable or input fails | Verify the Bughopper port, 115200/8N1 settings, and disabled hardware/software flow control. |
| A USB-to-serial cable on `JCTL` shows only zero bytes or nothing | The `JCTL` console uses 1.8 V logic, which a 3.3 V adapter such as a PL2303 can't read. Use a Bughopper or a 1.8 V adapter, or the [HDMI console](flash.md#with-an-hdmi-display-and-a-usb-keyboard). |
| No ADB device with an image built from meta-foundries' `ci/uno-q.yml`, including its CI images | Those images don't enable ADB; only the guide's `kas/uno-q-wrynose.yml` adds it. Use a Bughopper or the HDMI console. |
| A USB-C hub's Ethernet port shows no network interface on the board | Most hubs use a Realtek RTL8153 chip, and the image doesn't include its `r8152` driver. Connect through Wi-Fi. |
| Commands ran on the wrong board | With one hub or console moved between boards, check which board it's connected to. Every board boots as `uno-q`; set a distinct hostname on each with `hostnamectl set-hostname`. |
| The serial console stops responding to commands after `nmcli` | `nmcli` opened a pager. Press `q`, then repeat the command with `\| cat`. |
| `getent: command not found` on the board | The board's image has no `getent`. Use the `grep` and `ping` checks from [the name check](server-registration.md#give-the-server-a-name-reachable-from-the-board). |
| `curl: command not found` on the board | The board's image has no `curl`. Run the HTTP checks from the computer with the board's IP address. |
| Enrollment page opens but registration fails | Read the board's command result; check clock, server-terminal logs, and duplicate device name. |
| `fio-device-register` fails after waiting for authorization | The code expired: codes are valid for 10 minutes. Run the command again and approve the new code promptly. |
| The authorization page asks for a code that was no longer shown after signing in | Open `http://unoq-update.test:8080/auth/activate?user_code=CODE` with the code that `fio-device-register` or `fiocli login` printed. |
| Board cannot reach the server | Check the local-network IP address, hosts entry, listening ports, host firewall, and Wi-Fi client isolation. |
| TLS hostname error | Verify that `devserver-init --dnsname` matches the hostname in device configuration and hosts entries. |
| CLI works, but no device heartbeat | Check Transmission Control Protocol (TCP) port 8443 and certificate hostname resolution on the board; CLI access only establishes the 8080 path. |
| `fiocli` reports status 429, `rate-limit exceeded` | update-server release v1.0-rc1 allows about two requests per second. Use the server and CLI of [release v1.0-rc3](server-registration.md#download-and-initialize-the-server), and check that `fioserver --version` prints `v1.0-rc3`. With v1.0-rc1, wait 30 seconds, then check with `fiocli configs show` or `fiocli updates list`: the change may already be stored. |
| `sha256sum -c` reports `FAILED` for an update-server download | The file differs from the v1.0-rc3 release asset recorded in this guide, for example after an interrupted download. Delete it and download it again; do not run it. |
| An ARM64 build or `docker run --platform linux/arm64` fails with `exec format error` | Register ARM64 emulation with `docker run --privileged --rm tonistiigi/binfmt --install arm64`; it lasts until the computer reboots. |
| App packaging reports unauthorized | Check host Docker login, credential helpers, `DOCKER_CONFIG`, and package permissions. |
| `composectl publish` reports `app bundle size exceeds the maximum allowed` | `composectl publish` packs the current directory. Run it from the app's directory, for example `cd "$GUIDE_DIR/workspace/apps/v1"`. |
| `composectl pull` reports `server gave HTTP response to HTTPS client` | The registry serves plain HTTP. `composectl pull` needs HTTPS with a certificate the computer trusts; use a hosted registry. |
| App downloads fail after upload | Confirm the upload contained the entire `apps/` tree from `composectl pull --arch arm64`; a digest alone supplies no blobs. |
| An application-only rollout attempts an OS download or reboot | Confirm `--ostree-hash` exactly matches the board's booted OS checksum; an omitted hash does not mean retain the current OS. |
| An existing app disappears after adding another | Include both app digests in the target and both names in the device's explicit selection. |
| `fiocli updates tail` shows no events after a rollout | The board checks for updates every five minutes. Wait, then confirm the rollout with `fiocli updates list` (`DEVICE COUNT`) and the board's `journalctl -u aktualizr-lite`. |
| Uploaded update never installs | Check that a rollout targets the correct universally unique identifier (UUID), the tag is `wrynose`, the hardware ID is `uno-q`, and the version increases. |
| App exists but does not run | Check `--apps shellhttpd`, successful fioconfig delivery, updater logs, ARM64 packaging, and available board disk space. |
| `curl` to the board fails after the OS update reboot | The board's Wi-Fi address may have changed across the reboot. Run `ip -4 address` on the board and update `UNOQ_DEVICE_IP`. |
| Browser shows the server UI instead of shellhttpd | Use the UNO Q's IP for the app; `unoq-update.test:8080` is the server UI. |

Record the resolved configuration (`workspace/os-v1-resolved.yml`), host OS, board model, command output, and service logs when reporting a problem.
Exclude registry tokens, device private keys, and server private keys from shared logs.
