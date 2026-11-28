# Luckfox Pico RKNN Example — Cloud Agent 环境实施计划

> **For agentic workers:** 作者同意本 PR 不必严格遵循 superpowers 的执行流程（rkmpi 仓根 `AGENTS.md`「优先级」，本 PR 移植后即为本仓根 `AGENTS.md`）；按下面的 Task 顺序执行，复选框（`- [ ]`）用于跟踪。

**Goal:** 把 rkmpi 仓 `dev`（`cf2a0db`）的 Cloud Agent 环境移植到本仓：两份 Dockerfile（非注释行与 rkmpi 仓相同）、`environment.json`、`install.sh`、`start.sh`、三份 `AGENTS.md`；在当前 VM 临时安装 Docker 完成 V1–V4 验证。

**Architecture:** 推送 spec 与 plan、开草稿 PR，等作者审阅（Task 1）；两份 Dockerfile（Task 2）；JSON 与脚本（Task 3）；三份 `AGENTS.md`（Task 4）；V1 静态检查（Task 5）；V2–V4 镜像、交叉编译与 `install.sh`（Task 6）；整理提交、精简 plan、PR 正文（Task 7）；作者在 Cursor 中做 V5、V6（Task 8）。

**Tech Stack:** Dockerfile、bash、ShellCheck、Docker（临时安装）、CMake、`gcc-arm-linux-gnueabihf`、`arm-rockchip830-linux-uclibcgnueabihf`、`gh`。

**Spec:** [`docs/superpowers/specs/2026-11-27-rknn-example-cloudagent-env-design.md`](../specs/2026-11-27-rknn-example-cloudagent-env-design.md)

---

## 1. Global Constraints

- 分支 `pi/cloudagent-env`，起点 `origin/dev` `16ef54a`；PR 指向 fork `yuangezhizao/luckfox_pico_rknn_example` 的 `dev`，开发期为草稿，执行者不合并。
- 参考源一律取 rkmpi 仓 `dev` `cf2a0db`：本机检出 `/workspace` 中用 `git -C /workspace show cf2a0db:<路径>`；附录命令改用固定提交的 raw 链接，不依赖本机目录。
- 不改 `CMakeLists.txt`、`build.sh`、`example/`、`lib/`、`include/`、`3rdparty/`、`utils/`、`scripts/`、`install/`；`git add` 只写明确的路径。
- 验证构建不得改写工作区：容器内只读挂载仓库，部署目录用 `DESTDIR`；每个 Task 结束时 `git status --short` 只应出现本 Task 的文件。
- 提交作者与提交者均为 `Cursor Agent <cursoragent@cursor.com>`（`~/.gitconfig`），SSH 签名；所有创建或改写提交的 git 命令前加 `env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL`。本仓未配置 Cursor 钩子，推送前 `git log --format=%B origin/dev..HEAD | grep -i co-authored-by` 应无输出。
- rebase（含 fixup、reword）与 `--amend` 保留 author time，不用 `--reset-author`、`--ignore-date`、`--date`、`--committer-date-is-author-date`。
- 提交信息按 rkmpi 仓根 `AGENTS.md`「提交约定」：`<type>(<scope>): <emoji> <描述>`；正文为「- 」开头的要点，每条一行，写原因、不写过程；PR 写完整链接或 `owner/repo#N`；末尾 `Refs: docs/superpowers/specs/2026-11-27-rknn-example-cloudagent-env-design.md §X`。
- 推送：先试 remote 自带的凭据；403 时改用作者账户（附录 A.4）。`gh` 写操作（建 PR、改正文、发评论）用作者账户，`gh pr create` 加 `-R yuangezhizao/luckfox_pico_rknn_example --base dev`。输出中的令牌先打码。
- 新增或修改 `.sh` 先跑 `shellcheck -x -P SCRIPTDIR`，要求 0 条。
- 提交、推送前按 `docs/superpowers/AGENTS.md`「检查」运行 rkmpi 仓的 `check-headings.py`，不为 0 处时中止。
- Docker 只为验证临时安装，不写入任何仓库文件；镜像名用 `rknn-env:active`、`rknn-env:alt`。

## 2. Review Focus

