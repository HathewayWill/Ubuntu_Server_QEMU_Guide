# Ubuntu Server ARM64 on QEMU

A conversational, start-to-finish build and install guide

Host: clean Ubuntu x86_64 • Guest: Ubuntu Server 24.04.3 ARM64 • UEFI + SSH port forwarding

Updated: January 1, 2026

# 1. What you are building

You will build a recent QEMU from source on an x86_64 Ubuntu host, then use it to install and boot an ARM64 Ubuntu Server VM. The VM boots via UEFI firmware, uses virtio devices for disk and networking, and exposes SSH to the host via a simple port forward.

By the end, you will have:

* A QEMU binary that supports '-netdev user' (user-mode networking via SLiRP).
* A VM disk image (qcow2) containing Ubuntu Server.
* Two UEFI flash images: one for firmware (read-only) and one for UEFI variables (persistent).
* A repeatable QEMU command line for installer boot (VNC) and normal boot (terminal + SSH).

## Directory layout we will use

Everything lives under your home directory. Example paths:

```
~/qemu/                         # QEMU source tree
~/qemu/build/                   # QEMU build output (created by ./configure + ninja)
~/qemu-ubuntu-host/             # VM workspace (you create this)
  efi.img                       # UEFI firmware flash (contains QEMU_EFI.fd)
  varstore.img                  # UEFI variable store (NVRAM)
  disk.qcow2                    # VM disk
  ubuntu-24.04.3-live-server-arm64.iso  # installer ISO
```

# 2. Before you start

You will need:

* An x86_64 machine running Ubuntu (fresh install is fine).
* At least ~15 GB free disk space (more if you store multiple ISOs/snapshots).
* A working Internet connection (to download packages, QEMU source, and the ISO).
* Patience: cross-architecture emulation uses TCG (software CPU emulation) and is slower than running on real ARM hardware.

Tip: If you only need a fast ARM64 Ubuntu for development, consider cloud images + cloud-init. This guide focuses on a full installer-based setup because it matches a real system stack more closely.

# 3. Prepare a clean Ubuntu host

## 3.1 Update the host

Open a terminal and run:

```
sudo apt update
sudo apt -y upgrade
sudo reboot
```

After reboot, run:

```
sudo apt update
```

## 3.2 Install packages (build tools + firmware + VNC viewer)

Install the packages required to build QEMU, enable user-mode networking, and run the installer UI over VNC:

```
sudo apt install -y \
  build-essential git python3 pkg-config ninja-build \
  meson-1.5 \
  libglib2.0-dev libpixman-1-dev zlib1g-dev \
  libslirp-dev \
  qemu-efi-aarch64 \
  wget ca-certificates \
  tigervnc-viewer
```

What these are for:

* build-essential, git, python3: compilers and tooling for building from source.
* ninja-build, meson-1.5: QEMU uses Meson/Ninja as its build system.
* libglib2.0-dev, libpixman-1-dev, zlib1g-dev: core dependencies for QEMU system emulation.
* libslirp-dev: enables the 'user' network backend (-netdev user). Without it, QEMU will say: "network backend 'user' is not compiled into this binary".
* qemu-efi-aarch64: provides ARM64 UEFI firmware (QEMU_EFI.fd).
* tigervnc-viewer: lets you interact with the Ubuntu installer (Subiquity) easily.

Example output (abridged):

```
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  meson-1.5 ninja-build libslirp-dev qemu-efi-aarch64 ...
...
Setting up qemu-efi-aarch64 ...
Setting up meson-1.5 ...
```

Confirm Meson is new enough:

```
meson --version
```

Example output:

```
1.5.1
```

# 4. Build QEMU from source (with user networking enabled)

We build a recent QEMU because distro packages can be older, and because you already ran into missing network backends in custom builds. This section produces a qemu-system-aarch64 that supports '-netdev user' and 'hostfwd=' for SSH port forwarding.

## 4.1 Clone QEMU

```
cd ~
git clone https://git.qemu.org/git/qemu.git
cd qemu
```

Example output:

```
Cloning into 'qemu'...
remote: Enumerating objects: ...
Receiving objects: 100% (...), done.
```

## 4.2 Configure the build

Important: run QEMU's './configure'. Do not run 'meson setup' directly for this project. './configure' prepares build configuration files (including config-host.mak) and then drives Meson/Ninja.

```
rm -rf build
./configure --target-list=aarch64-softmmu --enable-slirp
```

