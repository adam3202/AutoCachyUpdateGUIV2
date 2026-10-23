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

(HERE IS THE ENTIRE CODE BASE FOR THE SCRIPT SO YOU CAN PERSONALLY AUDIT IT YOURSELF)

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

ANSI_ESCAPE = re.compile(r'\x1B\[[0-?]*[ -/]*[@-~]|\x1B\].*?(?:\x07|\x1B\\)')
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

    def authenticate_and_start(self):
        """Securely vaults the password into memory and locks the UI."""
        password = self.password_entry.get()
        if not password:
            self.status_label.configure(text="Please enter your password.", text_color="#e63946")
            return

        self.vault = password
        self.password_entry.delete(0, 'end')
        self.password_entry.configure(state="disabled")
        self.action_btn.configure(state="disabled", text="Updating...")
        self.start_update_thread()

    def start_update_thread(self):
        self.status_label.configure(text="Updating system... Please wait.", text_color="white")
        self.progress.configure(mode="indeterminate")
        self.progress.start()
        
        self.console.configure(state="normal")
        self.console.delete("0.0", "end")
        self.console.configure(state="disabled")
        self.write_to_console("Initializing cachy-update...\n")
        
        threading.Thread(target=self.run_update_process, daemon=True).start()

    def run_update_process(self):
        clean_env = os.environ.copy()
        clean_env.pop("LD_PRELOAD", None)
        clean_env["PAGER"] = "cat"
        
        master, slave = pty.openpty()
        
        # --- NEW: Forces Linux to recognize the pseudo-terminal as a true Controlling TTY ---
        def set_ctty():
            os.setsid()
            try:
                fcntl.ioctl(0, termios.TIOCSCTTY, 0)
            except Exception:
                pass
        
        try:
            process = subprocess.Popen("cachy-update", shell=True, stdin=slave, stdout=slave, stderr=subprocess.STDOUT, close_fds=True, env=clean_env, preexec_fn=set_ctty)
            os.close(slave) 
            
            error_detected = False
            terminal_buffer = ""
            
            while True:
                try:
                    data = os.read(master, 1024).decode('utf-8', errors='replace')
                    if not data:
                        break
                        
                    clean_logic_data = ANSI_ESCAPE.sub('', data)
                    clean_logic_data = SYSTEMD_TAGS.sub('', clean_logic_data)
                    
                    clean_lower = clean_logic_data.lower()
                    clean_stripped = clean_logic_data.strip()
                    
                    terminal_buffer += clean_lower
                    if len(terminal_buffer) > 50000:
                        terminal_buffer = terminal_buffer[-50000:]
                        
                    is_real_error = False
                    if "error has occurred" in clean_lower or "aborted" in clean_lower or "timeout" in clean_lower:
                        is_real_error = True
                    if "error:" in clean_lower and "ld.so" not in clean_lower:
                        is_real_error = True
                    # Catch wrong passwords gracefully
                    if "sorry, try again" in clean_lower or "incorrect password" in clean_lower:
                        is_real_error = True
                        
                    if is_real_error:
                        error_detected = True
                        
                    self.after(0, self.write_to_console, data)
                    
                    # --- THE UPGRADED VIRTUAL KEYBOARD ---
                    
                    # 1. Watch for the sudo prompt, inject the password, and instantly wipe it from memory
                    if "password" in clean_lower and clean_stripped.endswith(":") and self.vault:
                        os.write(master, (self.vault + "\n").encode())
                        self.vault = None 
                        self.after(0, self.write_to_console, "\n") # Force a clean line break visually
                    
                    elif "[y/n]" in clean_lower:
                        os.write(master, b"y\n")
                        
                    elif "default=" in clean_lower and clean_stripped.endswith((":", "?", ">")):
                        os.write(master, b"\n")
                        
                    elif "restart them all" in terminal_buffer and clean_stripped.endswith((":", "?", ">")):
                        os.write(master, b"0\n")
                        terminal_buffer = "" 
                        
                    elif 'press "enter" to proceed' in clean_lower:
                        os.write(master, b"\n")
                        
                    elif 'press "enter" to quit' in clean_lower:
                        os.write(master, b"\n")
                        
                except OSError:
                    break
                    
            process.wait()
            
            if process.returncode == 0 and not error_detected:
                self.after(0, self.finish_update, True)
            else:
                self.after(0, self.finish_update, False)
                
        except Exception as e:
            self.after(0, self.write_to_console, f"\n\n[ERROR] {str(e)}\n")
            self.after(0, self.finish_update, False)

    def finish_update(self, success):
        self.progress.stop()
        self.progress.configure(mode="determinate")
        self.progress.set(1)
        self.action_btn.configure(text="Close", state="normal", command=self.destroy, fg_color="#4a4a4a", hover_color="#333333")
        
        if success:
            self.status_label.configure(text="System is fully updated! Press X to close.", text_color="#32a852")
            self.write_to_console("\n\n[SUCCESS] System is fully updated!\n")
        else:
            self.status_label.configure(text="Update finished with errors. Check terminal.", text_color="#e63946")
            self.write_to_console("\n\n[WARNING] Process finished, but errors were detected.\n")
            if not self.console_visible:
                self.toggle_terminal()

if __name__ == "__main__":
    app = CachyUpdaterApp()
    app.mainloop()
