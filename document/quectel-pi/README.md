# Quectel Pi

**English** | [简体中文](https://github.com/quectel-developer/qpi-vscode-extension-doc/blob/master/document/quectel-pi/README.zh-CN.md)

A VS Code extension that drives the whole **Quectel Pi series** SDK workflow from a graphical interface: download the SDK → connect → set up the build environment → build → connect the board → flash → debug.

Supported SDKs:

| SDK | SoC | Build system | Output & flashing |
| --- | --- | --- | --- |
| **M2 SDK** | RK3576 | Native `build.sh` | `update.img`, written with a flashing tool |
| **H1 Simple SDK** | QCS6490 | Native `build.sh` | `efi.bin` + `dtb.bin` + `system.img`, whole-disk EDL flash |
| **H1 Yocto SDK** | QCS6490 | Yocto / BitBake | Firmware folder, flashed with the SDK's own `flash.sh` |

The extension shows the matching buttons for whichever SDK you are connected to.

---

# Part 1 · User Guide

> This part is for users. Everything is done in VS Code.

## 1. Before you start (what you need)

### Host machine (the computer with the VS Code extension installed)

Building, flashing and device operations all happen on this machine (or are forwarded over SSH to a remote build machine). The extension supports the three host environments below.

**Windows**

- **OS**: Windows 10 / Windows 11, with the **WSL2** subsystem installed (builds run inside a Docker container, so WSL1 will not do).
- **SDK location**: The SDK source **must live inside WSL** (for example `/home/<user>/qpi-sdk`), because the SDK's build scripts are Linux shell scripts.
- **Memory**: 16 GB or more recommended (WSL2 uses at most half of the host's memory by default; with too little memory a build may hit a kernel OOM).
- **Disk**: Downloading and building the M2 / H1 Simple SDK takes tens of GB; **downloading and building a full Yocto SDK takes hundreds of GB**.

**Ubuntu (Linux)**

- **OS**: **Ubuntu 22.04 or 24.04** recommended (the flashing tool qdl depends on glibc ≥ 2.34 and libzip.so, which Ubuntu 20.04 and older do not provide).
- **SDK location**: Any directory on this machine will do.
- **Memory**: 16 GB or more recommended.
- **Disk**: Downloading and building the M2 / H1 Simple SDK takes tens of GB; **downloading and building a full Yocto SDK takes hundreds of GB**.

**macOS**

- **OS**: **macOS 13 Ventura or newer** recommended; an ARM-based Mac is required.
- **SDK location**: Any directory on the Mac will do. The build does **not** run directly on the Mac — it runs inside a Docker container in the lightweight Linux VM (`colima`) that the extension prepares automatically (see section 7).
- **Memory**: 16 GB or more recommended (8 GB of it is used by the VM).
- **Disk**: The VM disk is 100 GB (sparse, grows as it is used), on top of the SDK and build output; **a full Yocto build needs more**.

### The board

The matching boards are **Quectel Pi M2** and **Quectel Pi H1**; the images you build are flashed onto the board and run there. Before you start, connect the board to the right machine with a USB data cable.

- **Quectel Pi M2** — Rockchip **RK3576** SoC.
- **Quectel Pi H1** — Qualcomm **QCS6490** SoC.

### What else you need

- **VS Code**: version 1.85 or newer.
- **An SDK source**: either download it inside the extension (see the steps below), or use an SDK directory you already have.
- **USB driver**: on Windows you need to install the USB driver before the board can be detected; see below for how to get it.
- **More information**: click Manage Resources → Documentation inside the extension to reach the official site, where you can get the driver and more.

![Click Manage Resources → Documentation to reach the official site for the driver and more](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/information_more_1.png)

Optional: a **remote Linux build machine**. If you build on a server, install VS Code and this extension on your own computer and connect over SSH — see section 6.

## 2. Install the extension

1. Open VS Code, click the **Extensions** icon in the activity bar (or press `Ctrl+Shift+X`).
2. Search for **Quectel Pi** and click **Install**.
3. A **Quectel Pi** icon appears in the activity bar — click it to open the tool panel.

![Search for Quectel Pi in the Marketplace and click Install](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/download_extension1.png)

The first time you open the panel, the extension prepares its own runtime in the background. The status chip in the panel header shows **Not connected** until you connect to an SDK.

> **Language**: the interface is available in Chinese and English. Click the 🌐 button in the panel header to switch. It follows your VS Code display language until you switch it manually, and remembers your choice afterwards.

## 3. Take a quick look around

Everything lives in three places (features an SDK does not have are hidden):

| Place | What is there |
| --- | --- |
| **Side bar** (the Quectel Pi icon) | The main UI: collapsible card sections — Connection / SDK, Application, Build, Flash & Device, Logs / Debug, Multimedia, AI Assistant |
| **Header buttons** | Three buttons to the right of the connection status: **Refresh** (re-check the connection and SDK platform), **Restart backend** (if the helper service gets stuck), **Toggle language** |
| **Status bar** (bottom edge of the window) | Five one-click entries: connection status, build app, flash boot, full flash, dmesg (live kernel log) |

Below the card sections, the same side bar shows the **Manage Resources** view — five entries you use before you start: **Documentation** (pick the English or Chinese official site), **Manage SDK**, **Open an existing application**, **Browse Samples** (opens the Quectel-Pi GitHub organisation), **Open terminal**.

A few things open as a **separate tab** in the editor area: the connection form, the AT command console, the device info page, the performance monitor, the SDK info page and the Yocto build configuration. Each tab carries the same name as the card that opened it.

The connection status refreshes every 5 seconds, so plugging in a board is picked up automatically.

## 4. Do it once: build → flash in five steps

This is the shortest complete path, from an empty computer to a board running your own build.

### Step 1 · Download the SDK

In the side bar, scroll to **Manage Resources** and click **Manage SDK**, then pick your SDK family:

![Select SDK version — M2 or H1](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step1-select-sdk.png)

*Pick the SDK family first. If you pick H1, a second prompt asks whether you want the **Simple SDK** or the **Yocto SDK**; M2 has only one variant.*

Next you are asked where to save it:

![Enter the local save path](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step1-save-path.png)

*Type an absolute path and press `Enter` — the SDK is `git clone`d into that directory in a dedicated terminal, with live progress. Closing that terminal aborts the download.*

- On Linux / WSL, use a path such as `/home/<username>/qpi-sdk`.
- On Windows, the path must be inside WSL (e.g. `/home/<username>/qpi-sdk`), see section 6.
- Depending on your network, the download may take a while.

### Step 2 · Connect to the SDK

Click **Connect SDK** in the **Connection / SDK** section. A form opens in the editor area:

![The Connect to QPi SDK form](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step2-connect-sdk.png)

*Leave the mode on **Local** and enter the SDK root directory (the folder you just cloned). If the SDK is on a remote build machine, choose **SSH remote** instead and fill in host / user / password. Click **Connect**.*

When it succeeds, the status chip in the panel header changes from **Not connected** to the SDK type and path (for example `Local @ /home/will/qpi/h1`), and the buttons in the **Build** and **Flash & Device** sections adapt to the detected SDK.

Your connection settings are remembered, so the extension reconnects automatically the next time you start VS Code.

### Step 3 · Set up the build environment

Click **Configure build env...** in the **Build** section:

![Configuring the build environment, with its terminal log](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step3-setup-env.png)

*This prepares a Docker build environment and then runs the SDK's own environment script inside it. Progress is streamed to the terminal.*

1. It checks for Docker. If it is missing it installs it — Docker Engine on Linux/WSL, or on **macOS** `colima` (a lightweight Linux VM; no Homebrew, no `sudo`, and Docker Desktop is never used even if installed).
2. It builds the build image from the SDK's own `docker_env/Dockerfile`. This image contains **every** build dependency, so your host does not need them.
3. It runs the SDK's `build.sh setenv` **inside that container**, which verifies/installs dependencies and downloads the prebuilt base images.

- On Linux/WSL you are asked for your login password once (installing Docker and building the image use `sudo`). On **macOS no password is needed**.
- The first run downloads a base image and can take several minutes.
- Run this once per SDK — but re-run it if a build complains about missing tools.
- Every later build reuses the same image, so the host toolchain is irrelevant. See section 7 for what this means on macOS.

### Step 4 · Build

Click **Build all** in the **Build** section:

![A running build with its log in the terminal](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step4-build-all.png)

*The build runs in its own terminal and streams the full log. The first build of a Yocto SDK takes a long time; later builds are incremental.*

- The build runs **inside the Docker container** prepared in step 3. Your SDK folder is mounted at the same absolute path in the container, so the artifacts (for example `update.img`) land directly in your SDK directory and the flashing tools can read them.
- You can also build just the bootloader (**Build boot**), or, on a Yocto SDK, build the bootloader and kernel together (**Build kernel**).
- **Clean** removes build intermediates if you want a build from scratch.
- Closing the terminal aborts the build — handy when you want to stop a long build.

### Step 5 · Connect the board and flash

1. Connect the board to your computer with a USB cable.
2. Click **Device info** and check that the board is detected (model, OS version, memory, storage).
3. Put the board into download mode:
   - M2: click **Enter Loader**
   - H1: click **Enter EDL** (Qualcomm 9008 download mode)
4. Click **Full flash** and wait for it to finish. On M2 you are asked which flashing tool to use — there is nothing extra to install.
5. Click **Reboot device**. The board boots into the system you just built.

## 5. The three SDKs in detail

### 5.1 M2 SDK (RK3576)

1. **Download**: Manage Resources → Manage SDK → **M2** → absolute path → `Enter`.
2. **Connect**: Connect SDK, mode **Local** (or **SSH remote** to build on a server).
3. **Configure build env...**: prepares the Docker build image and downloads the prebuilt base images.
4. **Build all**: produces `update.img`. **Build boot** builds only the bootloader.
5. **Device info**: plug in the board over USB and confirm it is detected.
6. **Enter Loader**: puts the board into the Rockchip loader mode.
7. **Full flash**: pick a flashing tool (`upgrade_tool` / `rkdeveloptool`) and write `update.img`; the boot partition can also be flashed on its own with **Flash boot**.
8. **Reboot device**.

### 5.2 H1 Simple SDK (QCS6490)

1. **Download**: Manage Resources → Manage SDK → **H1** → **H1 Simple SDK** → absolute path → `Enter`.
2. **Connect**: as above.
3. **Configure build env...**: prepares the Docker build image and downloads the prebuilt base image.
4. **Build all**: produces the partition images; **Build boot** builds only `efi.bin` / `dtb.bin`.
5. **Device info**: plug in the board over USB and confirm it is detected.
6. **Enter EDL**: puts the board into Qualcomm EDL 9008 download mode.
7. **Full flash**: runs the SDK's own `flash.sh` and writes the whole disk. Single-partition flashing is not supported on the Qualcomm platform.
8. **Reboot device**.

### 5.3 H1 Yocto SDK (QCS6490)

1. **Download**: Manage Resources → Manage SDK → **H1** → **H1 Yocto SDK** → absolute path → `Enter`.
2. **Connect**: as above.
3. **Configure build env...**: you fill in the project ID, the OS (`LINUX` / `UBUNTU` / `DEBIAN`) and the version (`STD` release / `DBG` debug), plus an optional secure-boot checkbox; the panel saves those parameters and every build applies them automatically (all compilation happens in the container, so no SDK dependencies are installed on the host). The same step checks the Docker / `colima` status and, if the build image is not ready yet, prepares the Docker build environment (on Linux/WSL it asks for your `sudo` password once — no password is needed on macOS).
4. **Build**:
   - **Build all**: produces the firmware folder used for flashing.
   - **Build kernel**: produces `dtb.bin` (bootloader) and `efi.bin` (kernel). This is much faster than a full build.
5. **Device info**: plug in the board over USB and confirm it is detected.
6. **Flash**:
   - **Full flash**: enter EDL first, then the firmware is written to the whole disk.
   - **Flash kernel**: no EDL needed — the kernel artifacts are written straight to the DTB and ESP partitions. If they do not exist yet, you are told to build first.
7. **Reboot device**.

## 6. Connection modes

| Extension host | Mode | Where the SDK lives |
| --- | --- | --- |
| **Linux** | Local | a directory on this Linux machine — the most common setup |
| **Linux** | SSH remote | a remote Ubuntu / Linux build machine |
| **macOS** | Local | a directory on the Mac (Apple Silicon or Intel) — builds run through a `colima` Docker VM (see section 7) |
| **macOS** | SSH remote | a remote Ubuntu / Linux build machine — the fastest option on Apple Silicon |
| **Windows** | Local | **inside WSL** (e.g. `/home/<user>/qpi-sdk`) — the build scripts need Linux |
| **Windows** | SSH remote | a remote Ubuntu / Linux build machine; your computer is only the UI |

### 6.1 Local connection (the SDK is on the machine running the extension)

**Linux local** — the most common setup.

- Open **Connect SDK**, leave the mode on **Local**, enter the absolute path to the SDK root (the level that contains `build.sh`) and click **Connect**.
- When the board is plugged into this machine over USB, `adb` must be available here. On native Linux, if it is missing the extension tells you and can install it for you; you can also install it yourself first:

  ```bash
  sudo apt-get install -y android-tools-adb
  ```

- The flashing tool qdl depends on `glibc ≥ 2.34` and `libzip.so`, so **Ubuntu 22.04 / 24.04** is recommended for the host (20.04 and older do not satisfy this).
- If you are not sure whether the environment is complete, run **QPi SDK: environment check** from the command palette — it lists the missing components.

**Windows (SDK inside WSL)**

- Prerequisite 1: install **WSL2** on Windows and put the SDK in a directory inside WSL (e.g. `/home/<user>/qpi-sdk`).
- Prerequisite 2: **enable systemd in WSL**. The Docker used for building is a systemd service inside WSL; without systemd, `docker` installs but will not start. Edit `/etc/wsl.conf` inside WSL:

  ```ini
  [boot]
  systemd=true
  ```

  Save it, then run `wsl --shutdown` back in Windows and enter WSL again.
- Prerequisite 3: keep WSL's **Windows interoperability (interop)** enabled (it is on by default). The board's USB is attached to Windows, and the extension connects to the board through the Windows-side `adb.exe` via interop — even if Windows has no platform-tools installed, the extension ships its own `adb`.
- Connect: choose the **Local** mode and enter the **path inside WSL** as the SDK root (e.g. `/home/<user>/qpi-sdk`) or the shared path (`\\wsl.localhost\<distro>\...`), **never a `C:\...` path**. The extension enters WSL automatically to run the build and other operations.
- Note: WSL2 uses at most half of the host's memory by default; with too little memory builds are slower and may even hit an OOM. You can raise the memory limit in `%UserProfile%\.wslconfig` on Windows.

**macOS local**

- Connect: choose the **Local** mode and enter the absolute path to the SDK on the Mac.
- Nothing else to prepare: the build is handled by the `colima` VM + Docker that the extension sets up automatically (see section 7); the `adb` used to talk to the board is bundled with the extension (a universal binary), so there is no need to install platform-tools.
- A full build on Apple Silicon is slow; if you build full images often, prefer SSH remote below.

### 6.2 SSH remote connection (the SDK is on a remote Linux build machine)

- Open **Connect SDK**, choose the **SSH remote** mode, and fill in the host IP / domain, the port (22 by default), the username, the authentication method (password or private key file), and then the **absolute path to the SDK root on the remote machine**.
- **The step most people overlook is "Device connection location".** The form has a **Device connection location** field that decides which machine actually runs the device operations — adb, AT, flashing and deploy:

  - **Board connected to this PC** (default): the board is plugged into your own computer over USB. Device operations run locally; if the firmware is produced on the remote build machine (for example H1's `flash.sh`), the extension **first pulls the firmware to a local temporary directory** and then flashes. This machine needs a working `adb` (Windows / macOS have a bundled adb, on Linux you install it yourself).
  - **Board connected to the remote machine**: the board is plugged into the remote build machine over USB. All device operations run remotely, and the remote machine **must have adb installed** (the extension will not install it there for you):

    ```bash
    sudo apt install android-tools-adb
    ```

    Otherwise the extension reports "adb command not found on the remote build machine".

- A typical symptom of getting this field wrong: the board is plugged into your own computer but you selected "board connected to the remote machine" — the commands then go looking for the device on the remote machine and of course do not find it.
- Other notes: the SDK on the remote machine also needs **Configure build env...** run once; if the connection fails, the extension prints a connection log in the terminal that you can use to track down a path or credential problem.

## 7. Building on macOS

On macOS the build does **not** run directly on the Mac. The extension downloads a lightweight Linux VM (`colima`) and builds inside a Docker container in that VM. The toolchain, the SDK's build dependencies and the scripts are all Linux, so nothing extra needs to be installed on the Mac itself.

### What the extension sets up for you

On the first **Configure build env...** it downloads, into `~/.qpi`:

- `colima` + `lima` (the VM manager), and the `docker` CLI + `buildx`;
- a base VM image (Ubuntu 24.04) — the first download is a few hundred MB;
- then it builds the SDK's `quectel-pi-builder` image inside that VM.

The VM is created with **4 CPUs, 8 GB of RAM and a 100 GB disk**. Re-running **Configure build env...** is idempotent — it only fills in what is missing.

> Docker Desktop is intentionally ignored: even if it is installed, the extension always drives Docker through its own `colima` VM (the docker context is pinned to `colima`). You do not need Docker Desktop, and you can leave it uninstalled.

### Apple Silicon vs Intel

| Mac | What happens | Effect |
| --- | --- | --- |
| **Intel** (x86_64) | The container matches the host architecture, so it runs natively | Normal build speed |
| **Apple Silicon** (arm64) | The SDK only ships an x86_64 toolchain, so the image is built for `linux/amd64` and run under QEMU emulation | Works, but builds are noticeably **slower** than on an Intel Mac or a native Linux machine |

On Apple Silicon the extension also fixes a known QEMU bug: the emulator that ships with the VM crashes (`QEMU internal SIGSEGV`) as soon as a build tool reads `/proc/self/maps`, which the kernel's `grep` / `flex` / assembler probes do. The extension installs a patched QEMU 9.x interpreter into the VM and registers it for x86_64 binaries, so builds finish instead of dying with exit code 2. This is automatic — you will see `[QPi] ... QEMU ...` lines in the setup log.

Because of the emulation overhead, a full build on Apple Silicon takes a long time. If you build full images regularly, consider an **SSH remote** Linux build machine instead (section 6) — the heavy compilation then runs natively there.

### Requirements for a successful build on macOS

| Requirement | Details |
| --- | --- |
| macOS | A recent macOS (**macOS 13 Ventura or newer** recommended). Both **Apple Silicon** and **Intel** Macs are supported |
| Free disk | The VM disk is 100 GB (sparse, grows as it is used), on top of the SDK and build output. Allow a few tens of GB for an M2 / H1 simple SDK, and **tens of GB more for a full Yocto build** |
| Memory | 8 GB is reserved for the VM, so **16 GB of Mac RAM** is recommended |
| Network | Needed on the first **Configure build env...** — it downloads `colima`, the base VM image and the build image |
| Nothing else | No Homebrew, no Docker Desktop, no toolchain, and no `sudo` password |
| Flashing from the Mac | Only the board's USB/serial driver, if your board needs one. The extension ships `adb` for macOS, so you do not need `platform-tools` |

## 8. Feature reference

Everything the cards do, section by section.

### Connection / SDK

- **Connect SDK** — connect to an SDK (Local or SSH). The form is described in step 2.
- **Disconnect** — drop the current connection. Nothing is deleted.
- **Refresh devices** — re-scan the boards connected over USB.
- **SDK info** — opens a tab showing the current branch, the current commit, whether your branch is behind the remote, and the recent commit history.
- **Backend info** — opens a tab with the address and state of the extension's helper service. Useful when something looks broken.
- **Setup env** — opens the environment bootstrap: it checks 8 components of your SDK directory (core SDK, kernel sources, build tools, prebuilt resources, cross toolchain, project templates, skill documents, device tree overlays) and tells you which ones are missing, with the option to clone them from a repository.

### Application

- **Create app** — a wizard: pick a template → fill in the template variables (values that have a dropdown must be picked from it) → name the project. Letters, digits, underscores and hyphens, 1–64 characters. The project is created under the SDK's `projects/` directory.
  - Templates come from the SDK itself (`docs/templates/`). If your SDK ships no templates, the wizard says so and stops.
- **App list** — every project under `projects/`, each with *open folder / build / delete* (deleting asks for confirmation).
- **Skill docs** — the documents shipped in the SDK's `skills/` directory, opened in a VS Code tab.

### Build

- **Configure build env...** — the one-time (or after-update) environment preparation described in step 3: it prepares Docker (or downloads `colima` on macOS), builds the container image, then runs the SDK's `setenv` inside it. On a Yocto SDK it instead opens the build configuration panel and, if needed, prepares the Docker environment.
- **Build all** — the full build, producing flashable images for your SDK.
- **Build boot** / **Build kernel** — only one of the two appears, depending on the SDK. On a simple SDK it builds the bootloader; on a Yocto SDK it builds the bootloader and the kernel together.
- **Clean** — delete build intermediates so the next build starts from scratch.

> **Building an application** (the *Build app* entry in the status bar) lists the projects that already exist under the SDK's `projects/` directory and builds the one you pick. If there is no project yet you are told so — create one first with **Create app**. A Yocto SDK does not take part in application builds: build applications with BitBake instead.

### Flash & Device

- **Full flash** — write the whole disk. M2 asks which tool to use (the tools come from the SDK); H1 runs the SDK's `flash.sh`.
- **Flash boot** / **Flash kernel** — the counterpart of *Build boot / Build kernel*. On the Qualcomm platform a single partition cannot be flashed, so *Flash boot* tells you to use **Full flash** instead; on a Yocto SDK *Flash kernel* writes the kernel artifacts to the DTB and ESP partitions.
- **Enter Loader / Enter EDL** — put the board into download mode (M2 → Loader, H1 → EDL 9008).
- **Deploy file** — copy a file to a directory on the board (the default is `/userdata/<file name>`).
- **Device info** — model, OS version, memory, storage, network and more.
- **Reboot device** — reboot the board.

### Logs / Debug

- **adb command** — a terminal already attached to the board, ready for `adb shell` commands.
- **dmesg** — the live kernel log.
- **journalctl** — the live system log.
- **AT command** — a serial console: pick the port and baud rate and exchange AT commands with the module. It works over the board's own serial port, the computer's serial port, or the remote build machine's serial port.
- **Shell** — run shell commands in a terminal attached to the board.
- **Audio diagnostics** — collect audio logs and configuration for troubleshooting.

### Multimedia

- **Audio playback** / **Video playback** — play a file that is on the board.
- **Screen record** / **Screenshot** — capture the board's screen. The board needs a display attached with the desktop logged in.
- **Performance monitor** — a live chart tab showing frame rate, CPU usage and frequency, GPU usage and frequency and temperature, over adb. The sampling interval is adjustable and you can pause at any time. Metrics a board does not expose show as "no data".

### AI Assistant

- **AI assistant** — opens GitHub Copilot Chat (and offers to install it if it is missing).
- **API key settings** — opens the settings page used to plug in a third-party AI service (base URL / API key).

## 9. FAQ

**The panel keeps showing "Not connected".**
Click **Connect SDK** and enter the SDK directory. After the first successful connection the extension remembers the settings and reconnects by itself.

**It asks for my password.**
Installing Docker and building the image — and flashing, on Linux — need administrator rights on the machine that holds the SDK. Enter your login password; if passwordless `sudo` is configured, just press `Enter`. **On macOS no password is ever needed**: the setup downloads `colima` and runs without `sudo`.

**On macOS, do I need to install Docker or Docker Desktop?**
No. On the first **Configure build env...** the extension downloads `colima` (its own lightweight Linux VM) and the docker CLI into `~/.qpi` — no Homebrew and no `sudo`. Docker Desktop is ignored even if it is installed. See section 7.

**Why is the build so slow on my Apple Silicon Mac?**
The SDK only provides an x86_64 toolchain, so the container is built for `linux/amd64` and runs under QEMU emulation. It works — the extension fixes the QEMU bug that would otherwise abort the build — but a full build takes a long time. If you build full images often, use an **SSH remote** Linux machine instead (section 7).

**"No usable flashing tool found".**
The Rockchip flashing tools come with the SDK (`tools/flash/`, declared by `FLASH_TOOLS_*` in the SDK's `tools/VERSION.txt`), so first check that your SDK is complete. Otherwise install them on the machine that does the flashing (`upgrade_tool`, `rkdeveloptool`) — the extension tells you which ones, and where, it looked for.

**How do I switch between Chinese and English?**
The 🌐 button in the panel header. The side bar, the status bar and every open tab switch immediately.

**How do I change the extension's own settings?**
Search for `qpiSdk` in the VS Code settings:

| Setting | Default | Description |
| --- | --- | --- |
| `qpiSdk.backend.python` | empty (auto-detect) | path to the Python interpreter used by the helper service |
| `qpiSdk.backend.autoStart` | true | start the helper service together with VS Code |
| `qpiSdk.backend.mode` | local | `local` = helper service on this machine; `remote` = use a helper service on a remote machine |
| `qpiSdk.backend.remoteUrl` | http://127.0.0.1:8765 | address of the helper service in `remote` mode |
| `qpiSdk.backend.remoteKey` | empty | key of the helper service in `remote` mode |
| `qpiSdk.backend.pythonMirror` | empty (chosen by your network) | mirror used to fetch the bundled Python runtime |
| `qpiSdk.backend.pipMirror` | empty (chosen by your network) | package index used to install the helper service's dependencies |
| `qpiSdk.download.proxy` | empty (direct) | proxy used for all downloads, e.g. `http://127.0.0.1:7890` or `socks5://127.0.0.1:7890` |
| `qpiSdk.download.mirror` | auto | the mirror acceleration policy, see the table below |

Values of `qpiSdk.download.mirror`:

| Value | Meaning |
| --- | --- |
| `auto` | detect the network environment and decide between Chinese mirrors and the official sources |
| `cn` | always prefer Chinese mirrors (Tsinghua / USTC / Aliyun / NJU) |
| `global` | always prefer the official sources (GitHub / PyPI / download.docker.com) |
| `off` | turn mirrors off completely and use only the official sources (can be combined with a proxy) |

**How the proxy and the mirror differ, and how they work together**: `download.proxy` is the traffic exit (it proxies every download — SDK repositories, the Python runtime, pip, the Docker toolchain, apt); `download.mirror` chooses the source site. Chinese mirrors are reached directly, so the two can be used together (for example, keep a proxy enabled and still use a Chinese mirror for speed).

The most convenient way to set both is the **QPi SDK: Configure download proxy / mirror** command in the command palette — it walks you through the proxy address and the mirror policy, which is easier than editing the settings by hand.

**Where are the screenshots in this document from?**
They were taken on a Linux (WSL) setup with an H1 simple SDK. The layout is identical on other platforms; only the SDK-specific buttons differ, as described in section 5.

## 10. Feedback

Found a bug, or have a question or a feature request? Please open an issue:

**https://github.com/quectel-developer/qpi-vscode-extension-doc/issues**

It helps to include: your operating system (and whether the SDK lives under WSL), the SDK type (M2 / H1 simple / H1 Yocto), what you did, what you expected, and what actually happened. The log from the terminal that ran the task is the most useful thing to attach.

---

# Part 2 · Development & Maintenance

> This part is for developers and maintainers of the extension.

## 1. Architecture

```
side bar / status bar / webview tabs ──postMessage──▶ extension.ts ──HTTP + SSE──▶ FastAPI (python)
              UI                                          │                            │
                                                 connection config              executor (local / SSH)
                                                 (workspaceState)                      │
                                                                 build.sh / bitbake / adb / qdl
```

- **Frontend**: TypeScript extension + webviews (plain HTML/CSS/JS, no framework, adapts to light and dark themes)
- **Backend**: Python FastAPI on a random `127.0.0.1` port; every request must carry an `X-API-Key`; the virtual environment and its dependencies are created on first start
- **Executor**: local (subprocess) or a remote Linux build machine over SSH (paramiko)
- **Task model**: each build / flash / log task opens a dedicated VS Code terminal and streams output over SSE; closing the terminal kills the backend task

## 2. Directory layout

```
src/                     extension frontend (TypeScript)
  extension.ts           entry point: registers commands, creates side bar / status bar / tabs
  sidebar.ts             side bar cards (sections + tile definitions)
  welcome.ts             the "Manage Resources" TreeView
  actions.ts             implementations of all commands
  api.ts                 HTTP client (mirrors the backend /api/*)
  backend.ts             backend process management (spawn / port / key / venv)
  statusbar.ts           status bar items
  i18n.ts                frontend strings (zh/en)
  *Panel.ts              webview tabs (connection / AT / device tree / device info / performance / SDK info)
  webview/               webview static assets (HTML/CSS/JS)
python/
  server.py              FastAPI entry point (prints READY port=<port> key=<key>)
  quecpi/                implementation: api.py routes / sdk_ops.py SDK operations / i18n.py backend strings
  requirements.txt       backend dependencies
tools/                   bundled binaries: Windows + macOS adb, and a patched QEMU for Apple Silicon builds
images/                  screenshots used by this document
```

## 3. Development environment

| Dependency | Version | Notes |
| --- | --- | --- |
| Node.js | ≥ 18 to compile, ≥ 20 to package | TypeScript compilation and vsce |
| Python | 3.9+ (3.12 recommended) | FastAPI backend; the venv is created automatically on first start |
| Git | any | the backend runs local git commands |

```bash
npm install                                   # 1. install TS dependencies
npm run compile                               # 2. compile TypeScript (into out/)
python3 -m venv python/.venv                  # 3. create the Python venv (optional, done automatically)
python/.venv/bin/python -m pip install -r python/requirements.txt
python/.venv/bin/python python/server.py      # 4. (optional) run the backend standalone to debug it
```

On success the backend prints `READY port=<port> key=<key>`; the extension uses that line to talk to it.

## 4. Build / package / install

```bash
npm run package        # recommended: wraps the vsce command below
# or
npx vsce package --no-dependencies --allow-missing-repository --skip-license --no-rewrite-relative-links
```

Two packaging details worth knowing:

- `--allow-missing-repository` is required because `package.json` has no `repository` field.
- `--no-rewrite-relative-links` keeps the relative link of this document (`README.zh-CN.md`) as it is. The screenshots are absolute URLs hosted in the `quectel-developer/qpi-vscode-extension-doc` repository, so vsce no longer needs a repository field to resolve image paths.

The result is `quectelpi-<version>.vsix`, installed through **Install from VSIX...** in VS Code.

One-shot build and install (rewrites the `version` in `package.json`, compiles, packages, uninstalls the old build and installs the new one):

```bash
build-install.bat <version>       # Windows
./build-install.sh <version>      # Linux / macOS
```

The version argument is optional (you are prompted) and accepts `vX.Y.Z` or `X.Y.Z`.

## 5. Debugging

Press **F5** in VS Code to launch the Extension Development Host. It starts the backend and opens the side bar; breakpoints work in both `src/*.ts` and `python/quecpi/*.py`.

## 6. Two conventions

**Localization.** Frontend strings live in `src/i18n.ts`, backend strings in `python/quecpi/i18n.py`. Switching language rebuilds the side bar, the status bar and every open tab; requests carry an `X-QPi-Lang` header and the backend answers in the matching language. Note that the `%key%` titles in `package.json` are resolved once when the extension loads, so a runtime switch must reset them in code (for example `view.title`).

**Packaging notes.** `tools/fix_vsix.py` and `tools/sshrun.py` are encrypted binaries (not plain Python); they cannot be executed and are not referenced by any script. `npm run package` used to call `fix_vsix.py`, which failed with `SyntaxError: source code cannot contain null bytes`; that step is gone. The package ships no flash tools: the Rockchip host tools come with the SDK under `tools/flash/`, declared by `FLASH_TOOLS_*` in the SDK's `tools/VERSION.txt`, and the extension references them in place. What *is* bundled is whitelisted one directory at a time in `.vscodeignore` (`tools/**` is otherwise excluded):

- **Windows `adb`** (exe + 2 DLLs), staged to `%LOCALAPPDATA%\QuectelPi\adb\` when Windows has no `adb` in `PATH`;
- **macOS `adb`** (a single universal Mach-O binary), executed in place from the extension so Mac users need no `platform-tools`;
- **a patched QEMU 9.x** (`tools/qemu/linux-arm64/qemu-x86_64`, an arm64 static-pie binary) that is pushed into the `colima` VM on Apple Silicon so the `--platform linux/amd64` build no longer dies with `QEMU internal SIGSEGV` (see Part 1, section 7).
