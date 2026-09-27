# Klipper Fresh Install and Recovery Guide

This guide documents the working fresh-install process used to rebuild the Klipper host for this printer on a Raspberry Pi 3. It covers the baseline operating system installation, staged Wi-Fi validation, KIAUH installation, Git-backed configuration restoration, Beacon support, Raspberry Pi UART setup for the SKR Mini E3 V3, and the Klipper Linux host MCU.

> **Scope:** This guide is specific to the current printer architecture:
>
> - Raspberry Pi 3 Model B
> - Raspberry Pi OS Lite, 32-bit
> - SKR Mini E3 V3 mainboard connected to the Pi by GPIO UART
> - Beacon RevH connected by USB
> - Mini12864/menu MCU using a Klipper STM32F042 USB device
> - Raspberry Pi configured as a Klipper Linux host MCU
> - Klipper, Moonraker, and Mainsail installed with KIAUH
> - Printer configuration stored in a GitHub repository

---

## 1. Preserve the Existing Installation

Use a new microSD card for the rebuild. Do not overwrite the old working card.

The old card provides a complete rollback if the new installation is incomplete or a printer-specific dependency is missed.

Recommended labeling:

- **Old card:** Bullseye installation, known printer configuration, historical Wi-Fi instability
- **New card:** Fresh current Raspberry Pi OS installation

---

## 2. Flash the Baseline Operating System

Use Raspberry Pi Imager on a PC.

### Image selection

- Device: **Raspberry Pi 3**
- OS: **Raspberry Pi OS Lite (32-bit)**
- Storage: New microSD card

### Imager customization

Configure the following before writing the card:

- Hostname: `pi`
- Username: `alwelch`
- Wi-Fi SSID: `MyRouter`
- Wi-Fi country: `US`
- SSH: Enabled
- Password or SSH authentication: Configure as preferred

Do not copy old files from these locations onto the new installation:

```text
/etc/NetworkManager/
/etc/wpa_supplicant/
/etc/network/
/etc/systemd/
```

The purpose of the fresh install is to start with a clean kernel, Wi-Fi stack, NetworkManager configuration, and operating-system service configuration.

---

## 3. First Boot and Bare-OS Wi-Fi Baseline

Install the new microSD card in the Pi and boot it. Find the Pi on the network and connect by SSH.

Example:

```bash
ssh alwelch@pi.local
```

If hostname resolution does not work, use the IPv4 address assigned by the router.

### Check the OS and Wi-Fi link

```bash
uname -a
```

```bash
iw dev wlan0 link
```

The successful fresh-install baseline was:

```text
Linux pi 6.18.50+rpt-rpi-v7
Connected to BSSID 8a:9e:68:73:1a:49
SSID MyRouter
Frequency 2412 MHz
Signal approximately -60 dBm
RX bitrate 72.2 MBit/s
TX bitrate 58.5 MBit/s
```

### Check for the historical Wi-Fi failure signature

```bash
journalctl -b | grep -Ei "CTRL-EVENT-DISCONNECTED|ASSOC-REJECT|supplicant-timeout"
```

A healthy result is no output.

### Start a continuous IPv4 ping from Windows

```cmd
ping -t 10.0.0.240
```

Stop the ping with `Ctrl+C` to display statistics.

Fresh bare-OS reference result:

```text
Sent: 2497
Received: 2492
Lost: 5
Actual loss: approximately 0.20%
Minimum: 2 ms
Maximum: 991 ms
Average: 27 ms
```

A small number of isolated ping timeouts did occur, but there were no matching association failures in the journal. The important distinction is between isolated packet loss and actual Wi-Fi association loss.

### Optional broader Wi-Fi journal check

```bash
journalctl -b | grep -Ei "wlan|brcmf|wpa|NetworkManager|deauth|disassoc|assoc"
```

---

## 4. Install KIAUH

Git is required to download KIAUH.

```bash
sudo apt-get update && sudo apt-get install git -y
```

Clone KIAUH into the home directory:

```bash
cd ~
git clone https://github.com/dw-0/kiauh.git
```

Run KIAUH:

```bash
./kiauh/kiauh.sh
```

Install the core components first:

1. Klipper
2. Moonraker
3. Mainsail

Do not install all optional services at once. Keep the continuous Windows IPv4 ping running during installation.

After the core installation, repeat:

