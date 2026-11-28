# Luckfox Pico RKNN Example — GitHub Actions CI 实施计划

> **For agentic workers:** 作者同意本 PR 不必严格遵循 superpowers 的执行流程（根 `AGENTS.md`「优先级」）；按下面的 Task 顺序执行，复选框（`- [ ]`）用于跟踪。Task 1–10 已完成，代码入库后本 plan 已精简：已完成的 Task 只留落地指针，保留验证证据（§5）、偏离（§6）、合并后的步骤（Task 11）与可复用的命令（附录 A）。

**Goal:** 以 rkmpi 仓 `dev`（`cf2a0db`）为底为本仓新增 GitHub Actions：GHCR 镜像 + 两种 libc × 4 变体的交叉编译与兼容性检查、ShellCheck、文档检查（脚本入库）、根 `AGENTS.md` 体积检查，以及 `.github/AGENTS.md` 与相关说明。

**Architecture:** 推送 spec 与 plan、开草稿 PR、grilling（Task 1）；Dockerfile 注释（Task 2）；检查脚本入库（Task 3）；主 workflow 的 `build-image`、`build-example`（Task 4）；`shellcheck`、`doc-check`（Task 5）；体积检查 workflow（Task 6）；五份 `AGENTS.md`（Task 7）；本地 V1–V3（Task 8）；开发期 CI（Task 9）；回填 PR #1 plan、整理提交与 V5（Task 10）；作者执行的项（Task 11）。

**Tech Stack:** GitHub Actions（`ubuntu-24.04`）、GHCR、`actions/checkout@v7`、`docker/login-action@v4`、`actions/attest@v4`、`actions/upload-artifact@v7`、`gh attestation verify`、CMake、`gcc-arm-linux-gnueabihf`、`arm-rockchip830-linux-uclibcgnueabihf`、Python 3 + markdown-it-py、ShellCheck 0.9.0、actionlint、Docker（临时安装）。

**Spec:** [`docs/superpowers/specs/2026-11-28-rknn-example-github-actions-ci-design.md`](../specs/2026-11-28-rknn-example-github-actions-ci-design.md)

---

## 1. Global Constraints

- 分支 `cursor/github-actions-ci-7892`，起点 `origin/dev` `1e5283e`；PR 指向 fork `yuangezhizao/luckfox_pico_rknn_example` 的 `dev`，开发期为草稿，执行者不合并、不改为 ready（作者确认后再改）。
- 参考源取 rkmpi 仓 `dev` `cf2a0db`（`/tmp/rkmpi` 的克隆或固定提交的 raw 链接）；lvgl 仓 `11497f3`、luckfox-pico 仓 `665c7f1` 只用于核对出处。
- 不改 `CMakeLists.txt`、`build.sh`、`example/`、`lib/`、`include/`、`3rdparty/`、`utils/`、`install/` 与 `scripts/` 下的现有文件（新增 `scripts/luckfox_onnx_to_rknn/AGENTS.md` 见 spec Q6）；PR #1 的 spec 与 plan 只在单独的提交中修改；`git add` 只写明确的路径；每个 Task 结束时 `git status --short` 只应出现本 Task 的文件。
- 本地验证构建不得改写工作区：仓库外的构建目录，`make install DESTDIR=…`（根 `AGENTS.md`）；容器内只读挂载或用 `git archive` 的副本。
- 提交身份 `Cursor Agent <cursoragent@cursor.com>`，SSH 签名；创建或改写提交的 git 命令前加 `env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL`。本次 Run 的两个 co-author 钩子已是 `远哥制造 <yuangezhizao@users.noreply.github.com>`；推送前 `git log --format=%B origin/dev..HEAD | grep -i co-authored-by` 只应出现该地址。
- rebase（含 fixup、reword）与 `--amend` 保留 author time，不用 `--reset-author`、`--ignore-date`、`--date`、`--committer-date-is-author-date`。
- 提交信息按根 `AGENTS.md`「提交约定」：`<type>(<scope>): <emoji> <描述>`，正文为「- 」开头的要点，PR 写完整链接或 `owner/repo#N`；末尾 `Refs: docs/superpowers/specs/2026-11-28-rknn-example-github-actions-ci-design.md §X`；spec 与 plan 放在 PR 的最后一个提交。
- 推送用 Cloud Agent 自带的凭据（`git push -u origin cursor/github-actions-ci-7892`）；创建、更新 PR 用 PR 管理工具，不用 `gh` 写操作；输出中的令牌先打码。
- 新增或修改的 `.sh` 跑 `shellcheck -x -P SCRIPTDIR`，要求 0 条；workflow 跑 actionlint。
- 提交、推送前运行文档检查：Task 3 之前用 `python3 /tmp/rkmpi/tests/tools/docs/check-headings.py`，之后用入库的 `tests/tools/docs/check-headings.py`；不为 0 处时中止。

