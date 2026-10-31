# table-peek

简体中文 | [English](README.md)

让 agent 在动手分析之前，先看清一张巨大的表格文件。

它是一个只依赖 Python 标准库的脚本加一份 skill 说明：读一遍表格，产出几个小文件，告诉模型这张表是什么结构、一行代表什么、数据在哪、哪里不确定。需要真实数据时，可以按条件取回有限几行，而不是把整张表塞进上下文。

仓库名沿用了旧名 `tabular-file-understanding-agent-skill`，skill 本体叫 `table-peek`。

## 为什么需要它

把一张几 GB 的表直接丢给模型，结果通常是：超出上下文、被截断，或者模型盯着前几百行原始数据却看不出结构。真正的难点是第一步，这张表到底怎么组织。

几种常见的坑：

- IMF BOP/PIP 导出文件：几列描述字段后面跟着 `1948` 到 `2025-Q4` 几百个期间列，值散落在这些列里，`PUBLICATION_DATE` 看着像时间列但不是观测期。
- World Bank 工作簿：前几行是标题和注释，真表头在第 5 行，还带合并单元格。
- EPU 全国家表：`Year`、`Month` 是键，后面几十个国家列是并列的数值列。

这些文件让模型硬读，`SCALE.ID` 容易被当成数值列，发布日容易被当成观测期。这个工具先把这些判断做掉，并列出需要确认的地方。

## 它会产出什么

每次运行生成 6 个文件：

| 文件 | 内容 |
|---|---|
| `table_manifest.json` | 文件清单、候选表或 sheet、能力等级、警告 |
| `table_profile.json` | 每列的类型、示例值、缺失率、统计量 |
| `table_digest.md` | 给模型读的结构摘要，回答"一行代表什么" |
| `ambiguities.md` | 需要人确认的疑点 |
| `data_locator_spec.json` | 机器可读的定位器：实体列、指标列、时间覆盖、观测键、数值列 |
| `data_locator_guide.md` | 上面那份的说明版，讲清下游怎么按国家码、指标码取数 |

加 `--extract-limit` 时多一个 `sample_extract.csv`，里面是最多 N 行真实数据。

agent 通常只读 `table_digest.md` 和 `ambiguities.md` 就够了，需要取数时再看 locator。

## 快速开始

```bash
git clone https://github.com/yanyintingyou/tabular-file-understanding-agent-skill.git
cd tabular-file-understanding-agent-skill

python skills/data-science/table-peek/scripts/profile_tabular_file.py \
  -i your_file.csv \
  -o ./peek
```

然后看这三个文件：

```text
./peek/table_digest.md
./peek/ambiguities.md
./peek/data_locator_guide.md
```

默认只采样：读 1MB 判编码和分隔符，最多扫 10 万行，列统计基于前 1000 行。文件再大也是这个节奏，几秒钟内出结果。

### 需要精确数字：`--full-scan`

```bash
python skills/data-science/table-peek/scripts/profile_tabular_file.py \
  -i your_file.csv -o ./peek --full-scan
```

流式扫完整份文件，给每列补 `exact_stats`：精确缺失计数和缺失率、数值 min/max/mean、有上限的去重值、按全量数据推断的类型。内存不涨，代价是一次完整读取。

本机实测 100 万行 × 7 列、31MB 的 CSV 约 34 秒，Python 峰值内存 5.2MB。按这个速度，几 GB 文件的全量扫描要按小时算，所以默认不开。

### 需要看真实几行：`--extract-limit`

```bash
python skills/data-science/table-peek/scripts/profile_tabular_file.py \
  -i your_file.csv -o ./peek \
  --extract-limit 20 \
  --extract-where COUNTRY.ID=US
```

`--extract-where` 可以写多次，多个条件按 AND 组合，去空格后精确匹配。扫到足够行数就停，结果写进 `sample_extract.csv`。这条路径只支持 CSV/TSV。

### 敏感文件

```bash
# 遮盖邮箱、电话、长数字
... --redact-samples

# 完全不写行级样本和列级取值，抽取同时被禁用
... --no-samples
```

`--no-samples` 会清掉 `samples`、`examples`、`top_values_sample` 和 locator 里由样本派生的内容，只保留理解 schema 所需的字段名、类型和结构警告。

## 能看懂哪些表

| 表格形态 | 识别结果 |
|---|---|
| 简单长表时间序列 | 时间列、数值列、行数和日期范围，FRED 单序列 CSV 属于这类 |
| 通用长表 / 面板 | 实体列、时间列、分类列、数值列，以及可能的一行粒度 |
| SDMX / 宏观长表 | entity、indicator、counterpart、frequency、time、value、unit/scale/status 九类角色 |
| IMF BOP/PIP 宽时间列 | 描述列加大量期间列，标出 `value_layout: wide_time_columns` 并建议 reshape |
| 宽指标矩阵 | `Year`/`Month` 或 `Date` 作键，其余列作为并列数值列 |
| 报告式 Excel | 搜索真实表头和数据起点，不假设第 1 行是表头，标注合并单元格和注释区 |
| lookup / 分类表 | 代码与标签的映射表，不硬当时间序列 |
| 矩阵 / 交叉表 | 行列都承载维度，标为需要确认 |
| PDF | 只报文件级元数据，不假装抽取了表格 |

