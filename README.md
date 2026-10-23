# CachyOS GUI Updater (Secure Edition)

A sleek, automated, dark-mode graphical interface for CachyOS system updates. Built with Python and CustomTkinter, this utility automates the terminal-based `cachy-update` process while maintaining 100% strict Arch Linux security compliance.

Designed for both traditional desktop environments and handheld gaming PCs (like the Lenovo Legion Go) running Steam Game Mode. It acts as a smart virtual keyboard, dynamically injecting your sudo password from a secure memory vault before automatically handling package confirmations, service restarts, and AUR reviews in the background.

## ✨ Features
* **Strict Security Compliance:** Requires NO `visudo` overrides or permanent `NOPASSWD` backdoors. Your password is held temporarily in RAM, dynamically injected into a true Controlling TTY session, and instantly wiped from memory.
* **Smart Virtual Keyboard:** Automatically answers `[Y/n]` prompts, bypasses Arch News blocks, and hits `0` for service restarts.
* **AUR Pager Bypass:** Automatically injects `PAGER="cat"` to prevent AUR package reviews from freezing the update process.
* **Steam Game Mode Immunity:** Natively strips `LD_PRELOAD` from the environment to prevent Steam's 32-bit overlay injection from causing false-positive errors during execution.
* **Handheld / Gamepad Ready:** Includes built-in hotkeys mapped to standard keyboard inputs (`X` to close, `Y` to toggle terminal) for seamless Steam Input controller integration.

## 📦 Prerequisites
This script requires the `python-customtkinter` UI library. Install it via the AUR:
```bash
paru -S python-customtkinter
```

## 🚀 Installation & Setup

**1. Download the Script**
Save the `updater_gui.py` file to your home directory (or any preferred application folder) and grant it executable permissions:
```bash
chmod +x ~/updater_gui.py
```

**2. Launch**
You can now run the app directly from your terminal or create a standard `.desktop` shortcut:
```bash
./updater_gui.py
```

## 🎮 Steam Game Mode Setup (Handhelds)
To use this natively on a Legion Go or Steam Deck UI:

1. Open Steam in Desktop Mode.
2. Click **Games > Add a Non-Steam Game to My Library**.
3. Click **Browse**, change the file filter to **"All Files"**, and select `updater_gui.py`.
4. Right-click the new shortcut in your library, select **Properties**, and rename it to "CachyOS Updater (Secure)".
5. Return to **Game Mode**.
6. Select the updater, open the **Controller Settings (Gear Icon > Edit Layout)**, and map your gamepad:
   * Bind your controller's **X Button** to the keyboard **X key** (Closes the app when finished).
   * Bind your controller's **Y Button** to the keyboard **Y key** (Toggles the hidden terminal output).
   * *Note: You will need to use the Steam virtual keyboard (STEAM + X) to enter your password when launching the app.*
