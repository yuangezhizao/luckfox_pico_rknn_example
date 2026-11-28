# spec 与 plan

写或改 `docs/superpowers/specs/`、`docs/superpowers/plans/` 下的文档前读本文件。

- **标题编号**：`## 2. 背景`、`### 2.2 小节`、`#### 2.2.1 小节`，最多四级，引用写成 §2.2.1；代码块内以 `#` 开头的行不是标题。
- **Task**：plan 的 Task 标题写成 `### Task N：…`（全角冒号），Task 号即序号，不叠章节号；Task 号只用数字，从 0 或 1 起逐个加 1，精简 plan 删掉 Task 后，后面的 Task 重新编号；执行中途插入 Task 时按位置插入，后面的 Task 顺延并同步引用；Task 放在单独的 Task 章（如 `## 4. Tasks`），Task 以外的小节照常编号，不与 Task 混在同一章。
- **附录与 QA**：附录写成 `## 附录 A`，附录内小节写成 `### A.1`，与正文分开编号；设计决策 QA 条目用加粗的 `**Q1. …**`，从 Q1 连续编号，删减后重新编号。
- **检查**：在仓库根目录运行 `python3 tests/tools/docs/check-headings.py [文件…]`，不带参数时检查全部 spec 与 plan，有问题时逐行打印并退出 1；修改检查器时再运行 `bash tests/tools/docs/check-samples.sh`（检查器的回归样例，全部符合时末行为「样例 N 个，不符合 0 个」；其中的 `mk` 样例改写 [yuangezhizao/luckfox_pico_rknn_example#1](https://github.com/yuangezhizao/luckfox_pico_rknn_example/pull/1) 的 spec 与 plan，修改这两份文档的原文时同步调整样例）。CI 的 `📝 文档检查` 运行这两个脚本。两者依赖 `python3-markdown-it`（`.cursor/Dockerfile` 已安装），本机缺失时 `sudo apt-get install -y python3-markdown-it`。