遇到不认识的结构，工具会把它标成假设而不是结论，`ambiguities.md` 里会写清哪里需要人确认。

## 跨 Agent 安装

仓库是双布局：根目录 `SKILL.md` 和 `skills/data-science/table-peek/SKILL.md` 内容一致，改一处要同步另一处。

- **Claude Code**：复制 `skills/data-science/table-peek/` 到 `~/.claude/skills/data-science/table-peek/`
- **OpenAI Codex**：在本仓库内直接读 `AGENTS.md`；要复用就复制到 `~/.agents/skills/data-science/table-peek/`
- **OpenClaw**：复制到 `<workspace>/skills/data-science/table-peek/`，或 `~/.openclaw/skills/` 下同名目录
- **Hermes Agent**：复制到 `~/.hermes/skills/data-science/table-peek/`，开新会话
- **Grok Build**：复制到 `~/.grok/skills/table-peek/`
- **通用 AgentSkills 加载器**：指向包含 `SKILL.md` 的目录，名字是 `table-peek`

## 宏观数据定位器

IMF、SDMX 这类国家 × 指标 × 时间的面板，可以加 `--domain-preset`：

```bash
... --domain-preset imf-bop
```

可选值：`auto`（默认）、`generic`、`macro-timeseries`、`sdmx`、`imf-bop`。

本地有 codelist 时传进来，定位器会补上标签；没有就只报代码，并注明需要外部 codelist：

```bash
... --domain-preset sdmx --indicator-mapping indicators.csv --entity-mapping areas.csv
```

映射文件用 `code,label,description` 这类列名。

想知道一个几 GB 的文件里到底有哪些指标，加 `--domain-full-scan`，额外产出 `domain_indicator_catalog.json`、`.csv`、`.md`，包含指标覆盖度、时间范围、单位，以及基于前缀规则的跨境资本流动指标分类。

## 输出体积控制

`--output-policy auto` 是默认值，按文件大小和列数自动选档，只影响生成的 JSON，不动原文件。

| 输入规模 | 策略 | 行为 |
|---|---|---|
| 小于 10MB，列数正常 | `full` | 保留全部 profiling 细节 |
| 中等文件或超过 200 列 | `balanced` | 减少示例和 top values |
| 超过 100MB 或 1000 列 | `compact` | 省略行级样本，保留关键角色列 |
| 超过 1GB 或 5000 列 | `very-compact` | 只保留约 300 列详细画像 |

阈值都能用参数改，也可以用 `--output-policy full|balanced|compact|very-compact` 手工指定。

如果下游是 RAG 或 SQL 流程，加 `--schema-preset generic|rag|sql`，会多写一个 `downstream_schema_hint.json`，给下游一个字段映射的起点。

## 文件类型

| 格式 | 支持程度 | 说明 |
|---|---|---|
| CSV / TSV / 分隔文本 | 强 | 标准库流式处理，支持 `--full-scan` 和抽取 |
| `.xlsx` / `.xlsm` | 实用级别 | 解包读 XML，识别 sheet、used range、合并单元格、候选表头；不执行公式和宏，不支持旧版 `.xls` |
| PDF | 元数据 | 只做保守的文件级检查，不做抽表或 OCR |

核心脚本只用标准库：`csv`、`zipfile`、`xml.etree`、`json`、`re`，Python 3.9+ 能跑。`requirements.txt` 里那五个包是可选的深度分析依赖，脚本本身不 import 它们，`--check-deps` 只负责报告在不在。

## 已知限制

- 默认统计基于样本；要精确数字得开 `--full-scan`，几 GB 文件按小时计。
- `sample_extract.csv` 取的是文件顺序上的前 N 条命中，不是随机抽样，也不是查询引擎。要按多键过滤做分析，把 reshape 后的数据交给 DuckDB 或 pandas。
- Excel 不支持抽取，`--extract-limit` 只作用于 CSV/TSV。
- `.xlsx` 里的日期可能显示为 Excel 序列号，因为不实现完整样式引擎。
- 多行表头、同一 sheet 堆叠多张表、扫描版 PDF、公式驱动的工作簿，仍可能需要定制解析。
- 这个 skill 只做结构理解，不做分析结论、不清洗数据、不改原文件。

## 开发与测试

```bash
python tests/test_profile_examples.py
```

13 个回归测试，只用标准库，覆盖 FRED 时间序列、IMF 宽时间列、EPU 宽指标、World Bank 报告式表头、精确统计、有界抽取、隐私选项和宽表截断。GitHub Actions 在 Python 3.9 到 3.12 上跑同一套。

改 CLI 行为、隐私逻辑或输出 schema 时，同步更新两份 `SKILL.md`、`references/` 下对应文档和 `CHANGELOG.md`。

## License

MIT。作者 [yanyintingyou](https://github.com/yanyintingyou)。
