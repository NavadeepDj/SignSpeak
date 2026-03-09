# Docker + Camera Setup Guide (Windows with USB Passthrough)

> **Complete documentation of setting up SignSpeak in Docker with webcam access on Windows**
> 
> This guide documents the entire journey, including problems faced and solutions found.

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Core Problem](#2-the-core-problem)
3. [Solution Architecture](#3-solution-architecture)
4. [Step-by-Step Setup](#4-step-by-step-setup)
5. [Problems Encountered & Solutions](#5-problems-encountered--solutions)
6. [Quick Reference Commands](#6-quick-reference-commands)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Overview

### What We Wanted
Run the SignSpeak Indian Sign Language recognition app in Docker on Windows, with working webcam access.

### Why It's Complicated
Docker Desktop on Windows runs Linux containers inside a WSL2 virtual machine. Windows webcams use DirectShow/MSMF APIs, but Linux containers expect V4L2 (`/dev/video0`). These are incompatible.

### The Solution
Use **usbipd-win** to forward the USB webcam from Windows → WSL2 (Ubuntu) → Docker container.

---

## 2. The Core Problem

### Initial Attempt: Direct Docker Run

```powershell
docker-compose up --build
```

**Error:**
```
error gathering device information while adding custom device "/dev/video0": no such file or directory
```

**Why it failed:** `/dev/video0` is a Linux device path. Windows doesn't have this — it uses different camera APIs.

### The Camera Access Flow

```
❌ DOESN'T WORK:
Windows Camera (DirectShow) ──X──► Docker Container (expects V4L2)

✅ WORKS:
Windows USB Camera 
    │
    ▼ (usbipd bind + attach)
WSL2 Ubuntu (/dev/video0)
    │
    ▼ (docker --device)
Docker Container (V4L2 works!)
```

---

## 3. Solution Architecture

### Components Needed

| Component | Purpose |
|-----------|---------|
| **Docker Desktop** | Runs Linux containers via WSL2 |
| **WSL2 + Ubuntu** | Full Linux environment that can receive USB devices |
| **usbipd-win** | Windows tool to forward USB devices to WSL |
| **usbutils** | Linux tools to verify USB devices |

### Why docker-desktop WSL Distro Doesn't Work

We tried using the `docker-desktop` WSL distro first:

```sh
# In docker-desktop WSL
ls /dev/bus/usb
# Result: ls: /dev/bus: No such file or directory

lsusb
# Result: unable to initialize libusb: -99
```

**Problem:** `docker-desktop` is a minimal Alpine Linux VM specifically for running Docker. It:
- Has no USB subsystem (`/dev/bus/usb` doesn't exist)
- Uses `apk` not `apt` (Alpine vs Debian/Ubuntu)
- Has no `sudo` command
- Cannot receive USB devices via usbipd

**Solution:** Install a full Ubuntu distro in WSL2.

---

## 4. Step-by-Step Setup

### Step 1: Install Ubuntu in WSL2

Open **PowerShell**:

```powershell
wsl --install -d Ubuntu
```

- Wait for download (~500MB)
- Create username and password when prompted
- This runs alongside docker-desktop (doesn't replace it)

### Step 2: Install USB Tools in Ubuntu

Open **Ubuntu** terminal:

```bash
sudo apt update
sudo apt install usbutils hwdata
```

> **Note:** We tried `linux-tools-generic` but got dependency errors:
> ```
> linux-tools-generic : Depends: linux-tools-6.8.0-100-generic but it is not installable
> ```
> This is fine — newer usbipd-win doesn't need it.

### Step 3: Install usbipd-win on Windows

Open **PowerShell (Admin)**:

```powershell
winget install usbipd
```

### Step 4: Find Your Webcam's Bus ID

```powershell
usbipd list
```

Output example:
```
BUSID  VID:PID    DEVICE                                          STATE
2-5    06cb:0124  Synaptics UWP WBDI                              Not shared
2-9    04f2:b765  HP True Vision 5MP Camera, Camera DFU Device    Not shared
2-10   0bda:b85c  Realtek Wireless Bluetooth Adapter              Not shared
```

In this case, the webcam is **BUSID 2-9**.

### Step 5: Bind and Attach the Camera

```powershell
# Bind (one-time, makes device shareable)
usbipd bind --busid 2-9

# Attach to Ubuntu (do this every time after restart)
usbipd attach --wsl=Ubuntu --busid 2-9 --force
```

#### Error: "Device busy"
```
WSL usbip: error: Attach Request for 2-9 failed - Device busy (exported)
usbipd: warning: The device appears to be used by Windows
```

**Solution:** Close any app using the camera (browser, Zoom, Teams, Camera app), then use `--force`:
```powershell
usbipd attach --wsl=Ubuntu --busid 2-9 --force
```

### Step 6: Verify Camera in Ubuntu

Open **Ubuntu** terminal:

```bash
ls /dev/video*
```

Expected output:
```
/dev/video0  /dev/video1
```

If you see video devices, the camera is now in WSL! 🎉

### Step 7: Enable Docker Integration with Ubuntu

1. Open **Docker Desktop**
2. Go to **Settings** → **Resources** → **WSL Integration**
3. Enable **Ubuntu** toggle
4. Click **Apply & Restart**

### Step 8: Run Docker from Ubuntu

```bash
cd /mnt/c/Users/navad/sign-to-text-and-speech

sudo docker-compose -f docker-compose.linux.yml up --build
```

#### Permission Error Fix
```
permission denied while trying to connect to the Docker daemon socket
```

**Solution:** Use `sudo` or add yourself to docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Step 9: Handle Camera Timeout Issues

If you see:
```
[ WARN:1@163.882] global cap_v4l.cpp:1048 tryIoctl VIDEOIO(V4L2:/dev/video0): select() timeout.
[WARNING] Camera test frame 1/5 failed
```

**Solution:** Try `/dev/video1` instead (video0 might be metadata-only):

```bash
sudo docker run --rm -it --privileged \
  -v /dev:/dev \
  -p 5000:5000 \
  -e CAMERA_INDEX=1 \
  -v $(pwd)/models:/app/models \
  sign-to-text-and-speech-signspeak
```

---

## 5. Problems Encountered & Solutions

### Problem 1: `version` attribute obsolete in docker-compose.yml

**Error:**
```
docker-compose.yml: the attribute `version` is obsolete
```

**Solution:** Remove the `version: '3.8'` line from docker-compose.yml (no longer needed in modern Docker Compose).

---

### Problem 2: Docker Desktop not running

**Error:**
```
error during connect: open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.
```

**Solution:** Start Docker Desktop application.

---

### Problem 3: Package not found in Dockerfile

**Error:**
```
E: Package 'libgl1-mesa-glx' has no installation candidate
```

**Solution:** Replace with `libgl1` in Dockerfile (package renamed in newer Debian):
```dockerfile
RUN apt-get install -y libgl1 libglib2.0-0 ...
```

---

### Problem 4: Python version mismatch

**Error:**
```
ERROR: No matching distribution found for contourpy==1.3.3
```

**Cause:** `contourpy==1.3.3` requires Python 3.11+, but Dockerfile used Python 3.10.

**Solution:** Change Dockerfile base image:
```dockerfile
FROM python:3.11-slim
```

---

### Problem 5: Windows-only TensorFlow package

**Error:**
```
ERROR: No matching distribution found for tensorflow-intel==2.15.0
```

**Cause:** `tensorflow-intel` is Windows-only, doesn't exist on Linux.

**Solution:** Remove from requirements.txt:
```
# tensorflow-intel is Windows-only; omit for Docker/Linux
```

---

### Problem 6: /dev/video0 not found (Windows Docker)

**Error:**
```
error gathering device information while adding custom device "/dev/video0": no such file or directory
```

**Cause:** Windows Docker can't directly access Windows cameras.

**Solution:** Use the full USB passthrough method documented in this guide.

---

### Problem 7: docker-desktop has no USB support

**Commands tried:**
```sh
# In docker-desktop WSL
ls /dev/bus/usb          # No such file or directory
lsusb                    # unable to initialize libusb: -99
sudo apt update          # sudo: not found
apt                      # apt: not found (uses apk instead)
```

**Cause:** docker-desktop is minimal Alpine, not a full Linux.

**Solution:** Install Ubuntu: `wsl --install -d Ubuntu`

---

### Problem 8: linux-tools-generic dependency errors

**Error:**
```
linux-tools-generic : Depends: linux-tools-6.8.0-100-generic but it is not installable
```

**Cause:** WSL2 kernel doesn't match Ubuntu's expected kernel tools.

**Solution:** Skip it — modern usbipd-win handles everything from Windows side. Just install:
```bash
sudo apt install usbutils hwdata
```

---

### Problem 9: usbipd attaching to wrong WSL distro

**Output:**
```
usbipd: info: Using WSL distribution 'docker-desktop' to attach
```

**Solution:** Specify the distro explicitly:
```powershell
usbipd attach --wsl=Ubuntu --busid 2-9 --force
```

---

### Problem 10: Camera timeout in Docker

**Error:**
```
VIDEOIO(V4L2:/dev/video0): select() timeout.
[WARNING] Camera test frame 1/5 failed
```

**Cause:** USB camera might have multiple video devices (video0 = metadata, video1 = actual stream).

**Solution:** 
1. Run with privileged mode and mount all `/dev`
2. Try CAMERA_INDEX=1

```bash
sudo docker run --rm -it --privileged \
  -v /dev:/dev \
  -p 5000:5000 \
  -e CAMERA_INDEX=1 \
  -v $(pwd)/models:/app/models \
  sign-to-text-and-speech-signspeak
```

---

## 6. Quick Reference Commands

### Windows PowerShell (Admin)

```powershell
# List USB devices
usbipd list

# Bind camera (one-time)
usbipd bind --busid 2-9

# Attach to Ubuntu (after each restart)
usbipd attach --wsl=Ubuntu --busid 2-9 --force

# Check WSL distros
wsl --list --verbose
```

### Ubuntu Terminal

```bash
# Check if camera is visible
ls /dev/video*
lsusb

# Navigate to project
cd /mnt/c/Users/navad/sign-to-text-and-speech

# Run with docker-compose
sudo docker-compose -f docker-compose.linux.yml up --build

# Run with specific camera index
sudo docker run --rm -it --privileged \
  -v /dev:/dev \
  -p 5000:5000 \
  -e CAMERA_INDEX=1 \
  -v $(pwd)/models:/app/models \
  sign-to-text-and-speech-signspeak
```

---

## 7. Troubleshooting

### Camera Disappears After Windows Restart

USB attachment doesn't persist. Re-run:
```powershell
usbipd attach --wsl=Ubuntu --busid 2-9 --force
```

### Camera Works in Ubuntu but Not in Docker

Check Docker WSL integration:
1. Docker Desktop → Settings → Resources → WSL Integration
2. Make sure Ubuntu is enabled

### "Permission denied" for Docker commands

Either use `sudo` or add to docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Still Getting Timeouts

Try these in order:
1. Detach and re-attach the camera via usbipd
2. Try CAMERA_INDEX=1 instead of 0
3. Run a quick test in Ubuntu first (outside Docker):
   ```bash
   sudo apt install ffmpeg
   ffmpeg -f v4l2 -i /dev/video0 -frames:v 1 test.jpg
   ```

---

## Summary Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  WINDOWS                                                            │
│                                                                     │
│  1. winget install usbipd                                          │
│  2. usbipd list                    → Find BUSID (e.g., 2-9)        │
│  3. usbipd bind --busid 2-9        → Make shareable                │
│  4. usbipd attach --wsl=Ubuntu ... → Forward to Ubuntu             │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  WSL2 Ubuntu                                                   │ │
│  │                                                                │ │
│  │  5. ls /dev/video*              → Verify camera visible        │ │
│  │  6. cd /mnt/c/.../project                                      │ │
│  │  7. sudo docker-compose -f docker-compose.linux.yml up --build │ │
│  │                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │  Docker Container                                        │  │ │
│  │  │                                                          │  │ │
│  │  │  8. App runs with /dev/video0 access                     │  │ │
│  │  │  9. Open http://localhost:5000                           │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Questions & Answers from Setup Session

### Q: Why is my built-in camera a USB device?

**A:** Modern laptops connect internal webcams via the internal USB bus, even though there's no external USB port. The camera controller chip communicates over USB protocol internally. This is why `usbipd list` shows your "integrated" camera as a USB device.

### Q: What's the difference between docker-desktop and Ubuntu WSL?

**A:** 
- `docker-desktop`: Minimal Alpine Linux VM that Docker Desktop uses internally. Has `apk` (not `apt`), no `sudo`, no USB subsystem. Not meant for direct use.
- `Ubuntu`: Full Linux distribution with complete tooling. Supports USB passthrough, has `apt`, `sudo`, and everything you'd expect.

### Q: Why do I need to re-attach the camera after restart?

**A:** USB device forwarding via usbipd is not persistent by design. Each Windows restart resets the attachment. The `bind` is persistent (one-time), but `attach` must be redone.

### Q: Why use `--force` with usbipd attach?

**A:** If any Windows application has the camera open (even in background), the device is "busy". `--force` disconnects it from Windows and forwards to WSL.

---

*Document created: February 2026*
*Last updated: February 6, 2026*