## 2. Review Focus

- **与 rkmpi 仓的差异是否只有预期部分**：`check-headings.py` 逐字相同；`check-samples.sh` 只差两份文档的路径与开头注释；`build-image` 只差镜像名、signer workflow、出处注释；`agents-md-size.yml` 只差预算出处（附录 A.2）。
- **变体清单**：`VARIANTS` 与 `build.sh` 的选项一一对应，部署目录名与入库的 `install/<libc>/` 相同，spidev 的两个变体覆盖 Plus、Pro、Max 三款板子（spec §2.1.1）；漏加例程时在编译前失败（spec §3.4、Q2）。
- **门禁不会被跳过**：每行 4 个变体都过兼容性检查，某个变体失败时 job 失败；反例本地验证过（spec §4 V3）。
- **PR 上不构建、不覆盖正式 tag**：最终形态下 `pull_request` 只复用 dev 签名镜像；开发期的临时提交（spec §2.3）不进最终历史。
- **根 `AGENTS.md` 的压缩不删信息**：逐句对照改前改后，移走的「模型转换」在 `scripts/luckfox_onnx_to_rknn/AGENTS.md` 中完整保留（Task 7、spec Q6）。

## 3. 提交

提交按主题拆分，以 `git log` 为准（spec §5）。开发期 spec 与 plan 最先推送，grilling 后的修订以 fixup、`amend!` 并入对应提交；V4 用的临时提交叠在实现提交之上，Task 10 用 `git rebase -i` 删去，运行记录写入 §5。

## 4. Tasks

### Task 1：spec 与 plan，推送并开草稿 PR，grilling ✅

