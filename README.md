# Game Mode GUI Updater (Secure Edition)

A sleek, gamepad-friendly graphical interface designed specifically to update CachyOS Handheld inside Steam Game Mode—without compromising Arch Linux security standards.

Traditional desktop package managers (like Shelly or Pamac) often crash Gamescope or freeze completely when asking for a root password via Polkit. This utility bypasses those UI freezes entirely. It acts as a smart virtual keyboard, dynamically injecting your sudo password from a secure memory vault into a true pseudo-terminal, then automatically handles package confirmations and service restarts in the background.

## ✨ Why This Exists (The Handheld Advantage)
* **Strict Security Compliance:** Requires NO `visudo` overrides or permanent `NOPASSWD` backdoors. Your password is held temporarily in RAM, injected into a Controlling TTY session across multiple AUR prompts, and securely wiped upon completion.
* **Gamescope Immunity:** Natively strips Steam's 32-bit `LD_PRELOAD` overlay injections and bypasses the Polkit GUI freeze entirely.
* **Smart Virtual Keyboard:** Automatically answers `[Y/n]` prompts, bypasses Arch News blocks, and hits `0` for service restarts.
* **Gamepad Native:** Includes built-in hotkeys mapped to standard controller inputs (`X` to close, `Y` to toggle the hidden terminal output) for a seamless console-like experience.

## 📦 Prerequisites
This script requires the `python-customtkinter` UI library. Install it via the terminal:
```bash
sudo pacman -S python-customtkinter
```

## 🚀 Installation & Setup

**1. Download the Script**
Save the `updater_gui.py` file to your preferred folder and grant it executable permissions:
```bash
chmod +x ~/updater_gui.py
```

**2. Launch**
You can now run the app directly from your terminal or create a standard `.desktop` shortcut:
```bash
./updater_gui.py
```

## 🎮 Steam Game Mode Setup 
To use this natively on a Legion Go, Steam Deck, or HTPC:

1. Open Steam in Desktop Mode.
2. Click **Games > Add a Non-Steam Game to My Library**.
3. Click **Browse**, select **"All Files"**, and choose `updater_gui.py`.
4. Right-click the new shortcut in your library, select **Properties**, and rename it to "System Updater (Secure)".
5. Return to **Game Mode**.
6. Select the updater, open **Controller Settings (Gear Icon > Edit Layout)**, and map your gamepad:
   * Bind your controller's **X Button** to the keyboard **X key** (Closes the app when finished).
   * Bind your controller's **Y Button** to the keyboard **Y key** (Toggles the terminal output).
   * *Note: You will use the Steam virtual keyboard (STEAM + X) to enter your password when initially launching the app.*

***

## 🔍 Source Code Audit

For complete transparency, the full `updater_gui.py` script is provided below. You can verify exactly how the pseudo-terminal (PTY) and `termios.TIOCSCTTY` flags are utilized to vault your password securely in RAM and wipe it upon completion.

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

        self.title("System Updater")
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
        clean_text = clean_text.replace('\r', '')
        
        self.console.configure(state="normal")
        self.console.insert("end", clean_text)
        self.console.see("end") 
        self.console.configure(state="disabled")

    def authenticate_and_start(self):
        """Vaults the password into memory and locks the UI."""
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
        self.write_to_console("Initializing update...\n")
        
        threading.Thread(target=self.run_update_process, daemon=True).start()

    def run_update_process(self):
        clean_env = os.environ.copy()
        clean_env.pop("LD_PRELOAD", None)
        clean_env["PAGER"] = "cat"
        
        master, slave = pty.openpty()
        
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
                    if "sorry, try again" in clean_lower or "incorrect password" in clean_lower:
                        is_real_error = True
                        
                    if is_real_error:
                        error_detected = True
                        
                    self.after(0, self.write_to_console, data)
                    
                    # 1. Watch for any sudo prompt and inject the password from the vault
                    if "password" in clean_lower and clean_stripped.endswith(":") and self.vault:
                        os.write(master, (self.vault + "\n").encode())
                        self.after(0, self.write_to_console, "\n")
                    
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

        finally:
            # Wipe the password from memory ONLY after the entire update is finished
            self.vault = None

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
```
</details>
