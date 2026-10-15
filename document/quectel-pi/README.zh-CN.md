# Quectel Pi

[English](https://github.com/quectel-developer/qpi-vscode-extension-doc/blob/master/document/quectel-pi/README.md) | **简体中文**

VS Code 插件，把 **Quectel Pi 系列** SDK 的整套流程搬进图形界面：下载 SDK → 连接 → 配置构建环境 → 编译 → 连接开发板 → 烧录 → 调试。

支持的 SDK：

| SDK | 芯片 | 构建体系 | 产物与烧录方式 |
| --- | --- | --- | --- |
| **M2 SDK** | RK3576 | 原生 `build.sh` | `update.img`，用烧录工具写入 |
| **H1 Simple SDK** | QCS6490 | 原生 `build.sh` | `efi.bin` + `dtb.bin` + `system.img`，EDL 全盘烧录 |
| **H1 Yocto SDK** | QCS6490 | Yocto / BitBake | firmware 目录，用 SDK 自带 `flash.sh` 烧录 |

插件会根据你连接的是哪套 SDK 展示相应的操作按钮

---

# 第一部分 · 用户指南

> 这一部分写给使用者。所有操作都在 VS Code 里完成。

## 1. 开始之前（需要准备）

### 宿主机（安装 VS Code 插件的电脑）

SDK 的编译、烧录与设备操作都在这台机器上完成（或由它经 SSH 转发到远程编译机）。插件支持以下三种宿主机环境。

**Windows**

- **系统**：Windows 10 / Windows 11，需安装 **WSL2** 子系统（编译在 Docker 容器里进行，WSL1 不可用）。
- **SDK 位置**：SDK 源码**必须放在 WSL 内**（例如 `/home/<用户名>/qpi-sdk`），因为 SDK 的构建脚本是 Linux shell 脚本。
- **内存**：建议 16 GB 及以上（WSL2 默认最多只使用宿主机一半内存，内存过小时编译可能触发内核 OOM）。
- **磁盘**：M2 / H1 Simple SDK 下载到构建需要几十 GB；**完整 Yocto 下载到构建需要几百 GB**。

**Ubuntu（Linux）**

- **系统**：推荐 **Ubuntu 22.04 或 24.04**（烧录工具 qdl 依赖 glibc ≥ 2.34 与 libzip.so，Ubuntu 20.04 及更早版本不满足）。
- **SDK 位置**：直接放在本机目录即可。
- **内存**：建议 16 GB 及以上。
- **磁盘**：M2 / H1 Simple SDK 下载到构建需要几十 GB；**完整 Yocto 下载到构建需要几百 GB**。

**macOS**

- **系统**：推荐 **macOS 13 Ventura 或更高**；要求 ARM 架构。
- **SDK 位置**：直接放在 Mac 里即可。编译不会在 Mac 上直接进行，而是在插件自动准备的轻量 Linux 虚拟机（`colima`）内的 Docker 容器里完成（见第 7 节）。
- **内存**：建议 16 GB 及以上（其中 8 GB 归虚拟机使用）。
- **磁盘**：虚拟机磁盘 100 GB（稀疏，按需增长），此外还要留给 SDK 与编译产物；**完整 Yocto 构建需要更多**。

### 开发板

配套的开发板有 **Quectel Pi M2** 与 **Quectel Pi H1** 两块，编译出的镜像最终要烧录到开发板上运行。使用前请用一根 USB 数据线把开发板连到对应机器；
- **Quectel Pi M2** —— 主控瑞芯微 **RK3576**。
- **Quectel Pi H1** —— 主控高通 **QCS6490**。

### 你还需要准备

- **VS Code**：版本 1.85 或更高。
- **SDK 源码**：可以在插件里直接下载（见下方步骤），也可以使用你已有的 SDK 目录。
- **USB 驱动**：Windows 下需要安装 USB 驱动，才能识别开发板，驱动获取方式见下方
- **更多信息**：可在插件内点击管理资源——文档指南 跳转官网获取驱动等更多信息

![点击管理资源——文档指南，可跳转官网获取驱动等更多信息](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/information_more_1.png)

可选：一台**远程 Linux 编译机**。如果你在服务器上构建，就在自己的电脑上装 VS Code 和本插件，通过 SSH 连接过去——见第 6 节。


## 2. 安装插件

1. 打开 VS Code，点击活动栏的 **扩展** 图标（或按 `Ctrl+Shift+X`）。
2. 搜索 **Quectel Pi**，点击 **安装**。
3. 活动栏会出现 **Quectel Pi** 图标——点它打开工具面板。

![在扩展市场搜索 Quectel Pi 并点击安装](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/download_extension1.png)


第一次打开面板时，插件会在后台准备自己的运行环境。在连接 SDK 之前，面板头部的状态标签一直显示 **未连接**。

> **语言**：界面支持中文和英文。点击面板头部的 🌐 按钮即可切换。默认跟随你的 VS Code 显示语言，一旦手动切换过，就会记住你的选择。

## 3. 界面速览

所有功能分布在三个地方（有些SDK不具备的功能会隐藏）：

| 位置 | 有什么 |
| --- | --- |
| **侧边栏**（Quectel Pi 图标） | 主界面：可折叠的卡片分组——连接 / SDK、应用管理、编译、烧录 & 设备、日志 / 调试、多媒体、AI 助手 |
| **头部按钮** | 连接状态右侧的三个按钮：**刷新**（重新检查连接与 SDK 平台）、**重启后端**（辅助服务卡住时用）、**切换语言** |
| **状态栏**（窗口底部） | 五个一键入口：连接状态、编译应用、烧录 Boot、全量烧录、dmesg（实时内核日志） |

在卡片分组下方，同一个侧边栏还有 **管理资源** 视图——五个开工前要用的入口：**文档指南**（选择英文或中文官网）、**管理 SDK**、**打开已有应用**、**示例工程**（打开 Quectel-Pi 的 GitHub 组织页）、**打开终端**。

有些内容会以**独立标签页**的形式在编辑器区域打开：连接表单、AT 指令台、设备信息页、性能监控、SDK 信息页以及 Yocto 构建配置。每个标签页的名字和打开它的卡片一致。

连接状态每 5 秒刷新一次，所以插上开发板会被自动识别。

## 4. 走一遍完整流程：五步从编译到烧录

这是最短的完整路径，从一台空电脑到开发板跑起你自己编译的系统。

### 步骤 1 · 下载 SDK

在侧边栏找到 **管理资源**，点击 **管理 SDK**，然后选择 SDK 家族：

![选择 SDK 版本 —— M2 或 H1](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step1-select-sdk.png)

*先选 SDK 。如果选 H1，会再问一次要 **Simple SDK** 还是 **Yocto SDK**；M2 只有一种。*

接着会问你保存到哪里：

![输入本地保存路径](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step1-save-path.png)

*输入一个绝对路径后按 `Enter`——SDK 会在一个专用终端里 `git clone` 到该目录，进度实时可见。关闭那个终端即中止下载。*

- Linux / WSL 下可以用类似 `/home/<用户名>/qpi-sdk` 的路径。
- Windows 下路径必须在 WSL 内（例如 `/home/<用户名>/qpi-sdk`），见第 6 节。
- 下载视网络情况可能需要一段时间。

### 步骤 2 · 连接 SDK

点击 **连接 / SDK** 分组里的 **连接 SDK**，编辑器区域会打开一个表单：

![连接 QPi SDK 表单](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step2-connect-sdk.png)

*模式保持 **本机**，填入 SDK 根目录（就是你刚克隆下来的那个文件夹）。如果 SDK 在远程编译机上，改选 **SSH 远程** 并填写主机 / 用户 / 密码。点击 **连接**。*

连接成功后，面板头部的状态标签会从 **未连接** 变为 SDK 类型和路径（例如 `Local @ /home/will/qpi/h1`），**编译** 与 **烧录 & 设备** 分组里的按钮也会自动适配识别到的 SDK。

连接配置会被记住，下次启动 VS Code 时插件会自动重连。

### 步骤 3 · 配置构建环境

点击 **编译** 分组里的 **配置构建环境...**：

![配置构建环境，以及它的终端日志](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step3-setup-env.png)

*这一步会先准备 Docker 编译环境，再在容器里跑 SDK 自带的环境脚本。进度会实时输出到终端。*

1. 检查 Docker。没有就自动安装——Linux/WSL 装 Docker Engine；**macOS** 上下载 `colima`（一个轻量 Linux 虚拟机，不用 Homebrew、不用 `sudo`，即便装了 Docker Desktop 也不会用它）。
2. 用 SDK 自带的 `docker_env/Dockerfile` 构建编译镜像。镜像里预置了**全部**编译依赖，所以宿主机不需要这些依赖。
3. 在这个容器里执行 SDK 的 `build.sh setenv`：补齐/校验依赖，并下载预编译的底包镜像。

- Linux/WSL 下会要你输入一次登录密码（安装 Docker、构建镜像都要 `sudo`）。**macOS 下全程不需要密码**。
- 首次会下载基础镜像，可能要几分钟。
- 每套 SDK 跑一次即可；但如果构建时报缺工具，就再跑一次。
- 之后的每次编译都复用同一个镜像，宿主机的工具链环境无关紧要。macOS 上的具体差异见第 7 节。

### 步骤 4 · 编译

点击 **编译** 分组里的 **全量编译**：

![正在编译，日志在终端里滚动](https://raw.githubusercontent.com/quectel-developer/qpi-vscode-extension-doc/master/images/quectel-pi/step4-build-all.png)

*编译在独立终端里进行，完整日志实时输出。Yocto SDK 的首次构建耗时很长，之后是增量构建。*

- 编译在步骤 3 准备好的 **Docker 容器内**进行。你的 SDK 目录会以相同绝对路径挂进容器，所以产物（例如 `update.img`）直接落在你的 SDK 目录里，烧录工具能直接读取。
- 也可以只编译 bootloader（**编译 Boot**）；在 Yocto SDK 上则可以把 bootloader 和内核一起编译（**编译内核**）。
- 想从零开始构建时，用 **清理产物** 删除中间产物。
- 关闭终端即中止编译——想停掉一次漫长的构建时很好用。

### 步骤 5 · 连接开发板并烧录

1. 用 USB 线把开发板连到电脑。
2. 点击 **设备信息**，确认开发板已被识别（型号、OS 版本、内存、存储）。
3. 让开发板进入下载模式：
   - M2：点击 **进 Loader**
   - H1：点击 **进 EDL**（高通 9008 下载模式）
4. 点击 **全量烧录**，等待完成。M2 会让你选烧录工具，无需另外安装。
5. 点击 **重启设备**。开发板就会启动到你刚编译好的系统。

## 5. 三套 SDK 详解

### 5.1 M2 SDK（RK3576）

1. **下载**：管理资源 → 管理 SDK → **M2** → 绝对路径 → `Enter`。
2. **连接**：连接 SDK，模式选 **本机**（或在服务器上构建时选 **SSH 远程**）。
3. **配置构建环境...**：准备 Docker 编译镜像并下载预编译底包镜像。
4. **全量编译**：产出 `update.img`。**编译 Boot** 只编译 bootloader。
5. **设备信息**：用 USB 插上开发板，确认已被识别。
6. **进 Loader**：让开发板进入 Rockchip loader 模式。
7. **全量烧录**：选择烧录工具（`upgrade_tool` / `rkdeveloptool` / `RKDevTool`）并写入 `update.img`；也可以单独用 **烧录 Boot** 写 boot 分区。
8. **重启设备**。

### 5.2 H1 Simple SDK（QCS6490）

1. **下载**：管理资源 → 管理 SDK → **H1** → **H1 Simple SDK** → 绝对路径 → `Enter`。
2. **连接**：同上。
3. **配置构建环境...**：准备 Docker 编译镜像并下载预编译底包镜像。
4. **全量编译**：产出各分区镜像；**编译 Boot** 只编译 `efi.bin` / `dtb.bin`。
5. **设备信息**：用 USB 插上开发板，确认已被识别。
6. **进 EDL**：让开发板进入高通 EDL 9008 下载模式。
7. **全量烧录**：运行 SDK 自带的 `flash.sh` 写整盘。高通平台不支持单分区烧录。
8. **重启设备**。

### 5.3 H1 Yocto SDK（QCS6490）

1. **下载**：管理资源 → 管理 SDK → **H1** → **H1 Yocto SDK** → 绝对路径 → `Enter`。
2. **连接**：同上。
3. **配置构建环境...**：你需要填写项目 ID、操作系统（`LINUX` / `UBUNTU` / `DEBIAN`）和版本（`STD` 发布 / `DBG` 调试），还可勾选安全启动；面板会保存这些参数，之后每次构建自动套用（编译全部在容器内进行，宿主机不安装 SDK 依赖）。同一步骤会检测 Docker / `colima` 状态，若编译镜像尚未就绪则一并准备 Docker 编译环境（Linux/WSL 下会要你输入一次 `sudo` 密码——macOS 上无需密码）。
4. **编译**：
   - **全量编译**：产出用于烧录的 firmware 目录。
   - **编译内核**：产出 `dtb.bin`（bootloader）和 `efi.bin`（内核）。比全量构建快得多。
5. **设备信息**：用 USB 插上开发板，确认已被识别。
6. **烧录**：
   - **全量烧录**：先进入 EDL，然后把固件写入整盘。
   - **烧录内核**：不需要 EDL——内核产物直接写入 DTB 和 ESP 分区。如果产物还不存在，会提示你先编译。
7. **重启设备**。

## 6. 连接方式

| 插件安装环境 | 模式 | SDK 在哪里 |
| --- | --- | --- |
| **Linux** | 本机 | 这台 Linux 机器上的某个目录——最常见的情况 |
| **Linux** | SSH 远程 | 一台远程 Ubuntu / Linux 编译机 |
| **macOS** | 本机 | Mac（Apple Silicon 或 Intel）上的某个目录——编译通过 `colima` 的 Docker 虚拟机进行（见第 7 节） |
| **macOS** | SSH 远程 | 一台远程 Ubuntu / Linux 编译机——Apple Silicon 上最快的选择 |
| **Windows** | 本机 | **在 WSL 内**（例如 `/home/<用户>/qpi-sdk`）——构建脚本需要 Linux |
| **Windows** | SSH 远程 | 一台远程 Ubuntu / Linux 编译机；你的电脑只作为界面 |

### 6.1 本机连接（SDK 就在装插件的这台机器上）

**Linux 本机** —— 最常见的情况。

- 打开 **连接 SDK**，模式保持 **本机**，填入 SDK 根目录的绝对路径（含 `build.sh` 的那一层），点 **连接** 即可。
- 板子用 USB 接本机时本机需要有 `adb`。原生 Linux 上若缺失，插件会提示并可直接代为安装；也可以自己先装好：

  ```bash
  sudo apt-get install -y android-tools-adb
  ```

- 烧录用的 qdl 依赖 `glibc ≥ 2.34` 与 `libzip.so`，所以宿主机推荐 **Ubuntu 22.04 / 24.04**（20.04 及更早不满足）。
- 不确定环境是否齐全时，在命令面板运行 **QPi SDK: 环境自检**，它会列出缺失的组件。

**Windows（SDK 在 WSL 内）**

- 前置 1：Windows 上装好 **WSL2**，并把 SDK 放在 WSL 内的目录（如 `/home/<用户>/qpi-sdk`）。
- 前置 2：**让 WSL 启用 systemd**。编译用的 Docker 在 WSL 内是 systemd 服务，未启用时 `docker` 装得上却起不来。编辑 WSL 里的 `/etc/wsl.conf`：

  ```ini
  [boot]
  systemd=true
  ```

  保存后回到 Windows 执行 `wsl --shutdown`，再重新进入 WSL。
- 前置 3：保持 WSL 的 **Windows 互操作（interop）** 开启（默认即开启）。板子的 USB 是接在 Windows 上的，插件通过 interop 调用 Windows 侧的 `adb.exe` 连板子——即使 Windows 没装 platform-tools，插件也内置了一份 `adb`。
- 连接：模式选 **本机**，SDK 根目录填 **WSL 内的路径**（如 `/home/<用户>/qpi-sdk`）或共享路径（`\\wsl.localhost\<发行版>\...`），**不要用 `C:\...`**。插件会自动进入 WSL 执行编译等操作。
- 注意：WSL2 默认最多只用宿主机一半内存，内存偏小时编译更慢甚至触发 OOM；可在 Windows 的 `%UserProfile%\.wslconfig` 里调整内存上限。

**macOS 本机**

- 连接：模式选 **本机**，填入 Mac 上的 SDK 绝对路径。
- 不需要额外准备：编译由插件自动装好的 `colima` 虚拟机 + Docker 完成（见第 7 节）；连板子用的 `adb` 插件已内置（universal 二进制），无需安装 platform-tools。
- Apple Silicon 上全量编译偏慢，若经常做全量构建，建议改用下面的 SSH 远程。

### 6.2 SSH 远程连接（SDK 在远程 Linux 编译机上）

- 打开 **连接 SDK**，模式选 **SSH 远程**，依次填写主机 IP / 域名、端口（默认 22）、用户名、认证方式（密码或私钥文件），再填 **远端 SDK 根目录**的绝对路径。
- **最容易忽略的一步是「设备连接位置」**。表单里有一栏 **设备连接位置**，它决定 adb / AT / 烧录 / 部署这些设备操作到底在哪台机器上执行：

  - **板子接在本机**（默认）：板子用 USB 插在你自己的电脑上。设备操作在本机执行；若固件产生在远端编译机上（例如 H1 的 `flash.sh`），插件会**先把固件拉取到本机临时目录**再烧录。本机需要有可用的 `adb`（Windows / macOS 有内置 adb，Linux 需自行安装）。
  - **板子接在远端**：板子用 USB 插在远程编译机上。所有设备操作都在远端执行，远端**必须自己装好 adb**（插件不会代远端安装）：

    ```bash
    sudo apt install android-tools-adb
    ```

    否则插件会提示「远端编译机未找到 adb 命令」。

- 选错这一栏的典型症状：板子明明插在本机，却选了「板子接在远端」——命令跑到远端机器上去找设备，自然找不到。
- 其余注意：远端机器上的 SDK 同样需要先跑一次 **配置构建环境...**；连接失败时插件会在终端里给出连接日志，可据此排查路径或凭据问题。


## 7. macOS 上的编译

在 macOS 上，编译**不会**直接在 Mac 上跑。插件会下载一个轻量 Linux 虚拟机（`colima`），并在该虚拟机里的 Docker 容器内编译。工具链、SDK 的构建依赖和脚本全都是 Linux 的，因此 Mac 本身不需要额外装任何东西。

### 插件会自动准备什么

首次点击 **配置构建环境...** 时，它会把下面这些下载到 `~/.qpi`：

- `colima` + `lima`（虚拟机管理器），以及 `docker` CLI + `buildx`；
- 一个基础虚拟机镜像（Ubuntu 24.04）——首次下载约几百 MB；
- 然后在虚拟机里构建 SDK 的 `quectel-pi-builder` 编译镜像。

虚拟机按 **4 核 CPU、8 GB 内存、100 GB 磁盘**创建。重复点击 **配置构建环境...** 是幂等的——只补缺的部分。

> 插件有意忽略 Docker Desktop：即便你装了它，插件也始终通过自己的 `colima` 虚拟机来驱动 Docker（docker context 固定为 `colima`）。你不需要 Docker Desktop，不装也可以。

### Apple Silicon 与 Intel 的差异

| Mac | 实际情况 | 影响 |
| --- | --- | --- |
| **Intel**（x86_64） | 容器与宿主机架构一致，原生化运行 | 正常编译速度 |
| **Apple Silicon**（arm64） | SDK 只提供 x86_64 工具链，因此镜像按 `linux/amd64` 构建，并在 QEMU 模拟下运行 | 能编过，但明显比 Intel Mac 或原生 Linux 机器**慢** |

在 Apple Silicon 上，插件还会修复一个已知的 QEMU 缺陷：虚拟机自带的模拟器一旦遇到编译工具读取 `/proc/self/maps`（内核的 `grep` / `flex` / 汇编器探测都会读）就会崩溃（`QEMU internal SIGSEGV`）。插件会往虚拟机里装入一个打过补丁的 QEMU 9.x 解释器，并为 x86_64 二进制注册它，于是编译能正常跑完，而不会以退出码 2 失败。这一步是自动的——你会在环境日志里看到 `[QPi] ... QEMU ...` 字样。

由于模拟开销，Apple Silicon 上的全量编译耗时很长。如果你经常做全量构建，建议改用 **SSH 远程** Linux 编译机（见第 6 节）——繁重的编译会在远端原生化执行。

### 想在 macOS 上编译成功，需要满足这些环境

| 要求 | 说明 |
| --- | --- |
| macOS 版本 | 较新的 macOS（**推荐 macOS 13 Ventura 或更高**）。**Apple Silicon** 与 **Intel** Mac 都支持 |
| 可用磁盘 | 虚拟机磁盘为 100 GB（稀疏，按需增长），此外还要留给 SDK 与编译产物。M2 / H1 Simple SDK 建议预留几十 GB，**完整 Yocto 构建还要再多几十 GB** |
| 内存 | 其中 8 GB 归虚拟机使用，因此建议 Mac 内存 **16 GB** 及以上 |
| 网络 | 首次 **配置构建环境...** 需要联网——要下载 `colima`、基础虚拟机镜像和编译镜像 |
| 其他 | 不需要 Homebrew、不需要 Docker Desktop、不需要工具链，也不需要 `sudo` 密码 |
| 仅在烧录时 | 只有开发板需要时，才装它的 USB/串口驱动。插件内置了 macOS 版 `adb`，所以无需安装 `platform-tools` |

## 8. 功能详解

按分组逐条说明每个卡片能做什么。

### 连接 / SDK

- **连接 SDK** —— 连接一个 SDK（本机或 SSH）。表单见步骤 2。
- **断开** —— 断开当前连接，不会删除任何东西。
- **刷新设备** —— 重新扫描通过 USB 连接的开发板。
- **SDK 信息** —— 打开一个标签页，显示当前分支、当前提交、你的分支是否落后于远端，以及最近的提交记录。
- **后端信息** —— 打开一个标签页，显示插件辅助服务的地址与状态。看起来不对劲时很有用。
- **环境引导** —— 打开环境引导：检查 SDK 目录下的 8 个组件（核心 SDK、内核源码、构建工具、预编译资源、交叉工具链、工程模板、SKILL 文档、设备树 Overlay），告诉你缺哪些，并支持从仓库克隆补齐。

### 应用管理

- **创建应用** —— 向导式：选模板 → 填写模板变量（有下拉的必须从下拉里选）→ 给工程命名。字母、数字、下划线、连字符，1–64 个字符。工程创建在 SDK 的 `projects/` 目录下。
  - 模板来自 SDK 本身（`docs/templates/`）。如果你的 SDK 没有附带模板，向导会明确提示并终止。
- **应用列表** —— 列出 `projects/` 下的每个工程，各自带 *打开目录 / 编译 / 删除*（删除会要求确认）。
- **SKILL 文档** —— SDK `skills/` 目录下随 SDK 附带的文档，在 VS Code 标签页中打开。

### 编译

- **配置构建环境...** —— 步骤 3 描述的一次性（或更新后的）环境准备：准备 Docker（macOS 上下载 `colima`）、构建编译镜像，然后在容器里执行 SDK 的 `setenv`。Yocto SDK 上则改为打开构建配置面板，并按需准备 Docker 环境。
- **全量编译** —— 完整构建，产出可用于烧录的镜像。
- **编译 Boot** / **编译内核** —— 两者只会出现一个，取决于 SDK。Simple SDK 上编译 bootloader；Yocto SDK 上把 bootloader 和内核一起编译。
- **清理产物** —— 删除构建中间产物，让下次构建从零开始。

> **编译应用**（状态栏里的 *编译应用* 入口）会列出 SDK `projects/` 目录下已有的工程并编译你所选的那个。如果还没有工程，会提示你先用 **创建应用** 建一个。Yocto SDK 不参与应用编译：应用请改用 BitBake 构建。

### 烧录 & 设备

- **全量烧录** —— 写整盘。M2 会让你选工具（工具来自 SDK）；H1 运行 SDK 的 `flash.sh`。
- **烧录 Boot** / **烧录内核** —— 与 *编译 Boot / 编译内核* 对应。高通平台上无法烧写单个分区，所以 *烧录 Boot* 会提示你改用 **全量烧录**；在 Yocto SDK 上 *烧录内核* 会把内核产物写入 DTB 和 ESP 分区。
- **进 Loader / 进 EDL** —— 让开发板进入下载模式（M2 → Loader，H1 → EDL 9008）。
- **部署文件** —— 把电脑上的一个文件复制到开发板的某个目录（默认 `/userdata/<文件名>`）。
- **设备信息** —— 型号、OS 版本、内存、存储、网络等等。
- **重启设备** —— 重启开发板。

### 日志 / 调试

- **adb 命令** —— 一个已经连上开发板的终端，可以直接敲 `adb shell` 命令。
- **dmesg** —— 实时内核日志。
- **journalctl** —— 实时系统日志。
- **AT 指令** —— 串口控制台：选择端口和波特率，与模块收发 AT 指令。可以走开发板自己的串口、电脑的串口，或远程编译机的串口。
- **Shell** —— 在连上开发板的终端里执行 shell 命令。
- **音频诊断** —— 收集音频日志与配置，用于排查问题。

### 多媒体

- **音频播放** / **视频播放** —— 播放开发板上的文件。
- **录屏** / **截图** —— 抓取开发板屏幕。开发板需要接了显示器并已登录桌面会话。
- **性能监控** —— 一个实时曲线标签页，通过 adb 展示帧率、CPU 占用与频率、GPU 占用与频率以及温度。采样间隔可调，随时可暂停。开发板未暴露的指标显示为「无数据」。

### AI 助手

- **AI 助手** —— 打开 GitHub Copilot Chat（未安装时会询问是否安装）。
- **API 密钥设置** —— 打开用于接入第三方 AI 服务（Base URL / API Key）的设置页。

## 9. 常见问题

**面板一直显示「未连接」。**
点击 **连接 SDK** 并填入 SDK 目录。首次连接成功后插件会记住配置并自动重连。

**它向我要密码。**
安装 Docker、构建镜像（以及 Linux 上的烧录）都需要存放 SDK 那台机器的管理员权限。请输入你的登录密码；如果配置了免密 `sudo`，直接按 `Enter` 即可。**macOS 下全程不需要密码**：环境准备会下载 `colima` 并在无 `sudo` 的情况下完成。

**macOS 上需要我装 Docker 或 Docker Desktop 吗？**
不需要。首次 **配置构建环境...** 时，插件会把 `colima`（它自己的轻量 Linux 虚拟机）和 docker CLI 下载到 `~/.qpi`——不用 Homebrew、不用 `sudo`。即便装了 Docker Desktop 也会被忽略。见第 7 节。

**为什么我的 Apple Silicon Mac 编译这么慢？**
SDK 只提供 x86_64 工具链，所以容器按 `linux/amd64` 构建并在 QEMU 模拟下运行。它能编过（插件修复了会导致编译中断的 QEMU 缺陷），但全量编译耗时很长。如果你经常做全量构建，建议改用 **SSH 远程** Linux 机器（见第 7 节）。

**提示「未找到可用的烧录工具」。**
Rockchip 的烧录工具随 SDK 分发（在 `tools/flash/` 下，由 SDK 的 `tools/VERSION.txt` 里的 `FLASH_TOOLS_*` 声明），所以先确认你的 SDK 是否完整。否则请在执行烧录的那台机器上安装它们（`upgrade_tool`、`rkdeveloptool`）——插件会告诉你它找过哪些、以及找过哪里。

**怎么在中英文之间切换？**
面板头部的 🌐 按钮。侧边栏、状态栏以及所有已打开的标签页会立即切换。

**怎么改插件本身的设置？**
在 VS Code 设置里搜索 `qpiSdk`：

| 设置项 | 默认值 | 说明 |
| --- | --- | --- |
| `qpiSdk.backend.python` | 空（自动探测） | 辅助服务使用的 Python 解释器路径 |
| `qpiSdk.backend.autoStart` | true | 随 VS Code 一起启动辅助服务 |
| `qpiSdk.backend.mode` | local | `local` = 辅助服务在本机；`remote` = 使用远端机器上的辅助服务 |
| `qpiSdk.backend.remoteUrl` | http://127.0.0.1:8765 | `remote` 模式下辅助服务的地址 |
| `qpiSdk.backend.remoteKey` | 空 | `remote` 模式下辅助服务的密钥 |
| `qpiSdk.backend.pythonMirror` | 空（按网络自动选择） | 下载内置 Python 运行时所用的镜像 |
| `qpiSdk.backend.pipMirror` | 空（按网络自动选择） | 安装辅助服务依赖所用的包索引 |
| `qpiSdk.download.proxy` | 空（直连） | 所有下载统一使用的代理，如 `http://127.0.0.1:7890` 或 `socks5://127.0.0.1:7890` |
| `qpiSdk.download.mirror` | auto | 下载的镜像加速策略，见下表 |

`qpiSdk.download.mirror` 的取值：

| 取值 | 含义 |
| --- | --- |
| `auto` | 自动探测网络环境，决定用国内镜像还是官方源 |
| `cn` | 始终优先国内镜像（清华 / 中科大 / 阿里云 / 南大） |
| `global` | 始终优先官方源（GitHub / PyPI / download.docker.com） |
| `off` | 完全关闭镜像，只走官方源（需要时可配合代理） |

**代理与镜像的区别与配合**：`download.proxy` 是流量出口（代理所有下载——SDK 仓库、Python 运行时、pip、Docker 工具链、apt）；`download.mirror` 是源站选择。国内镜像本身走直连，因此两者可以并用（例如开了代理的同时仍用国内镜像提速）。

最直观的做法是用命令面板里的 **QPi SDK: 配置下载代理 / 镜像** 命令来设置这两项——它会引导你填写代理地址并选择镜像策略，比手改设置项省事。


## 10. 问题反馈

遇到问题、有疑问或者想提建议？欢迎提交 issue：

**https://github.com/quectel-developer/qpi-vscode-extension-doc/issues**

反馈时建议附上：你的操作系统（以及 SDK 是否在 WSL 内）、SDK 类型（M2 / H1 Simple / H1 Yocto）、你做了什么、期望的结果以及实际现象。最有用的材料是执行任务那个终端里的日志。

---

# 第二部分 · 开发与维护

> 这一部分写给插件的开发者和维护者。

## 1. 架构

```
侧边栏 / 状态栏 / webview 标签页 ──postMessage──▶ extension.ts ──HTTP + SSE──▶ FastAPI (python)
                界面                                │                              │
                                            连接配置                         executor (local / SSH)
                                          (workspaceState)                        │
                                                              build.sh / bitbake / adb / qdl
```

- **前端**：TypeScript 插件 + webview（纯 HTML/CSS/JS，不依赖框架，自适应亮色与暗色主题）
- **后端**：Python FastAPI，监听随机 `127.0.0.1` 端口；每个请求都必须携带 `X-API-Key`；虚拟环境及其依赖在首次启动时创建
- **执行器**：本机（子进程）或通过 SSH 连接到远程 Linux 编译机（paramiko）
- **任务模型**：每个编译 / 烧录 / 日志任务都会打开一个专用的 VS Code 终端并通过 SSE 输出流；关闭终端会杀掉后端任务

## 2. 目录结构

```
src/                     插件前端（TypeScript）
  extension.ts           入口：注册命令，创建侧边栏 / 状态栏 / 标签页
  sidebar.ts             侧边栏卡片（分组 + 磁贴定义）
  welcome.ts             "管理资源" TreeView
  actions.ts             所有命令的实现
  api.ts                 HTTP 客户端（对应后端 /api/*）
  backend.ts             后端进程管理（spawn / 端口 / 密钥 / venv）
  statusbar.ts           状态栏项
  i18n.ts                前端文案（中/英）
  *Panel.ts              webview 标签页（连接 / AT / 设备树 / 设备信息 / 性能 / SDK 信息）
  webview/               webview 静态资源（HTML/CSS/JS）
python/
  server.py              FastAPI 入口（打印 READY port=<port> key=<key>）
  quecpi/                实现：api.py 路由 / sdk_ops.py SDK 操作 / i18n.py 后端文案
  requirements.txt       后端依赖
tools/                   随包分发的二进制文件：Windows + macOS 版 adb，以及 Apple Silicon 编译用的补丁版 QEMU
images/                  本文档使用的截图
```

## 3. 开发环境

| 依赖 | 版本 | 说明 |
| --- | --- | --- |
| Node.js | 编译需 ≥ 18，打包需 ≥ 20 | TypeScript 编译与 vsce |
| Python | 3.9+（推荐 3.12） | FastAPI 后端；venv 首次启动时自动创建 |
| Git | 任意 | 后端会执行本地 git 命令 |

```bash
npm install                                   # 1. 安装 TS 依赖
npm run compile                               # 2. 编译 TypeScript（输出到 out/）
python3 -m venv python/.venv                  # 3. 创建 Python venv（可选，会自动创建）
python/.venv/bin/python -m pip install -r python/requirements.txt
python/.venv/bin/python python/server.py      # 4.（可选）单独运行后端以便调试
```

成功后后端会打印 `READY port=<port> key=<key>`；插件用这一行来与它通信。

## 4. 构建 / 打包 / 安装

```bash
npm run package        # 推荐：封装了下面的 vsce 命令
# 或者
npx vsce package --no-dependencies --allow-missing-repository --skip-license --no-rewrite-relative-links
```

有两个打包细节值得了解：

- 需要 `--allow-missing-repository`，因为 `package.json` 里没有 `repository` 字段。
- `--no-rewrite-relative-links` 会保持本文档中的相对链接（`README.zh-CN.md`）不变。截图已改为托管在 `quectel-developer/qpi-vscode-extension-doc` 仓库中的绝对 URL，因此 vsce 不再需要仓库地址来解析图片路径。

产物是 `quectelpi-<version>.vsix`，通过 VS Code 的 **从 VSIX 安装...** 安装。

一键构建并安装（改写 `package.json` 里的 `version`、编译、打包、卸载旧版本并安装新版本）：

```bash
build-install.bat <version>       # Windows
./build-install.sh <version>      # Linux / macOS
```

版本号参数可选（不填会提示输入），接受 `vX.Y.Z` 或 `X.Y.Z`。

## 5. 调试

在 VS Code 中按 **F5** 启动扩展开发宿主。它会启动后端并打开侧边栏；`src/*.ts` 和 `python/quecpi/*.py` 里都能正常命中断点。

## 6. 两个约定

**本地化。** 前端文案在 `src/i18n.ts`，后端文案在 `python/quecpi/i18n.py`。切换语言会重建侧边栏、状态栏以及所有已打开的标签页；请求会携带 `X-QPi-Lang` 头，后端以对应语言应答。注意 `package.json` 里的 `%key%` 标题在扩展加载时只解析一次，所以运行时切换语言必须在代码里重置它们（例如 `view.title`）。

**打包注意事项。** `tools/fix_vsix.py` 和 `tools/sshrun.py` 是加密后的二进制（不是明文 Python），无法执行，也没有被任何脚本引用。`npm run package` 曾经会调用 `fix_vsix.py`，但它会以 `SyntaxError: source code cannot contain null bytes` 失败；这一步已经移除。包内不再分发烧录工具：Rockchip 的宿主机烧录工具由 SDK 随包分发在 `tools/flash/`，通过 SDK 的 `tools/VERSION.txt` 里的 `FLASH_TOOLS_*` 声明，扩展直接就地引用。真正随包分发的文件在 `.vscodeignore` 里逐目录放行（`tools/**` 默认排除）：

- **Windows 版 `adb`**（exe + 2 个 DLL），当 Windows 的 `PATH` 里没有 `adb` 时暂存到 `%LOCALAPPDATA%\QuectelPi\adb\`；
- **macOS 版 `adb`**（单个 universal Mach-O 二进制），直接从扩展目录就地执行，Mac 用户因此无需安装 `platform-tools`；
- **补丁版 QEMU 9.x**（`tools/qemu/linux-arm64/qemu-x86_64`，arm64 static-pie 二进制），在 Apple Silicon 上注入 `colima` 虚拟机，使 `--platform linux/amd64` 编译不再以 `QEMU internal SIGSEGV` 中断（见第一部分第 7 节）。