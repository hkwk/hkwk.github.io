---
author: HKL
categories:
- Default
date: "2026-04-23T19:08:00+08:00"
slug: build-and-run-mmdvm-on-windows
draft: false
tags:
- Programming
- Radio
title: Building and Running MMDVMHost + DMRGateway (2026) on Windows 
---

### Native MSYS2 Build + Modern BrandMeister Setup Guide


## Introduction

If you are running a personal DMR hotspot on Windows, modern versions of **MMDVMHost (2026)** no longer behave reliably with direct BrandMeister connections.

The recommended architecture is now:

```text
Radio / Hotspot
      ↓
  MMDVMHost
      ↓
  DMRGateway
      ↓
 BrandMeister
````

This guide combines:

* **Windows Native Build Guide**
* **Modern MMDVMHost + DMRGateway Setup Guide**

and shows how to:

✅ Compile from source on Windows
✅ Build standalone `.exe` binaries
✅ Connect successfully to BrandMeister
✅ Use OpenGD77 hotspot hardware
✅ Run latest 2026 versions stably



---

# Why Use DMRGateway in 2026?

Older MMDVMHost versions could connect directly to BrandMeister.

Modern builds increasingly expect:

* MMDVMHost handles **RF / modem**
* DMRGateway handles **network login / TG routing**

So instead of:

```text
MMDVMHost → BrandMeister
```

Use:

```text
MMDVMHost → DMRGateway → BrandMeister
```

---

# Part 1 – Build on Windows (MSYS2 UCRT64)

---

## Step 1 – Install MSYS2

Download:

👉 [https://www.msys2.org/](https://www.msys2.org/)

Install 64-bit version.

Then open:

```text
MSYS2 UCRT64 Shell
```

---

## Step 2 – Install Required Packages

```bash
pacman -Syu
pacman -Su

pacman -S --needed base-devel mingw-w64-ucrt-x86_64-toolchain git
pacman -S mingw-w64-ucrt-x86_64-libmosquitto
pacman -S mingw-w64-ucrt-x86_64-nlohmann-json
pacman -S mingw-w64-ucrt-x86_64-openssl
pacman -S mingw-w64-ucrt-x86_64-libserialport
```

---

## Step 3 – Clone Source Code

```bash
cd /d/workspace

git clone https://github.com/g4klx/MMDVMHost.git
git clone https://github.com/g4klx/DMRGateway.git
```

---

## Step 4 – Patch Makefiles for Windows

Both projects need:

* remove `-lutil`
* add Winsock libraries

---

### MMDVMHost

```bash
cd /d/workspace/MMDVMHost

sed -i 's/-lutil //' Makefile
sed -i 's/LIBS    = -lpthread -lmosquitto/LIBS    = -lpthread -lmosquitto -lws2_32 -liphlpapi/' Makefile
```

---

### DMRGateway

```bash
cd /d/workspace/DMRGateway

sed -i 's/-lutil //' Makefile
sed -i 's/LIBS    = -lpthread -lmosquitto/LIBS    = -lpthread -lmosquitto -lws2_32 -liphlpapi/' Makefile
```



---

## Step 5 – Compile

```bash
cd /d/workspace/MMDVMHost
make clean
make -j$(nproc)

cd /d/workspace/DMRGateway
make clean
make -j$(nproc)
```

---

# Part 2 – Make Standalone Windows EXE

Copy runtime DLLs beside each EXE:

```bash
cp /ucrt64/bin/libgcc_s_seh-1.dll .
cp /ucrt64/bin/libstdc++-6.dll .
cp /ucrt64/bin/libwinpthread-1.dll .
cp /ucrt64/bin/libmosquitto.dll .
cp /ucrt64/bin/libssl-3-x64.dll .
cp /ucrt64/bin/libcrypto-3-x64.dll .
```

Now they run in:

* PowerShell
* CMD
* Double click
* Other Windows PCs

---

# Part 3 – Install MQTT Broker

Modern builds require MQTT support.

Install Mosquitto:

👉 [https://mosquitto.org/download/](https://mosquitto.org/download/)

Run:

```powershell
mosquitto.exe
```

---

# Part 4 – Configure MMDVMHost

## MMDVMHost.ini

```ini
[General]
Callsign=BG7KUH
Id=4615604
Duplex=0

[Modem]
Protocol=uart
UARTPort=\\.\COM8
UARTSpeed=115200

[DMR]
Enable=1
Id=461560401
ColorCode=1

[DMR Network]
Enable=1
GatewayAddress=127.0.0.1
GatewayPort=62031
LocalAddress=127.0.0.1
LocalPort=62032
Slot1=0
Slot2=1
```

---

# Part 5 – Configure DMRGateway

## DMRGateway.ini

```ini
[General]
Callsign=BG7KUH
Id=4615604

[DMR Network 1]
Enabled=1
Name=BM
Address=4602.master.brandmeister.network
Port=62031
Password=YOUR_HOTSPOT_PASSWORD
Location=1
Debug=0

TGRewrite=2,9,2,9,1
TGRewrite=2,46007,2,46007,1
TGRewrite=2,46020,2,46020,1

PassAllTG=2
PassAllPC=1
PassAllPC=2
```

---

# Part 6 – Start Programs

Launch in this order:

```powershell
DMRGateway.exe DMRGateway.ini
MMDVMHost.exe MMDVMHost.ini
```

---

# Expected Output

```text
DMR, Sending authorisation
DMR, Logged into the master successfully
```

---

# Recommended Single Hotspot Settings (China)

## Always-On Talkgroups

| TG    | Use            |
| ----- | -------------- |
| 9     | Local          |
| 46007 | China National |
| 46020 | Guangdong      |
| 46001 | China Regional |

---

# OpenGD77 Users

Use:

```ini
Duplex=0
Slot2=1
```

Because OpenGD77 hotspot mode is **single simplex**, not real duplex.

---

# Troubleshooting

---

## Cannot connect to BM

Check:

* hotspot password
* BrandMeister SelfCare enabled
* firewall UDP 62031
* MQTT running

---

## Some TG need PTT first

That means Dynamic TG subscription.

Use BM SelfCare Static TG:

```text
46020 TS2 Static
```

---

## EXE Missing DLL

Copy runtime DLLs from:

```text
/ucrt64/bin/
```

---

# Download Ready-to-Use Package

👉 [http://qsl.net/bg7kuh/mmdvmhost.html](http://qsl.net/bg7kuh/mmdvmhost.html)

---

# Final Notes

Modern MMDVM on Windows is now very stable when using:

```text
MMDVMHost + DMRGateway + MQTT
```

This architecture gives:

✅ Better BM compatibility
✅ Cleaner TG routing
✅ Future-proof setup
✅ Stable OpenGD77 hotspot support

---

# 73 de BG7KUH