What the configure flags mean:

* --target-list=aarch64-softmmu: build the ARM64 system emulator (qemu-system-aarch64).
* --enable-slirp: compile in the user-mode networking backend ('-netdev user').

Example output (abridged):

```
Using './build' as the build directory
...
Target list     : aarch64-softmmu
SLIRP support   : yes
...
```

## 4.3 Build

```
ninja -C build
```

Example output:

```
ninja: Entering directory `build'
[1/2487] Compiling C object ...
...
[2487/2487] Linking target aarch64-softmmu/qemu-system-aarch64
```

## 4.4 Verify the 'user' networking backend is present

This is the quick check that prevents the earlier 'network backend user is not compiled' error:

```
./build/qemu-system-aarch64 -help | grep -n "^-netdev user"
```

Example output:

```
272:-netdev user,id=str[,ipv4=on|off]...[,...][,hostfwd=rule]...
```

## If the check fails

If you do not see '-netdev user' in the help output, the usual causes are:

* libslirp-dev was not installed before configuring.
* You configured without '--enable-slirp'.

Fix by installing libslirp-dev and re-running configure + build:

```
sudo apt install -y libslirp-dev
cd ~/qemu
rm -rf build
./configure --target-list=aarch64-softmmu --enable-slirp
ninja -C build
```

# 5. Create the VM workspace (disk + UEFI flash)

## 5.1 Create a directory for the VM

```
mkdir -p ~/qemu-ubuntu-host
cd ~/qemu-ubuntu-host
pwd
```

Example output:

```
/home/youruser/qemu-ubuntu-host
```

## 5.2 Create a qcow2 disk

qcow2 grows on demand, so the file starts small and expands as the guest writes data.

```
~/qemu/build/qemu-img create -f qcow2 disk.qcow2 40G
```

Example output:

```
Formatting 'disk.qcow2', fmt=qcow2 ... size=42949672960 ...
```

## 5.3 Prepare UEFI firmware flash images

We create two flash images:

* efi.img: contains the UEFI firmware code. We mark it read-only when launching QEMU.
* varstore.img: the persistent UEFI variable store (NVRAM). This remembers boot entries and other firmware settings.

Run these commands exactly:

```
truncate -s 64m varstore.img
truncate -s 64m efi.img
dd if=/usr/share/qemu-efi-aarch64/QEMU_EFI.fd of=efi.img conv=notrunc
```

Example output:

```
0+1 records in
0+1 records out
2097152 bytes (2.1 MB, 2.0 MiB) copied, 0.004 s, 485 MB/s
```

Check the files:

```
ls -lh efi.img varstore.img disk.qcow2
```

Example output:

```
-rw-r--r-- 1 youruser youruser  64M Jan  1 12:00 efi.img
-rw-r--r-- 1 youruser youruser  64M Jan  1 12:00 varstore.img
-rw-r--r-- 1 youruser youruser 196K Jan  1 12:01 disk.qcow2
```

Do not recreate varstore.img after installation unless you intentionally want to reset firmware state.

# 6. Download Ubuntu Server ARM64 24.04.3 install media

Ubuntu publishes both desktop and server ARM64 ISOs. For this guide we use the server installer ISO:

```
wget https://cdimage.ubuntu.com/releases/noble/release/ubuntu-24.04.3-live-server-arm64.iso
```

Example output (abridged):

```
--2026-01-01 12:05:10--  https://cdimage.ubuntu.com/.../ubuntu-24.04.3-live-server-arm64.iso
HTTP request sent, awaiting response... 200 OK
Length: 3000000000 (2.8G)
Saving to: 'ubuntu-24.04.3-live-server-arm64.iso'
...
100%[===================>] 2.80G  25.3MB/s    in 1m 54s
```

Optional checksum verification:

```
wget https://cdimage.ubuntu.com/releases/noble/release/SHA256SUMS
grep ubuntu-24.04.3-live-server-arm64.iso SHA256SUMS
sha256sum ubuntu-24.04.3-live-server-arm64.iso
```

# 7. Boot the installer (VNC)

Ubuntu Server uses a full-screen installer UI. The easiest way to interact with it under QEMU is to expose a local VNC server and connect with a VNC viewer.

## 7.1 Launch QEMU for installation

Run QEMU from your VM directory (where the images and ISO live):

```
~/qemu/build/qemu-system-aarch64 \
  -machine virt,gic-version=3,virtualization=true \
  -accel tcg,thread=multi \
  -cpu max,pauth-impdef=on \
  -smp 8 \
  -m 32768 \
  -drive if=pflash,format=raw,file=efi.img,readonly=on \
  -drive if=pflash,format=raw,file=varstore.img \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8022-:22 \
  -device virtio-scsi-pci,id=scsi0 \
  -drive if=none,id=cd,format=raw,file=ubuntu-24.04.3-live-server-arm64.iso \
  -device scsi-cd,drive=cd \
  -device virtio-gpu-pci \
  -device qemu-xhci \
  -device usb-kbd \
  -device usb-tablet \
  -display none \
  -vnc 127.0.0.1:1
