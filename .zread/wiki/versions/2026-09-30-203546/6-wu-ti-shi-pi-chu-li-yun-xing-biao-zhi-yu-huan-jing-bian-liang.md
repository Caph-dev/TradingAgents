交互式 CLI 会依次提出一组问题（标的、日期、语言、分析师、研究深度、供应商、模型等），适合人坐在终端前逐步确认。但当你要把一次分析放进定时任务、CI 流水线或容器编排里时，没有任何人能回答这些问题。本页说明 TradingAgents 的两条"无人值守"通道：**命令行标志**负责回答每次运行都会变化的问题，**`TRADINGAGENTS_*` 环境变量**负责回答相对稳定、可以随部署固定的问题。二者配合，一个不弹任何提示的完整分析运行就成型了。

Sources: [cli/main.py](cli/main.py#L33-L96), [cli/selections.py](cli/selections.py#L42-L69)

## 设计原则：每个标志只跳过它自己那一步

整套机制建立在一个简单约定上——**"每个标志只跳过它自己的问题"**。交互流程把设置拆成若干步骤（Step 1 到 Step 8），每个步骤独立地判断"这个问题是否已经有答案了"：若对应标志存在就使用标志值，否则若对应环境变量已设置就使用 `DEFAULT_CONFIG`（已折叠环境覆盖）中的值，两者都没有才会真正弹出提示。因此你不必一次把全部问题都关掉，可以只跳过那些在批处理场景中明确已知的步骤。

Sources: [cli/selections.py](cli/selections.py#L80-L128), [cli/run.py](cli/run.py#L97-L103)

下面的流程图展示了单次运行在进入模型调用之前如何决定"是否需要提问"：

```mermaid
flowchart TD
    A[启动 tradingagents] --> B{stdin 是终端?}
    B -- 否 --> C[unattended_gaps 收集所有未答问题]
    C --> D{还有缺口?}
    D -- 是 --> E[打印需设置的标志/变量并退出(1)]
    D -- 否 --> G[进入选择流程]
    B -- 是 --> G
    G --> H{该步骤有标志?}
    H -- 是 --> I[用标志值, 跳过提示]
    H -- 否 --> J{该步骤有环境变量?}
    J -- 是 --> K[用 DEFAULT_CONFIG 中的值, 跳过提示]
    J -- 否 --> L[弹出交互提示]
    I --> M[组装 run config]
    K --> M
    L --> M
    M --> N[构建图并运行]
```

Sources: [cli/run.py](cli/run.py#L101-L111), [cli/selections.py](cli/selections.py#L129-L207)

## 命令行标志：回答每次运行都会变化的问题

`--ticker`、`--date`、`--analysts` 这些值每次都不同，最自然的归宿就是命令行标志。它们挂在与交互式运行相同的入口（`analyze` 回调）上，因此裸 `tradingagents` 与带标志的 `tradingagents --ticker ...` 走的是同一条执行路径——只是前者的每个问题都需要人来答。

Sources: [cli/main.py](cli/main.py#L33-L61), [cli/main.py](cli/main.py#L84-L85)

| 标志 | 类型 | 回答的问题 | 说明 |
| --- | --- | --- | --- |
| `--ticker` | str | Step 1 标的 | 如 `NVDA`、`0700.HK`；会做规范化与合法性校验 |
| `--date` | str | Step 2 分析日期 | `YYYY-MM-DD`，不允许未来日期 |
| `--analysts` | str | Step 4 分析师团队 | 逗号分隔，如 `market,news`；按资产类型校验 |
| `--save/--no-save` | bool | 运行后的"保存报告?" | 省略则保留提问 |
| `--show/--no-show` | bool | 运行后的"显示完整报告?" | 省略则保留提问 |
| `--checkpoint/--no-checkpoint` | bool | 断点续跑开关 | 省略则遵循 `TRADINGAGENTS_CHECKPOINT_ENABLED` |
| `--clear-checkpoints` | bool | — | 运行前清空所有已保存检查点 |
| `--portfolio` | str | — | 持仓与现金的 JSON 文件路径 |

标志值并非直接使用，而是先经过与交互提示**完全相同的校验函数**（`_from_flag` → `parse_ticker` / `parse_analysis_date` / `parse_analysts`）。这保证了批处理与交互运行对非法输入的反应一致：例如空标的、未来日期、未知分析师名称都会在起步阶段报错退出，而不是带着坏配置进入模型调用。

Sources: [cli/main.py](cli/main.py#L52-L61), [cli/selections.py](cli/selections.py#L71-L78), [cli/prompts.py](cli/prompts.py#L66-L116)

值得注意的是 `--save/--show` 被声明为 `bool | None`（三态）。`None` 表示"没说"，运行结束后照常提问；显式的 `--save`/`--no-save` 才把该问题关掉。这样批处理既能照常拿到报告，又不会在无人应答时卡住。

Sources: [cli/main.py](cli/main.py#L57-L65), [cli/run.py](cli/run.py#L389-L414)

## 环境变量：回答可随部署固定的问题

语言、研究深度、LLM 供应商、模型、以及各家供应商的 thinking/reasoning 旋钮，这些通常在一次部署内保持不变，适合放进 `.env` 或容器的环境。它们统一由 `tradingagents/default_config.py` 中的 `_ENV_OVERRIDES` 表驱动——这是**环境变量到配置键映射的唯一真相来源**：要新增一个可用环境变量覆盖的键，只需在此表加一行，无需改动任何入口脚本。

Sources: [tradingagents/default_config.py](tradingagents/default_config.py#L9-L31), [tradingagents/default_config.py](tradingagents/default_config.py#L73-L77)

| 环境变量 | 覆盖的配置键 | 跳过/影响的交互步骤 |
| --- | --- | --- |
| `TRADINGAGENTS_LLM_PROVIDER` | `llm_provider` | Step 6 供应商 |
| `TRADINGAGENTS_DEEP_THINK_LLM` | `deep_think_llm` | Step 7 深度思考模型 |
| `TRADINGAGENTS_QUICK_THINK_LLM` | `quick_think_llm` | Step 7 快速思考模型 |
| `TRADINGAGENTS_LLM_BACKEND_URL` | `backend_url` | 供应商端点（可与 Step 6 组合） |
| `TRADINGAGENTS_OUTPUT_LANGUAGE` | `output_language` | Step 3 输出语言 |
| `TRADINGAGENTS_MAX_DEBATE_ROUNDS` | `max_debate_rounds` | Step 5 研究深度（需两个都设） |
| `TRADINGAGENTS_MAX_RISK_ROUNDS` | `max_risk_discuss_rounds` | Step 5 研究深度（需两个都设） |
| `TRADINGAGENTS_MAX_TOOL_ROUNDS` | `max_tool_rounds` | 分析师工具调用轮数上限 |
| `TRADINGAGENTS_CHECKPOINT_ENABLED` | `checkpoint_enabled` | 断点续跑开关 |
| `TRADINGAGENTS_BENCHMARK_TICKER` | `benchmark_ticker` | Alpha 基准 |
| `TRADINGAGENTS_TEMPERATURE` | `temperature` | 采样温度 |
| `TRADINGAGENTS_LLM_MAX_RETRIES` | `llm_max_retries` | SDK 重试预算 |
| `TRADINGAGENTS_MAX_TOKENS` | `max_tokens` | 最大输出 token |
| `TRADINGAGENTS_GOOGLE_THINKING_LEVEL` | `google_thinking_level` | Step 8（Google） |
| `TRADINGAGENTS_OPENAI_REASONING_EFFORT` | `openai_reasoning_effort` | Step 8（OpenAI） |
| `TRADINGAGENTS_ANTHROPIC_EFFORT` | `anthropic_effort` | Step 8（Anthropic） |

有一处细节值得留意：**研究深度需要"两个环境变量同时设置"才会被跳过**，这是由 `depth_from_env()` 判定的——因为它同时映射到辩论轮数与风控轮数两个键，只设其一会让该问题失去明确定义。当两个都设时，Step 5 的提问被跳过，并直接采用 `DEFAULT_CONFIG` 中的值。

Sources: [cli/selections.py](cli/selections.py#L49-L54), [cli/selections.py](cli/selections.py#L195-L207), [tradingagents/default_config.py](tradingagents/default_config.py#L10-L31)

## 无终端时的快速失败：先列出所有缺口

这是批处理体验中最关键的一块设计。许多工具在无终端时会卡在第一个提问上、或直�题掉进异常堆栈。TradingAgents 改为**起步即审计**：一旦检测到标准输入不是终端（`sys.stdin` 缺失或其 `isatty()` 为 `False`），`run_analysis` 就会调用 `unattended_gaps(flags)` 收集*所有*仍未回答的问题，一次性打印出来并 `exit(1)`——发生在任何模型被调用之前。

Sources: [cli/run.py](cli/run.py#L97-L103), [cli/selections.py](cli/selections.py#L55-L69)

这个"缺口清单"同时覆盖标志与环境变量两处，让你一眼看到该补什么：

```
No terminal to answer the setup questions. Set:
  --date
  --analysts
  --save or --no-save
  --show or --no-show
  TRADINGAGENTS_OUTPUT_LANGUAGE
  TRADINGAGENTS_MAX_DEBATE_ROUNDS and TRADINGAGENTS_MAX_RISK_ROUNDS
  TRADINGAGENTS_LLM_PROVIDER
  TRADINGAGENTS_QUICK_THINK_LLM or TRADINGAGENTS_DEEP_THINK_LLM
```

注意清单只列出真正未答的项——若 `--ticker` 已给出，它就不会出现在列表里；若某个标志被赋了空字符串，则视为"已回答"，交由该步骤自己的校验报错，而非报"缺失"。

Sources: [cli/selections.py](cli/selections.py#L55-L69), [tests/test_cli_headless.py](tests/test_cli_headless.py#L62-L77), [tests/test_cli_headless.py](tests/test_cli_headless.py#L140-L148)

一个可用作端到端验证的"无人值守环境变量集合"如下，只要这些变量加标志齐备，`unattended_gaps` 就会返回空列表，运行不再有任何提问：

| 类别 | 需要设置 |
| --- | --- |
| 语言 | `TRADINGAGENTS_OUTPUT_LANGUAGE` |
| 深度 | `TRADINGAGENTS_MAX_DEBATE_ROUNDS` + `TRADINGAGENTS_MAX_RISK_ROUNDS` |
| 供应商 | `TRADINGAGENTS_LLM_PROVIDER` |
| 模型 | `TRADINGAGENTS_QUICK_THINK_LLM` 或 `TRADINGAGENTS_DEEP_THINK_LLM` |
| 标志 | `--ticker --date --analysts --save/--no-save --show/--no-show` |

Sources: [tests/test_cli_headless.py](tests/test_cli_headless.py#L9-L13), [tests/test_cli_headless.py](tests/test_cli_headless.py#L16-L17)

## 类型强制：把字符串环境变量还原成正确的类型

环境变量本质是字符串，而配置里的值是布尔、整数、浮点或字符串。`_coerce` 依据**现有默认值的类型**做转换：布尔接受 `true/1/yes/on` 与 `false/0/no/off`（大小写不敏感），整数与浮点分别用 `int()` / `float()` 解析，其余按字符串处理。这让你可以在 `.env` 里直接写 `TRADINGAGENTS_CHECKPOINT_ENABLED=false` 而无需关心内部类型。

Sources: [tradingagents/default_config.py](tradingagents/default_config.py#L33-L57)

强制转换的取舍是**"响亮失败"**：拼错的布尔值（如 `treu`）或非数字的整数会在启动阶段抛出 `ValueError`，而不是悄悄回退到默认值——对无人值守运行而言，静默误配比直接报错危险得多。空字符串则是例外：它被当作"未设置"直接跳过，从而保留内置默认值。

Sources: [tradingagents/default_config.py](tradingagents/default_config.py#L44-L68), [tests/test_env_overrides.py](tests/test_env_overrides.py#L33-L48), [tests/test_env_overrides.py](tests/test_env_overrides.py#L88-L96)

**显式优先**是贯穿整条链路的优先级规则。当环境变量设置了某个值、而用户又在 CLI 上做了选择时，环境侧的值被保留，交互选择不会覆盖它（例如研究深度被选，但 `TRADINGAGENTS_MAX_DEBATE_ROUNDS` 已设，则辩论轮数仍取环境值）。`_build_run_config` 在组装运行配置时明确实现了这一规则。

Sources: [cli/run.py](cli/run.py#L58-L96), [tests/test_cli_headless.py](tests/test_cli_headless.py#L151-L162)

| 优先级 | 来源 | 何时生效 |
| --- | --- | --- |
| 最高 | 命令行标志 | 显式给出 `--checkpoint` / `--no-checkpoint` 等 |
| 高 | `TRADINGAGENTS_*` 环境变量 | 环境已设且标志未显式覆盖 |
| 中 | 交互选择 | 无标志、无环境变量时由用户选择 |
| 低 | 内置默认值 | 以上皆无 |

Sources: [cli/run.py](cli/run.py#L58-L96), [tradingagents/default_config.py](tradingagents/default_config.py#L59-L71)

## 运行后的无提示行为：保存、展示与评级

无提示运行的最后一段是运行结束后的收尾询问。`_offer_reports` 以 `save`/`show` 是否为 `None` 来判断"是否该问"：如果给了 `--save`/`--no-save`，保存问题被跳过；如果给了 `--show`/`--no-show`，展示问题被跳过。保存目标默认落在 `results_dir` 之下（而非进程工作目录），因此在 Docker 里报告会写进已挂载的卷，随数据一起持久化。

Sources: [cli/run.py](cli/run.py#L371-L414)

此外，如果最终决策里读不出可评级的结论，运行会被标记为"待复核"而非记作一个持仓，并给出明确提示——这样批处理结果不会伪装成一次正常产出，避免把坏数据当成仓位信号误用。

Sources: [cli/run.py](cli/run.py#L357-L368)

## API 密钥与公告：无终端时的替代路径

无提示环境里还有两处本来需要交互的环节，各有对应处理。其一是 **API 密钥**：`ensure_api_key` 会校验所选供应商对应的环境变量（映射见 `PROVIDER_API_KEY_ENV`）是否已设置；如果缺失且没有终端可粘贴，它不会尝试提问，而是打印"`<VAR> 未设置且无终端可询问`"并退出。因此批处理运行必须提前把密钥注入环境。

Sources: [cli/prompts.py](cli/prompts.py#L627-L655), [tradingagents/llm_clients/api_key_env.py](tradingagents/llm_clients/api_key_env.py#L14-L45)

其二是**启动公告**：需要用户确认的公告只在存在终端时才通过 `getpass` 等待回车；无终端时直接打印面板并继续，不会阻塞流水线。

Sources: [cli/announcements.py](cli/announcements.py#L32-L54), [tests/test_cli_headless.py](tests/test_cli_headless.py#L121-L130)

## 端到端示例：一次完整的无提示运行

把前三部分合起来，一个可用于定时任务或容器入口的完整调用如下。标志回答每次变化的项，环境变量回答部署级固定的项：

```bash
export TRADINGAGENTS_LLM_PROVIDER=openai \
       TRADINGAGENTS_QUICK_THINK_LLM=gpt-6-luna \
       TRADINGAGENTS_DEEP_THINK_LLM=gpt-6-sol
export TRADINGAGENTS_OUTPUT_LANGUAGE=English \
       TRADINGAGENTS_MAX_DEBATE_ROUNDS=1 \
       TRADINGAGENTS_MAX_RISK_ROUNDS=1

tradingagents --ticker NVDA --date 2026-09-23 \
              --analysts market,news,fundamentals \
              --save --no-show
```

上述命令不产生任何提问，运行结束后报告自动写入 `results_dir`、不在屏幕上打印全文。若在无终端环境下漏掉了某个标志或变量，运行会在起步阶段列出所有缺口并退出，而不是静默挂起。

Sources: [README.md](README.md#L209-L215), [.env.example](.env.example#L39-L56)

交互式运行与批处理运行的关键差异可对照如下：

| 维度 | 交互式 CLI | 无提示批处理 |
| --- | --- | --- |
| 入口 | 裸 `tradingagents` | `tradingagents --ticker ... --date ...` |
| 每次变化的设置 | 逐项提问 | 命令行标志 |
| 部署级固定的设置 | 提问（默认值可回车） | `TRADINGAGENTS_*` 环境变量 |
| 无终端时的行为 | 不适用 | 起步即列出全部缺口并退出 |
| 运行后保存/展示 | 提问 | 由 `--save`/`--no-save`、`--show`/`--no-show` 决定 |
| API 密钥缺失 | 提示粘贴并写入 `.env` | 报错退出，要求预置环境变量 |

Sources: [cli/main.py](cli/main.py#L33-L96), [cli/selections.py](cli/selections.py#L42-L69), [cli/prompts.py](cli/prompts.py#L627-L655)

## 故障排查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 提示 `No terminal to answer the setup questions` | 无终端且有未答步骤 | 按清单补 `--ticker/--date/--analysts/--save/--no-save/--show/--no-show` 或相应环境变量 |
| 深度问题仍被询问 | 只设了辩论轮数或风控轮数之一 | 两个 `MAX_*_ROUNDS` 都要设置 |
| 启动即 `ValueError: Invalid value for TRADINGAGENTS_...` | 环境变量类型无法强制转换 | 修正值：布尔用 true/false 等，整数写数字 |
| 无终端报某 `*_API_KEY` 未设置 | 密钥未预置 | 在环境中导出对应密钥变量 |
| `--no-save` 后找不到报告 | 保存被显式关闭 | 改用 `--save`，报告写入 `results_dir` 下的标的/日期目录 |
| 供应商已选但仍问端点 | OpenAI 兼容端点无默认值 | 设 `TRADINGAGENTS_LLM_BACKEND_URL` |
| 想在批处理里改模型端点 | 端点被菜单默认覆盖 | 用 `TRADINGAGENTS_LLM_BACKEND_URL`（显式环境值优先） |

Sources: [cli/selections.py](cli/selections.py#L55-L69), [tradingagents/default_config.py](tradingagents/default_config.py#L44-L57), [cli/prompts.py](cli/prompts.py#L394-L408)

## 下一步

如果你已经能让分析无提示地跑起来，接下来可以了解如何**把一次运行的历史结果拿来评估决策质量**，参见 [回测命令行与结果解读](8-hui-ce-ming-ling-xing-yu-jie-guo-jie-du)。想从代码层面以编程方式驱动同样的分析流程（而不是走 CLI），参见 [Python API 与配置调整](9-python-api-yu-pei-zhi-diao-zheng)。若需要在容器中持久化这些运行产物，参见 [Docker 部署与数据持久化](4-docker-bu-shu-yu-shu-ju-chi-jiu-hua)。