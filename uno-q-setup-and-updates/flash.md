---
title: Flash UNO Q and Connect to the Board
description: Flash the Qualcomm Linux image, access UNO Q with or without a Bughopper, and connect to the network.
type: page
doc-category: instructional
authors: David Griego, Caio Pereira, Codex:GPT-6, Andre Detsch, Claude:Opus-5.5
last-edited: 2026-11-09
license: MIT
access: public
references:
  - https://github.com/arduino/docs-content/tree/dab66ecbd6ad52cd742da6c19c5b6330a7f2caae
  - https://github.com/qualcomm/pytactl/tree/6ee43578f983487821ade4feebebad94198da622
  - https://github.com/foundriesio/meta-foundries/commit/3ed349554814f5596891ac155d58a55f9a636df7
  - https://github.com/qualcomm-linux/meta-qcom-arduino/blob/8457c3dbb5dc650c7b0ac45c99ccb004a05f9d4f/conf/machine/uno-q.conf
  - https://github.com/linux-msm/qdl/tree/v2.8
relations:
  prev: build.md
  next: server-registration.md
  parent: README.md
---

# Flash UNO Q and Connect to the Board

[Guide overview](README.md) · [Previous: Build the image](build.md) · **Part 2 of 4**

Complete [the image build](build.md) first.
On your local x86-64 Linux® computer, open a Bash shell in the guide directory and set `GUIDE_DIR` to its absolute path.
Confirm that `workspace/unoq-os1-flash.tar.gz` and `workspace/releases/1/ostree_repo` are available there.

Choose Bughopper serial access, Android Debug Bridge (ADB), or an HDMI display and keyboard, then connect Arduino® UNO Q to your network.
The board will run Qualcomm® Linux® (QLI) after flashing.

> **Validation status:** The Bughopper flashing and serial-console steps completed on an UNO Q on October 11 and 13, 2026, with the previous revision's edk2 UEFI boot chain.
> That image was flashed and booted on October 13, and ADB listed the board and opened a root shell.
> On October 28, this revision's image, with the stock U-Boot boot chain, was flashed and booted through the Bughopper path, and the M-05 check passed.
> ADB has not been run with this revision's kernel, and the jumper path has not been run.
> See [sources and validation](validation.md).

