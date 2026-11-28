# Luckfox Pico RKNN Example — Cloud Agent 环境镜像（自建 Ubuntu 24.04 方案 / environment.json 引用本文件）
#
# 目标：把开发环境「代码化」，不依赖任何人的个人快照（Personal snapshot）；任何人从带本文件的分支起 Cloud Agent，都能得到可复现的交叉编译环境：glibc 例程设置 `GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-` 后即可 cmake 构建（不设置 GLIBC_COMPILER 或 LUCKFOX_SDK_PATH 时 CMakeLists 提前 return、不产出任何 target，详见 AGENTS.md）。
#
# 说明：
# - Cloud Agent 本身就运行在「一个容器」里；本 Dockerfile 只是「声明这个容器长什么样」，并非 docker-in-docker。本仓安装 Tailscale，Agent Run 使用 kernel 模式（luckfox_pico_lvgl_example 仓 docs/superpowers/specs/2026-10-06-lvgl-example-cloudagent-align-luckfox-pico-design.md §3.2；理由见 luckfox-pico 08-30 spec §2.1）；Cursor 官方 userspace 路径只作回退。
# - 本仓是 RKNN 例程集（非 SDK）：3 个 RKNN 推理例程（人脸识别、目标检测，共 4 个构建变体），OpenCV-Mobile、rknnmrt 按 libc 预编译于 lib/{glibc,uclibc}，屏驱动源码在 lib/ 下，librga 等在 3rdparty/，无需系统库；
#     · glibc 交叉编译：ARM 交叉工具链【由本 Dockerfile apt 安装】（gcc/g++-arm-linux-gnueabihf），CMakeLists 走 $GLIBC_COMPILER 前缀。
#     · uClibc 交叉编译：工具链 arm-rockchip830 随 luckfox-pico SDK 仓库提供，不在本镜像中，取法见 AGENTS.md「项目与构建」；CMakeLists 走 $LUCKFOX_SDK_PATH。
# - 本仓有交互式 build.sh（选 libc、例程与板型），但它与 `make install` 都会改写已入库的 install/，`build.sh clean` 更会删除整个 install/；Agent 验证构建的推荐写法见 AGENTS.md。
# - 基于 ubuntu:24.04（贴合默认 Cloud Agent 与本机 24.04 LTS）：官方仅就 computer use 声明镜像须为 Debian/Ubuntu 系；本仓不做 computer use、也没有桌面 GUI，选 Ubuntu 基底是为贴合默认环境与 apt 生态，并与 luckfox_pico_rkmpi_example、luckfox_pico_lvgl_example 仓同构。
# - 不要 COPY 项目源码：Cursor 会自行 checkout 正确的 commit 到工作区。
# - FROM 同时锁定 tag 与 digest：tag（24.04）便于人读，digest 保证镜像内容字节级不可变。
#     获取/更新 digest：docker buildx imagetools inspect ubuntu:24.04（取 index digest）。当前对应 24.04.4 LTS。
FROM ubuntu:24.04@sha256:4fbb8e6a8395de5a7550b33509421a2bafbc0aab6c06ba2cef9ebffbc7092d90

# DEBIAN_FRONTEND=noninteractive：无人值守安装，避免 tzdata/keyboard 交互卡住构建。仅构建期需要，用 ARG（不持久化到运行期）。
ARG DEBIAN_FRONTEND=noninteractive
# TZ：设运行期时区环境变量，glibc 据此让 date/localtime() 走 CST，影响容器内时间与日志时间戳（注：它并不改镜像的 /etc/localtime，该文件仍是 tzdata 默认的 Etc/UTC；让 tzdata 免交互安装的是上面的 DEBIAN_FRONTEND）。
ENV TZ=Asia/Shanghai

