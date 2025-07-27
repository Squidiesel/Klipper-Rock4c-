# MainsailOS on DietPi for Rock Pi 4C+

## Overview

This guide walks you through running Klipper + Moonraker + Mainsail on a Rock Pi 4C+ using DietPi as the base OS. It’s optimized for lightweight, headless 3D printer control.


### Troubleshooting in #8. Fixes and Tweaks
---

## 1. Flash DietPi to SD Card

### Download Image

> **Note:** Rock Pi 4C+ is not officially supported by DietPi. Use the community-maintained image instead.

* Go to the official DietPi website: [https://dietpi.com/](https://dietpi.com/)
* Navigate to the **Download** section.
* Find the **Rock Pi 4C+ (ARMv8)** image and download the latest `.img` file.

### Flash with Balena Etcher or Raspberry Pi Imager

* Download and install [Balena Etcher](https://www.balena.io/etcher/)
* Insert your SD card into your PC’s card reader.
* Open Etcher.
* Select the downloaded DietPi image file.
* Select your SD card as the target.
* Click **Flash** (Etcher) and wait for completion.
* Once done, safely eject the SD card.

### Enable SSH (Headless Setup)

* Remove and re-insert the SD card to mount its **boot** partition (usually labeled `boot`).
* Open the boot partition folder in your file explorer.
* Create a new empty notepad file named exactly `ssh` (no file extension).

> **Windows Tip:** When saving the empty text file named ssh change the file type from the default .Txt to All Files.

  ```bash
  touch /path/to/boot/ssh
  ```


### (Optional) WiFi Setup

* In the boot partition, create a file named `dietpi.txt` using any text editor.

* Add these lines, replacing with your WiFi credentials:

  ```ini
  AUTO_SETUP_NET_WIFI_ENABLED=1
  AUTO_SETUP_NET_WIFI_SSID=YourSSID
  AUTO_SETUP_NET_WIFI_KEY=YourPassword
  AUTO_SETUP_NET_WIFI_COUNTRY_CODE=AU
  ```

* Save and close the file.

---

## 2. First Boot and Configuration

### Boot Rock Pi 4C+

* Insert the flashed SD card into the Rock Pi 4C+.
* Connect power and network (Ethernet recommended for first boot, or WiFi if set).
* **Wait 2-3 minutes** for initial boot and setup.

### Login via SSH

* Find the Rock Pi IP address:

  * Check your router’s DHCP list.
  * Or scan the network with a tool like [Fing (mobile)](https://www.fing.com/).

* Connect:

  ```bash
  ssh root@<rockpi-ip>
  ```

* Default password: `dietpi`

* If prompted to accept the host key, type `yes`.

### Initial DietPi Setup

* On first login, DietPi will run setup scripts:

  * Set locale, timezone, keyboard, etc.
  * Update packages.
  * Change root password when prompted.
* Follow on-screen prompts carefully.
* Reboot if required.

---

## 3. Install Klipper Stack via KIAUH

### Install Dependencies

```bash
apt update && apt upgrade -y
apt install git curl python3-pip -y
```

### Install KIAUH

```bash
git clone https://github.com/th33xitus/kiauh.git
cd kiauh
chmod +x kiauh.sh
./kiauh.sh
```

### Use KIAUH

* Select **Install Klipper**.
* Then **Install Moonraker**.
* Then **Install Mainsail**.
* KIAUH will handle dependencies and configuration.

---

## 4. Flash Firmware to SKR Mini E3 V3.0

### Compile Klipper Firmware

```bash
cd ~/klipper
make menuconfig
```

* In the menu:

  * Micro-controller: `STMicroelectronics STM32`
  * Processor model: `STM32G0B1`
  * Bootloader offset: `8KiB`
  * Communication interface: `USB (on PA11/PA12)`

* Exit saving the config.

* Compile:

```bash
make
```

### Transfer Firmware to SKR Mini E3 V3.0

#### Option A: microSD Card

```bash
cp out/klipper.bin /mnt/SDCARD/firmware.bin
```

* If `/mnt/SDCARD` is not available, copy manually.
* Rename `klipper.bin` to `firmware.bin`.
* Insert into board and power cycle.

#### Option B: WinSCP

* Connect via SFTP to Rock Pi IP as `root`.
* Navigate to `~/klipper/out/`, download `klipper.bin`, rename to `firmware.bin`, and copy to a FAT32 microSD.

* For transferring firmware via microSD, see troubleshooting in 8. Fixes and Tweaks if flashing fails.

### If SFTP Fails

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

## 5. Connect MCU

### Find MCU Device Path

```bash
ls /dev/serial/by-id/
```

Expected output:

```
usb-Klipper_stm32f103xe_123456789ABC-if00 -> ../../ttyACM0
```

### Update `printer.cfg`

```bash
nano ~/printer_data/config/printer.cfg
```

Add or modify:

```ini
[mcu]
serial: /dev/serial/by-id/usb-Klipper_stm32f103xe_123456789ABC-if00
```

* Save: Ctrl+O, Enter
* Exit: Ctrl+X

### Restart Klipper Service

```bash
sudo service klipper restart
```

## 5.1 Verify USB Serial Device Paths

* Use USB 2.0 ports, avoid USB 3.0.

```bash
dmesg | grep tty
```

```bash
sudo apt install usbutils -y
lsusb
```

## 6. Access Mainsail Web UI

Open browser to:

```
http://<rockpi-ip>
```

> Ensure Rock Pi and your PC are on the same network.


## 7. NGINX Setup, Config Fixes, and WinSCP Access

### NGINX Setup

```bash
sudo mkdir -p /var/log/nginx
sudo touch /var/log/nginx/error.log
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Config Fixes

```bash
wget https://github.com/Klipper3d/klipper/blob/master/config/generic-bigtreetech-skr-mini-e3-v3.0.cfg -O /home/dietpi/printer_data/config/printer.cfg
```

> Backup existing `printer.cfg` first.

### WinSCP Access Fix

```bash
sudo nano /etc/ssh/sshd_config
```

Add or change:

```
PermitRootLogin yes
PasswordAuthentication yes
```

```bash
sudo systemctl restart ssh
```

## 8. Fixes and Tweaks

### Boot Partition Not Visible?

* Boot once in Rock Pi, wait, then remove/reinsert card to PC.

### USB Serial Not Found?

* Always use `serial/by-id` paths.
* Try different USB ports.

### MCU Flash Failed?

* Rename to `firmware.bin`.
* Use FAT32 cards only.

### Auto-Start Services

```bash
sudo systemctl enable klipper
sudo systemctl enable moonraker
sudo systemctl enable mainsail
sudo systemctl enable ssh
```

---

## Using `cd ~` and Reconnecting via SSH as Root

### Returning to Your Home Directory

```bash
cd ~
```

### Reconnecting via SSH

```bash
ssh root@<rockpi-ip-address>
```

* Default password: `dietpi`

### Example Workflow

```bash
cd ~
exit
ssh root@192.168.1.100
```

## 9. Find Rock Pi IP Address

```bash
ip a
```

Look for `inet` under `eth0` or `wlan0`:

```
inet 192.168.1.xxx/24
```

Or:

```bash
arp -a
hostname -I
```

## 10. Check Klipper MCU Status

```bash
sudo journalctl -u klipper -f
```

```bash
tail -f ~/printer_data/logs/klippy.log
```

```bash
curl http://localhost/api/printer/info
```

## Conclusion

DietPi + Rock Pi 4C+ gives you a lean and powerful Klipper setup with Mainsail. Ideal for headless printers and custom enclosures. Tested and working as of June 2025.

**Contributors:** ChatGPT + Mitchell