```

## Adjusting CPU cores and memory (the knobs you’ll change most)

You control “how big” the VM is with **two flags**:

* **CPU cores:** `-smp <N>`
* **RAM:** `-m <MiB>`

### Quick examples

**2 cores, 4 GiB RAM (lightweight testing):**

```bash
-smp 2 -m 4096
```

**4 cores, 8 GiB RAM (comfortable for installs + basic dev):**

```bash
-smp 4 -m 8192
```

**8 cores, 32 GiB RAM (heavier workloads):**

```bash
-smp 8 -m 32768
```

### Notes that save time

* `-m` is in **MiB**. So `8192` = ~8 GiB, `32768` = ~32 GiB.
* On an x86 host emulating ARM with **TCG**, adding more cores can help some workloads but won’t scale like real hardware. If it feels slower with more cores, try fewer.
* If you use a **custom DTB** (`-dtb ...`) later, keep in mind QEMU still uses `-smp` and `-m` to define the virtual hardware; the DTB won’t override those values.

What to expect:

* The terminal may print nothing and appear to 'hang'. That is normal: the UI is on VNC.
* VNC is bound to localhost only (127.0.0.1). It is not exposed to the network.

## 7.2 Connect to the installer over VNC

```
vncviewer 127.0.0.1:5901
```

Example output:

```
TigerVNC Viewer 64-bit v1.13.x
CConn: connected to host 127.0.0.1 port 5901
```

## 7.3 Install Ubuntu Server

Inside the installer:

1. Choose language/keyboard/timezone as desired.
2. Partitioning: the default guided install is fine for most cases.
3. Networking: user-mode networking provides outbound Internet access; keep DHCP defaults.
4. SSH setup: select 'Install OpenSSH server' so you can SSH in after first boot.
5. Finish installation and allow the system to reboot.

When the installer finishes and reboots, stop QEMU with Ctrl+C in the terminal running QEMU. Next we boot without the ISO.

# 8. Boot the installed system (terminal + SSH)

## 8.1 Launch QEMU (no ISO)

```
cd /vms/ubuntu-arm64

~/qemu/build/qemu-system-aarch64 \
  -machine virt,gic-version=3,virtualization=true \
  -accel tcg,thread=multi \
  -cpu max,pauth-impdef=on \
  -smp 8 \
  -m 32768 \
  -drive if=pflash,format=raw,file=efi.img,readonly=on \
  -drive if=pflash,format=raw,file=varstore.img \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8022-:22 \
  -nographic
```

Example output (boot messages abridged):

```
[    0.000000] Booting Linux on physical CPU 0x0000000000 ...
...
Ubuntu 24.04.3 LTS ubuntu ttyAMA0

