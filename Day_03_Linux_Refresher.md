# Day 3: Linux Refresher, Networking & Process Management
**Date:** 31st October

## 1. Basic Linux Filesystem Hierarchy (FHS)
*   `/bin`: Essential binaries (ls, cp).
*   `/sbin`: System binaries (reboot, ip).
*   `/etc`: Configuration files.
*   `/home`: User directories.
*   `/root`: Root user's home (Security).
*   `/tmp`: Temporary files (Cleared on reboot).
*   `/var`: Variable data (Logs, databases, spool).
*   `/proc`: Virtual filesystem mapping kernel/process info (`/proc/cpuinfo`).
*   `/dev`: Device nodes (`/dev/sda`).

## 2. Basic Command Mastery
*   `mkdir -p dir/subdir`: Make directory (Parents too).
*   `cd -`: Go back to previous directory.
*   `touch filename`: Update timestamp or create empty file.
*   `pwd`: Print Working Directory.
*   `echo "text" > file`: Redirection overwrite.
*   `echo "text" >> file`: Redirection append.
*   `man <command>`: Manual pages.
*   `history`: View last commands.
*   `alias ll="ls -la"`: Create shortcut.

---

## 3. File Management Deep Dive
*   `ls -lah`: List all, human-readable sizes.
*   `cp -r`: Recursive copy for folders.
*   `mv`: Move or Rename.
*   `rm -rf`: Force remove recursive (Dangerous!).
*   `cat`: Concatenate and print.
*   `less`: Scrollable view. (Use `q` to quit).
*   `head -n 5 file`: First 5 lines.
*   `tail -f file`: Follow log updates in real-time.
*   `grep -r "error" /var/log`: Recursive search for text.
*   `find / -name "*.conf" -type f`: Find files named *.conf.
*   `ln -s target link`: Create soft link (Shortcut).

---

## 4. Advanced Permissions (SUID/SGID)
Standard: `rwx` (Read/Write/Execute).
**Octal:** 4(r) + 2(w) + 1(x). `chmod 755 file`.

### The Special bits:
1.  **SUID (Set User ID):** `chmod u+s file`.
    *   Example: `/usr/bin/passwd`.
    *   *Effect:* When executed, the process runs as the **Owner** (root), not the user. This allows normal users to update `/etc/shadow` indirectly.
2.  **SGID (Set Group ID):** `chmod g+s dir`.
    *   *Effect:* Files created in this directory inherit the group of the directory, not the user's primary group. Used for shared folders.
3.  **Sticky Bit:** `chmod +t dir`.
    *   Example: `/tmp`.
    *   *Effect:* Users can only delete their **own** files in a shared directory.

---

## 5. Process Management & Signals
Every running program is a process.
*   `ps aux`: Snapshot of all processes.
*   `top` / `htop`: Real-time Resource usage.
*   **Load Average:** Average number of processes waiting for CPU (1 min, 5 min, 15 min).
    *   *Rule of Thumb:* Load > Cores = Constraint.

### Signals (IPC)
*   **SIGTERM (15):** Polite kill. "Please stop." (Default `kill PID`).
*   **SIGKILL (9):** Force kill. Kernel removes process immediately. Can cause corruption.
*   **SIGHUP (1):** Hang up. Often used to "Reload Configuration" without restarting.

### Background Jobs
*   `command &`: Run in background.
*   `jobs`: List background jobs.
*   `fg %1`: Bring job 1 to foreground.
*   `bg %1`: Resume stopped job in background.

---

## 6. Networking Power Tools
*   `ip addr`: Show interfaces/IPs (Modern `ifconfig`).
*   `ss -tulpn`: Show listening ports (Modern `netstat`).
*   `ping -c 4 google.com`: Connectivity test.
*   `curl -vI google.com`: Verbose header check.
*   `wget`: Download file.
*   `nslookup / dig`: DNS query debugging.
*   `nc -zv host port`: Netcat. Check if port is open.
*   `tcpdump -i eth0 port 80`: Packet capture (Wireshark for CLI).

---

## 7. The Boot Process
1.  **BIOS/UEFI:** POST.
2.  **Bootloader (GRUB):** Loads Kernel.
3.  **Kernel:** Mounts root filesystem, starts PID 1.
4.  **Init (Systemd/SysVinit):** Parents all processes.
    *   `systemctl start service`
    *   `systemctl enable service` (Start on boot).
    *   `journalctl -u service`: View logs.

---

## 8. Writing a Hello World Program in Linux Shell

### Step 1: Create the file
```bash
touch hello.sh
```

### Step 2: Add executable permissions
```bash
chmod +x hello.sh
```

### Step 3: Write the script
Open with an editor (`nano hello.sh` or `vim hello.sh`) and write:
```bash
#!/bin/bash
# Shebang line tells kernel which interpreter to use

# Variable
who="World"

# Command
echo "Hello $who from Linux Shell!"
echo "Current Date: $(date)"
```

### Step 4: Run it
```bash
./hello.sh
```