# ┌─ 平台会「自动安装」的包 ───────────────────────────────────────────────────
# │ Cursor 在 Environment Build 阶段按容器内 /usr/local/share/vnc-desktop.Aptfile，
# │ 以 `apt-get install --no-install-recommends` 安装下列包；Agent Run 时 desktop-init.sh 只负责启动桌面。
# │ 2026-11-27 在 luckfox_pico_lvgl_example 仓的 Cloud Agent（同一 FROM 的自建 Ubuntu 24.04 镜像）复核：sha256=819987b7…，58 个包名与下列清单一致。
# │   VNC     : tigervnc-standalone-server tigervnc-common tigervnc-tools
# │   XFCE    : xfce4 xfce4-terminal xfce4-settings thunar
# │   X11     : x11-utils x11-xserver-utils xdg-utils xdotool xclip procps
# │   D-Bus   : dbus-x11 at-spi2-core
# │   应用    : mousepad seahorse sudo ffmpeg
# │   主题    : adwaita-icon-theme gnome-themes-extra gnome-keyring plank sassc libglib2.0-dev-bin libxml2-utils dconf-cli xz-utils
# │   locale  : locales
# │   字体    : xfonts-base xfonts-terminus fonts-noto fonts-wqy-microhei fonts-droid-fallback fonts-noto-color-emoji fonts-liberation fonts-croscore fonts-cantarell fonts-jetbrains-mono
# │   桌面库  : libx11-dev libxkbfile-dev libsecret-1-dev libgbm-dev libnotify4 libnss3 libxss1
# │   Mesa    : libgl1-mesa-dri libglx-mesa0 libgl1（Aptfile 原文单列「Mesa software rendering for WebGL」，分组同原文）
# │ 本仓注释掉了 libsdl2-dev，镜像在平台安装前没有 Mesa 三包与 libgbm-dev（2026-11-27 实测），由平台按 Aptfile 安装；本仓没有桌面 GUI（例程直接写板上 framebuffer 或 SPI 屏），用不到其软件 OpenGL。
# │   Python  : python3-minimal python3-numpy
# │   资产依赖: bash ca-certificates coreutils curl findutils gzip tar
# │   浏览器  : google-chrome-stable（单独脚本 install-google-chrome 安装）
# │ 平台 Build 步骤还会配置 git 身份与 SSH 提交签名（含 git-cleanup 子步骤）、把 gh 软链到 /usr/local/bin/gh、创建 /opt/cursor/artifacts；
# │ Agent Run 启动时 artifacts 被替换为指向 /cursor/stores/self/artifacts 的链接。
# │ ⚠️ 本仓只做交叉编译、没有 GUI，也不用桌面 computer use，但平台仍会装上述包。该清单只作 Cloud Agent 环境记录，不作为本文件 apt 取舍依据。
# └──────────────────────────────────────────────────────────────────────────────

