# AGENTS

## 交互约定

- 与本仓库交互时，请始终使用中文回复（代码、路径、命令除外）。

## 优先级

- 作者在对话中的当次指令优先于本文件，本文件优先于技能中的流程要求。superpowers 等技能的流程在本仓库只作可选参考，作者已明确同意不必为满足技能流程而停下或补做步骤；spec 与 plan 仍按 [`docs/superpowers/AGENTS.md`](docs/superpowers/AGENTS.md) 的格式写。
- 因本文件或技能而停下、请求确认或偏离任务时，说明是哪个文件的哪一句。

## 项目与构建

- Luckfox Pico 系列（RV1103 / RV1106）的 RKNN 例程集：`example/` 下 3 个人脸识别 / 目标检测例程，C++ + OpenCV-Mobile + RKNN，CMake 构建；依赖按 libc 预编译于 `lib/{glibc,uclibc}` 与 `3rdparty/`，不需要系统库。本仓是 `LuckfoxTECH/luckfox_pico_rknn_example` 的 fork，默认分支 `dev`。
- 只能交叉编译，产物是 32 位 ARM。`cmake` 必须传 `-DLIBC_TYPE=glibc|uclibc`、`-DEXAMPLE_DIR=example/<例程>`、`-DEXAMPLE_NAME=<例程>`；对应的 `GLIBC_COMPILER` 或 `LUCKFOX_SDK_PATH` 未设置时 `CMakeLists.txt` 打印 WARNING 后 `return()`，不产生任何构建目标。`luckfox_pico_retinaface_facenet_spidev` 按板型另传 `-DLUCKFOX_PICO_PLUS=ON` 或 `-DLUCKFOX_PICO_PRO_MAX=ON`，每种 libc 共 4 个变体。产物的 libc 要与板上固件一致。
- **`install/` 已入库**：`CMakeLists.txt` 把安装目录写死为源码树内的 `install/<libc>/<例程>[_plus|_pro_max]_demo/`（命令行 `-DCMAKE_INSTALL_PREFIX` 无效），`make install` 与 `build.sh` 都会改写已跟踪的二进制，`build.sh clean` 删除整个 `install/`。验证构建用仓库外的构建目录与 `make`，需要部署目录时用 `make install DESTDIR=/tmp/stage`（写到 `/tmp/stage/<仓库绝对路径>/install/…`）；`install/` 只在作者要求更新发布产物时改写，误改用 `git restore install/` 还原。
  - glibc：`export GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-` 后 `cmake -S . -B /tmp/build-glibc-<例程> -DLIBC_TYPE=glibc -DEXAMPLE_DIR=example/<例程> -DEXAMPLE_NAME=<例程> && make -C /tmp/build-glibc-<例程> -j`。工具链由 `.cursor/Dockerfile` 安装，其他环境 `sudo apt-get install -y gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf`。
  - uClibc：无需完整 SDK，稀疏检出工具链：`git clone --depth 1 --filter=blob:none --sparse -b dev https://github.com/yuangezhizao/luckfox-pico.git /tmp/luckfox-pico && git -C /tmp/luckfox-pico sparse-checkout set tools/linux/toolchain`，再以 `LUCKFOX_SDK_PATH=/tmp/luckfox-pico`、`-DLIBC_TYPE=uclibc` 按上一条构建。
- 产物不能在 x86 VM 上验证功能：例程启动即调用 `RkLunch-stop.sh`，并打开摄像头、NPU 与屏幕，权威验证是物理板子。
- **模型转换**（`scripts/luckfox_onnx_to_rknn/`）只能用 RKNN-Toolkit2 **1.6.0**（与固件的 `librknnmrt` 一致；PyPI 只有 2.x，转出的模型板上无法加载）：whl 取 `airockchip/rknn-toolkit2` 的 `v1.6.0` 标签，需 Python ≤ 3.11（`.cursor/Dockerfile` 为 3.12）；镜像不预装。
- C/C++ 代码没有 linter，编译告警只记录、不作门禁。

## 测试与检查

- 本仓暂无测试与 CI。新增或修改 shell 脚本先在本地跑 `shellcheck -x -P SCRIPTDIR <脚本>`，要求 0 条（含 style 级）。
- 按改动范围选检查：只改文档按 [`docs/superpowers/AGENTS.md`](docs/superpowers/AGENTS.md)「检查」一条检查 spec 与 plan；改 `example/` 至少交叉编译改动的例程，改 `CMakeLists.txt`、`lib/`、`include/`、`3rdparty/`、`utils/` 交叉编译两种 libc 的全部 4 个变体。plan 另有要求时以 plan 为准。

## 提交约定