- **与 rkmpi 仓的差异是否只有预期部分**：`environment.json`、`install.sh` 逐字相同；`start.sh` 只差 `repo_slug`；Dockerfile 去掉注释行后无差异（附录 A.1）。
- **注释是否都改成了本仓**：两份 Dockerfile 中不再出现 RKMPI、RTSP、5 个例程、rkmpi 仓 spec 路径；「32 位运行」不照搬 rkmpi 仓的 qemu 实测结论（spec §3.2）。
- **`AGENTS.md` 的事实**：4 个变体与板型开关、`DESTDIR`、`build.sh clean`、模型转换的版本约束（1.6.0、非 PyPI、Python ≤ 3.11）都与实测一致；不引用本仓不存在的文件；根文件 ≤ 8,192 字节。
- **V2 的复核结果**：两个 FROM 的包与 rkmpi 仓记录是否一致（不一致时改 apt 行并记录）；备选镜像的 `pip`、`venv`。

## 3. 提交

按改动对象拆分（spec §5），以 `git log` 为准。开发期 spec 与 plan 最先推送供作者审阅，审阅后的修订以 fixup 并入，Task 7 用 `git rebase -i --autosquash` 整理并把 spec 与 plan 的提交移到末尾（附录 A.4）。

## 4. Tasks

### Task 1：spec 与 plan，推送并开草稿 PR，等作者审阅 ✅

