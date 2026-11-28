# GitHub Actions

改 `.github/workflows/` 下的 workflow 或排查 CI 失败前读本文件。CI 成功只证明交叉编译与兼容性检查通过，不能代替真机；摄像头、NPU 与屏幕只能在物理板子上验证。

## 交叉编译

- `🛠️ glibc · Ubuntu` 只设 `GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-`；`🛠️ uClibc · Buildroot` 只设 `LUCKFOX_SDK_PATH`，指向 CI 内 `yuangezhizao/luckfox-pico@dev` 的稀疏检出（`tools/linux/toolchain/`，跟随 `dev`，日志打印 SHA），不引入 submodule。
- 每行按 `build-example` 的 job 级 `env` 中的 `VARIANTS` 编译 4 个变体：每行「例程 [板型开关]」，与 `build.sh` 的选项一一对应，spidev 按 `LUCKFOX_PICO_PLUS`、`LUCKFOX_PICO_PRO_MAX` 各编译一次。新增例程或板型时同步修改 `VARIANTS`；`example/` 下有例程不在清单中时，编译前即报错。某个变体失败时继续编译其余变体，最后列出失败的变体，每个变体的输出折叠在同名分组中。
- 编译前删去检出中已入库的 `install/<libc>/`，`make install` 后它只含本次产物：4 个部署目录，各含可执行文件、`lib/librga.so`（`3rdparty/` 的预编译库）与 `model/`。artifact `luckfox_pico_rknn_example_<libc>-<rootfs>_on-ubuntu-24.04` 就是该目录，其中的 `SHA256SUMS` 覆盖全部文件。CI 产物由 CI 镜像的工具链编译，与入库的二进制不逐字节相同（入库的 glibc 二进制还多依赖 `libfunc_none.so`，24.04 工具链默认 `--as-needed`，CI 产物不依赖它）；`install/` 仍只在作者要求时更新（根 [`AGENTS.md`](../AGENTS.md)）。
- 兼容性检查（步骤名按行显示为 `✅ 兼容性检查（含版本上限）` 或 `✅ 兼容性检查（uClibc 不查版本上限）`）：每个可执行文件与 `lib/*.so` 为 ELF32、ARM，`NEEDED` 中的 libc 符合本行，可执行文件的 interpreter 符合本行（共享库没有 interpreter）；glibc 行再要求 Version needs 中 `GLIBC_`、`GLIBCXX_`、`CXXABI_`、`GCC_` 的最高版本不超过板上 Ubuntu 22.04 的库，上限写在该步骤的 `env` 中。CI 工具链来自 Ubuntu 24.04，例如调用 `sscanf`、`strtol` 就会要求 `GLIBC_2.38`，编译没有告警，板上动态链接器直接拒绝启动；超限时报错列出引用它的符号。上限的出处、实测与调整方法见 [luckfox_pico_rkmpi_example 仓 CI spec](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/blob/cf2a0dbb99416858454bef9f9108a028362e75d9/docs/superpowers/specs/2026-11-17-rkmpi-example-github-actions-ci-design.md) §2.6。

## CI 镜像

- CI 镜像 `ghcr.io/<owner>/luckfox-rknn-example-ci` 只在 `dev` 上构建并 attest；`pull_request` 只复用 dest-lock（signer/source=`dev`）通过的 digest，失败即退出，不覆盖 GHCR。在非 `dev` 分支手动触发时同样只能复用：勾选 `force_rebuild`，或 tag 不存在（如该分支改了 Dockerfile）需要首建时，`build-image` 在推送前报错退出，避免 feature 分支签名的镜像占用正式 tag、使之后 `dev` 的 dest-lock 一直失败。
- 镜像标签取 `.cursor/Dockerfile` 内容哈希（含注释），所以修改该文件的 PR 在合并前 `build-image` 必然失败（找不到 dev 签名镜像），依赖它的容器 job 被跳过，合并后 dev push 重建镜像才恢复。合并前要在 PR 上验证时，用只在开发期存在的临时提交把镜像推到 `pr-<PR 号>-<哈希>` 这类临时 tag，合并前用 rebase 删去（spec [`2026-11-28-rknn-example-github-actions-ci-design.md`](../docs/superpowers/specs/2026-11-28-rknn-example-github-actions-ci-design.md) §2.3）。
- dest-lock 失败不一定是投毒：正式 tag 存在但没有 `dev` 签名（如 run 在推送之后、签名之前中断），同样报「强验证未通过」，宽松复验打印实际的 signer/source 或「未取到 attestation」。确认 tag 不是外部推送的之后，在 `dev` 上手动触发并勾选 `force_rebuild`，重建并重新签名即可恢复。`dev` 推送、手动触发与 `schedule` 的 run 排队而不互相取消（只有 `pull_request` 取消旧 run），以免在推送与签名之间被打断。
- 镜像末尾 `USER ubuntu`，`build-example` 与 `doc-check` 以 `options: --user 0` 运行（Actions 要求 job 容器为超级用户）。
- `build-image` 把触发事件、分支、镜像来源（复用、首建、强制重建）与完整镜像引用写入 run 摘要页，可从 artifact 所在 run 追溯编译用的镜像。
- `schedule` 每月 1 日北京时间 08:00 无缓存重建镜像并 attest，随后用新镜像编译与做文档检查；在 `dev` 上手动触发并勾选 `force_rebuild` 同样无缓存重建。fork 中 scheduled workflow 默认禁用，公开仓库 60 天无活动也会停用，停用时在 Actions 页或用 `gh workflow enable` 启用。

## 各 job

- `🐚 ShellCheck` 在 runner 上直接运行（不用容器、不依赖 `build-image`），对 `git ls-files` 列出的 `*.sh` 执行 `shellcheck -x -P SCRIPTDIR`，要求 0 条（含 style 级）；上游的 `build.sh` 暂时排除，修复它的 PR 删去排除。两份 Dockerfile 都不装 `shellcheck`，本地检查前 `sudo apt-get install -y shellcheck`。
- `📝 文档检查`（`doc-check`）用同一镜像，每次完整运行，不按改动文件跳过：先 `tests/tools/docs/check-headings.py` 检查全部 spec 与 plan，再 `check-samples.sh` 运行检查器自身的样例（用法见 [`docs/superpowers/AGENTS.md`](../docs/superpowers/AGENTS.md)）。修改 `.cursor/Dockerfile` 的 PR 上它与其他容器 job 一同被跳过；只改 `.md` 的 dev push 因 `paths-ignore` 不触发 workflow，文档在 PR 上已检查。
- `📏 检查根 AGENTS.md 的体积` 是单独的 workflow `agents-md-size.yml`：`dev` 的推送与指向 `dev` 的 PR 中改了根 [`AGENTS.md`](../AGENTS.md) 时触发，在 runner 上直接运行，所以只改 `.md` 的 dev push 与改 Dockerfile 的 PR 上也照常检查；超过 8,192 字节时失败。超限时先按 lvgl 仓的[放置原则](https://github.com/yuangezhizao/luckfox_pico_lvgl_example/blob/11497f3d57d1e74eeade30a07e79a19c0f7f2369/docs/superpowers/specs/2026-11-01-lvgl-example-agents-md-slim-design.md) §3.1 把内容移到子目录说明或代码注释，再考虑调整预算。
