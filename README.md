# Ubuntu 24.04.4 LTS Clean Install Guide

## Using dd, Autoinstall, GitHub Config, and Android USB Tethering

---

## Overview

This guide walks you through wiping your entire computer (Windows + Ubuntu) and installing a fresh Ubuntu 24.04.4 LTS using:

- `dd` to flash the USB installer
- An autoinstall YAML config hosted on GitHub
- Android phone USB tethering for internet during installation
- Manual Wi-Fi setup after installation (to avoid exposing credentials on GitHub)

---

## What You Need

- A USB drive (8GB minimum) — **all data on it will be erased**
- Your Android phone with a USB cable
- A GitHub account
- Your Wi-Fi network name and password (for post-install setup)

---

## Phase 1: Prepare the Autoinstall Config on GitHub

### Step 1: Generate your password hash

Open a terminal on your current system and run:

```bash
openssl passwd -6
```

It will ask you to type a password. Type it, press Enter, confirm it. It outputs a long hash string starting with `$6$`. Copy this entire string — you'll need it in the next step.

### Step 2: Create a GitHub repository

1. Go to https://github.com and log in
2. Click **New Repository**
3. Name it something like `ubuntu-autoinstall`
4. Set it to **Public** (the installer needs to access it without authentication)
5. Click **Create repository**

### Step 3: Create the `meta-data` file

In your new repo, click **Add file > Create new file**. Name it `meta-data` and leave the content completely empty. Commit the file.

### Step 4: Create the `user-data` file

Click **Add file > Create new file** again. Name it `user-data` and paste the following content. **Edit the values marked with comments before committing.**

```yaml
#cloud-config
autoinstall:
  version: 1

  # --- Language & Keyboard ---
  locale: en_US.UTF-8
  keyboard:
    layout: us

  # --- Your Identity ---
  identity:
    hostname: "my-ubuntu"
    username: "username here"
    password: "YOUR_HASH_HERE"

  # --- Timezone ---
  timezone: Africa/Nairobi

  # --- Disk Setup (WIPES EVERYTHING) ---
  storage:
    layout:
      name: lvm
      match:
        size: largest

  # --- Snaps ---
  snaps:
    - name: spotify
      classic: false
    - name: obsidian
      classic: true
    - name: code
      classic: true
    - name: node
      classic: true
    - name: docker
      classic: false
    - name: slack
      classic: false
    - name: telegram-desktop
      classic: false
    - name: discord
      classic: false
    - name: chromium
      classic: false
    - name: obs-studio
      classic: false
    - name: gimp
      classic: false
    - name: postman
      classic: false
    - name: alacritty
      classic: true

  # --- Packages to Install ---
  packages:
    - git
    - curl
    - wget
    - htop
    - vim
    - openssh-server
    - build-essential
    - net-tools
    - ufw
    - nano

  # --- Commands to Run After Installation ---
  late-commands:
    - curtin in-target -- systemctl enable ssh
    - curtin in-target -- ufw allow ssh
    - curtin in-target -- ufw enable
    - curtin in-target -- bash -c "curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg"
    - curtin in-target -- bash -c "echo 'deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg] https://brave-browser-apt-release.s3.brave.com/ stable main' > /etc/apt/sources.list.d/brave-browser-release.list"
    - curtin in-target -- apt update
    - curtin in-target -- apt install -y brave-browser
    - curtin in-target -- bash -c "wget -O /tmp/protonvpn.deb https://repo2.protonvpn.com/debian/dists/stable/main/binary-all/protonvpn-stable-release_1.0.3-3_all.deb"
    - curtin in-target -- dpkg -i /tmp/protonvpn.deb
    - curtin in-target -- apt update
    - curtin in-target -- apt install -y proton-vpn-gnome-desktop

  # --- Updates ---
  updates: security
```

**Note:** Wi-Fi credentials are NOT included in this file to avoid exposing them on a public GitHub repo. Wi-Fi will be configured manually after installation.

Commit the file.

### Step 5: Get your raw GitHub URL

Your URL will follow this pattern:

```
https://raw.githubusercontent.com/YOUR_USERNAME/ubuntu-autoinstall/main/
```

Replace `YOUR_USERNAME` with your actual GitHub username. Save this URL — you'll need it during boot.

---

## Phase 2: Download and Verify the Ubuntu ISO

### Step 6: Download the ISO

```bash
cd ~/Downloads
wget https://releases.ubuntu.com/noble/ubuntu-24.04.4-desktop-amd64.iso
```

This is a 6.2GB download. Wait for it to complete.

### Step 7: Verify the download

```bash
cd ~/Downloads
wget https://releases.ubuntu.com/noble/SHA256SUMS
sha256sum -c SHA256SUMS 2>/dev/null | grep desktop
```

You should see:

```
ubuntu-24.04.4-desktop-amd64.iso: OK
```

If it says **FAILED**, the download is corrupted. Delete it and download again.

