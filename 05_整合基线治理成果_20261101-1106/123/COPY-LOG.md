# COPY-LOG — Desktop\123

**时间**：2026-11-06 16:56  
**方式**：robocopy（目录）/ Copy-Item（文件）；源文件只读，未修改  
**校验**：见 MANIFEST-SHA256.txt（318 个文件）

| 目标目录 | 来源 | 文件数 | MB |
|---|---|---:|---:|
| 01-当前基线-canonical | C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design（robocopy /E，排除 node_modules/pydeps/__pycache__/.pnpm） | 121 | 1.57 |
| 02-实施规格 | C:\Users\[USER]\Desktop\work\ 根目录 10 份：00,01,02,04,05,06,07,13,14,15 | 10 | 0.19 |
| 03-治理与裁决 | C:\Users\[USER]\Desktop\work\ 根目录 5 份：08,09-05,10-06,11,12 | 5 | 0.08 |
| 04-模块设计-R3 | work\03-m01…m05 + 03-m01-problem + data-samples（2） | 8 | 1.77 |
| 05-历史机制-V2.4.2 | 时间流\2026-10-31_04_V2.4.2设计收敛\2.42（robocopy） | 13 | 0.18 |
| 06-历史机制-R2 | 时间流\2026-10-31_03_R2架构基线冻结\R2架构-最新-20261101\设计文档（robocopy） | 21 | 0.16 |
| 07-人格管线 | 时间流\…\R3-MemoryCore-全套产物-20261102-1723\04-人格管线方案（4 份） | 4 | 0.10 |
| 08-心理模型 | 时间流\…\F_moren\psy-model 顶层 44 个文件（不含 _backup） | 44 | 0.48 |
| 09-情绪与框架 | 时间流 R3情绪体系整合方案 + 06-情绪体系整合方案 + 04-框架结构优化方案 | 3 | 0.07 |
| 10-独立审查-B | 时间流\2026-11-03_13_B独立审查交付\B（18 份） | 18 | 0.68 |
| 11-压缩桥-CMB | E:\[PROJECT]\CMB-v0.1-review-pack（robocopy，40 份） | 40 | 0.78 |
| 12-本轮审查报告 | C:\Users\[USER]\Desktop\fin（11 份） | 11 | 0.19 |
| 13-上一轮能力审查 | E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026（8 md + EVIDENCE） | 20 | 0.21 |

**合计**：320 文件 / 6.51 MB

## 未复制（有意排除）

| 类别 | 体量 | 原因 |
|---|---:|---|
| 明文凭据（apikey.csv / SecretKey.csv / API密钥 txt） | <1 KB | 安全，建议轮换 |
| ChatGPT 私人对话导出 | 34.4 MB | 隐私，与项目分离 |
| companion sqlite 运行库（5+ 副本） | ~12.7 MB | 私人运行数据 |
| node_modules / pydeps / pyc | 19.4 MB | 可重建 |
| WAL/SHM / .bak / 字体 / 媒体 | ~34 MB | 垃圾与副本 |
| zip 压缩包（82 个） | 48 MB | 多数已有解包目录，未逐一比对 |
| 时间流全量（6,116 文件） | 249 MB | 只取机制文档；全量保留原处 |
## 本轮新写（非复制）

| 文件 | 说明 |
|---|---|
| 00-README-本目录说明.md | 本目录导航与边界 |
| 14-历史进程与关键转折.md | 八阶段时间轴/版本谱系/九条主线/六轮裁决/遗留项 |
| COPY-LOG.md / MANIFEST-SHA256.txt | 复制记录与校验 |
