# 模型转换

转换或重新量化 `scripts/luckfox_onnx_to_rknn/` 下的模型前读本文件；步骤见同目录的 [`README_CN.md`](README_CN.md)。

- 只能用 RKNN-Toolkit2 **1.6.0**：板上固件的 `librknnmrt` 与本仓 `lib/*/librknnmrt.so` 都是 1.6.0，只认 RKNN v5 模型；2.x 转出的 v6 模型板上报 `incompatible model version`。PyPI 只有 2.x，`pip install rknn-toolkit2` 会装错版本。
- whl 取 `airockchip/rknn-toolkit2` 的 `v1.6.0` 标签（`rknn-toolkit2/packages/`，cp36–cp311、x86_64），需 Python ≤ 3.11；`.cursor/Dockerfile` 为 3.12，要另装解释器。两份镜像都不预装 toolkit，也没有 `pip`、`venv`。
- 背景与依据（版本对应、两个 rknn-toolkit2 仓库的关系、升级到 2.x 涉及的范围）见 spec [`2026-11-27-rknn-example-cloudagent-env-design.md`](../../docs/superpowers/specs/2026-11-27-rknn-example-cloudagent-env-design.md) §2.7、§2.8。