```bash
journalctl -b | grep -Ei "CTRL-EVENT-DISCONNECTED|ASSOC-REJECT|supplicant-timeout"
```

The successful core-install checkpoint had no matching Wi-Fi events.

---

## 5. Restore the Git-Backed Printer Configuration

KIAUH creates this directory:

```text
/home/alwelch/printer_data/config
```

If the GitHub repository represents the entire live Klipper configuration directory, preserve the new KIAUH-generated directory and clone the repository into its place.

```bash
cd ~/printer_data
mv config config_fresh
```

The `mv` command renames the newly created `config` directory to `config_fresh`. It does not delete it.

Clone the configuration repository:

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git config
```

Verify:

```bash
cd ~/printer_data/config
git remote -v
git status
```

### Configure Git identity

```bash
git config --global user.name "Austin Welch"
git config --global user.email "YOUR_GITHUB_EMAIL"
```

Verify:

```bash
git config --global --list
```

---

## 6. Configure Non-Interactive GitHub Credentials

A GitHub personal access token is used as the HTTPS password.

> **Security note:** Prefer a fine-grained token limited to this configuration repository with only the permissions needed to pull and push. Never place the token directly in `autocommit.sh`, `printer.cfg`, or the remote URL.

Run these commands in order:

```bash
cd ~/printer_data/config
```

```bash
git config --global credential.helper store
```

```bash
git config --global credential.helper
```

Expected output:

```text
store
```

Authenticate once:

```bash
git push
```

When prompted:

```text
Username: GitHub username
Password: GitHub personal access token
```

Nothing is displayed while the token is pasted. This is normal.

After successful authentication, secure the credential file:

```bash
chmod 600 ~/.git-credentials
```

Verify the permissions without displaying the file contents:

```bash
ls -l ~/.git-credentials
```

Expected permission prefix:

```text
-rw-------
```

Test non-interactive authentication:

```bash
git push
```

Expected result without another prompt:

```text
Everything up-to-date
```

Do not run `cat ~/.git-credentials`; the token is stored in that file.

---

## 7. Review Restored Paths and Includes

Older Klipper installations may reference the deprecated directory:

```text
/home/alwelch/klipper_config
```

The current live configuration directory is:

```text
/home/alwelch/printer_data/config
```

Search for outdated paths:

```bash
grep -R "/home/alwelch/klipper_config" ~/printer_data/config
```

Example update for saved variables:

```ini
[save_variables]
filename: /home/alwelch/printer_data/config/variables.cfg
```

Check for missing or broken symbolic links:

```bash
cd ~/printer_data/config
ls -la
```

```bash
find ~ -name "variables.cfg" -print
```

Correct or temporarily comment unresolved includes until their associated software is installed. Do not recreate the old `~/klipper_config` directory solely to hide obsolete paths.

---

## 8. Install Beacon Support

Clone and install the Beacon Klipper module:

```bash
cd ~
git clone https://github.com/beacon3d/beacon_klipper.git
./beacon_klipper/install.sh
```

Verify USB serial devices:

```bash
ls -l /dev/serial/by-id/
```

The working installation showed entries similar to:

```text
usb-Beacon_Beacon_RevH_... -> ../../ttyACM0
usb-Klipper_stm32f042x6_... -> ../../ttyACM1
```

Do not assign the STM32F042 device to the main `[mcu]`; that device is the separate menu/display MCU in this printer.

### Resolve the NumPy/OpenBLAS dependency if required

If Klipper reports:

```text
Importing the numpy C-extensions failed
libopenblas.so.0: cannot open shared object file
```

install OpenBLAS:

```bash
sudo apt update
sudo apt install libopenblas-dev
```

Verify the library:

```bash
ldconfig -p | grep openblas
```

Test NumPy inside Klipper's Python environment:

```bash
~/klippy-env/bin/python -c "import numpy; print(numpy.__version__)"
```

Restart Klipper:

```bash
sudo systemctl restart klipper
```

---

## 9. Enable Raspberry Pi GPIO Serial for the SKR Mini E3 V3

The SKR Mini E3 V3 is connected through the Pi GPIO UART, not through the USB devices listed under `/dev/serial/by-id/`.

The intended Klipper configuration is:

```ini
[mcu]
serial: /dev/serial0
restart_method: command
```

On a fresh Raspberry Pi OS installation, `/dev/serial0` may not exist until serial hardware is enabled.

Run:

```bash
sudo raspi-config
```

Select:

1. **Interface Options**
2. **Serial Port**
3. Login shell over serial: **No**
4. Serial port hardware enabled: **Yes**
5. Finish and reboot

After reboot, verify:

```bash
ls -l /dev/serial0
```

The working result was:

```text
/dev/serial0 -> ttyS0
```

Verify the boot configuration:

```bash
grep -Ei "enable_uart|console=serial" /boot/firmware/config.txt /boot/firmware/cmdline.txt
```

Expected relevant setting:

```text
enable_uart=1
```

There should not be a serial login console competing for the UART.

List the UART device:

```bash
ls -l /dev/ttyAMA* /dev/ttyS* 2>/dev/null
```

Restart Klipper:

```bash
sudo systemctl restart klipper
```

A successful connection identifies the main MCU as `stm32g0b1xx` at `250000` baud.

---

## 10. Verify the Menu/Display MCU

The restored configuration uses a separate menu MCU. The fresh installation detected it as a USB Klipper STM32F042 device.

List persistent USB identifiers:

```bash
ls -l /dev/serial/by-id/
```

Use the exact persistent `/dev/serial/by-id/...` path already stored in the menu MCU configuration. Do not replace it with `/dev/ttyACM1`, because `ttyACM` numbering can change across boots.

Successful Klipper startup identifies the menu MCU as:

```text
MCU=stm32f042x6
```

---

## 11. Build and Install the Raspberry Pi Host MCU

The restored configuration includes:

```ini
[mcu host]
serial: /tmp/klipper_host_mcu
```

If Klipper reports that `/tmp/klipper_host_mcu` does not exist, install the Linux host MCU service.

### Install and enable the service

```bash
cd ~/klipper
sudo cp ./scripts/klipper-mcu.service /etc/systemd/system/
sudo systemctl enable klipper-mcu.service
```

### Configure the host MCU build

```bash
cd ~/klipper
make menuconfig
```

Set:

```text
Micro-controller Architecture: Linux process
```

Save and exit.

### Build and install

```bash
sudo service klipper stop
make flash
sudo service klipper start
```

### Verify

```bash
systemctl status klipper-mcu --no-pager
```

Expected status:

```text
Active: active (running)
```

The running command should reference:

```text
/usr/local/bin/klipper_mcu -r -I /tmp/klipper_host_mcu
```

Verify the interface:

```bash
ls -l /tmp/klipper_host_mcu
```

Restart Klipper:

```bash
sudo systemctl restart klipper
```

---

## 12. Validate Basic Printer Operation

Before installing optional services, confirm:

- Mainsail opens successfully
- SKR Mini E3 V3 connects
- Menu/display MCU connects
- Raspberry Pi host MCU connects
- Beacon connects
- Temperatures are reported
- X, Y, and Z can home correctly
- Bed heater operates
- Nozzle heater operates
- Main printer controls respond

Do not perform unattended heating or motion tests. Remain available to stop the printer if restored pin assignments, direction settings, limits, or macros are incorrect.

---

## 13. Wi-Fi Health Checks During Restoration

Run the primary failure-signature check after each major installation stage:

```bash
journalctl -b | grep -Ei "CTRL-EVENT-DISCONNECTED|ASSOC-REJECT|supplicant-timeout"
```

A healthy result is no output.

For a broader check:

```bash
journalctl -b | grep -Ei "wlan|brcmf|wpa|NetworkManager|deauth|disassoc|assoc"
```

For current link information:

```bash
iw dev wlan0 link
```

From Windows:

```cmd
ping -t 10.0.0.240
```

The fresh installation may show occasional isolated ICMP timeouts. The historical failure signature to watch for is actual association failure, especially:

```text
CTRL-EVENT-DISCONNECTED
ASSOC-REJECT status_code=16
supplicant-timeout
```

At the working basic-printer checkpoint, none of those events had occurred on the fresh installation.

---

## 14. Restore Optional Components in Stages

Do not install every optional service simultaneously. Restore in logical groups and recheck Wi-Fi after each group.

Suggested order:

1. Shell scripts and associated Moonraker configuration
2. Remaining macros and helper files
3. Crowsnest and camera configuration
4. Remote monitoring integrations such as Obico, if used
5. Additional optional services
6. Sonar last

After each stage:

```bash
journalctl -b | grep -Ei "CTRL-EVENT-DISCONNECTED|ASSOC-REJECT|supplicant-timeout"
```

Keep the default fresh-OS network configuration while establishing stability. Do not immediately reapply the old Bullseye workarounds such as BSSID locking, band locking, or forced power-save changes.

---

## 15. Autocommit Script Template

After Git push authentication works non-interactively, an `autocommit.sh` file can back up configuration changes.

Example:

```bash
#!/bin/bash