- [Prepare the equipment and flashing tools](#prepare-the-equipment-and-flashing-tools)
- [Flash the UNO Q and open a shell](#flash-the-uno-q-and-open-a-shell)

## Prepare the Equipment and Flashing Tools

You need an UNO Q, a USB-C data cable, and a network the board can join.
Some USB-C cables carry only power.
With such a cable the board powers up, but the computer logs no USB event at all, in EDL mode or after boot.
If `journalctl -k` on the computer shows no new USB device when you connect the board, try another cable before anything else.
For Bughopper boot control and serial access, add a Bughopper and a second USB data cable.
Without a Bughopper, keep a jumper cap or female-to-female jumper available for Emergency Download (EDL) mode.

Install [`qdl` v2.8](https://github.com/linux-msm/qdl/releases/tag/v2.8) on the **local computer**.
For Ubuntu 24.04 on x86-64:

```bash
mkdir -p "$HOME/qdl-v2.8"
cd "$HOME/qdl-v2.8"
curl -fLO https://github.com/linux-msm/qdl/releases/download/v2.8/qdl-binary-ubuntu-24-x64-v2.8.zip
unzip -o qdl-binary-ubuntu-24-x64-v2.8.zip
chmod +x qdl-binary-ubuntu-24-x64/qdl
export PATH="$HOME/qdl-v2.8/qdl-binary-ubuntu-24-x64:$PATH"
qdl --version
```

On Ubuntu 22.04, replace `ubuntu-24` with `ubuntu-22` in the archive and directory names.
Install `unzip` first if it is missing (`sudo apt install unzip`).
The last command must print `qdl version v2.8`; an older `qdl` elsewhere on `PATH` prints a different version.
The archive keeps `qdl` with the libraries it needs, so run it from that directory through `PATH`.
The `PATH` setting lasts for this shell; repeat the `export` line in any new shell used for flashing.

Install the console tool for your chosen connection:

- **With a Bughopper:** Install the boot-control and serial tools described below. ADB is optional for this path.
- **Without a Bughopper:** Install ADB. On Ubuntu, run `sudo apt install adb usbutils`.

Both paths use `qdl` and the UNO Q's own USB-C port for flashing.
Bughopper controls EDL entry and reset; `qdl` transfers the image.

### Install Bughopper Tools

For the Bughopper path, install [`pytactl` 2.0.0](https://github.com/qualcomm/pytactl/tree/6ee43578f983487821ade4feebebad94198da622).
It requires Python 3.10 or newer and supports Bughopper V1 and V2 without extra configuration files.
Run these commands on the **local Linux host**. On Ubuntu:

```bash
sudo apt update
sudo apt install pipx picocom usbutils libusb-1.0-0 libhidapi-hidraw0 libhidapi-libusb0
pipx install 'pytactl==2.0.0'
export PATH="$HOME/.local/bin:$PATH"
pytactl --help
```

The native libraries provide USB and Human Interface Device (HID) access.
Install these Bughopper USB permissions rules before connecting the hardware:

```bash
sudo tee /etc/udev/rules.d/60-bughopper.rules > /dev/null <<'EOF'
SUBSYSTEM=="usb", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6015", MODE="0660", GROUP="plugdev", TAG+="uaccess"
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="2341", ATTRS{idProduct}=="b001", MODE="0660", GROUP="plugdev", TAG+="uaccess"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger
```

These rules cover Bughopper V1 USB control and V2 HID control, using the
[upstream device permissions](https://github.com/qualcomm/pytactl/blob/4c47a6172e3ee6fabc16fb8b7042004ec9cd140f/60-pytactl.rules).
They grant access to the active local session and the `plugdev` group.
For a remote host session, ask your administrator to provide the required device access.
If the Bughopper was already connected, reconnect it after installing the rules.
Serial-console permissions are described below.

## Flash the UNO Q and Open a Shell

> **Flashing replaces the board's Linux installation and stored data.** Save anything you need before proceeding.
> This installs the QLI image from the build page; Arduino App Lab workflows are outside this guide.

### Choose Your Console Connection

```mermaid
flowchart TD
    H{"Have a Bughopper?"}
    H -->|Yes| S["Fit Bughopper to JCTL<br/>Connect both USB cables<br/>Enter EDL with pytactl"]
    H -->|No| A["Short EDL pins<br/>Connect UNO Q USB-C<br/>Remove short after EDL entry"]
    S --> E["Verify EDL: 05c6:9008"]
    A --> E
    E --> F["qdl writes image over UNO Q USB-C"]
    F --> R{"Bughopper attached?"}
    R -->|Yes| B["Reset with pytactl<br/>Open serial console"]
    R -->|No| P["Power cycle with jumper removed<br/>Open adb shell after boot"]
    B --> N["Connect Wi-Fi and verify networking"]
    P --> N
```

#### With a Bughopper

With power disconnected, align the Bughopper with the UNO Q's `JCTL` header as shown in
[Arduino's Bughopper manual](https://docs.arduino.cc/tutorials/bughopper/user-manual/).
Connect its USB port to the computer, and connect the UNO Q's own USB-C port separately for flashing and power.

![Official Arduino illustration showing a Bughopper fitted to the UNO Q JCTL connector.](https://raw.githubusercontent.com/arduino/docs-content/dab66ecbd6ad52cd742da6c19c5b6330a7f2caae/content/hardware/accessories/accessories/bughopper/tutorials/01.user-manual/assets/connection-1.png)

*Hardware illustration: Arduino documentation. Use the linked manual for connector orientation.*

Open the Bughopper serial port at **115200 baud, 8 data bits, no parity, 1 stop bit, no flow control**.

Identify the Bughopper port under `/dev/serial/by-id/` and open it with
`picocom -b 115200 /dev/serial/by-id/<your-port>`.
Your account may need membership in the serial device's group, normally `dialout`.
Check with `ls -lL /dev/serial/by-id/<your-port>` and `id -nG`, add the group if needed, then log out and back in:

```bash
sudo usermod -aG dialout "$USER"
```

After flashing, this console displays boot output and provides a Linux shell.
The supplied development configuration enables root serial autologin.

The `JCTL` console signals use 1.8 V logic.
A generic 3.3 V USB-to-serial adapter, such as a PL2303 cable, can't read them: the terminal shows only zero bytes or nothing.
Use a Bughopper or a 1.8 V adapter; don't connect an adapter's transmit line to the board without level shifting.

#### Without a Bughopper

The guide's build configuration enables `image-adbd` and `enable-adbd` and installs the `libcomposite` and `usb_f_fs` kernel modules that the ADB USB gadget uses.
Keep the UNO Q's USB-C data cable connected after flashing.
After QLI boots, use ADB to open a shell as described below.
ADB becomes available after Linux starts; a Bughopper can also show earlier boot output.

ADB comes from the guide's `kas/uno-q-wrynose.yml`, not from meta-foundries itself.
An image built from meta-foundries' `ci/uno-q.yml` alone, such as an image from meta-foundries' continuous integration, has no ADB.
For such an image, use a Bughopper or the HDMI console below.

#### With an HDMI Display and a USB Keyboard

The UNO Q's USB-C port can also drive a USB-C hub with HDMI output.
After flashing, connect a hub with an HDMI display and a USB keyboard to the UNO Q's USB-C port, and power the hub through its USB-C Power Delivery (PD) input.
The board shows a login prompt on the display after boot; log in as `root`, which has no password in this development image.
This console doesn't show early boot output.

Many hubs include an Ethernet port with a Realtek RTL8153 chip.
The image doesn't include its `r8152` driver, so that port doesn't work; connect the board through Wi-Fi as described below.

If you set up more than one board with a single hub, check which board the hub is connected to before typing commands.
Every board boots with the hostname `uno-q`, so the prompt doesn't tell them apart.

### Enter EDL Mode

Choose the procedure for your connection, then verify EDL enumeration before flashing.

#### Enter EDL With Bughopper

Keep the Bughopper attached to `JCTL`, with its USB cable connected to the host.
Keep the UNO Q powered through its own USB-C connection to the same host.
Run the following in a **host terminal**, separate from the board's serial console:

```bash
pytactl list
```

Find the Bughopper entry and copy its USB serial number from `serial=...`.
Set `BUGHOPPER_SERIAL` to that value and retain it for the reset step:

```bash
export BUGHOPPER_SERIAL='PASTE_BUGHOPPER_SERIAL'
pytactl oneshot bootToEDL --serial "$BUGHOPPER_SERIAL"
```

`--serial` identifies the Bughopper USB device; it is separate from the UNO Q's ADB identifier and serial-port path.
The command cycles board power and asserts EDL through Bughopper.
Keep both USB cables connected throughout EDL entry and flashing.
The board can take up to 15 seconds to appear in EDL mode; continue with the check below.

#### Enter EDL With a Jumper

Use this procedure when flashing without a Bughopper.

Follow Arduino's [UNO Q EDL procedure](https://docs.arduino.cc/software/app-lab/configure/flash/#step-1-set-your-board-to-edl-mode).
Disconnect board power and short the designated EDL pins.
Then connect the UNO Q USB-C port to your computer.
Remove the short once the board has entered EDL.

![Official Arduino illustration identifying the UNO Q EDL pins to short before connecting USB power.](https://raw.githubusercontent.com/arduino/docs-content/dab66ecbd6ad52cd742da6c19c5b6330a7f2caae/content/software/app-lab/assets/flashing/flashing-pins.png)

*EDL illustration: Arduino documentation. Follow the marked pins rather than shorting an unidentified header pair.*

#### Verify EDL Enumeration

For either procedure, check for the EDL USB device on the host:

```bash
lsusb -d 05c6:9008
```

The command prints one line for the board once it is in EDL mode.
If it prints nothing, wait five seconds and run it again, for up to 30 seconds.
If the device still does not appear, repeat the EDL entry for your connection.

`qdl` must have USB access; consult its [host instructions](https://github.com/linux-msm/qdl/blob/v2.8/README.md)
if it cannot open the device.

Identify the UNO Q's EDL serial number so that `qdl` writes to this board only:

```bash
qdl list
```

The UNO Q's line begins `05c6:9008`, followed by a tab and its serial number.
Lines for other kinds of devices begin with a different ID; ignore them.
Without a serial, `qdl` writes to the first EDL device it finds, which may be another connected board.
If more than one `05c6:9008` line appears, disconnect the other EDL devices, or run `qdl list` before and after entering EDL
and take the serial that appeared with the UNO Q.
**Stop** if the UNO Q's serial is blank or missing, or if you cannot tell which line is the UNO Q's.
Record the serial for the flash command:

```bash
export UNOQ_EDL_SERIAL='PASTE_UNOQ_EDL_SERIAL'
```

### Write the Image

On the **local computer**, extract the flash archive:

```bash
mkdir -p "$GUIDE_DIR/workspace/flash-os1"
tar -xzf "$GUIDE_DIR/workspace/unoq-os1-flash.tar.gz" \
  -C "$GUIDE_DIR/workspace/flash-os1"
```

```bash
cd "$GUIDE_DIR/workspace/flash-os1/core-image-full-cmdline-uno-q.rootfs.qcomflash"
qdl --storage emmc --serial "$UNOQ_EDL_SERIAL" prog_firehose_ddr.elf rawprogram0.xml patch0.xml
```

`--storage emmc` is required: `qdl` defaults to UFS storage, and without the option it stops with `ERR: failed to setup programming`.
A successful run ends with a summary line such as `flashed 38 partitions (1.9 GiB written in 1 min 1 s, 31.7 MiB/s), applied 13 patches`.
If flashing fails, repeat the command with `--debug` added; it prints every firehose exchange, so keep that output for a bug report rather than reading it in full.

Use the [Arduino USB permissions setup](https://docs.arduino.cc/software/app-lab/setup/linux/)
if `qdl` reports insufficient permissions.

Wait for `qdl` to complete successfully before resetting or disconnecting the board.

### Return to Normal Boot

**With Bughopper**, run this in the host terminal where you set `BUGHOPPER_SERIAL`:

```bash
pytactl oneshot reset --serial "$BUGHOPPER_SERIAL"
```

The reset command clears the EDL assertion and cycles board power for normal boot.
Keep both USB cables connected and observe the serial console.

**With a jumper**, disconnect board power, remove the EDL short if it is still connected,
and reconnect the board for normal boot.

### Open the QLI Shell and Connect Wi-Fi

With a Bughopper, use the serial shell.
With an HDMI display and keyboard, log in as `root` on the display.
Otherwise, run these commands on the host after the board finishes booting:

```bash
adb devices
adb root
adb shell
```

If more than one ADB device appears, add `-s <serial>` to each command.

Run the following **on the UNO Q**, as root:

```bash
cat /etc/os-release
command -v fio-device-register
command -v aktualizr-lite
docker version
nmcli device wifi rescan
nmcli device wifi list | cat
nmcli --ask device wifi connect "YOUR_WIFI_SSID"
ip -4 address
ip route
date -u
```

The `| cat` stops `nmcli` from opening a pager on the serial console.
If a pager opens anyway, press `q` before typing the next command.
Enter the Wi-Fi password when prompted.
Confirm that `/etc/os-release` identifies Qualcomm Linux and `BUILD_ID="unoq-wrynose-os-1"`.

Find the board's local-network IPv4 address in the `ip -4 address` output, on the Wi-Fi interface.
On the **local computer**, record it for the later steps:

```bash
export UNOQ_DEVICE_IP='PASTE_THE_BOARD_IP'
```

The board's clock can take tens of seconds to synchronize after Wi-Fi connects.
Run `date -u` on the local computer too, and repeat `date -u` on the board until the two agree to within a few seconds.
Enroll the board only after its clock is correct.

If you set up more than one board, give each one its own hostname so that prompts and logs tell them apart:

```bash
hostnamectl set-hostname unoq-01
```

Use `unoq-02` for the second board, and so on.
The hostname is stored in `/etc`, which OS updates preserve.

### Verify the Selected Memory Workaround

If you selected the [M-05 overlay](build.md#select-the-boards-memory-configuration) for an affected 4 GB board,
check the **booted device tree on the UNO Q** before enrollment or any update:

```bash
M05_NODE=/sys/firmware/devicetree/base/reserved-memory/m05-boundary-guard@7b8ff000
od -An -tx1 "$M05_NODE/reg"
test -e "$M05_NODE/no-map" && printf 'no-map present\n'
```

The `reg` output must contain these 16 bytes, in order:
`00 00 00 00 7b 8f f0 00 00 00 00 00 01 10 00 00`.
They encode the start address `0x7b8ff000` and size `0x1100000` in big-endian cells.
The second command must print `no-map present`.
If either property is missing or differs, stop and resolve the build or flashed-image mismatch before continuing.
Save the output in `workspace/board-record.txt`, with the board's RAM size, firmware revision, and resolved build configuration.
Repeat this check after the OS version 2 reboot if the workaround is selected.
The live properties are one checkpoint; successful memory and update testing is still required.

**Next:** [Set up the server and register UNO Q](server-registration.md).
Keep the guide directory, OS repository, build records, and Linux build environment for the later update exercises.