- 标题按 [cz-conventional-emoji v1.1.8](https://github.com/gaoac/cz-conventional-emoji/blob/v1.1.8/engine.js#L21) 的格式 `<type>(<scope>): <emoji> <描述>`：先选与改动相符的 gitmoji，type 由 emoji 决定（取该文件的 `GITMOJI_TYPE_MAP`，未列出的取 gitmoji 名称），如 ✨ `feat`、🐛 `fix`、📝 `docs`、✅ `test`、👷 `ci`、🔧 `chore`、🔨 `hammer`、💡 `comments`；scope 必填，取改动所在的模块，常用的见 `git log`；描述用一句动宾短语，不加句号。
- 一个提交只做一件事，一句话说不清就拆开。
- 正文写 diff 看不出的内容：要解决的问题与根因、为什么选这个做法（必要时写被否决的替代）、副作用与合并后须知；不复述改了哪些文件，不写过程（谁发现的、第几次修改），引用外部讨论时概括要点；简单改动可以不写正文。
- 正文写成「- 」开头的要点，每条一行，不硬折行，这与 Git 按 72 列折行的惯例不同。
- 本仓是 fork，提交、PR 正文与评论中的 PR 或 issue 写完整链接或 `owner/repo#N`，不写裸 `#N`：GitHub 会把它解析到本 fork 或上游。
- 关联的 spec 写成文末脚注 `Refs: docs/superpowers/specs/<文件>.md §X`，它不能代替正文中的原因。
- spec 与 plan 放在 PR 的最后一个提交。

## 去向表

| 要做的事 | 先读 |
|---|---|
| 改 Cloud Agent 环境：`.cursor/` 下的 Dockerfile、`install.sh`、`start.sh`、`environment.json` | [`.cursor/AGENTS.md`](.cursor/AGENTS.md) |
| 写或改 `docs/superpowers/` 下的 spec、plan | [`docs/superpowers/AGENTS.md`](docs/superpowers/AGENTS.md) |
| 了解例程的用途、板端部署与运行 | [`README_CN.md`](README_CN.md) |
| 转换模型（版本见上方） | [`scripts/luckfox_onnx_to_rknn/README_CN.md`](scripts/luckfox_onnx_to_rknn/README_CN.md) |
| 新增给 agent 的约定 | 本文件只写几乎每个任务都用得到、读代码推不出来的内容，其余写进对应目录的 `AGENTS.md`（[放置原则](https://github.com/yuangezhizao/luckfox_pico_lvgl_example/blob/dev/docs/superpowers/specs/2026-11-01-lvgl-example-agents-md-slim-design.md) §3.1）；本文件不超过 8 KiB |

## Cursor Cloud specific instructions

只适用于 Cursor Cloud Agent 的 VM。

- **身份**：提交身份为仓库配置的 `Cursor Agent <cursoragent@cursor.com>`，带 SSH 签名。环境会注入 `GIT_AUTHOR_*`、`GIT_COMMITTER_*`，提交、rebase 前要加 `env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL`。
- **`Co-authored-by` 尾注**：`.git/config` 的 `core.hooksPath` 指向 `~/.cursor/agent-hooks/<id>/`，其中的钩子由 Cursor 在每次 Agent Run 启动时生成，`commit-msg` 与 `prepare-commit-msg` 会把发起本次 Run 的 Cursor 账户追加为 `Co-authored-by`（`prepare-commit-msg` 在 `--no-verify` 时也运行）。本仓允许带该尾注，但地址只能是 `远哥制造 <yuangezhizao@users.noreply.github.com>`，不能写进工作邮箱。
  - 每次 Run 第一次提交前，查看 `commit-msg.cursor.co-author` 与 `prepare-commit-msg.cursor.co-author` 中的 `CO_AUTHORS`，不是该地址就改成它；只对本次 Run 有效，新 Run 会重新生成。
  - 内容提交照常经过钩子，`pre-commit` 会做密钥扫描，不要用 `--no-verify`；推送前用 `git log --format=%B origin/dev..HEAD | grep -i co-authored-by` 确认没有其他地址。
  - `GIT_EDITOR` 由 shell 解释，参数中有 `(` 时要加引号。
- **保留作者时间**：rebase（含 fixup、reword）和 `--amend` 都要保留 author time，committer time 可以变化。不要用 `--reset-author`、`--ignore-date`、`--date`，也不用 `--committer-date-is-author-date`。
- **PR 与 GitHub 账户**：推送可以使用 Cloud Agent 自带的 `cursor` 账户凭据，创建、更新 PR 用其自带的 PR 管理工具。在本环境用 `gh`（`cursor` 账户）创建 PR、改 PR 正文、发评论会返回 HTTP 403 `Resource not accessible by integration`，需要 `gh` 写操作时请作者 `gh auth login` 登录本人账户；环境刷新令牌时会改写 `~/.config/gh/hosts.yml`，登录随时可能失效。本仓是 fork，`gh pr create` 要加 `-R yuangezhizao/luckfox_pico_rknn_example --base dev`。
- **打码**：`git remote -v`、push 报错可能显示带令牌的地址，输出前要先打码。
