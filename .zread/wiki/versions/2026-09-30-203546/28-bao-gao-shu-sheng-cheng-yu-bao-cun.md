一次完整的交易分析会在多个智能体之间流转，最终产出一系列 Markdown 报告。本页说明这些报告如何被组织成"报告树"并落盘保存。无论你从交互式 CLI 触发，还是通过 Python API 以编程方式调用，最终写入磁盘的内容都来自同一个共享写入器，因此两种入口得到的文件结构完全一致。本页聚焦于**报告树的结构、写入逻辑、保存路径与调用方式**；各智能体报告的**具体内容**请参见[分析师团队](15-fen-xi-shi-tuan-dui-ji-ben-mian-shi-chang-xin-wen-yu-qing-xu)与[研究员辩论与结构化输出](16-yan-jiu-yuan-bian-lun-yu-jie-gou-hua-shu-chu)。

## 报告树是什么

报告树是一次运行的产物目录。它把分散在最终状态（`final_state`）各字段中的文本，按分析阶段拆分为编号子目录，并额外生成一份把所有内容拼接在一起的 `complete_report.md`。代码顶部的文档字符串直接点明了它的定位：CLI 与 `TradingAgentsGraph.save_reports` 都调用它，因此 headless / API 运行产生与 CLI 运行相同的磁盘报告树。

树的根目录由 `save_path` 决定，其下按阶段编号排列五个子目录，加一个合并报告文件：

```mermaid
graph TD
    Root["save_path/ (一次运行的根目录)"]
    Root --> A["1_analysts/"]
    Root --> B["2_research/"]
    Root --> C["3_trading/"]
    Root --> D["4_risk/"]
    Root --> E["5_portfolio/"]
    Root --> F["complete_report.md"]

    A --> A1["market.md"]
    A --> A2["sentiment.md"]
    A --> A3["news.md"]
    A --> A4["fundamentals.md"]

    B --> B1["bull.md"]
    B --> B2["bear.md"]
    B --> B3["manager.md"]

    C --> C1["trader.md"]

    D --> D1["aggressive.md"]
    D --> D2["conservative.md"]
    D --> D3["neutral.md"]

    E --> E1["decision.md"]

    F --> F1["报告头 + 全部阶段正文"]
```

每个子目录中的文件名对应最终状态里的一个键。下表给出"目录 / 文件"与"状态字段"的完整映射，这是理解写入逻辑的核心：

| 阶段目录 | 文件名 | 来源状态字段 |
|---|---|---|
| `1_analysts/` | `market.md` | `market_report` |
| `1_analysts/` | `sentiment.md` | `sentiment_report` |
| `1_analysts/` | `news.md` | `news_report` |
| `1_analysts/` | `fundamentals.md` | `fundamentals_report` |
| `2_research/` | `bull.md` | `investment_debate_state.bull_history` |
| `2_research/` | `bear.md` | `investment_debate_state.bear_history` |
| `2_research/` | `manager.md` | `investment_plan` |
| `3_trading/` | `trader.md` | `trader_investment_plan` |
| `4_risk/` | `aggressive.md` | `risk_debate_state.aggressive_history` |
| `4_risk/` | `conservative.md` | `risk_debate_state.conservative_history` |
| `4_risk/` | `neutral.md` | `risk_debate_state.neutral_history` |
| `5_portfolio/` | `decision.md` | `final_trade_decision` |

这些状态字段的完整定义可参见智能体状态模型 `AgentState`，例如分析师报告字段与辩论状态字段都声明在其中。

