# MHS 3.5" Touchscreen Setup Guide (Raspberry Pi OS Bookworm)

This guide provides a reliable, manual method to configure an **MHS 3.5-inch SPI display** on a **Raspberry Pi 3B** running **Raspberry Pi OS (Bookworm)**. It configures the panel in **Portrait Orientation (rotate=0)** with a verified working touch calibration matrix and includes toggle scripts to switch cleanly between the 3.5" screen and a standard HDMI monitor.

---

## Step 1: Edit the Boot Configuration File

Open your system's boot configuration profile via your SSH session:

```bash
sudo nano /boot/firmware/config.txt
```

1. **Disable default 3D graphics acceleration** (Wayland-based acceleration will crash mini SPI panels on older hardware):
   Find the line `dtoverlay=vc4-kms-v3d` and comment it out by adding a `#` to the front:
   ```text
   #dtoverlay=vc4-kms-v3d
   ```

2. **Add the 3.5" Display Hardware block**:
   Scroll all the way down to the very bottom of the file (`[all]` section) and paste these exact lines:
   ```text
   dtparam=spi=on
   dtoverlay=piscreen,drm,speed=16000000,rotate=0
   hdmi_force_hotplug=1
   hdmi_group=2
   hdmi_mode=87
   hdmi_cvt=320 480 60 6 0 0 0
   ```

Save changes by pressing `Ctrl + O`, then `Enter`. Exit the editor with `Ctrl + X`.

---

## Step 2: Lock the Screen Resolution Viewport

Force the Linux kernel to stick strictly to the physical display boundaries so the splash logo and desktop canvas center correctly.

1. Open the boot command configuration file (Note: All parameters inside this file **must remain on a single continuous line**):
   ```bash
   sudo nano /boot/firmware/cmdline.txt
   ```
2. Scroll to the very end of that single line, add a single trailing **space**, and append this text:
   ```text
   video=SPI-1:320x480@60
   ```
3. Save changes (`Ctrl + O` $\\rightarrow$ `Enter`) and exit (`Ctrl + X`).

---

## Step 3: Switch the Desktop Display Engine to X11

The modern default window server (Wayland) drops processing frames on mini-SPI layers. Force the desktop environment back onto a reliable X11 rendering backend.

1. Open the Raspberry Pi Configuration menu:
   ```bash
   sudo raspi-config
   ```
2. Navigate to: **Advanced Options** $\\rightarrow$ **Wayland**
3. Select **X11** (or **Openbox**)
4. Choose **Finish**, and select **No** if asked to reboot immediately (we still need to configure the touch layers).

---

## Step 4: Install and Set Up the Touchscreen Matrix

Configure the X server to apply the `evdev` event driver to track your panel's coordinate registrations accurately.

1. Update your index packages and fetch the input handling module:
   ```bash
   sudo apt update
   sudo apt install -y xserver-xorg-input-evdev
   ```
2. Force the X system to prioritize legacy `evdev` mapping rules:
   ```bash
   sudo mkdir -p /etc/X11/xorg.conf.d
   sudo cp /usr/share/X11/xorg.conf.d/10-evdev.conf /etc/X11/xorg.conf.d/45-evdev.conf
   ```
3. Create your custom hardware calibration rule:
   ```bash
   sudo nano /etc/X11/xorg.conf.d/99-calibration.conf
   ```
4. Paste this precise block to map your structural workspace boundaries:
   ```text
   Section "InputClass"
           Identifier      "calibration"
           MatchProduct    "ADS7846 Touchscreen"
           Driver          "evdev"
           Option          "Calibration"   "3880 268 3936 227"
           Option          "SwapAxes"      "0"
           Option          "InvertX"       "1"
           Option          "InvertY"       "0"
   EndSection
   ```
5. Save changes (`Ctrl + O` $\\rightarrow$ `Enter`) and exit (`Ctrl + X`).

---

## Step 5: Create Display Toggle Automation Scripts

Build two executable scripts to switch output displays over SSH without manual text manipulation.

### Script A: Switch to Standard HDMI Monitor
1. Create and open the file:
   ```bash
   nano ~/switch-to-hdmi.sh
   ```
2. Paste this recovery sequence inside:
   ```bash
   #!/bin/bash
   # Re-enable standard graphics acceleration and disable SPI screen
   sudo sed -i 's/#dtoverlay=vc4-kms-v3d/dtoverlay=vc4-kms-v3d/g' /boot/firmware/config.txt
   sudo sed -i 's/dtoverlay=piscreen.*/#dtoverlay=piscreen,drm,speed=16000000,rotate=0/g' /boot/firmware/config.txt
   
   # Remove the forced video resolution limit from cmdline
   sudo sed -i 's/ video=SPI-1:320x480@60//g' /boot/firmware/cmdline.txt
   
   echo "Switching to HDMI output. The system will now reboot..."
   sudo reboot
   ```
3. Save, exit, and mark it as an executable binary:
   ```bash
   chmod +x ~/switch-to-hdmi.sh
   ```

### Script B: Switch to 3.5" LCD Touchscreen
1. Create and open the file:
   ```bash
   nano ~/switch-to-screen.sh
   ```
2. Paste this setup restoration script inside:
   ```bash
   #!/bin/bash
   # Turn off HDMI graphics acceleration and turn on the 3.5-inch panel
   sudo sed -i 's/^dtoverlay=vc4-kms-v3d/#dtoverlay=vc4-kms-v3d/g' /boot/firmware/config.txt
   sudo sed -i 's/#dtoverlay=piscreen.*/dtoverlay=piscreen,drm,speed=16000000,rotate=0/g' /boot/firmware/config.txt
   
   # Put the resolution block back into cmdline if it isn't there
   if ! grep -q "video=SPI-1:320x480@60" /boot/firmware/cmdline.txt; then
       sudo sed -i 's/$/ video=SPI-1:320x480@60/' /boot/firmware/cmdline.txt
   fi
   
   echo "Switching to 3.5 inch display. The system will now reboot..."
   sudo reboot
   ```
3. Save, exit, and mark it as an executable binary:
   ```bash
   chmod +x ~/switch-to-screen.sh
   ```

---

## Usage Instructions

Run the respective script from your home folder over SSH to toggle setups:

* **To output to an external HDMI monitor/TV:** Run `./switch-to-hdmi.sh`
* **To switch back to the mini 3.5" portable layout:** Run `./switch-to-screen.sh`