- [x] 草稿 PR [yuangezhizao/luckfox_pico_rknn_example#2](https://github.com/yuangezhizao/luckfox_pico_rknn_example/pull/2)；grilling 的设计决策写入 spec §8（Q1–Q7）。

### Task 2：两份 Dockerfile 的注释 ✅

- [x] 只改「文档检查」组说明与分组总述的 spec 指针，非注释行与 rkmpi 仓相同（§5.1）。

### Task 3：文档检查脚本入库 ✅

- [x] `check-headings.py` 与 rkmpi 仓逐字相同；`check-samples.sh` 的 `mk` 样例改取本仓 PR #1 的 spec 与 plan，115 个（§5.1）。

### Task 4：主 workflow 的 `build-image`、`build-example` ✅

- [x] `build-image` 与 rkmpi 仓只差显示名、注释与镜像名、signer workflow；`build-example` 按 job 级 `env` 的 `VARIANTS` 构建，编译前核对 `example/` 目录都在清单中，兼容性检查覆盖可执行文件与 `lib/*.so`（spec §3.4、Q2、Q3）；`run:` 只用环境变量，可在 CI 镜像中原样执行（附录 A.3）。

### Task 5：`shellcheck`、`doc-check` ✅

- [x] 照搬 rkmpi 仓，ShellCheck 排除上游 `build.sh`（spec §3.5）。

### Task 6：体积检查 workflow ✅

- [x] 照搬 rkmpi 仓，只改预算出处。

### Task 7：五份 `AGENTS.md` ✅

- [x] 新增 `.github/AGENTS.md`；`.cursor/AGENTS.md`、`docs/superpowers/AGENTS.md` 改为指向 CI 与入库的脚本；根 `AGENTS.md` 压缩措辞后按 spec Q6 把「模型转换」移到新建的 `scripts/luckfox_onnx_to_rknn/AGENTS.md`，改后 8,192 字节（spec §3.9）。

### Task 8：本地 V1–V3 ✅

- [x] 结果见 §5.1–§5.3，命令见附录 A.1–A.3。

### Task 9：开发期 CI（V4）✅

- [x] 两个临时提交的 run 见 §5.4，做法见附录 A.4，最终历史中没有它们。

### Task 10：回填 PR #1 plan、整理提交、V5、精简 plan ✅

- [x] PR #1 plan Task 8 勾选、PR #1 spec 状态行改为 V5、V6 已通过（均指向本 spec §2.4），PR #1 spec §2.1、§3.4 改正 spidev 板型说明，合为单独一个提交。
- [x] `git rebase -i --autosquash`：fixup 并入，spec 与 plan 的提交移到末尾（附录 A.5）；逐个提交检查见 §5.6；V5 见 §5.5；精简本 plan，更新 PR 正文。

### Task 11：作者执行的项

- [x] tailnet SSH：作者从另一台机器登录成功、root 被拒（spec §2.4）。
- [ ] 合并后（V6）：`push(dev)` 的 `build-image` 走可信首建并 attest；可选在 GHCR 包页面删除临时 tag `pr-2-92806b15473f7871e71e7c7fa5e858a0`；下一个 PR 的 CI 走 dest-lock 复用，结果记入下一个 PR。

## 5. 验证证据

Task 1 前的现状实测见 spec §2.1、§2.4。

### 5.1 V1 静态检查（2026-11-28，附录 A.2）

- actionlint 1.7.12 对两个 workflow 0 条；ShellCheck 0.9.0 对 `.cursor/install.sh`、`.cursor/start.sh`、`tests/tools/docs/check-samples.sh` 0 条。
- `check-headings.py` 检查 4 个文件 0 处；`check-samples.sh` 样例 115 个、不符合 0 个，从其他目录运行结果相同。
- `check-headings.py` 与 rkmpi 仓逐字相同；`check-samples.sh` 只差开头注释与两份文档的变量名、路径；`build-image` 及其以上部分只差显示名、schedule 注释中的「例程」改「变体」与出处注释（镜像名、signer workflow 经名称替换后相同）；`agents-md-size.yml` 只差两处注释；两份 Dockerfile 去掉注释后与 rkmpi 仓相同，`.cursor/Dockerfile` 内容哈希前 32 位为 `92806b15473f7871e71e7c7fa5e858a0`。

### 5.2 V2 镜像（附录 A.3）

- Docker 28.5.2（fuse-overlayfs）；`docker build --check` 两份均无告警；`rknn-ci:active` 956 MB、`rknn-ci:alt` 1.08 GB。
- `doc-check` 的步骤：活动镜像 Python 3.12.3、markdown-it-py 3.0.0，备选镜像 Python 3.10.12、markdown-it-py 1.1.0，均为 4 个文件 0 处、样例 115 个全部符合。
- 在活动镜像中以 `--user 0` 原样执行 `build-example` 的 4 段 `run:`（`git archive HEAD` 的副本）：两行各 4 个变体编译成功、通过兼容性检查；glibc 行可执行文件最高 `GLIBC_2.34`、`GLIBCXX_3.4.29`，`librga.so` 最高 `GLIBC_2.4`；编译告警 glibc 3 条、uClibc 0 条（spec §2.1）；`install/<libc>/` 恰为 4 个部署目录，`SHA256SUMS` 各 20 行、`sha256sum -c` 通过；uClibc 工具链 `665c7f1`。本仓工作区无改动。

### 5.3 V3 门禁的反例（附录 A.3）

| 反例 | 结果 |
|---|---|
| glibc 行检查 uClibc 产物 | 退出 1：可执行文件报 interpreter、`NEEDED` 缺 `libc.so.6` 且有 `libc.so.0`，`librga.so` 报后两项 |
| uClibc 行检查 glibc 产物 | 退出 1，各项相反 |
| 调用 `sscanf`、`strtol` 的程序替换 yolov5 | 退出 1：`要求 GLIBC_2.38，超过板上的 GLIBC_2.35；引用它的符号：__isoc23_sscanf@GLIBC_2.38 __isoc23_strtol@GLIBC_2.38`，汇总 1/4 个变体未通过 |
| 删去 yolov5 的可执行文件 | 退出 1：`产物不存在或为空` |
| uClibc 的 `librga.so` 放进 glibc 部署目录 | 退出 1：`librga.so` 报 `NEEDED` 中有 `libc.so.0`、没有 `libc.so.6` |
| `VARIANTS` 删去 yolov5 一行 | 编译前退出 1：`example/ 下的例程未加入 VARIANTS：luckfox_pico_yolov5`，没有生成任何构建目录 |
| 正常产物 | 退出 0 |

### 5.4 V4 开发期 CI（2026-11-28，北京时间）

- 首次推送 workflow 后，「构建 Luckfox Pico RKNN 例程」即为 `active`，PR 上自动触发；`agents-md-size.yml` 只在改了根 `AGENTS.md` 时触发，此时尚未出现在 workflow 列表中。
- 临时提交 1（PR 上构建临时 tag）：[run 46124805343](https://github.com/yuangezhizao/luckfox_pico_rknn_example/actions/runs/46124805343) 16:13–16:15 全部 5 个 job 通过，约 2 分 16 秒。`build-image` 推送 `pr-2-92806b15473f7871e71e7c7fa5e858a0`（digest `sha256:6393104c…`，不 attest），约 1.5 分钟；两行编译各约 35 秒，「4 个变体编译成功」「4 个变体通过兼容性检查」，uClibc 工具链 `665c7f1`；`📝 文档检查` 用 markdown-it-py 3.0.0，4 个文件 0 处、样例 115 个全部符合。
- 两个 artifact 各 21 个文件（4 个部署目录的 20 个文件与 `SHA256SUMS`）、33 MB，`sha256sum -c` 全部通过。
- 临时提交 2（spec 的 `## 4.` 改为 `## 9.`）：[run 46125037888](https://github.com/yuangezhizao/luckfox_pico_rknn_example/actions/runs/46125037888) 中只有 `📝 文档检查` 失败，报 2 处章号不连续，退出码 1。
- 两个临时提交随后用 `git reset --keep` 去掉，最终历史中没有它们。

### 5.5 V5 最终形态 CI

- 去掉临时提交后的 `64afb4a`：[run 46125262801](https://github.com/yuangezhizao/luckfox_pico_rknn_example/actions/runs/46125262801) 中 `build-image` 报 `pull_request：只复用 dev 签名镜像` 与 `ghcr.io/yuangezhizao/luckfox-rknn-example-ci:92806b15473f7871e71e7c7fa5e858a0: not found` 后失败（临时 tag 不被复用路径使用），两行编译与 `📝 文档检查` 被跳过，`🐚 ShellCheck` 通过；与 spec §2.3 的预期相同。根 `AGENTS.md` 的改动并入后，体积检查的结果见 §5.6。

### 5.6 整理提交后（2026-11-28）

- `git rebase -i --autosquash`（把 spec 与 plan 的提交连同 fixup 移到末尾）前后文件树相同，各提交的 author time 不变，8 个提交都带 SSH 签名，`Co-authored-by` 只有 noreply 地址。
- 逐个提交检出（rkmpi 仓 PR #2 plan 附录 A.8 的循环）：有 workflow 的提交 actionlint 0 条，ShellCheck 均为 0 条，文档检查均为 0 处，有样例的提交均为 115 个全部符合，根 `AGENTS.md` 依次为 8,184 与 8,192 字节。
- 推送 `cf1d292` 后：[run 46132989482](https://github.com/yuangezhizao/luckfox_pico_rknn_example/actions/runs/46132989482) 体积检查通过（`AGENTS.md: 8192 / 8192 字节`）；[run 46132989457](https://github.com/yuangezhizao/luckfox_pico_rknn_example/actions/runs/46132989457) 与 V5 相同，`build-image` 失败、容器 job 跳过、ShellCheck 通过。此后只改了注释与文档（spec §2.1.1、Q7），每次推送的结果相同。

### 5.7 spidev 板型对比（spec §2.1.1，2026-11-28）

| 构建 | glibc：GNU build-id / sha256 前 16 位 | uClibc：sha256 前 16 位（无 build-id） |
|---|---|---|
| 不带板型 | `4d059cda…` / `187e50ce2bb2a55f` | `64fdcbf45ba5a3a9` |
| `-DLUCKFOX_PICO_PRO_MAX=ON` | `4d059cda…` / `187e50ce2bb2a55f` | `64fdcbf45ba5a3a9` |
| `-DLUCKFOX_PICO_PLUS=ON` | `54e01ad6…` / `1e62e2d7695660af` | `4069147090e70ac4` |

不带板型与 `PRO_MAX` 两种 libc 下都 `cmp` 相同；三者文件大小相同（glibc 3,406,216 字节），`PLUS` 的差异在 build-id 与代码中的引脚、SPI 速率常量。复现：

```bash
# glibc；uClibc 把 GLIBC_COMPILER 换成 LUCKFOX_SDK_PATH=/tmp/luckfox-pico、LIBC_TYPE 换成 uclibc
for v in "" -DLUCKFOX_PICO_PRO_MAX=ON -DLUCKFOX_PICO_PLUS=ON; do
  b=/tmp/cmp-${v//[^A-Z]/}; rm -rf "$b"
  GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf- cmake -S . -B "$b" -DLIBC_TYPE=glibc -DEXAMPLE_DIR=example/luckfox_pico_retinaface_facenet_spidev -DEXAMPLE_NAME=luckfox_pico_retinaface_facenet_spidev $v > /dev/null
  make -C "$b" -j"$(nproc)" > /dev/null; f=$b/luckfox_pico_retinaface_facenet_spidev
  echo "${v:-无板型} $(readelf -n "$f" | grep -o 'Build ID: .*') $(sha256sum < "$f" | cut -c1-16)"
done
cmp /tmp/cmp-/luckfox_pico_retinaface_facenet_spidev /tmp/cmp-DLUCKFOXPICOPROMAXON/luckfox_pico_retinaface_facenet_spidev && echo same
```

### 5.8 合并前检查（2026-11-28）

- **workflow 是否需要复跑**：V4 之后 workflow 只改了 `VARIANTS` 上方的注释（spec §2.1.1），其余代码改动是 `AGENTS.md` 的文字；V4 的 run 即最终 workflow 的行为，不再用临时提交复跑。
- **链接**：spec、plan、五份 `AGENTS.md`、两份 Dockerfile、两个 workflow、检查脚本与 PR 正文中共 37 个链接（rkmpi 仓 PR #2 plan 附录 A.9 的循环），35 个返回 200；PR 正文中指向 `dev` 的本 spec 与 plan 为 404，合并后才在 `dev` 上。
- **引用**：spec 与 plan 中的 §、附录、Task 与 Q 引用都指向存在的位置；脚本列出的都是指向 rkmpi 仓、lvgl 仓、PR #1 文档的引用或提交模板中的 `§X`。
- **最终审查**：Codex 代码审查无意见；Codex 安全审查 1 条 P2，与 spec Q4 相同。一名只读审查者（`gemini-4-neon-high`）在 `/tmp` 复跑附录 A.1、A.2 与 §5.7，数值与文档一致。

## 6. 与计划的偏离

- 根 `AGENTS.md` 只压缩措辞放不进 8 KiB（超出 311 字节）：作者在 spec Q6 中选择把「模型转换」移到 `scripts/luckfox_onnx_to_rknn/AGENTS.md`，新增该文件并同步 `.cursor/AGENTS.md` 的指针，并入 Task 7 的提交；去向表一行用「须用」以恰好留在 8,192 字节。
- spidev 的板型不只影响 SPI 速率，Plus 还切换引脚（spec §2.1.1、Q7）：workflow 的 `VARIANTS` 注释与 PR #1 spec §2.1、§3.4 按此改写，CI 变体不变。
- 本 VM 与两份镜像都没有 `shellcheck`，按 spec Q5 临时 `apt-get install`（0.9.0），不改 Dockerfile。

## 附录 A

只留以后还会用到、容易踩坑、模型不易直接写对的命令，都在本仓根目录运行。需要：uClibc 工具链稀疏检出到 `/tmp/luckfox-pico`（根 `AGENTS.md`「项目与构建」）；同步 rkmpi 仓时克隆到 `/tmp/rkmpi`（`git clone -q -b dev https://github.com/yuangezhizao/luckfox_pico_rkmpi_example.git /tmp/rkmpi`）；本 VM 与两份镜像都没有 `shellcheck`，用前 `sudo apt-get install -y shellcheck`。日常的文档检查、ShellCheck 命令见根 `AGENTS.md`。

### A.1 本地构建两种 libc × 4 变体并读取 ABI（改例程、依赖或版本上限时复用）

spidev 必须带板型，部署目录要用 `DESTDIR`，否则改写已入库的 `install/`；`top` 读取每个前缀的最高版本要求，调整兼容性检查的上限时对照。

```bash
git status --short > /tmp/st-before
for libc in glibc uclibc; do
  for v in luckfox_pico_retinaface_facenet: luckfox_pico_retinaface_facenet_spidev:-DLUCKFOX_PICO_PLUS=ON luckfox_pico_retinaface_facenet_spidev:-DLUCKFOX_PICO_PRO_MAX=ON luckfox_pico_yolov5:; do
    ex=${v%%:*}; opt=${v#*:}; b=/tmp/build-$libc-$ex-${opt//[^A-Z]/}; rm -rf "$b"
    if [ $libc = glibc ]; then e="GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-"; else e="LUCKFOX_SDK_PATH=/tmp/luckfox-pico"; fi
    { env $e cmake -S . -B "$b" -DEXAMPLE_DIR=example/$ex -DEXAMPLE_NAME=$ex -DLIBC_TYPE=$libc $opt && env $e make -C "$b" -j"$(nproc)" && make -C "$b" install DESTDIR=/tmp/stage-$libc; } > "$b.log" 2>&1
    echo "$libc $ex $opt rc=$? warn=$(grep -c 'warning:' "$b.log")"
  done
done
git status --short | diff /tmp/st-before - && echo worktree-unchanged
top() { readelf -V "$1" | awk '/Version needs/,0' | grep -oE '(GLIBC|GLIBCXX|CXXABI|GCC)_[0-9.]+' | sort -uV | awk -F_ '{m[$1]=$0} END{for(k in m) printf "%s ", m[k]; print ""}'; }
for f in $(find /tmp/stage-* -type f ! -path '*/model/*') $(git ls-files install | grep -v /model/); do
  echo "$f: $(top "$f")| $(readelf -d "$f" | sed -n 's/.*(NEEDED).*\[\(.*\)\]/\1/p' | tr '\n' ' ')"
done
```

### A.2 actionlint 与 rkmpi 仓的差异（改 workflow 或同步 rkmpi 仓时复用）

actionlint 不在镜像中，取 release 的二进制；`build-image` 按名称替换后对齐比较，差异应只有显示名与注释；ShellCheck 的文件列表与 CI 相同，排除上游 `build.sh`。

```bash
V=$(gh api repos/rhysd/actionlint/releases/latest -q .tag_name)
curl -sSfL "https://github.com/rhysd/actionlint/releases/download/${V}/actionlint_${V#v}_linux_amd64.tar.gz" | tar xz -C /tmp actionlint
/tmp/actionlint && echo actionlint-0
mapfile -d '' files < <(git ls-files -z '*.sh' | grep -zvx 'build\.sh'); shellcheck -x -P SCRIPTDIR "${files[@]}" && echo shellcheck-0
K=/tmp/rkmpi
diff $K/tests/tools/docs/check-headings.py tests/tools/docs/check-headings.py && echo headings-same
diff $K/tests/tools/docs/check-samples.sh tests/tools/docs/check-samples.sh
diff $K/.github/workflows/agents-md-size.yml .github/workflows/agents-md-size.yml
diff <(sed -n '1,/^  build-example:/p' $K/.github/workflows/build-luckfox-rkmpi-example.yml | sed 's/rkmpi-example/X/g') \
     <(sed -n '1,/^  build-example:/p' .github/workflows/build-luckfox-rknn-example.yml | sed 's/rknn-example/X/g')
for f in Dockerfile Dockerfile.luckfox_pico; do diff <(grep -v '^\s*#' $K/.cursor/$f) <(grep -v '^\s*#' .cursor/$f) && echo "$f non-comment-same"; done
```

### A.3 在 CI 镜像中原样执行 workflow 的步骤与门禁反例（改 workflow 时复用）

前提：按 PR #1 plan 附录 A.2 临时安装 Docker（官方 Running Docker 的写法，`fuse-overlayfs` 与 `iptables-legacy`）。Dockerfile 不 `COPY`，用空目录作构建上下文；容器 job 要 `--user 0`；uClibc 行的工具链在 `$GITHUB_WORKSPACE/luckfox-pico`；多行的 `VARIANTS` 以单引号原样导出即可。

```bash
sudo apt-get install -y -qq python3-yaml
sudo docker build -q -f .cursor/Dockerfile -t rknn-ci:active "$(mktemp -d)"
cat > /tmp/extract-steps.py <<'EOF'
import os, sys, yaml
wf, job, out = sys.argv[1:]
j = yaml.safe_load(open(wf, encoding="utf-8"))["jobs"][job]
os.makedirs(out, exist_ok=True)
for i, s in enumerate(x for x in j["steps"] if "run" in x):
    env = {**j.get("env", {}), **s.get("env", {})}
    with open(f"{out}/{i:02d}.sh", "w", encoding="utf-8") as fh:
        fh.write(f"# {s['name']}\n")
        fh.writelines(f"export {k}='{v}'\n" for k, v in env.items() if "${{" not in str(v))
        fh.write(s["run"])
EOF
python3 /tmp/extract-steps.py .github/workflows/build-luckfox-rknn-example.yml build-example /tmp/ci-steps   # 00 编译、01 清单、02 兼容性检查、03 SHA256SUMS
for l in glibc uclibc; do
  ws=/tmp/v2-ws-$l; sudo rm -rf $ws; mkdir -p $ws; git archive HEAD | tar -x -C $ws
  extra=""; [ $l = uclibc ] && extra="-v /tmp/luckfox-pico:/ws/luckfox-pico:ro"
  sudo docker run --rm --user 0 -v $ws:/ws $extra -v /tmp/ci-steps:/steps:ro -w /ws -e LIBC=$l -e GITHUB_WORKSPACE=/ws rknn-ci:active \
    bash -c 'for s in /steps/*.sh; do bash "$s" || exit 1; done' > /tmp/v2-$l.log 2>&1; echo "$l rc=$?"
done
# 反例：在部署目录的副本上只跑兼容性检查；t 为 rkmpi 仓 PR #2 plan 附录 A.3 的 t.cpp（调用 sscanf、strtol，要求 GLIBC_2.38）交叉编译所得
v3() {  # $1=名称 $2=检查用的 LIBC $3=产物来源 $4=可选：在部署目录中执行的改动
  local ws=/tmp/v3-$1; sudo rm -rf "$ws"; mkdir -p "$ws/install/$2"; cp -r "$3"/. "$ws/install/$2/"
  [ -n "${4:-}" ] && (cd "$ws/install/$2" && eval "$4")
  (cd "$ws" && LIBC=$2 bash /tmp/ci-steps/02.sh > out.log 2>&1); echo "== $1 rc=$?"; grep -E '::error|❌' "$ws/out.log" | sort -u | head -5
}
G=/tmp/v2-ws-glibc/install/glibc; U=/tmp/v2-ws-uclibc/install/uclibc
v3 glibc-on-uclibc glibc $U; v3 uclibc-on-glibc uclibc $G
v3 glibc-2.38 glibc $G 'cp /tmp/abi/t luckfox_pico_yolov5_demo/luckfox_pico_yolov5'
v3 wrong-librga glibc $G "cp $U/luckfox_pico_yolov5_demo/lib/librga.so luckfox_pico_yolov5_demo/lib/"
sed '/^luckfox_pico_yolov5$/d' /tmp/ci-steps/00.sh > /tmp/00-missing.sh   # VARIANTS 漏例程：在 git archive 的副本中运行，应在编译前失败
```

### A.4 在 PR 上验证容器 job（改 `.cursor/Dockerfile` 的 PR 复用）

最终形态下 PR 只复用 dev 签名镜像，改了 Dockerfile 的 PR 上容器 job 必然被跳过。开发期加一个临时提交：`build-image` 构建步骤的 `env` 加 `PR_NUMBER: ${{ github.event.pull_request.number }}`，`pull_request` 分支换成下面几行（推送临时 tag，`REBUILT` 保持 `false`，不 attest），其余分支不变：

```bash
          if [ "${GITHUB_EVENT_NAME}" = "pull_request" ]; then
            docker buildx build --push -f .cursor/Dockerfile -t "${IMAGE_REF}:pr-${PR_NUMBER}-${TAG}" --metadata-file /tmp/bx-meta.json .cursor
            DIGEST=$(jq -er '."containerimage.digest" // empty' /tmp/bx-meta.json)
            assert_digest "临时" "${DIGEST}"
            SOURCE="🚧 临时 tag pr-${PR_NUMBER}-${TAG}（不 attest）"
```

- 同一 PR 的新推送会取消正在运行的 run，下一个临时提交要等上一个 run 结束再推。
- 验证后用 `git reset --keep <临时提交之前的提交>` 去掉临时提交，工作区中未提交的改动保留；run 与 artifact 的核对命令见 rkmpi 仓 PR #2 plan 附录 A.7。

### A.5 整理提交与 PR 正文（以后的 PR 复用）

```bash
E="env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL"
# spec 与 plan 的提交连同其 fixup 移到末尾，其余按 autosquash
cat > /tmp/seq.py <<'EOF'
import os, sys
p = sys.argv[1]; key = os.environ["SPEC_SUBJECT"]
todo = [l for l in open(p, encoding="utf-8").read().splitlines() if l and not l.startswith("#")]
start = next(i for i, l in enumerate(todo) if l.startswith("pick ") and key in l)
end = start + 1
while end < len(todo) and todo[end].split()[0] in ("fixup", "f", "squash", "s"):
    end += 1
open(p, "w", encoding="utf-8").write("\n".join(todo[:start] + todo[end:] + todo[start:end]) + "\n")
EOF
before_tree=$(git rev-parse HEAD^{tree}); git log --format='%s|%aI' origin/dev..HEAD | grep -v '^fixup!\|^amend!' | sort > /tmp/at-before
SPEC_SUBJECT='新增 GitHub Actions CI spec 与 plan' GIT_SEQUENCE_EDITOR='python3 /tmp/seq.py' $E git rebase -i --autosquash origin/dev
[ "$(git rev-parse HEAD^{tree})" = "$before_tree" ] && echo tree-same; git log --format='%s|%aI' origin/dev..HEAD | sort | diff /tmp/at-before - && echo authortime-same
# 只改某个提交的说明：amend! 提交，autosquash 时连同内容并入；amend.txt 首行为「amend! <原标题>」，空行后为新的完整说明
GIT_EDITOR="cp /tmp/amend.txt" $E git commit --fixup=amend:<提交>
```

- 逐个提交检出后的检查用 rkmpi 仓 PR #2 plan 附录 A.8 的循环：`git archive` 中没有 `.git`，actionlint 要显式传 `.github/workflows/*.yml`；检查脚本入库之前的提交改用 `/tmp/rkmpi` 的 `check-headings.py`。
- Cloud Agent 的 PR 管理工具不读取 `$file:` 路径（会把路径原样写进正文），要直接传全文；改正文前先用 `gh pr view --json body` 读取当前正文，保留作者在 GitHub 上的修改。
