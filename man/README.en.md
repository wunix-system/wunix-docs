[ 🇷🇺 Русский ](README.ru.md) | [ 🇬🇧 English ](README.en.md)

# Wunix Manual Pages Guide

Wunix features a standard UNIX offline documentation subsystem — manual pages (**man pages**). All userland utilities are documented using the traditional roff/groff format in both Russian and English.

---

## 1. Using the `man` Command

Inside the interactive `wsh` shell, the `man` tool provides syntax, option details, and examples for all available commands:

```bash
# View documentation for a command:
man ls
man ping
man mount

# Explicitly specify manual section:
man 1 cat
man 8 fdisk
```

The display language is automatically determined by the system locale:
- Russian locale (`ru`) loads pages from `/usr/share/man/ru/`.
- English locale (`en`) loads pages from `/usr/share/man/en/`.

---

## 2. Manual Sections

Wunix organizes man pages into standard UNIX sections:

- **Section 1 (`man1`)**: User commands and interactive shells.
- **Section 8 (`man8`)**: System administration and hardware control utilities.

---

## 3. Section 1 Utilities (`man1`)

| Utility | Description |
|:---|:---|
| [`cat(1)`](man1/cat.1) | Concatenates and prints file contents to standard output. |
| [`chmod(1)`](man1/chmod.1) | Changes file mode bits (octal and symbolic permission modes). |
| [`chown(1)`](man1/chown.1) | Changes user owner and group of files. |
| [`clear(1)`](man1/clear.1) | Clears terminal screen buffer and resets cursor position. |
| [`curl(1)`](man1/curl.1) | Performs HTTP requests and fetches web content over the network. |
| [`date(1)`](man1/date.1) | Displays and formats the current system date and time. |
| [`echo(1)`](man1/echo.1) | Prints command-line argument strings to the terminal. |
| [`fbset(1)`](man1/fbset.1) | Displays framebuffer resolution and color mode settings. |
| [`free(1)`](man1/free.1) | Displays total, used, and available physical memory. |
| [`host(1)`](man1/host.1) | DNS lookup utility for resolving host names and IP addresses (nslookup). |
| [`hwclock(1)`](man1/hwclock.1) | Reads and synchronizes hardware CMOS RTC time. |
| [`id(1)`](man1/id.1) | Prints user and group information (UID/GID and supplementary groups). |
| [`ln(1)`](man1/ln.1) | Creates symbolic links within the VFS hierarchy. |
| [`login(1)`](man1/login.1) | Initializes a user session and authenticates user credentials. |
| [`ls(1)`](man1/ls.1) | Lists directory contents and file metadata. |
| [`lsdev(1)`](man1/lsdev.1) | Lists registered character and block devices. |
| [`lsmod(1)`](man1/lsmod.1) | Displays loaded kernel drivers and hardware modules. |
| [`lspci(1)`](man1/lspci.1) | Scans and lists devices on the PCI bus. |
| [`lshw(1)`](man1/lshw.1) | Generates comprehensive CPU, memory, and bus hardware inventory. |
| [`man(1)`](man1/man.1) | Interactive manual pages viewer. |
| [`mkdir(1)`](man1/mkdir.1) | Creates directories in the VFS tree. |
| [`modinfo(1)`](man1/modinfo.1) | Displays driver metadata and version attributes. |
| [`passwd(1)`](man1/passwd.1) | Updates user authentication password in /etc/shadow. |
| [`pwd(1)`](man1/pwd.1) | Prints canonical path of the current working directory. |
| [`readlink(1)`](man1/readlink.1) | Resolves and prints symbolic link targets. |
| [`rm(1)`](man1/rm.1) | Removes files from the filesystem. |
| [`rmdir(1)`](man1/rmdir.1) | Deletes empty directories. |
| [`su(1)`](man1/su.1) | Runs a shell with substitute user and group ID (default root). |
| [`system(1)`](man1/system.1) | Manages and supervises system services and daemons. |
| [`time(1)`](man1/time.1) | Measures execution duration of specified commands. |
| [`timedatectl(1)`](man1/timedatectl.1) | Configures time zones and system clock parameters. |
| [`top(1)`](man1/top.1) | Dynamic process and memory activity monitor. |
| [`touch(1)`](man1/touch.1) | Creates empty files or touches modification timestamps. |
| [`uname(1)`](man1/uname.1) | Prints operating system name, kernel release, and CPU architecture. |
| [`whoami(1)`](man1/whoami.1) | Prints current effective user name (EUID). |
| [`wsh(1)`](man1/wsh.1) | Wunix interactive command shell. |

---

## 4. Section 8 System Administration Utilities (`man8`)

| Utility | Description |
|:---|:---|
| [`fdisk(8)`](man8/fdisk.8) | Inspects and edits MBR partition tables on block storage. |
| [`init(8)`](man8/init.8) | Kernel root supervisor process (PID 1). |
| [`ip(8)`](man8/ip.8) | Configures network interfaces, IP addresses, and default routes. |
| [`lsblk(8)`](man8/lsblk.8) | Lists block storage devices and disk partitions. |
| [`mount(8)`](man8/mount.8) | Mounts block filesystems (FAT32, RamFS) onto the VFS. |
| [`ping(8)`](man8/ping.8) | Tests network reachability using ICMP Echo Request packets. |
| [`reboot(8)`](man8/reboot.8) | Flushes dirty buffers and restarts the system. |
| [`shutdown(8)`](man8/shutdown.8) | Shuts down the operating system and powers down via ACPI. |
| [`ss(8)`](man8/ss.8) | Displays active TCP/UDP socket connections and listening ports. |
| [`sudo(8)`](man8/sudo.8) | Executes a command with superuser (root) privileges. |
| [`sudoadd(8)`](man8/sudoadd.8) | Grants or revokes administrative sudo privileges (wheel group). |
| [`umount(8)`](man8/umount.8) | Unmounts filesystems from the VFS. |
| [`useradd(8)`](man8/useradd.8) | Creates a new user account and home directory. |
| [`userdel(8)`](man8/userdel.8) | Deletes a user account and associated user files. |