set -e

CONFIG_DIR="/home/alwelch/printer_data/config"

cd "$CONFIG_DIR"

git add -A

if git diff --cached --quiet; then
    exit 0
fi

git commit -m "Auto backup: $(date '+%Y-%m-%d %H:%M:%S')"
git push
```

Make it executable:

```bash
chmod +x ~/printer_data/config/autocommit.sh
```

Test it manually:

```bash
~/printer_data/config/autocommit.sh
```

Inspect recent commits:

```bash
cd ~/printer_data/config
git log --oneline -5
```

Only automate the script after manual execution succeeds without requesting credentials.

---

## 16. Useful Diagnostic Commands

### Klipper service

```bash
systemctl status klipper --no-pager
```

### Moonraker service

```bash
systemctl status moonraker --no-pager
```

### Host MCU service

```bash
systemctl status klipper-mcu --no-pager
```

### Persistent USB serial devices

```bash
ls -l /dev/serial/by-id/
```

### GPIO UART

```bash
ls -l /dev/serial0
```

### Current Wi-Fi link

```bash
iw dev wlan0 link
```

### Wi-Fi association failure check

```bash
journalctl -b | grep -Ei "CTRL-EVENT-DISCONNECTED|ASSOC-REJECT|supplicant-timeout"
```

### Previous boot after a hard reset

```bash
journalctl -b -1 > ~/previous_boot.log
```

### Recent full journal capture

```bash
journalctl --since "10 minutes ago" > ~/recent_connection_issue.log
```

---

## 17. Known Working Architecture Summary

```text
Raspberry Pi 3
├── Raspberry Pi OS Lite 32-bit
├── NetworkManager 1.52.1 at the validated checkpoint
├── Klipper
├── Moonraker
├── Mainsail
├── Beacon Klipper module
├── Linux host MCU
│   └── /tmp/klipper_host_mcu
├── SKR Mini E3 V3
│   └── /dev/serial0 -> /dev/ttyS0
├── Beacon RevH
│   └── persistent /dev/serial/by-id/ USB path
└── Menu/display MCU STM32F042
    └── persistent /dev/serial/by-id/ USB path
