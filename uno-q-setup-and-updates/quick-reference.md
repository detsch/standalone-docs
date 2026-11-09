---
title: UNO Q Walkthrough Quick Reference
description: The walkthrough's commands in order, with the result each must produce, for readers and automation agents that already know the setup.
type: page
doc-category: reference
authors: Andre Detsch, Claude:Opus-5.5
last-edited: 2026-11-09
license: MIT
access: public
relations:
  parent: README.md
---

# UNO Q Walkthrough Quick Reference

[Guide overview](README.md)

This page lists the walkthrough's main commands in order, with the result each one must produce.
It's meant for readers repeating the setup and for automation agents, which can work from it without reading the full guide.
Each step links to the page that explains it; read that page when a result differs.

Run commands marked **host** on the Linux computer and commands marked **board** as root on the UNO Q.

## Flash

| Where | Command | Expected result |
| --- | --- | --- |
| host | Enter EDL: short the EDL pins, or run `pytactl oneshot bootToEDL --serial "$BUGHOPPER_SERIAL"`, then connect USB-C | — |
| host | `lsusb -d 05c6:9008` | One line. Nothing in `journalctl -k` means a power-only cable. |
| host | `qdl list` | `05c6:9008`, a tab and the serial for `UNOQ_EDL_SERIAL` |
| host | `qdl --storage emmc --serial "$UNOQ_EDL_SERIAL" prog_firehose_ddr.elf rawprogram0.xml patch0.xml`, in the extracted `.qcomflash` directory | Last line `flashed N partitions (...), applied N patches` |
| host | Remove the EDL short and reconnect, or run `pytactl oneshot reset --serial "$BUGHOPPER_SERIAL"` | The board boots normally |

Details: [Flash UNO Q and connect to the board](flash.md).

## Connect the Board

Open a shell through a Bughopper, `adb shell` (guide image only), or an HDMI display and keyboard, then:

| Where | Command | Expected result |
| --- | --- | --- |
| board | `nmcli --ask device wifi connect "YOUR_WIFI_SSID"` | `successfully activated` |
| board | `ip -4 -br address show wlan0` | The board's IP, for `UNOQ_DEVICE_IP` |
| board | `hostnamectl set-hostname unoq-01` | Only with more than one board |
| board | `date -u` | Matches the host within a few seconds |
| board | `printf '%s unoq-update.test\n' "$UPDATE_SERVER_HOST_IP" >> /etc/hosts` | `ping -c 1 unoq-update.test` replies |

Once the board is on the network, the host can run board commands with `ssh root@"$UNOQ_DEVICE_IP"`; root has no password in this development image.

## Server and Registration

| Where | Command | Expected result |
| --- | --- | --- |
| host | `./bin/fioserver --datadir=./datadir devserver-init --dnsname=unoq-update.test` | Run once per lab |
| host | `./bin/fioserver --datadir=./datadir serve` | Leave running; UI on port 8080, devices on 8443 |
| board | `DEVICE_FACTORY=unoq-lab fio-device-register --device-api=http://unoq-update.test:8080/v1/devices --oauth-api=http://unoq-update.test:8080/oauth2 --name=unoq-01 --tag=wrynose` | Prints a code; approve it at `http://unoq-update.test:8080/auth/activate?user_code=CODE` within 10 minutes |
| board | `systemctl is-active aktualizr-lite fioconfig docker` | `active` three times |
| host | `fiocli login unoq-local http://unoq-update.test:8080` | Approve the printed code the same way |
| host | `fiocli devices list` | One UUID per registered board, for `UNOQ_UUID` |

Details: [Set up the server and register UNO Q](server-registration.md).

## Update

| Where | Command | Expected result |
| --- | --- | --- |
| host | `fiocli updates upload wrynose NAME DIR --version N --hardware-id uno-q --name TARGET` | `fiocli updates list` shows `NAME` |
| host | `fiocli updates create-rollout NAME ROLLOUT --uuids "$UNOQ_UUID"` | Prints nothing on success |
| board | `systemctl restart aktualizr-lite` | Optional: checks now instead of within five minutes |
| host | `fiocli updates show-rollout NAME ROLLOUT` | `Installation completed; succeeded: 1` after an OS update's reboot |
| board | `ostree admin status` | The new commit marked `*`, the previous one as `(rollback)` |

`--version` must increase with every update.
A rollout reaches only the UUIDs it names; roll out to more boards with another rollout of the same update.

Details: [Deploy and update applications](application-updates.md).
For failures, see [Troubleshooting](troubleshooting.md).
