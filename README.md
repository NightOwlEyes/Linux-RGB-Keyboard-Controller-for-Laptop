# Linux-RGB-Keyboard-Controller-for-Laptop
**Linux on Gigabyte Laptops has few drivers that support LED activation for the keyboard. This project provides a complete solution based on the "tuxedo-keyboard" driver, allowing you to control your keyboard's RGB backlight via Terminal and native Fn keys.**

>[!CAUTION]
>**This guide was performed on Fedora Linux 42 (kernel 6.14.0-63.fc42.x86_64). Compatibility with the latest kernels (e.g., 6.15.x+) is not guaranteed. Other Linux distributions may require different package names.**

>[!IMPORTANT]
>**Secure Boot must be disabled.** Restart your computer and press `F2` to enter BIOS. Navigate to "Administer Secure Boot" and ensure "Secure Boot Status" is **Disabled**

## Features

*   **Full Backlight Control:** Activates and provides complete control over the RGB keyboard backlight on the Gigabyte G5 laptop, a feature not natively supported on most Linux distributions.

*   **Native `Fn` Key Support:** Restores the functionality of the hardware `Fn` keys for quick, on-the-fly adjustments, just like on Windows.
    *   `Fn` + `/`: Cycle through 7 preset colors.
    *   `Fn` + `*`: Toggle the backlight on/off.
    *   `Fn` + `-`: Decrease brightness.
    *   `Fn` + `+`: Increase brightness.

*   **Powerful Command-Line Interface (CLI):** Includes a user-friendly `led` command that goes beyond the hardware limitations.
    *   **Full RGB Color Customization:** Set any of the 16.7 million possible colors using RGB values (e.g., `sudo led rgb 255 100 0` for orange).
    *   **Fine-grained Brightness Control:** Set brightness to any specific value between 0 and 255 for the perfect intensity.

*   **Persistent State with Auto-Save:** This is the core of the solution. The system automatically remembers your last used color and brightness setting.
    *   **Works Seamlessly:** Whether you change the color using the `Fn` keys or the `led` command, the state is automatically saved periodically.
    *   **Restores on Boot:** The saved state is automatically restored every time you start your computer, ensuring your keyboard always looks the way you want it to.

## Prerequisites
Before starting, make sure you have installed all the necessary tools for building kernel modules.<br>
Depending on the linux system, there will be different installation commands: `sudo apt`, `sudo dnf`,...<br>
```
sudo dnf install git dkms make gcc kernel-devel
```

### 1. Install the kernel-devel package for your current kernel</br>
This ensures the driver can be built for the exact kernel version you are running.
```
sudo dnf install kernel-devel-$(uname -r)
```

### 2. Download and Install the DKMS Driver<br>
This will download the `clevo-keyboard` driver source and use DKMS to install it. DKMS ensures the driver is automatically rebuilt after kernel updates.
```
# Clean up any previous attempts
rm -rf ~/clevo-keyboard

# Clone the repository
cd ~
git clone https://github.com/wessel-novacustom/clevo-keyboard.git

# Automatically get driver version and install via DKMS
DRIVER_VERSION=$(grep "PACKAGE_VERSION" ~/clevo-keyboard/dkms.conf | cut -d'=' -f2 | tr -d '"')
sudo cp -R ~/clevo-keyboard /usr/src/tuxedo-keyboard-${DRIVER_VERSION}
sudo dkms install -m tuxedo-keyboard -v ${DRIVER_VERSION}
```
>[!NOTE]
>You will see **`bad exit status: 1`**, don't worry it's not serious, do the next step.<br>
>`Executing post-transaction command......`**`(bad exit status: 1)`**<br>
>`Failed command:`<br>
>`dracut --regenerate-all --force`
>
>But if you see **`bad exit status: 2`**, it means the kernel does not support it so the next steps cannot be performed, in my case it was kernel version 6.15.x so I switched back to an older kernel.<br>
>`Building module(s)...`**`(bad exit status: 2)`**<br>
>`Failed command:`<br>
>`make -j12 KERNELRELEASE=6.15.10-200.fc42.x86_64 KDIR=/lib/modules/...`<br>
>`Error! Bad return status for module build on kernel:...`

### 3. Load and Verify the Driver<br>
Load the module into the kernel and check if the control interface was created.
```
# Load the module
sudo modprobe tuxedo_keyboard

# Check if the control directory exists
ls /sys/class/leds/
```
If you see rgb:kbd_backlight in the output, the driver is working! You can test it by running:
```
echo 255 | sudo tee /sys/class/leds/rgb:kbd_backlight/brightness
```

### 4. Install the Control Scripts and Systemd Services
This will download this repository and place the `led` script and systemd services in the correct system directories.
```
rm -rf ~/Linux-RGB-Keyboard-Controller-for-Gigabyte-G5-Laptop
cd ~
git clone https://github.com/NightOwlEyes/Linux-RGB-Keyboard-Controller-for-Gigabyte-G5-Laptop.git && \
cd Linux-RGB-Keyboard-Controller-for-Gigabyte-G5-Laptop && \
sudo cp led /usr/local/bin/ && \
sudo cp kbd-backlight-* /etc/systemd/system/ && \
echo tuxedo_keyboard | sudo tee /etc/modules-load.d/tuxedo.conf
```

### 5. Set Permissions<br>
We need to set the correct permissions and SELinux context so the systemd services can execute the control script.
```
# Make the script executable
sudo chmod +x /usr/local/bin/led

# Set the correct SELinux context
sudo chcon -t bin_t /usr/local/bin/led
```

### 6. Enable and Start the Services.<br>
This will enable the services to start on boot and start the auto-save timer immediately.
```
# Enable the restore-on-boot service
sudo systemctl enable kbd-backlight-color.service

# Enable and start the periodic auto-save timer
sudo systemctl enable --now kbd-backlight-autosave.timer
```

### 7. Finalize Installation<br>
Reload the systemd manager to apply all changes and reboot the system.
```
sudo systemctl daemon-reload
sudo reboot now
```
## Usage
### Key Combinations
The native hardware keys should now be active.<br>
  `Fn` + `/` (num lock) Cycle through 7 preset colors.<br>
  `Fn` + `*` (num lock) Toggle the backlight on/off.<br>
  `Fn` + `-` (num lock) Decrease brightness.<br>
  `Fn` + `+` (num lock) Increase brightness.<br>

### Terminal Command
For full control over any color and brightness, use the led command.

**Usage:** `sudo led` `<command>` `[parameters]`

| Syntax| Command | Parameters | Description |
|------|------|---------|-------------|
| `sudo led` | `on` `off`  |  | Turn on/off the light. |
| `sudo led` | `bright` | `0` - `255` | Set custom brightness. |
| `sudo led` | `rgb` | `r` `g` `b` | Custom RGB color mixing. |
| `sudo led` | `save` |  | Save current state. |
| `sudo led` | `restore` |  | Restore saved state. |
| `sudo led` | `help` |  | To help. |

**For example:**<br>
- `sudo led rgb 128 128 225`<br>
- `sudo led bright 50`<br>
- `sudo led off`
