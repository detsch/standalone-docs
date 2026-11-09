---
title: Set Up the Server and Register UNO Q
description: Run the local update server, register the device, and select its applications.
type: page
doc-category: instructional
authors: David Griego, Codex:GPT-6, Andre Detsch, Claude:Opus-5.5
last-edited: 2026-11-09
license: MIT
access: public
references:
  - https://github.com/foundriesio/update-server/releases/tag/v1.0-rc3
  - https://github.com/foundriesio/update-server/blob/e307a38b702da479b971b8286d534f0545956dec/docs/quick-start.md
  - https://github.com/foundriesio/update-server/blob/e307a38b702da479b971b8286d534f0545956dec/cmd/server/dev_server.go
  - https://github.com/foundriesio/update-server/blob/e307a38b702da479b971b8286d534f0545956dec/docs/fiocli.md
  - https://github.com/foundriesio/meta-foundries/blob/ee25a99b3f2ef66af36a51da583f8223bcaf58d1/recipes-sota/fio-device-register/fio-device-register_97.bb
  - https://github.com/foundriesio/lmp-device-register/tree/d8f39a6d95e20f5b64881ae014381682aa1c7aad
relations:
  prev: flash.md
  next: application-updates.md
  parent: README.md
---

# Set Up the Server and Register UNO Q

[Guide overview](README.md) · [Previous: Flash and connect](flash.md) · **Part 3 of 4**

Complete [flashing and board setup](flash.md) first.
The board must be running OS version 1, connected to your local area network (LAN), and accessible through your chosen console.

Use your local Linux® computer, with the complete guide directory and `GUIDE_DIR` set to its absolute path.
Keep this host Bash session open for the application exercises so `GUIDE_DIR` and `UNOQ_UUID` remain available.

> **Validation status:** These steps completed on an UNO Q on October 11 and 13, 2026, with update-server release v1.0-rc1 and then a source build of `e30762e`.
> On October 13, back-to-back CLI commands hit v1.0-rc1's request-rate limit (HTTP 429); `e30762e` raised that limit, and release v1.0-rc3 includes the change.
> On October 28, these steps completed on hardware with this revision's image and the v1.0-rc3 binaries, with no HTTP 429.
> See [sources and validation](validation.md).