落地：本 spec 与 plan；草稿 PR [yuangezhizao/luckfox_pico_rknn_example#1](https://github.com/yuangezhizao/luckfox_pico_rknn_example/pull/1)。

- [x] `check-headings.py` 0 处；推送 spec 与 plan，用作者账户开草稿 PR。
- [x] 作者审阅：spec 的 Q5–Q9 与 §2.7–§2.9 重复，并入 Q3 的「答」（§6 第 5 条）；作者审阅完成后开始实施。

### Task 2：两份 Dockerfile ✅

落地：`.cursor/Dockerfile`、`.cursor/Dockerfile.luckfox_pico`。

- [x] 以 rkmpi 仓 `cf2a0db` 的同名文件为底，用一次性脚本按行替换注释（不保存，§6 第 6 条）；非注释行逐字相同；残留的 rkmpi 字样只有指向 rkmpi 仓 spec 的出处。
- [x] V2 复核后把注释中的复核日期改为 2026-11-27。

### Task 3：`environment.json`、`install.sh`、`start.sh` ✅

落地：`.cursor/environment.json`、`.cursor/install.sh`、`.cursor/start.sh`；`environment.json`、`install.sh` 的 blob 与 rkmpi 仓相同（`47922a7`、`99af772`）。

### Task 4：三份 `AGENTS.md` ✅

落地：根 `AGENTS.md`（8,184 字节）、`.cursor/AGENTS.md`、`docs/superpowers/AGENTS.md`；相对链接均存在。

### Task 5：V1 静态检查 ✅

- [x] 附录 A.1，结果见 §5.1。

### Task 6：V2–V4 ✅

- [x] Docker 按附录 A.2 安装，交叉编译按附录 A.3，结果见 §5.2–§5.4。

### Task 7：整理提交、精简 plan、PR 正文 ✅

- [x] `git rebase -i --autosquash origin/dev`：fixup 并入，spec、plan 的提交移到末尾，说明改为最终状态；author time 不变、每个提交带签名、逐个提交 `check-headings.py` 为 0、无 `Co-authored-by`，rebase 前后文件树相同（附录 A.4）。
- [x] 精简 plan：已完成 Task 只留落地指针与勾选。
- [x] 链接检查：spec、plan、三份 `AGENTS.md`、Dockerfile 注释中的 31 个链接，30 个为 200；[Luckfox 论坛](https://forums.luckfox.com/viewtopic.php?t=2268)对脚本返回 Cloudflare 质询（403、`cf-mitigated: challenge`），浏览器可访问，保留。
- [x] `--force-with-lease` 推送（`gh` 的作者登录被环境刷新覆盖，`cursor[bot]` 对本仓无写权限，请作者重新登录后推送，§6 第 1 条）；更新 PR 正文，改为 ready，不合并。
- [x] 作者要求按根 `AGENTS.md`「提交约定」自查提交说明，3 处不符，用 rebase 的 `exec … --amend -F` 只改说明（附录 A.4；文件树与 author time 不变）：一条写了过程（「已重新复核」）、两处复述文件内容，改写为原因；spec 与 plan 提交中的裸「PR #2」改为 `yuangezhizao/luckfox_pico_rkmpi_example#2`。Dockerfile 的提交与 PR 标题按作者决定沿用 rkmpi、lvgl 仓 PR #1 的 🐳（不在 gitmoji 中，spec §5）。
- [x] 作者要求对照 GPT-6.2 的两篇官方指南评估 `AGENTS.md`：候选改动作者决定都不改，记为 spec Q5；本 VM 上临时安装的 Docker 与 `rknn-env:*` 镜像保留。
- [x] 再次完整复查 spec、plan 与 PR 正文，按上述修改同步（spec 的 PR 状态、提交标题的取舍，plan 的本 Task 与 §6，PR 正文的 QA 数与评审注意）；按「以后能沿用、能避坑、模型推不出来」精简附录 A（§6 第 8 条）。

### Task 8：作者在 Cursor 中执行 V5、V6（下一个 PR 开发时）✅

合并后，作者在 Cursor 中为本 fork 首次启动 Cloud Agent（合并前不要启动，spec §6）：

- [x] Dashboard 中新建的 saved environment 的 **environment.json location** 指向本仓 `.cursor/environment.json`。
- [x] V5：首个 Build `SUCCEEDED`，日志中 `install.sh` exit 0（grilling `curl` 成功、`openssh-server` 安装、host key 生成）。
- [x] V6：`whoami` 为 `ubuntu`；`tailscale status` 的主机名为 `cursor-agent-for-luckfox-pico-rknn-example-<bcId 前 8 位>`；`tailscale serve status` 有 5901 / 1054 / 1055；经 tailnet `ssh ubuntu@<主机名>` 成功、`root` 被拒；`~/.cursor` 与 grilling 属主 `ubuntu`；按根 `AGENTS.md` 的命令完成两种 libc 的构建。
- [x] 结果记入下一个 PR 的 spec 或 plan；发现问题时在下一个 PR 修复。

2026-11-28 在 CI PR 的 Cloud Agent 中核对，全部符合，结果见 spec [`2026-11-28-rknn-example-github-actions-ci-design.md`](../specs/2026-11-28-rknn-example-github-actions-ci-design.md) §2.4。

## 5. 验证证据

2026-11-27 在当前 VM（pi）执行；Docker 按附录 A.2 临时安装（`docker-ce` 28.5.2，存储驱动 `fuse-overlayfs`）。

### 5.1 V1 静态检查

- `environment.json`、`install.sh` 与 rkmpi 仓 `cf2a0db` 逐字相同；`start.sh` 只差第 3 行 `repo_slug`。
- 两份 Dockerfile 去掉注释行后与 rkmpi 仓无差异。
- `.cursor/` 6 个文件模式均为 `100644`；ShellCheck 0.9.0 0 条；`check-headings.py` 2 个文件 0 处；根 `AGENTS.md` 8,184 字节。

### 5.2 V2 镜像

| 项 | `rknn-env:active` | `rknn-env:alt` |
|---|---|---|
| `docker build --check` | 无告警 | 无告警 |
| 体积 | 956 MB（与 rkmpi 仓 PR #1 的同名镜像相同） | 1.08 GB |
| 默认用户 | `ubuntu` uid 1000，`HOME=/home/ubuntu`，`sudo -n` 可用 | 同左（`useradd` 创建） |
| 需求包 36 个 | 齐全 | 齐全 |
| 注释掉的包与 Mesa 三包、`libgbm-dev` | 均不在 | 只有 `pkg-config`（FROM 已有） |
| 工具版本 | gcc 13.3.0、cmake 3.28.3、make 4.3、tailscale 1.104.7 | gcc 11.4.0、cmake 3.22.1、make 4.3、tailscale 1.104.7 |
| Python、`python3-markdown-it` | 3.12.3、3.0.0-2 | 3.10.12、1.1.0-2 |
| `pip`、`venv` | 均不可用 | 均不可用 |
| 文档检查 | rkmpi 仓 `check-headings.py` 0 处，`check-samples.sh` 115 个样例全部符合 | 同左 |

- FROM 复核：`ubuntu:24.04` 92 个包、`luckfox_pico:1.0` 366 个包；rkmpi 仓 apt 清单中「已在 FROM」的包为 `luckfox_pico:1.0` 的 `ca-certificates` `git` `cmake` `pkg-config` `file` `openssh-client` `openssl` `rsync` `unzip` `vim` `wget` `xz-utils`、`openssh-server`，两者都有 `procps`，与 rkmpi 仓记录相同；`luckfox_pico:1.0` 另有 `python3`，两个 FROM 都没有 `python3-pip`、`python3-venv`。
- 平台 Aptfile：sha256 `819987b7…`，58 个包，与 rkmpi 仓相同。

### 5.3 V3 容器内交叉编译

两份镜像中以 `ubuntu` 身份、仓库只读挂载，稀疏检出 `yuangezhizao/luckfox-pico` `dev`（`665c7f1`）：两种 libc × 4 个变体共 16 次全部成功；glibc 产物为 ELF32 ARM PIE、interpreter `/lib/ld-linux-armhf.so.3`，uClibc 产物为 ELF32 ARM、interpreter `/lib/ld-uClibc.so.0`，属主 `ubuntu`；告警只有 glibc 下 `retinaface_facenet`、两个 `spidev` 变体各一条 `-Wformat-overflow`（spec §2.3）；前后 `git status --short` 相同。

### 5.4 V4 `install.sh`

两份镜像均为：预建 `~/.cursor`（755）后连续运行两次，退出码 0 / 0；host key 指纹不变；`~/.cursor/skills`、`grilling/` 为 `ubuntu:ubuntu 750`，`SKILL.md` 为 `ubuntu:ubuntu 644`，SHA-256 `10ff989e7498b23b5acb49d5048f11dcd906757d2f79c5cdf8a00001381296f2`（与 rkmpi 仓相同）；`openssh-server` 活动版 `1:9.6p1-3ubuntu13.20`、备选版 `1:8.9p1-3ubuntu0.18`。

## 6. 与计划的偏离

1. **推送账户**：remote 自带的凭据推送报 `Invalid username or token`，改用作者账户推送（附录 A.4）；Task 7 时 `gh` 的作者登录被环境刷新覆盖，`cursor[bot]` 推送返回 403，请作者重新 `gh auth login` 后推送。
2. **Docker 安装方式**：初稿用 Ubuntu 的 `docker.io`，按作者提示改为官方文档 Running Docker 的写法（Docker 官方 apt 源，`docker-ce` 28.5.2）。
3. **根 `AGENTS.md` 体积**：初稿 8,469 字节，超过 8 KiB；精简 `lib/` 的描述、模型转换与 x86 两条的措辞，删去「测试与检查」中与去向表重复的 `.cursor/AGENTS.md` 指针、`Cursor Cloud specific instructions` 开头的环境指针（rkmpi 仓 PR #2 也已删去），为 8,184 字节。
4. **备选镜像的 Python**：spec §2.7 初稿写 FROM 自带 3.10.6，构建后为 3.10.12；两份镜像都没有 `pip`、`venv`，spec §2.7 已改。
5. **spec 的 QA**：作者审阅时指出 Q5–Q9 与 §2.7–§2.9 重复，删去并在 Q3 的「答」中留一句汇总（Task 1）。
6. **Dockerfile 注释改写脚本不保存**：一次性按 rkmpi 仓 `cf2a0db` 的原文替换；以后 rkmpi 仓的 Dockerfile 改动按其 diff 同步即可，用附录 A.1 核对。
7. **额外验证**：在两份镜像中运行 rkmpi 仓的 `check-headings.py` 与 `check-samples.sh`，确认备选镜像注释中「检查结果与 3.0.0 相同」属实（§5.2）。
8. **附录精简**：删去模型能直接写出的命令（逐包 `dpkg-query`、`docker build`、ShellCheck、文档检查、`git log` 核对等），只留与 rkmpi 仓逐字比对、临时安装 Docker、4 个变体的只读构建、提交整理与推送中容易踩坑的部分。

## 附录 A

只留以后能沿用、容易踩坑的命令，都在本仓根目录运行；ShellCheck、文档检查等日常命令见根 `AGENTS.md` 与 `docs/superpowers/AGENTS.md`。

### A.1 与 rkmpi 仓逐字比对（同步 rkmpi 仓的环境改动时复用）

取固定提交的 raw 文件，不依赖本机检出；Dockerfile 只比非注释行，注释按本仓改写过。

```bash
ref() { curl -fsSL "https://raw.githubusercontent.com/yuangezhizao/luckfox_pico_rkmpi_example/cf2a0dbb99416858454bef9f9108a028362e75d9/$1"; }
for f in environment.json install.sh start.sh; do diff <(ref ".cursor/$f") ".cursor/$f" && echo "$f same"; done   # start.sh 只差 repo_slug
for f in Dockerfile Dockerfile.luckfox_pico; do
  diff <(ref ".cursor/$f" | grep -v '^\s*#') <(grep -v '^\s*#' ".cursor/$f") && echo "$f non-comment-same"
done
```

### A.2 在 Cloud Agent 的 VM 中临时安装 Docker

按官方文档 [Running Docker](https://cursor.com/docs/cloud-agent/setup#running-docker)：VM 本身是容器，要用 `fuse-overlayfs` 与 `iptables-legacy`；Docker 官方 apt 源，不用 Ubuntu 的 `docker.io`。不写入仓库的 Dockerfile。

```bash
sudo bash -euo pipefail <<'SH'
export DEBIAN_FRONTEND=noninteractive
install -m 0755 -d /etc/apt/keyrings
curl --retry 3 -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" > /etc/apt/sources.list.d/docker.list
apt-get update
# --force-confold：遇 /etc/fuse.conf 等 conffile 提问时保留现有文件，否则无人值守安装会卡住
apt-get install -y -o Dpkg::Options::=--force-confold docker-ce=5:28.5.2-1~ubuntu.24.04~noble docker-ce-cli=5:28.5.2-1~ubuntu.24.04~noble containerd.io docker-buildx-plugin fuse-overlayfs iptables
mkdir -p /etc/docker && echo '{ "storage-driver": "fuse-overlayfs" }' > /etc/docker/daemon.json
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
service docker start
SH
```

构建镜像时用空目录作构建上下文（Dockerfile 不 `COPY`）；两个 FROM 的包复核见 rkmpi 仓 PR #1 plan 附录 A.5。

### A.3 容器内只读构建两种 libc × 4 个变体（CI PR 的矩阵可沿用）

仓库只读挂载，`CMakeLists.txt` 的安装目录写死在源码树内，直接 `make install` 会失败，必须用 `DESTDIR`；spidev 的两个板型要分别构建。

```bash
for tag in active alt; do
  sudo docker run --rm -v "$PWD:/src:ro" "rknn-env:$tag" bash -euo pipefail -c '
    git clone -q --depth 1 --filter=blob:none --sparse -b dev https://github.com/yuangezhizao/luckfox-pico.git /tmp/luckfox-pico
    git -C /tmp/luckfox-pico sparse-checkout set tools/linux/toolchain
    for libc in glibc uclibc; do
      for v in luckfox_pico_retinaface_facenet: luckfox_pico_retinaface_facenet_spidev:-DLUCKFOX_PICO_PLUS=ON luckfox_pico_retinaface_facenet_spidev:-DLUCKFOX_PICO_PRO_MAX=ON luckfox_pico_yolov5:; do
        ex=${v%%:*}; opt=${v#*:}; b=/tmp/build-$libc-$ex-${opt//[^A-Z]/}
        if [ $libc = glibc ]; then e="GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-"; else e="LUCKFOX_SDK_PATH=/tmp/luckfox-pico"; fi
        { env $e cmake -S /src -B $b -DEXAMPLE_DIR=example/$ex -DEXAMPLE_NAME=$ex -DLIBC_TYPE=$libc $opt && env $e make -C $b -j"$(nproc)" && env $e make -C $b install DESTDIR=/tmp/stage; } > $b.log 2>&1
        echo "$libc $ex $opt rc=$? warnings=$(grep -c "warning:" $b.log || true)"
      done
    done'
done
```

`install.sh` 的幂等验证（预建 `~/.cursor` 后连续运行两次）见 rkmpi 仓 PR #1 plan 附录 A.4；不预建时 grilling 对 `ubuntu` 不可读，平台会预建。

### A.4 提交整理与推送

- autosquash 并把 spec 与 plan 的提交移到末尾：`GIT_SEQUENCE_EDITOR` 用 rkmpi 仓 PR #1 plan 附录 A.7 的 `seq.py`。
- 只改某几个提交的说明：在 todo 中对应的 `pick` 后插入 `exec git commit --amend -F <新说明>`，比逐个 `edit` 停下省事；`--amend` 保留 author time。
- `cursor[bot]` 对本仓没有写权限，`gh` 的作者登录又会被环境刷新覆盖；推送用作者账户的令牌，失效时请作者重新 `gh auth login`：

```bash
E="env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL"
SPEC_SUBJECT='新增 Cloud Agent 环境 spec 与 plan' GIT_SEQUENCE_EDITOR='python3 /tmp/seq.py' $E git rebase -i --autosquash origin/dev
old=$(git ls-remote https://github.com/yuangezhizao/luckfox_pico_rknn_example.git refs/heads/pi/cloudagent-env | cut -f1)
GIT_TERMINAL_PROMPT=0 env -u GIT_ASKPASS git -c credential.helper= \
  -c credential.helper='!f(){ test "$1" = get || exit 0; echo username=yuangezhizao; echo "password=$(gh auth token -u yuangezhizao)"; }; f' \
  push --force-with-lease=pi/cloudagent-env:$old https://yuangezhizao@github.com/yuangezhizao/luckfox_pico_rknn_example.git pi/cloudagent-env 2>&1 \
  | sed 's/gh[a-z]_[A-Za-z0-9_]*/***/g'
```