ubuntu login:
```

## 8.2 SSH into the guest

From another host terminal:

```
ssh -p 8022 <your-username>@localhost
```

Example first connection output:

```
The authenticity of host '[localhost]:8022 ([127.0.0.1]:8022)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[localhost]:8022' (ED25519) to the list of known hosts.
<your-username>@localhost's password:
```

# 9. Networking choices: user vs tap/bridge vs passt

This guide uses user-mode networking because it requires no root setup and works anywhere. However, you have options:

## 9.1 Option A (recommended): user-mode networking

Keep these flags:

```
-device virtio-net-pci,netdev=net0
-netdev user,id=net0,hostfwd=tcp::8022-:22
```

Pros:

* Zero host networking configuration.
* Easy SSH from host using hostfwd.

Cons:

* NAT-like behavior; inbound connections require explicit port forwards.
* Some advanced networking features are limited compared to a bridged setup.

## 9.2 Option B: TAP/bridge networking (more real networking)

Use this when you want the guest to be on the same L2 network (or when you want normal inbound access without hostfwd). Requires host setup and often root.

Example host setup (creates tap0 owned by your user):

```
sudo ip tuntap add dev tap0 mode tap user "$USER"
sudo ip link set tap0 up
```

Then run QEMU with:

```
-device virtio-net-pci,netdev=net0 \
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no
```

If you want guest Internet via host NAT, you must also enable forwarding and add NAT rules on the host (iptables/nftables).

## 9.3 Option C: passt backend (if present)

Some QEMU builds support a 'passt' backend. If your help output shows '-netdev passt', you can use it as another host-friendly approach. Whether it is ideal depends on your environment; user-mode networking is the most universally available and easiest to reason about.

# 10. QEMU flags explained line-by-line (what it maps to, and what you can delete)

This section explains every QEMU flag used in the installer and normal-boot commands. For each flag we cover: what it does, what virtual hardware it represents (if any), and whether you can remove it safely.

| Flag / example                                           | What it does                                                                                                      | Maps to virtual hardware?                                                              | Can you remove it?                                                                                                                         |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| -machine virt,gic-version=3,virtualization=true          | Selects the ARM 'virt' machine and sets board properties (interrupt controller version, virtualization exposure). | Yes: board model (virt), GIC interrupt controller, virtualization capability exposure. | Must keep '-machine virt'. You can usually drop 'gic-version=3' and 'virtualization=true' for a normal server VM.                          |
| -accel tcg,thread=multi                                  | Uses TCG software CPU emulation. 'thread=multi' may improve throughput in some cases.                             | No direct guest hardware; affects how QEMU executes guest code on the host.            | Optional. Safe to omit (QEMU will typically fall back to TCG anyway in cross-arch).                                                        |
| -cpu max,pauth-impdef=on                                 | Selects a feature-rich CPU model and enables a pointer authentication feature flag.                               | Yes: guest CPU model and CPU feature set.                                              | Optional. You can omit '-cpu' to use defaults, or choose a conservative CPU (e.g., cortex-a72). 'pauth-impdef=on' is usually safe to drop. |
| -smp 8                                                   | Exposes 8 virtual CPUs to the guest.                                                                              | Yes: vCPU count/topology.                                                              | Optional (default is small). Keep for performance/testing.                                                                                 |
| -m 32768                                                 | Allocates 32 GiB RAM to the guest.                                                                                | Yes: guest RAM size.                                                                   | Optional (default is small). Keep for realistic workloads.                                                                                 |
| -drive if=pflash,... file=efi.img,readonly=on            | Attaches firmware flash containing UEFI code and makes it read-only.                                              | Yes: flash device holding UEFI firmware.                                               | Keep if you want UEFI boot. Without it you lose this UEFI firmware path.                                                                   |
| -drive if=pflash,... file=varstore.img                   | Attaches UEFI variable store (NVRAM) for boot entries, settings, etc.                                             | Yes: flash region for UEFI variables.                                                  | Strongly recommended to keep. Removing it causes non-persistent firmware state (boot entry problems).                                      |
| -drive if=virtio,format=qcow2,file=disk.qcow2            | Attaches the main VM disk using virtio (fast paravirtual disk).                                                   | Yes: virtio-blk disk device (often /dev/vda).                                          | Keep for installed OS. Required unless you boot from something else.                                                                       |
| -device virtio-net-pci,netdev=net0                       | Adds a virtio network card connected to backend 'net0'.                                                           | Yes: virtio network interface.                                                         | Optional only if you do not need networking. Must match a '-netdev ... id=net0'.                                                           |
| -netdev user,id=net0,hostfwd=tcp::8022-:22               | Creates a user-mode (SLiRP) NAT network backend and forwards host TCP 8022 to guest TCP 22 (SSH).                 | Host-side network backend (not a guest device).                                        | Optional if you switch to tap/bridge/passt. 'hostfwd' is optional but required for easy host->guest SSH.                                   |
| -device virtio-scsi-pci,id=scsi0                         | Adds a virtio SCSI controller so we can attach a SCSI CD-ROM.                                                     | Yes: virtio SCSI HBA.                                                                  | Installer-only in this guide. If you attach the ISO another way, you can drop it.                                                          |
| -drive if=none,id=cd,format=raw,file=...iso              | Defines the ISO as a backend named 'cd'. 'format=raw' avoids QEMU's probing warning.                              | Host-side block backend definition.                                                    | Installer-only. Remove after installation.                                                                                                 |
| -device scsi-cd,drive=cd                                 | Attaches the ISO backend as a SCSI CD-ROM device.                                                                 | Yes: CD-ROM device.                                                                    | Installer-only. Remove after installation.                                                                                                 |
| -device virtio-gpu-pci                                   | Provides a framebuffer device for graphical output (useful for VNC installer UI).                                 | Yes: virtio GPU device.                                                                | Installer convenience. Optional for headless/serial installs.                                                                              |
| -device qemu-xhci / -device usb-kbd / -device usb-tablet | Adds USB controller, keyboard, and absolute pointing device for reliable VNC input.                               | Yes: USB controller and USB HID devices.                                               | Installer convenience. Usually safe to remove once you use SSH or -nographic.                                                              |
| -display none                                            | Prevents QEMU from opening a local GUI window (SDL/GTK).                                                          | Host UI behavior.                                                                      | Optional. Keep if you only want VNC, drop if you want a local window.                                                                      |
| -vnc 127.0.0.1:1                                         | Starts a VNC server on localhost display :1 (port 5901).                                                          | Host UI behavior.                                                                      | Installer convenience. Remove if using -nographic or local GUI.                                                                            |
| -nographic                                               | Disables graphics and routes serial console to your terminal.                                                     | Host console wiring (serial over stdio).                                               | Optional. Great for server usage. If you want VNC/GUI console, omit it.                                                                    |

## 10.1 Minimal recommended post-install command (safe defaults)

Once Ubuntu is installed and you mostly live in SSH, you can simplify to the essentials:

```
~/qemu/build/qemu-system-aarch64 \
  -machine virt \
  -accel tcg \
  -smp 8 -m 32768 \
  -drive if=pflash,format=raw,file=efi.img,readonly=on \
  -drive if=pflash,format=raw,file=varstore.img \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8022-:22 \
  -nographic