# apt 取舍只看当前 FROM digest 的 dpkg：不在 FROM 则显式安装，已在 FROM 则不写。2026-11-27 对该 digest 执行 docker run --rm --entrypoint dpkg-query ubuntu:24.04@sha256:4fbb8e6a8395de5a7550b33509421a2bafbc0aab6c06ba2cef9ebffbc7092d90 -W（amd64，92 个包），下列包均不在 FROM。
# 下方 apt 包分组与 luckfox_pico_rkmpi_example、luckfox_pico_lvgl_example 仓的同名文件相同（docs/superpowers/specs/2026-11-27-rknn-example-cloudagent-env-design.md §3.2）：先本仓构建依赖（基础工具、平台引导、ARM 交叉工具链、构建系统，本仓暂不安装、以注释保留的 native 验证、32 位运行、preview 工具三组，以及 docs/superpowers/specs/2026-11-28-rknn-example-github-actions-ci-design.md §3.8 所述供 CI 使用的文档检查），再 A / B / C 类；每组前一行组名注释，组内按字母序。被注释的包行由引入相应用途的 PR 取消注释。
# - 基础工具    : git（构建期即需——本文件末尾的 git config 依赖它）、sudo（容器内提权）、ca-certificates（运行期 https：git clone / curl / gh；注意 Ubuntu 官方 apt 源走 http，本行 apt 自身并不需要它）。
# - 平台引导    : curl（Cursor 安装 exec-daemon 时用它从 S3 拉取 tar.gz；该步早于平台按 Aptfile 补装 curl，故必须镜像内自带——ubuntu:24.04 基础镜像默认不含 curl）；curl 还被下一层拉 Tailscale 密钥使用。
# - ARM 交叉工具链: gcc-arm-linux-gnueabihf、g++-arm-linux-gnueabihf（编 32-bit ARM 目标；本仓工具链在此安装）。
# - 构建系统    : cmake；build-essential 提供 make（cmake 生成 Makefile）与 native x86 gcc/g++。
# - native 验证 : 本仓暂不安装。lvgl 仓用于 native 渲染验证（libcjson-dev、libdrm-dev、libsdl2-dev、pkg-config）；本仓的依赖已按 libc 预编译于 lib/，没有 native 构建。
# - 32 位运行 : 本仓暂不安装 qemu-user。本仓例程启动即调用 RkLunch-stop.sh，并打开摄像头、NPU 与 /dev/fb0 或 /dev/spidev0.0，qemu 至多做加载冒烟，本仓未实测；由引入该冒烟的 PR 评估并取消注释（docs/superpowers/specs/2026-11-27-rknn-example-cloudagent-env-design.md §3.2）。
# - preview 工具: 本仓暂不安装 imagemagick、xdotool、xvfb。lvgl 仓用于 GUI 截图、取色与点击；本仓的输出在板上屏幕或终端，没有桌面 GUI。xdotool 在 Cloud Agent 中仍由平台 Aptfile 安装。
# - 文档检查  : python3-markdown-it（tests/tools/docs/check-headings.py 依赖 markdown-it-py；CI 的 doc-check job 在本镜像中运行它与 check-samples.sh，docs/superpowers/specs/2026-11-27-rknn-example-cloudagent-env-design.md §2.6）。不在 FROM，FROM 也没有 python3，由本包的依赖装上；noble 中为 3.0.0，Python 3.12（2026-11-27 dpkg-query）。
# - A 类 iproute2 / jq：`ss` 与 JSON 摘要供 .cursor/start.sh 使用；FROM 没有。procps 已在 FROM，不重复。
# - B 类 tcpdump / htop / neofetch：抓包与排查 CLI。neofetch 在 universe；本 RUN 带 --no-install-recommends，避免拉上 neofetch 推荐的 chafa、w3m-img 等包。
# - B 类 tree / iftop / iotop / screen / ncdu / traceroute / nmap / ngrep / conntrack / psmisc / fping / dmidecode：诊断 CLI。apt 写 conntrack（AlmaLinux 包名 conntrack-tools）；不写 conntrack-tools / conntrackd / iotop-c。nmap/ngrep/ncdu 等在 universe，与 neofetch 相同前提。
# - C 类 通用 CLI：vim / file / rsync / unzip / wget / xz-utils / openssh-client / openssl / locales：与 luckfox-pico 仓库对齐的通用 CLI。
# - 绝不写 which：which 无独立实体包（命令由 essential 包 debianutils 内置、FROM 已有）；ubuntu:24.04（universe 默认开启）上写它会解析到虚包 gnu-which 并安装成功（退出 0）、徒增冗余包，官方镜像 Ubuntu 22.04 上则报 "E: Unable to locate package which" 致退出码 100。
# - 公开 Dockerfile 层不安装 openssh-server：FROM 也没有；server 由 .cursor/install.sh 写入私有 Build 快照，避免 host key 进入公开层。
RUN apt-get update && apt-get install -y --no-install-recommends \
      # 基础工具
      ca-certificates git sudo \
      # 平台引导
      curl \
      # ARM 交叉工具链
      g++-arm-linux-gnueabihf gcc-arm-linux-gnueabihf \
      # 构建系统
      build-essential cmake \
      # native 验证（本仓暂不安装，原因见上方说明）
      # libcjson-dev libdrm-dev libsdl2-dev pkg-config \
      # 32 位运行（本仓暂不安装，原因见上方说明）
      # qemu-user \
      # preview 工具（本仓暂不安装，原因见上方说明）
      # imagemagick xdotool xvfb \
      # 文档检查
      python3-markdown-it \
      # A 类：start.sh 依赖
      iproute2 jq \
      # B 类：诊断 CLI
      conntrack dmidecode fping htop iftop iotop ncdu neofetch ngrep \
      nmap psmisc screen tcpdump traceroute tree \
      # C 类：通用 CLI
      file locales openssh-client openssl rsync unzip vim wget xz-utils \
    && rm -rf /var/lib/apt/lists/*

# Tailscale 使用官方 Ubuntu 24.04 Noble stable apt 仓库，不锁定版本；节点认证只在 Agent Run 的 start 阶段进行。本层与上方 apt RUN 一样带 --no-install-recommends。
RUN install -d -m 0755 /usr/share/keyrings \
    && curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg \
       -o /usr/share/keyrings/tailscale-archive-keyring.gpg \
    && curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list \
       -o /etc/apt/sources.list.d/tailscale.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends tailscale \
    && rm -rf /var/lib/apt/lists/*

# Cursor 平台在运行时提供 ubuntu 用户；为 OpenSSH 登录后的提权固化免密 sudo，不在此镜像层安装 openssh-server。
RUN printf '%s\n' 'ubuntu ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ubuntu \
    && chmod 0440 /etc/sudoers.d/ubuntu \
    && visudo -cf /etc/sudoers.d/ubuntu

# 允许对检出的工作区执行 git（避免 "detected dubious ownership" 报错）。
RUN git config --system --add safe.directory '*'

# Cloud Agent 默认运行用户。须写在 docker build 的全部 RUN 之后：USER 会改变后续 RUN 的身份，构建期没有 sudo -n -E；JSON 的 sudo -n -E 只用于镜像建成后的 install/start。
USER ubuntu
