# CachyOS GUI Updater (Secure Edition)

A sleek, automated, dark-mode graphical interface for CachyOS system updates. Built with Python and CustomTkinter, this utility automates the terminal-based `cachy-update` process while maintaining 100% strict Arch Linux security compliance.

Designed for both traditional desktop environments and handheld gaming PCs (like the Lenovo Legion Go) running Steam Game Mode. It acts as a smart virtual keyboard, dynamically injecting your sudo password from a secure memory vault before automatically handling package confirmations, service restarts, and AUR reviews in the background.

## ✨ Features
* **Strict Security Compliance:** Requires NO `visudo` overrides or permanent `NOPASSWD` backdoors. Your password is held temporarily in RAM, dynamically injected into a true Controlling TTY session across multiple AUR prompts, and securely wiped upon completion.
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

***

## 🔍 Source Code Audit

For complete transparency, the full `updater_gui.py` script is provided below. You can verify exactly how the pseudo-terminal (PTY) and `termios.TIOCSCTTY` flags are utilized to vault your password securely in RAM and dynamically handle standard input.

<details>
<summary><b>Click to expand and view the full Python source code</b></summary>

```python
#!/usr/bin/env python
import customtkinter as ctk
import subprocess
import threading
import pty
import os
import re
import fcntl
import termios

# Configure the window theme
ctk.set_appearance_mode("Dark")
ctk.set_default_color_theme("blue")

ANSI_ESCAPE = re.compile(r'\x1B\[[0-?]*[ -/]*[@-~]\vert{}\x1B\].*?(?:\x07|\x1B\\)')
SYSTEMD_TAGS = re.compile(r'\]?\d{4};(?:start|end)=[a-f0-9]+(?:;[a-z]+=[^;\n]+)*')

class CachyUpdaterApp(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.title("CachyOS System Updater")
        self.geometry("650x480")
        self.resizable(False, False)
        self.vault = None

        self.label = ctk.CTkLabel(self, text="System Update Manager", font=ctk.CTkFont(size=22, weight="bold"))
        self.label.pack(pady=(20, 5))

        self.status_label = ctk.CTkLabel(self, text="Ready to update.", font=ctk.CTkFont(size=14), text_color="gray")
        self.status_label.pack(pady=(0, 10))

        self.progress = ctk.CTkProgressBar(self, width=450, height=15)
        self.progress.pack(pady=(5, 15))
        self.progress.set(0)
        
        self.password_entry = ctk.CTkEntry(self, width=250, placeholder_text="Enter sudo password...", show="*")
        self.password_entry.pack(pady=(0, 10))

        self.btn_frame = ctk.CTkFrame(self, fg_color="transparent")
        self.btn_frame.pack(pady=(5, 10))

        self.action_btn = ctk.CTkButton(self.btn_frame, text="Start Update", height=40, font=ctk.CTkFont(size=14, weight="bold"), command=self.authenticate_and_start)
        self.action_btn.grid(row=0, column=0)

        self.console_visible = False
        self.toggle_btn = ctk.CTkButton(self, text="▼ Show Terminal Output", command=self.toggle_terminal, fg_color="transparent", text_color="gray", hover_color="#2b2b2b")
        self.toggle_btn.pack(pady=(10, 5))

        self.console = ctk.CTkTextbox(self, width=600, height=200, state="disabled", font=ctk.CTkFont(family="monospace", size=12), fg_color="#1e1e1e")

        # HOTKEYS
        self.bind("<Key-x>", self.safe_close)
        self.bind("<Key-X>", self.safe_close)
        self.bind("<Escape>", self.safe_close)
        self.bind("<Key-y>", self.safe_toggle)
        self.bind("<Key-Y>", self.safe_toggle)
        self.bind("<Return>", lambda event: self.authenticate_and_start())

    def toggle_terminal(self):
        if self.console_visible:
            self.console.pack_forget()
            self.toggle_btn.configure(text="▼ Show Terminal Output")
            self.console_visible = False
        else:
            self.console.pack(padx=20, pady=5)
            self.toggle_btn.configure(text="▲ Hide Terminal Output")
            self.console_visible = True

    def safe_close(self, event=None):
        if self.action_btn.cget("text") == "Close":
            self.destroy()

    def safe_toggle(self, event=None):
        self.toggle_terminal()

    def write_to_console(self, text):
        clean_text = ANSI_ESCAPE.sub('', text)
        clean_text = SYSTEMD_TAGS.sub('', clean_text)
        clean_text = clean_text.replace('==> Press "enter" to quit', '')
        
        self.console.configure(state="normal")
        self.console.insert("end", clean_text)
        self.console.see("end") 
        self.console.configure(state="disabled")

    def authenticate_