- [Start the update server on your computer](#start-the-update-server-on-your-computer)
- [Register the UNO Q](#register-the-uno-q)
- [Select the applications the board should run](#select-the-applications-the-board-should-run)
- [Stop and restart the local server](#stop-and-restart-the-local-server)

## Start the Update Server on Your Computer

Run this section in a Bash shell on your **local Linux host** using your regular account.
It follows the upstream [development-server bootstrap](https://github.com/foundriesio/update-server/blob/e307a38b702da479b971b8286d534f0545956dec/docs/quick-start.md).

This is a local development server.
Its enrollment UI uses HTTP, and the UNO Q image enables development console access.
Use a trusted lab network and keep these ports off the public Internet.

### Give the Server a Name Reachable From the Board

Use the hostname **`unoq-update.test`** throughout this guide.
Run `ip -4 address` and find the computer's local area network (LAN) IPv4 address.
Reserve that address through your router's Dynamic Host Configuration Protocol (DHCP) settings if possible.

| Name used in this guide | Address it identifies |
| --- | --- |
| `UPDATE_SERVER_HOST_IP` | The computer running the Foundries Update Server |
| `UNOQ_DEVICE_IP` | The UNO Q board |

Enter the server computer's LAN address when prompted:

```bash
read -r -p "Enter the update-server computer's LAN IPv4 address: " UPDATE_SERVER_HOST_IP
printf '%s unoq-update.test\n' "$UPDATE_SERVER_HOST_IP"
```

Copy the **printed output** into `/etc/hosts` on both the computer and the UNO Q.
On the computer, edit the file with administrator privileges.
On the **UNO Q**, edit it as root:

```bash
vi /etc/hosts
```

Check resolution on the **computer**:

```bash
getent hosts unoq-update.test
```

The board's image has no `getent`, so check it on the **UNO Q** with:

```bash
grep unoq-update.test /etc/hosts
ping -c 1 unoq-update.test
```

All three commands must show the computer's LAN address, and `ping` must receive a reply.
If that address changes, update both hosts files; keeping the hostname preserves the certificate name.

Allow inbound Transmission Control Protocol (TCP) connections on ports **8080 and 8443** through the computer's firewall.
Ubuntu's firewall tool is `ufw`; some distributions use `firewalld`. Check which one is active:

```bash
sudo ufw status
sudo firewall-cmd --state
```

If `ufw` reports `Status: active`, run `sudo ufw allow 8080/tcp` and `sudo ufw allow 8443/tcp`.
If `firewalld` reports `running`, run `sudo firewall-cmd --add-port=8080/tcp --add-port=8443/tcp`.
If neither tool is installed or active, the computer has no firewall from either tool to change.
If your organization manages the computer's firewall, ask its administrator to open the two ports.

### Download and Initialize the Server

This walkthrough uses the server and CLI of update-server release
[v1.0-rc3](https://github.com/foundriesio/update-server/releases/tag/v1.0-rc3), built from commit
[`e307a38`](https://github.com/foundriesio/update-server/tree/e307a38b702da479b971b8286d534f0545956dec).
It includes the higher request-rate limit that the CLI commands below need; release v1.0-rc1 rejects some of them with HTTP 429.
The release publishes no checksum file, so the commands check each download against the SHA-256 checksum recorded for this guide.

Download both tools on the **local computer**:

```bash
mkdir -p "$GUIDE_DIR/workspace/bin"
cd "$GUIDE_DIR/workspace/bin"
curl -fLO https://github.com/foundriesio/update-server/releases/download/v1.0-rc3/fioserver-linux-amd64
curl -fLO https://github.com/foundriesio/update-server/releases/download/v1.0-rc3/fiocli-linux-amd64
sha256sum -c <<'EOF'
5dac161d79f679c15dcfe1ba0aff275e979893fb683a30f4777acf664439aa27  fioserver-linux-amd64
3e87a42d862433d7d12258c2c3b9d16ad9f55ea7368421b476a485c2f78dca62  fiocli-linux-amd64
EOF
```

`sha256sum` must print `fioserver-linux-amd64: OK` and `fiocli-linux-amd64: OK`.
If either line reports `FAILED`, delete that file and download it again; do not run a file that fails the check.
Then install the tools under the names the guide uses:

```bash
mv fioserver-linux-amd64 fioserver
mv fiocli-linux-amd64 fiocli
chmod +x fioserver fiocli
export PATH="$GUIDE_DIR/workspace/bin:$PATH"
fioserver --version
fiocli version
```

Both version commands must print `v1.0-rc3`.
The server needs GNU C Library 2.34 or newer, which Ubuntu 22.04 and 24.04 provide.
Keep both tools at this release for the walkthrough.

Initialize the server's data directory:

```bash
cd "$GUIDE_DIR/workspace"
./bin/fioserver --datadir=./datadir devserver-init --dnsname=unoq-update.test
```

Initialize the data directory **once**.
The bootstrap creates the public key infrastructure (PKI), The Update Framework (TUF) metadata,
and a local `admin` account with development password `admin`.
The explicit hostname sets the gateway certificate name used by the board.

Open a **second host terminal for the server**.
Set `GUIDE_DIR` to the same guide directory, then run:

```bash
cd "$GUIDE_DIR/workspace"
./bin/fioserver --datadir=./datadir serve
```

Leave this terminal open; the server runs in the foreground and prints its logs here.
Its data, keys, enrollment records, and updates persist in `workspace/datadir/`.
Open [the local server](http://unoq-update.test:8080/) and sign in as `admin` / `admin`.

Use a browser on the computer running the server, where the hosts entry resolves `unoq-update.test`.
A browser on another computer needs the same hosts entry.
Without that entry, use the server computer's IP address in place of `unoq-update.test`.
This applies to the server address and to the authorization links printed later; for example, `http://UPDATE_SERVER_HOST_IP:8080/`.

Use the **first host terminal** for CLI and application commands throughout the guide.
Keep `GUIDE_DIR` and the tools directory on `PATH` in that terminal.
When opening another host shell for these commands, set `GUIDE_DIR` again and run
`export PATH="$GUIDE_DIR/workspace/bin:$PATH"`.
Keep the board's serial or Android Debug Bridge (ADB) shell separate from both host terminals.

## Register the UNO Q

On the **UNO Q**, stop the updater before enrollment, then run:

```bash
systemctl stop aktualizr-lite
DEVICE_FACTORY=unoq-lab fio-device-register \
  --device-api=http://unoq-update.test:8080/v1/devices \
  --oauth-api=http://unoq-update.test:8080/oauth2 \
  --name=unoq-01 \
  --tag=wrynose
```

Open the authorization link printed by the command in your computer's browser.
Sign in to the local update server, enter the displayed code if requested, and approve this device.
Approve it promptly: the code expires after 10 minutes, and `fio-device-register` then fails and must be run again for a new code.
After you sign in, the page may no longer show the code; open `http://unoq-update.test:8080/auth/activate?user_code=CODE` with the printed code to fill it in.
Wait for `fio-device-register` to report success on the board.
Its `Device Name:` line may show the device's identifier in place of its name; the server still lists the device as `unoq-01`.
The `wrynose` tag must match the updates you upload later.

At this meta-foundries pin, `fio-device-register` is built with `REQUIRE_FACTORY` disabled.
Its supported flags include `--device-api`, `--oauth-api`, `--name`, and `--tag`; it has no `--factory` option.
The environment assignment sets the client-side factory label to `unoq-lab` for this command.
That label enters the authorization scope and certificate organization unit; it does not create a separate factory namespace in the local server.
Without the assignment, this build defaults to `fio-device-register`, regardless of an `LMP_FACTORY` image setting.
The image sets `LMP_FACTORY_TAG="wrynose"`; the explicit `--tag=wrynose` selects the same update tag during enrollment.

Check the services on the **UNO Q**:

```bash
systemctl start aktualizr-lite fioconfig
systemctl --no-pager status aktualizr-lite fioconfig docker
journalctl -u aktualizr-lite -n 50 --no-pager
```

All three services must be `active (running)`.
Until you upload and roll out the first update, the updater log reports `No Targets are available for the device`; that is expected.
The server terminal may log `Not found TUF role` for `/repo/2.root.json`.
That is also expected: the board checks for a newer version of the server's root metadata, and none exists yet.

The image's registration tool requests Compose application support.
Confirm `/var/sota/sota.toml` contains `type = "ostree+compose_apps"` in its `[pacman]` section.
Do not print or share the private key from this directory.

Print only the package-manager type, tags, and OS name:

```bash
awk '/^\[/{pacman=($0 == "[pacman]")} pacman && /^(type|tags|os) = /{print}' /var/sota/sota.toml
```

Expect `type = "ostree+compose_apps"` and `tags = "wrynose"`, in either order, and **no** `os = ` line.
Leave `pacman.os` unset so the pinned client's OS-name derivation is exercised.

In the server UI, open **Devices** and find `unoq-01`.
Confirm a recent device heartbeat, then copy its universally unique identifier (**UUID**).
A completed browser authorization alone is insufficient if the board command failed.

Return to the terminal on the **computer running the update server**.
Use the first host terminal, where the tools directory is on `PATH`, and log the CLI into the server:

```bash
fiocli login unoq-local http://unoq-update.test:8080
```

Complete the browser authorization printed by `fiocli`.
Its login state persists in `$HOME/.config/fiocli.yaml`.
The login selects `unoq-local` as the active CLI context for the following commands.

```bash
fiocli devices list
export UNOQ_UUID='PASTE_THE_DEVICE_UUID'
```

## Select the Applications the Board Should Run

In the same **update-server computer terminal**, select `shellhttpd` for this device and keep its update tag `wrynose`:

```bash
fiocli configs updates --device "$UNOQ_UUID" \
  --tag wrynose --apps shellhttpd
```

The command prints nothing when it succeeds.
Check the stored selection:

```bash
fiocli configs show --device "$UNOQ_UUID"
```

The listed file must mention `shellhttpd` and `wrynose`.
After the board's next configuration poll, `fiocli configs applied "$UNOQ_UUID"` shows the configuration it applied.

This records the desired application selection.
The application starts after an update containing `shellhttpd` is uploaded and rolled out.
The first selection uses the CLI because a newly enrolled device may not yet have an app catalog in the UI.

For later changes, use these alternatives:

| Desired result | Value for `--apps` |
| --- | --- |
| Run shellhttpd | `shellhttpd` |
| Run two apps present in the update | `shellhttpd,another-app` |
| Run no apps | `,` |
| Remove this override and use the device's configured defaults | `-` |

For example, `fiocli configs updates --device "$UNOQ_UUID" --apps ','` disables all applications.
Changing the selection does not create application content; selected names must exist in the update.
Allow the configuration client and updater time to poll the server.

## Register More Boards

To use more than one UNO Q with the same server, repeat [registration](#register-the-uno-q) on each board with its own name, for example `--name=unoq-02`.
Each board needs the same `/etc/hosts` entry for `unoq-update.test`.
Record each board's UUID from `fiocli devices list`, for example as `UNOQ_UUID_2`.
`fiocli devices list` shows UUIDs, not names; match each UUID to a board with the **Devices** page of the server UI.
Then select applications for each UUID with `fiocli configs updates`, as for the first board.

A rollout reaches only the UUIDs it names.
To update boards in stages, create one rollout per stage, as described in [the version 2 update](application-updates.md#upload-and-roll-out-version-2).

## Stop and Restart the Local Server

In the **server terminal**, press Ctrl+C to stop the foreground process.
To restart it with the saved data, run:

```bash
cd "$GUIDE_DIR/workspace"
./bin/fioserver --datadir=./datadir serve
```

Keep `workspace/datadir/` to retain the server's keys, enrollment records, and updates.
Run `serve` against that same directory each time; initialize a new directory only when starting a new lab.

**Next:** [Deploy and update applications](application-updates.md).
Leave the server running in its terminal and continue in the first host Bash session.