```

This removes installer-only devices (CD-ROM, VNC GPU/input) and keeps a stable UEFI + disk + network + SSH workflow.

# 11. Troubleshooting quick hits

## Error: network backend 'user' is not compiled into this binary

Your QEMU build is missing SLiRP/user networking. Install libslirp-dev, reconfigure with --enable-slirp, rebuild.

```
sudo apt install -y libslirp-dev
cd ~/qemu
rm -rf build
./configure --target-list=aarch64-softmmu --enable-slirp
ninja -C build
```

## Error: Meson version too old (project requires >= 1.5.0)

Install meson-1.5 (or newer) and make sure 'meson --version' shows 1.5+.

```
sudo apt install -y meson-1.5
meson --version
```

## Boot returns to installer

You are still booting with the ISO attached. Boot without the CD drive lines.

```
(Remove '-drive ...iso' and '-device scsi-cd,...' from the normal boot command)
```

## SSH does not respond on port 8022

Make sure OpenSSH server is installed and running inside the guest. If not, install and enable it.

```
sudo systemctl status ssh
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

# 12. Confirm system architecture (host vs guest)

This is a quick sanity check that answers two questions:

* Is my **host** really x86_64 (amd64)?
* Is my **guest** really ARM64 (aarch64/arm64)?

Run these on the **host** (your Ubuntu machine):

```
uname -m
dpkg --print-architecture
```

Typical expected output on the host:

* `uname -m` -> `x86_64`
* `dpkg --print-architecture` -> `amd64`

Now run the same commands **inside the guest** (in the VM console or over SSH):

```
uname -m
dpkg --print-architecture
```

Typical expected output in the guest:

* `uname -m` -> `aarch64`
* `dpkg --print-architecture` -> `arm64`

# References

Ubuntu ARM64 ISO directory (24.04.3): [https://cdimage.ubuntu.com/releases/24.04.3/release/](https://cdimage.ubuntu.com/releases/24.04.3/release/)

Ubuntu Server ARM64 ISO used here: [https://cdimage.ubuntu.com/releases/noble/release/ubuntu-24.04.3-live-server-arm64.iso](https://cdimage.ubuntu.com/releases/noble/release/ubuntu-24.04.3-live-server-arm64.iso)

QEMU user networking documentation (for hostfwd and behavior): [https://www.qemu.org/docs/master/system/invocation.html#hxtool-4](https://www.qemu.org/docs/master/system/invocation.html#hxtool-4) and [https://www.qemu.org/docs/master/system/net.html](https://www.qemu.org/docs/master/system/net.html)