---

## Phase 3: Flash the USB Drive

### Step 8: Plug in the USB drive

Insert the USB drive into your computer.

### Step 9: Identify the USB drive

```bash
lsblk -p -e 7
```

This shows all physical drives, excluding loop devices. Your USB will show up as something like `/dev/sdb` or `/dev/sdc`. Check the **SIZE** column to confirm it matches your USB drive's capacity.

**CRITICAL: Triple-check this. If you pick the wrong device, you will wipe the wrong drive.**

### Step 10: Unmount the USB drive

```bash
sudo umount /dev/sdb*
```

Replace `sdb` with your actual USB device name.

### Step 11: Flash the ISO with dd

```bash
sudo dd if=~/Downloads/ubuntu-24.04.4-desktop-amd64.iso of=/dev/sdb bs=4M status=progress oflag=sync
```

Replace `sdb` with your actual USB device name.

**What each flag means:**
- `if=` — input file (the Ubuntu ISO)
- `of=` — output file (your USB drive, NOT a partition like sdb1)
- `bs=4M` — block size of 4 megabytes (faster write speed)
- `status=progress` — shows write progress
- `oflag=sync` — ensures all data is physically written before finishing

This takes 5-15 minutes depending on USB speed. Wait for it to complete and return to the terminal prompt.

### Step 12: Eject the USB safely

```bash
sync
sudo eject /dev/sdb
```

---

## Phase 4: Set Up Android USB Tethering

### Step 13: Plug in your Android phone

Connect your Android phone to the computer via USB cable.

### Step 14: Enable USB tethering

1. On your Android phone, go to **Settings > Connections > Mobile Hotspot and Tethering** (path varies by phone brand — Samsung, Pixel, etc. may have slightly different menu names)
2. Toggle on **USB tethering**

That's it. No trust prompts or special drivers needed. Android USB tethering works out of the box with Linux.

### Step 15: Verify tethering works (optional)

On your current system, check that the phone appears as a network interface:

```bash
ip link show
```

Look for a new interface that wasn't there before — usually `usb0` or something starting with `enx`. If you see it, tethering is working.

---

## Phase 5: Boot and Install

### Step 16: Plug everything in

Before rebooting, make sure you have:

- The flashed USB drive plugged in
- Your Android phone plugged in via USB with USB tethering enabled

### Step 17: Reboot and enter boot menu

```bash
sudo reboot
```

As the computer restarts, press the boot menu key repeatedly:

- **F12** — most Dell, Lenovo
- **F2** — ASUS, some Acer
- **F9** — HP
- **Esc** — varies

Select your USB drive from the list. **Choose the UEFI option** if two entries appear.

### Step 18: Edit the GRUB boot entry

When the Ubuntu installer GRUB menu appears:

1. Press **`e`** to edit the boot entry
2. Find the line that starts with `linux`
3. At the **end** of that line, add a space and then:

```
autoinstall ds=nocloud-net;s=https://raw.githubusercontent.com/YOUR_USERNAME/ubuntu-autoinstall/main/
```

Replace `YOUR_USERNAME` with your GitHub username.

4. Press **Ctrl+X** or **F10** to boot

### Step 19: Let it run

The installer will:

1. Connect to the internet via Android USB tethering
2. Fetch your autoinstall config from GitHub
3. Wipe your entire hard drive (Windows, old Ubuntu, everything)
4. Install Ubuntu 24.04.4 LTS with LVM
5. Set your hostname, username, and password
6. Install all apt packages and snaps you specified
7. Install Brave browser and ProtonVPN
8. Enable SSH and the firewall
9. Set your timezone to Africa/Nairobi
10. Reboot automatically when done

**You may see a prompt asking to confirm the autoinstall.** Press Enter or type `yes` to proceed.

The installation takes roughly 15-30 minutes depending on your hardware and internet speed.

### Step 20: Remove USB and phone

When the installer finishes and the machine reboots:

1. Remove the USB drive
2. Unplug the Android phone

---

## Phase 6: Post-Install — Connect to Wi-Fi

### Step 21: Log in

Log in with the username `leviscoffie` and the password you set during hash generation.

### Step 22: Connect to Wi-Fi

Your machine has no internet yet since Wi-Fi wasn't configured in the YAML. Connect now using one command:

```bash
nmcli device wifi connect "YourWiFiName" password "YourWiFiPassword"
```

Replace with your actual Starlink Wi-Fi name and password.

Verify the connection:

```bash
ping -c 3 google.com
```

### Step 23: Make Wi-Fi persist across reboots

The `nmcli` command above automatically saves the connection. Your machine will now reconnect to this Wi-Fi network on every boot without you needing to do anything.

Verify it's saved:

```bash
nmcli connection show
```

You should see your Wi-Fi network listed.

---

## Phase 7: Post-Install Verification

### Step 24: Verify everything is set up

