---

## 📌 **What is tmux?**

**tmux** (Terminal MUltipleXer) is a powerful tool that allows you to manage multiple terminal sessions within a single terminal window or remote session. It helps you keep your terminal organized, lets sessions persist even if you disconnect, and improves productivity when working with command-line interfaces.

---

## 🛠 **How to Install tmux**

Open your terminal and run the command specific to your operating system:

- **Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install tmux -y
```

- **Arch Linux:**
```bash
sudo pacman -S tmux
```

- **Fedora:**
```bash
sudo dnf install tmux
```

- **macOS (Homebrew):**
```bash
brew install tmux
```
---

## 🚀 **Basic Usage**

- **Start a new tmux session:**
```bash
tmux new -s mysession
```

- **Detach from a session (leave running in background):**  
Press `Ctrl + b`, then `d`

- **List existing sessions:**
```bash
tmux ls
```

- **Attach to a session:**
```bash
tmux attach -t mysession
```

- **Split terminal window into panes:**
  - Horizontally: `Ctrl + b`, then `%`
  - Vertically: `Ctrl + b`, then `"`

- **Navigate panes:**  
`Ctrl + b`, then arrow keys (`← ↑ → ↓`)

- **Close current pane:**  
Type `exit` or press `Ctrl + d`

- **Kill a tmux session completely:**
```bash
tmux kill-session -t mysession
```

---

## 🌟 **Why use tmux?**

- **Persistent sessions:**  
  Sessions keep running, even if your SSH connection drops or your terminal closes.

- **Efficient multitasking:**  
  Manage multiple terminals or processes simultaneously without opening multiple terminal windows.

- **Improved workflow:**  
  Easily organize and switch contexts within your terminal.

---

## 🔗 **Further Reading**

- [Official tmux GitHub Wiki](https://github.com/tmux/tmux/wiki)
- [tmux Cheat Sheet](https://tmuxcheatsheet.com/)

---
