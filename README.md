# DedupeFiles

文本文件去重工具（Swift 单文件实现）。两道流程：**完全相同**检测与**近重复**检测，支持 dry-run 预览、安全移动（不删除）、以及 `--undo` 撤销。

## 特性

- **第一道 · 完全相同**：基于 `size + 双 FNV-1a 128 位指纹`，同组保留路径排序最靠前的文件。
- **第二道 · 近重复**：代表文本（默认首部 + 尾部各取一半，共 `--max-chars` 字符；`--full-text` 用全文）→ 字符清理（汉字保留、英文转小写、剔除标点/空白）→ k-shingle（默认 3-gram，滑窗去重）→ 停用词过滤 → 文档级停用词（DF ≥ `--df-cut` 剔除）→ MinHash 128 维签名 → LSH 8 bands × 16 rows 召回 → 桶内签名粗筛（相同位数 / 128 > `--sim`）→ 连通分量内贪心选代表（每个被删文档都与保留代表直接相似）。
- **大小比保护**：开头相同但体积差异超过 `--size-ratio` 倍的文件（如合集 vs 单册）改标「部分重复」，只报告不移动，避免误删。
- **缓存**：提取阶段结果写入 `--cache`，按 `size-mtime` 指纹失效，重跑增量复用。
- **安全**：`--apply` 仅把重复文件移入重复文件夹，绝不删除；重名自动加序号；写 manifest 支持 `--undo` 恢复。

## 构建

依赖 `CoreWordKit.swift` 提供的 `WordFilters`（停用词表），需与本文件一同编译：

```bash
swiftc -O -o /tmp/bookdedupe tools/CoreWordKit.swift tools/DedupeFiles.swift
```

## 用法

```
DedupeFiles <目录> [选项]
```

典型用法：

```bash
# 预览（dry-run，不移动任何文件）
bookdedupe ./my_books

# 实际去重（重复文件移入 <目录>/_重复）
bookdedupe ./my_books --apply

# 撤销上次移动（按 manifest 把文件移回原位）
bookdedupe --undo ./out/manifest.json
```

### 选项

| 选项 | 说明 | 默认 |
|------|------|------|
| `--apply` | 执行移动（默认 dry-run 只报告） | off |
| `--undo PATH` | 按 manifest 撤销上次移动 | — |
| `--limit N` | 只处理前 N 个文件 | 0（全部） |
| `--max-chars N` | 代表文本长度：首部+尾部各一半，合计 N 字符；N=0 等价全文 | 20000 |
| `--full-text` | 不截取，用全文计算签名（更慢更占内存） | off |
| `--shingle-k N` | shingle 窗口长度 | 3 |
| `--sim X` | 签名粗筛阈值（相同位数/128） | 0.85 |
| `--size-ratio X` | 判重的大小比上限；超过改标「部分重复」不移动；0=关闭 | 3.0 |
| `--df-cut X` | 文档级停用词阈值；0=关闭 | 0.5 |
| `--no-stopwords` | 关闭停用词过滤 | off |
| `--stopwords PATH` | 追加自定义停用词文件 | — |
| `--ext LIST` | 扩展名列表，逗号分隔 | md,txt |
| `--max-bucket-size N` | LSH 单桶上限，超过跳过 | 2000 |
| `--jobs N` | 并发数（默认 = CPU 核数） | 0 |
| `--cache PATH` | 提取缓存路径 | — |
| `--report PATH` | 报告输出路径（JSON） | — |
| `--out-root PATH` | 重复文件夹创建位置（默认 = 输入目录） | — |
| `--duplicates-name NAME` | 重复文件夹名 | `_重复` |
| `--manifest PATH` | 移动清单路径（`--apply` 时写，供 `--undo`） | — |
| `--self-test` | 运行内置自检 | — |
| `--help` | 显示帮助 | — |

## 算法说明

### 代表文本

默认截取「前 `maxChars/2` + 后 `maxChars - maxChars/2`」两段，分段分别生成 shingle，避免首/尾拼接处产生虚假跨段 k-gram。清理后全文 ≤ `maxChars` 时直接使用全文。两遍线性扫描，无需整本大书常驻内存。

### 指纹与签名

- **FNV-1a 128**：对文件字节流用两个不同种子计算双指纹，配合文件大小判定完全重复。
- **MinHash 128**：128 组独立仿射哈希函数（a·x+b mod 2^61-1）下各取最小值，估计 Jaccard 相似度。
- **LSH**：签名切 8 段 × 16 维，段内全同进同桶召回候选对，再对候选对按签名 agreement 粗筛。

### 贪心保留

连通分量内强制选**体积最大**的文件作唯一代表，只删与最大文件**直接相似**（共享边）的成员——被删文件必与保留代表直接高度相似，且绝不会比保留文件更大；与代表无直接相似的链末端成员则保留，不误删。

## 自检

```bash
bookdedupe --self-test
```

运行字符清理、代表文本截取、shingle、停用词、DF 过滤、指纹、MinHash/agreement、LSH、贪心保留以及端到端（dry-run → apply → undo）等全部断言，全部通过则输出 `all checks passed`。

## 输出

dry-run / apply 结束后会在 stderr 打印统计（文件数、完全重复、近重复、部分重复、移除数量、节省空间、耗时）；若指定 `--report`，额外写出 JSON 报告（含每个重复组的代表与成员、类型、大小等）。