Sources: [reporting.py](tradingagents/reporting.py#L1-L7), [reporting.py](tradingagents/reporting.py#L42-L119), [state.py](tradingagents/agents/state.py#L45-L73)

## 核心写入器：`write_report_tree`

所有保存行为都汇聚到一个函数 `write_report_tree(final_state, ticker, save_path, settings=None)`。它的返回类型是 `Path`，指向生成的 `complete_report.md`。

函数的执行遵循一个固定模式：**先建目录，再逐节判断字段是否存在，存在才写文件、才把该节加入合并报告**。开头的 `save_path.mkdir(parents=True, exist_ok=True)` 保证根目录存在，随后每一节内部再按需创建自己的子目录（例如 `analysts_dir.mkdir(exist_ok=True)`）。这种"按需创建"的写法意味着——如果某个字段在状态中缺失，对应的目录不会被创建，合并报告里也不会出现空章节。

```mermaid
flowchart TD
    Start([调用 write_report_tree]) --> Mk["mkdir(save_path, exist_ok=True)"]
    Mk --> S1{"market_report 存在?"}
    S1 -->|是| W1["写 market.md, 收集到 analyst_parts"]
    S1 -->|否| S2
    W1 --> S2{"sentiment_report 存在?"}
    S2 -->|是/否| S3{"news_report / fundamentals_report"}
    S3 --> Sec1["有内容则拼成 '## I. Analyst Team Reports'"]
    Sec1 --> S4{"investment_debate_state 存在?"}
    S4 -->|是| W4["写 bull/bear/manager 到 2_research/"]
    S4 -->|否| S5
    W4 --> S5{"trader_investment_plan 存在?"}
    S5 -->|是| W5["写 trader.md 到 3_trading/"]
    S5 -->|否| S6
    W5 --> S6{"risk_debate_state 存在?"}
    S6 -->|是| W6["写 aggressive/conservative/neutral 到 4_risk/"]
    S6 -->|否| S7
    W6 --> S7{"final_trade_decision 存在?"}
    S7 -->|是| W7["写 decision.md 到 5_portfolio/"]
    S7 -->|否| Merge
    W7 --> Merge["写 complete_report.md = 报告头 + 各节正文"]
    Merge --> Ret(["返回 complete_report.md 路径"])
```

以分析师一节为例，代码逐字段判断 `final_state.get("market_report")` 等，每写一个文件就把它以 `("Market Analyst", 文本)` 的形式追加进 `analyst_parts`；当 `analyst_parts` 非空时，才用 `"### 名称\n正文"` 的格式拼接并作为一个 `## I. Analyst Team Reports` 大节加入 `sections` 列表。研究员、风险、交易、组合管理各节都复用了这一"收集—非空则拼接"的模式。

特别值得一提的是**文件编码**：所有 `write_text` 调用都显式传入 `encoding="utf-8"`，保证中文等多字节字符不会因系统默认编码而损坏（这对 `output_language` 设为中文的运行尤为重要）。

Sources: [reporting.py](tradingagents/reporting.py#L32-L40), [reporting.py](tradingagents/reporting.py#L42-L63), [reporting.py](tradingagents/reporting.py#L65-L119)

## 报告头：说明"这次运行是什么"

`complete_report.md` 的最前面由内部函数 `_header(ticker, final_state, settings)` 生成。它不是把正文简单堆叠，而是先给出一段"元数据头"，回答"哪只标的、哪一天分析、由什么配置产生"。

头部包含以下信息，且每一步都做了存在性保护（缺失则跳过对应行）：

| 头部行 | 内容来源 | 说明 |
|---|---|---|
| `# Trading Analysis Report: {ticker}` | 传入的 `ticker` | 报告标题 |
| `- Analysis date: ...` | `final_state["trade_date"]` | 仅在存在时输出 |
| `- Generated: ...` | 当前时间 `datetime.now()` | 生成时间戳 |
| `- TradingAgents {version}: {provider}, deep ..., quick ...` | `settings` | 版本与模型配置 |
| `- Analysts: ...; research debate rounds ..., risk debate rounds ...` | `settings` | 参与的分析师与辩论轮次 |
| `- Data vendors: ...` | `settings` 的 `data_vendors` + `tool_vendors` | 数据供应商 |

`settings` 参数是可选的。当从 CLI 或 `save_reports` 调用时会传入 `run_settings()`，头部就完整；若为 `None` 或部分字典，`_header` 用 `settings.get` 带默认值 `'?'` 的方式读取，缺失的字段退化为占位符而不会报错——这保证了即使配置不完整，报告仍能写出。

Sources: [reporting.py](tradingagents/reporting.py#L13-L29), [reporting.py](tradingagents/reporting.py#L121-L125), [test_reporting.py](tests/test_reporting.py#L95-L98)

## 两条调用路径：CLI 与 Python API

报告树的保存有且仅有两条入口，二者最终都调用 `write_report_tree`。这样的设计是为了消除"CLI 能生成报告、但编程接口不能"的不一致。

```mermaid
flowchart LR
    subgraph CLI["交互式 CLI 路径"]
        A1["run_analysis()"] --> A2["_offer_reports()"]
        A2 --> A3["graph.save_reports()"]
    end
    subgraph API["Python API 路径"]
        B1["调用方直接调用"] --> B2["graph.save_reports()"]
    end
    A3 --> C["write_report_tree()"]
    B2 --> C
    C --> D["磁盘报告树 + complete_report.md"]
```

`TradingAgentsGraph.save_reports(final_state, ticker, save_path=None)` 是编程接口的核心方法。当调用方不传 `save_path` 时，它落到默认路径 `self.default_report_path(ticker)`；然后以 `settings=self.run_settings()` 调用共享写入器。因此无论哪条路径，写入逻辑与报告头都保持一致。

Sources: [trading_graph.py](tradingagents/graph/trading_graph.py#L290-L298), [reporting.py](tradingagents/reporting.py#L1-L7)

## 默认保存位置与路径安全

默认保存路径由 `default_report_path(ticker)` 决定：它在配置的 `results_dir` 下创建 `reports/` 子目录，并给每个标的拼上时间戳，形成一次运行独占的目录。

```
{results_dir}/reports/{safe_ticker_component(ticker)}_{YYYYMMDD_HHMMSS}/
```

其中 `results_dir` 的默认值是 `~/.tradingagents/logs`，可通过环境变量 `TRADINGAGENTS_RESULTS_DIR` 覆盖。时间戳用 `datetime.now().strftime("%Y%m%d_%H%M%S")` 生成，保证同一标的的多次运行互不覆盖。

**为什么要经过 `safe_ticker_component`？** 标的符号既可能来自用户 CLI 输入，也可能来自 LLM 工具调用，而后者可能被攻击者控制的新闻内容影响（提示注入）。如果没有校验，像 `"../../../etc/foo"` 这样的值一旦经 `Path /` 拼接就会逃逸出配置目录。`safe_ticker_component` 用正则 `^[A-Za-z0-9._\-\^=+]+$` 做白名单校验，只允许字母、数字、点、连字符、下划线、脱字符、等号、加号（覆盖 `^GSPC`、`GC=F`、`XAUUSD+` 等合法符号），并额外拒绝仅由点组成的值（如 `..`），否则抛出 `ValueError`。

在 Docker 部署中，`results_dir` 位于挂载卷 `/home/appuser/.tradingagents` 内（`docker-compose.yml` 将宿主数据目录挂载到该路径），因此报告随卷持久化，不会因容器销毁而丢失；这也是 CLI 特意把报告写到 `results_dir` 而非当前工作目录的原因——容器内的工作目录会随容器一起消失。

Sources: [trading_graph.py](tradingagents/graph/trading_graph.py#L300-L303), [default_config.py](tradingagents/default_config.py#L3-L3), [default_config.py](tradingagents/default_config.py#L80-L80), [symbols.py](tradingagents/dataflows/symbols.py#L147-L180), [docker-compose.yml](docker-compose.yml#L6-L7), [run.py](cli/run.py#L395-L398)

## 运行元数据：`run_settings()` 的白名单机制

报告头里的版本、模型、分析师等信息来自 `run_settings()`。这个方法返回一个**白名单字典**：只挑选对"复现这次运行"有意义的字段，绝不含端点、密钥或本地路径。

其注释明确指出：`backend_url` 可能携带凭证，因此连同密钥、本地路径一起被排除在记录之外。返回的字段包括 `version`、`llm_provider`、`deep_think_llm`、`quick_think_llm`、`analysts`、`max_debate_rounds`、`max_risk_discuss_rounds`、`output_language`、`data_vendors` 与 `tool_vendors`。测试 `test_run_settings_record_the_run_without_endpoints_or_paths` 验证了即使配置中包含 `https://user:secret@relay.example/v1` 这样的后端地址与 `/home/me/results` 这样的本地路径，产出的 settings 中也不会出现 `secret` 或 `/home/me`。

Sources: [trading_graph.py](tradingagents/graph/trading_graph.py#L270-L288), [test_reporting.py](tests/test_reporting.py#L77-L92)

## CLI 交互流程：`_offer_reports`

交互式运行时，分析结束会通过 `_offer_reports(final_state, graph, ticker, save=None, show=None)` 询问是否保存与是否展示。这里有一个关键变量 `asked`：只要 `save` 为 `None`（即用户没有通过命令行标志预先回答），就认为需要交互提问。

```mermaid
flowchart TD
    Start([分析完成]) --> Q1{"save 参数为 None?"}
    Q1 -->|是| P1["提问 'Save report?' 默认 Y"]
    Q1 -->|否| Dec
    P1 --> Dec{"要保存?"}
    Dec -->|是| Path["save_path = graph.default_report_path(ticker)"]
    Path --> Chk{"asked 为真?"}
    Chk -->|是| P2["提问 'Save path' 允许自定义"]
    Chk -->|否| Save
    P2 --> Save["graph.save_reports(...) 并打印路径"]
    Save --> Q2{"show 参数为 None?"}
    Dec -->|否| Q2
    Q2 -->|是| P3["提问 'Display full report?' 默认 Y"]
    Q2 -->|否| Sh
    P3 --> Sh{"要展示?"}
    Sh -->|是| Disp["display_complete_report(final_state)"]
    Sh -->|否| End(["结束"])
    Disp --> End
```

保存成功后，CLI 用绿色文字打印 `save_path.resolve()`（绝对路径）与 `report_file.name`（即 `complete_report.md`），方便用户定位。若保存过程抛出异常，会被捕获并打印 `Error saving report: ...`，而不会中断整个程序。

命令行标志 `--save/--no-save` 与 `--show/--no-show` 直接映射到这两个参数（见 `cli/main.py` 的选项声明）。当用户用标志预先回答时，`save`/`show` 不再为 `None`，`_offer_reports` 便跳过相应提问——这正是无提示批处理运行能够完全非交互的机制之一。相关批处理约定可参见[无提示批处理运行](6-wu-ti-shi-pi-chu-li-yun-xing-biao-zhi-yu-huan-jing-bian-liang)。

Sources: [run.py](cli/run.py#L385-L413), [run.py](cli/run.py#L372-L386), [main.py](cli/main.py#L57-L62)

## 屏幕展示：`display_complete_report`

保存到磁盘之外，CLI 还能把完整报告直接渲染到终端。`display_complete_report(final_state)` 与报告树遵循**同一套分区结构**，但用 Rich 的 `Panel` 与 `Markdown` 逐段渲染，避免一次性长文本被截断。

它依次输出五个分区：`I. Analyst Team Reports`、`II. Research Team Decision`、`III. Trading Team Plan`、`IV. Risk Management Team Decision`、`V. Portfolio Manager Decision`，与报告树中的五个编号目录一一对应。与写入器一致，每个分区同样做存在性判断（如 `if final_state.get("market_report")`），缺失则整段跳过。

需要区分的是：`display_complete_report` 面向**终端即时查看**，`write_report_tree` 面向**磁盘持久化**；二者读取的是同一个 `final_state`，但互不依赖。

Sources: [display.py](cli/display.py#L379-L437)

## 部分状态与容错

报告树写入器对"不完整运行"有良好的容错。测试通过一个只包含部分字段的状态验证了这一点——即使缺少情绪报告、熊方研究员、风险历史等，只要其余字段存在，`write_report_tree` 仍能正常生成文件并返回 `complete_report.md` 路径。

这背后的设计原则是：**每一节独立判断、独立写入**，不存在"某个字段缺失就整体失败"的联动。下表总结了容错相关的关键行为：

| 场景 | 行为 |
|---|---|
| 某分析师报告缺失 | 不创建对应 `.md`，合并报告不含该小节 |
| 整个分析师阶段都缺失 | 不创建 `1_analysts/` 目录，合并报告无 `## I.` 小节 |
| 辩论/风险状态缺失 | 跳过 `2_research/` 或 `4_risk/` 的写入 |
| `settings` 为 `None` 或部分字典 | 报告头用默认值 `'?'` 兜底，不报错 |
| `save_path` 父目录不存在 | `mkdir(parents=True, exist_ok=True)` 自动创建 |
| 标的符号含非法字符 | `safe_ticker_component` 抛 `ValueError`（默认路径场景） |

Sources: [reporting.py](tradingagents/reporting.py#L38-L63), [test_reporting.py](tests/test_reporting.py#L30-L40), [test_reporting.py](tests/test_reporting.py#L95-L98)

## 小结与延伸阅读

报告树的生成与保存可以概括为一条清晰链路：**状态字段 → 共享写入器 `write_report_tree` → 编号子目录 + `complete_report.md`**，并由 CLI 与 Python API 两条入口统一驱动。默认落在 `results_dir/reports/{ticker}_{时间戳}/` 下，标的符号经安全校验，报告头通过白名单的 `run_settings()` 记录运行元数据而不泄露敏感信息。

如果你希望进一步了解与本页相邻的主题，可以继续阅读：报告的**磁盘结构与内存决策记录**如何互补，见[记忆日志与决策记录](25-ji-yi-ri-zhi-yu-jue-ce-ji-lu)；通过编程方式触发保存的**完整 API 用法**，见[Python API 与配置调整](9-python-api-yu-pei-zhi-diao-zheng)；以及决定默认保存目录的**配置与环境变量覆盖**机制，见[配置系统与环境变量覆盖](29-pei-zhi-xi-tong-yu-huan-jing-bian-liang-fu-gai)。