```bash
# Verify system info
hostnamectl

# Verify network connectivity
ping -c 3 google.com

# Verify Wi-Fi is connected
nmcli device status

# Verify installed packages
which git curl wget htop vim nano

# Verify SSH is running
systemctl status ssh

# Verify firewall is active
sudo ufw status

# Verify timezone
timedatectl

# Verify snaps installed
snap list

# Verify Brave installed
which brave-browser
```

### Step 25: Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 26: Clean up old UEFI boot entries

```bash
efibootmgr
```

If you see leftover Windows or old Ubuntu entries, remove them:

```bash
# Replace 0002 with the actual entry number to delete
sudo efibootmgr -b 0002 -B
```

### Step 27: Delete the GitHub repo

Now that installation is complete, delete the `ubuntu-autoinstall` repo from GitHub since you no longer need it and it contains your password hash.

1. Go to your repo on GitHub
2. Click **Settings**
3. Scroll to the bottom
4. Click **Delete this repository**

---

## Phase 8: Configure Netplan for Wi-Fi (Optional)

If you prefer Netplan over NetworkManager for Wi-Fi management (useful for headless/always-on setups where you want Wi-Fi to connect before login):

### Step 28: Find your Wi-Fi interface name

```bash
ip link show
```

Look for the interface starting with `wl` (likely `wlo1` based on your hardware).

### Step 29: Create a Netplan config

```bash
sudo nano /etc/netplan/01-wifi.yaml
```

Add:

```yaml
network:
  version: 2
  renderer: NetworkManager
  wifis:
    wlo1:
      dhcp4: true
      access-points:
        "YourWiFiName":
          password: "YourWiFiPassword"
```

### Step 30: Apply and secure the config

```bash
sudo netplan try
sudo netplan apply
sudo chmod 600 /etc/netplan/01-wifi.yaml
```

The `chmod 600` ensures only root can read the file, protecting your Wi-Fi password.

---

## Troubleshooting

### Autoinstall not detected

- Make sure the `user-data` and `meta-data` files are in the **root** of your GitHub repo (not in a subfolder)
- Make sure the repo is **public**
- Make sure the URL ends with a trailing `/`
- Double-check for YAML syntax errors — indentation must use spaces, not tabs

### Android tethering not working

- Make sure USB tethering is toggled on in your phone settings (not just Mobile Hotspot)
- Try unplugging and replugging the phone after the installer loads
- Try a different USB cable — some cables are charge-only and don't support data
- Check if the interface appears: press `Ctrl+Alt+F2` for a terminal and run `ip link show`

### Installation hangs at storage step

- The `match: size: largest` option picks your biggest disk. If you have multiple drives, make sure this picks the right one
- Try removing the `match` block and letting it use the default disk

### Wi-Fi doesn't connect after install

- Double-check the Wi-Fi name is exact (case-sensitive, including spaces)
- Make sure the password is correct
- Run `nmcli device wifi list` to see available networks
- If your interface name changed, check with `ip link show`

### GRUB edit doesn't stick

- Make sure you're editing the correct line (the one starting with `linux`)
- Make sure there's a space between the existing text and `autoinstall`
- The semicolon in the ds parameter is important: `ds=nocloud-net;s=...`

### Snaps fail to install

- This usually means internet was lost during installation
- After booting into Ubuntu, connect to Wi-Fi and install missed snaps manually:
  ```bash
  sudo snap install spotify
  sudo snap install obsidian --classic
  sudo snap install code --classic
  ```

---

## Quick Reference — Complete Command Sequence

```bash
# 1. Generate password hash
openssl passwd -6

# 2. Download Ubuntu ISO
cd ~/Downloads
wget https://releases.ubuntu.com/noble/ubuntu-24.04.4-desktop-amd64.iso

# 3. Verify download
wget https://releases.ubuntu.com/noble/SHA256SUMS
sha256sum -c SHA256SUMS 2>/dev/null | grep desktop

# 4. Identify USB drive
lsblk -p -e 7

# 5. Unmount USB
sudo umount /dev/sdb*

# 6. Flash ISO to USB
sudo dd if=~/Downloads/ubuntu-24.04.4-desktop-amd64.iso of=/dev/sda bs=4M status=progress oflag=sync

# 7. Eject USB
sync
sudo eject /dev/sdb

# 8. Enable Android USB tethering on phone

# 9. Reboot
sudo reboot

# === At GRUB menu, press 'e' and add: ===
# autoinstall ds=nocloud-net;s=https://raw.githubusercontent.com/USERNAME/ubuntu-autoinstall/main/
# === Press Ctrl+X to boot ===

# 10. After install — connect to Wi-Fi
nmcli device wifi connect "YourWiFiName" password "YourWiFiPassword"

# 11. Update system
sudo apt update && sudo apt upgrade -y

# 12. Clean up UEFI entries
efibootmgr
sudo efibootmgr -b 0002 -B

# 13. Delete the GitHub repo
```