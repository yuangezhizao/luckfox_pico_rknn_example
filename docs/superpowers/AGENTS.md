# spec 与 plan

写或改 `docs/superpowers/specs/`、`docs/superpowers/plans/` 下的文档前读本文件。

- **标题编号**：`## 2. 背景`、`### 2.2 小节`、`#### 2.2.1 小节`，最多四级，引用写成 §2.2.1；代码块内以 `#` 开头的行不是标题。
- **Task**：plan 的 Task 标题写成 `### Task N：…`（全角冒号），Task 号即序号，不叠章节号；Task 号只用数字，从 0 或 1 起逐个加 1，精简 plan 删掉 Task 后，后面的 Task 重新编号；执行中途插入 Task 时按位置插入，后面的 Task 顺延并同步引用；Task 放在单独的 Task 章（如 `## 4. Tasks`），Task 以外的小节照常编号，不与 Task 混在同一章。
- **附录与 QA**：附录写成 `## 附录 A`，附录内小节写成 `### A.1`，与正文分开编号；设计决策 QA 条目用加粗的 `**Q1. …**`，从 Q1 连续编号，删减后重新编号。
- **检查**：本仓的检查脚本尚未入库，随 CI PR 入库（spec [`2026-11-27-rknn-example-cloudagent-env-design.md`](specs/2026-11-27-rknn-example-cloudagent-env-design.md) §7）。在此之前用 luckfox_pico_rkmpi_example 仓的同名脚本：`git clone --depth 1 -b dev https://github.com/yuangezhizao/luckfox_pico_rkmpi_example.git /tmp/rkmpi` 后在本仓根目录运行 `python3 /tmp/rkmpi/tests/tools/docs/check-headings.py [文件…]`，不带参数时检查当前目录下全部 spec 与 plan，有问题时逐行打印并退出 1；该脚本的 QA 豁免名单为空，规则与本仓一致。依赖 `python3-markdown-it`（`.cursor/Dockerfile` 已安装），本机缺失时 `sudo apt-get install -y python3-markdown-it`。