```

---

## 18. Final Recovery Checklist

- [ ] Old microSD card preserved
- [ ] Fresh Raspberry Pi OS Lite 32-bit flashed
- [ ] SSH and Wi-Fi configured in Raspberry Pi Imager
- [ ] Bare-OS Wi-Fi baseline checked
- [ ] KIAUH installed
- [ ] Klipper installed
- [ ] Moonraker installed
- [ ] Mainsail installed
- [ ] Git-backed printer configuration restored
- [ ] Git identity configured
- [ ] GitHub PAT verified for push access
- [ ] Non-interactive credentials secured
- [ ] Old config paths updated
- [ ] Beacon module installed
- [ ] OpenBLAS installed if required
- [ ] GPIO serial enabled using `raspi-config`
- [ ] `/dev/serial0` verified
- [ ] SKR Mini E3 V3 connected
- [ ] Menu/display MCU connected
- [ ] Linux host MCU built and enabled
- [ ] Basic homing verified
- [ ] Temperature reporting verified
- [ ] Bed and nozzle heaters verified
- [ ] Wi-Fi failure-signature journal remains empty
- [ ] Optional services restored incrementally
- [ ] Sonar installed last, if still desired
- [ ] Autocommit tested manually before automation

---

## Important Safety and Security Notes

- Never hot-plug printer electronics unless the hardware documentation explicitly permits it.
- Keep the old SD card unchanged until the new installation is fully validated.
- Do not expose GitHub tokens in scripts, screenshots, Git remotes, logs, or repository files.
- Use persistent `/dev/serial/by-id/` paths for USB MCUs instead of `/dev/ttyACM0` or `/dev/ttyACM1`.
- Do not run `rpi-update` as part of this process.
- Avoid restoring old system-level network configuration into the fresh OS.
- Validate motion direction, endstops, homing, heaters, and thermal protection before attempting a print.
