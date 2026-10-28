# Chat-catch：宝武微聊采集与水晶魔方上传

获取源码请在Issues留言

面向 Windows 的宝武微聊群消息处理工具，支持两条采集路径，共用附件解析、群规则筛选和模型故障抽取：

- **截图 OCR**：进入群聊 → 滚动截图 → OCR 与页面去重 → 生成聊天原文。
- **数据库采集（试用）**：只读本机账号数据库及 WAL → 按会话和日期查询 → 输出结构化原文与 Markdown 预览。

桌面客户端 `desktop.py` 支持采集、抽取、eplat（水晶魔方）匹配上传、人工复核及 CSV 导出；命令行 `main.py` 完成采集与抽取，上传需另运行 `python -m api_upload`。

`--group` 用作群方案名称和处理阶段的 `chat_id`；OCR 还用它搜索群聊，数据库方式则用独立的会话三元组定位来源。采集日期自动传入后处理；截图及附件 OCR 共用配置中的服务类型、地址、模型版本和超时。

## 当前进展（2026-10-28）

| 模块 | 当前状态 |
| --- | --- |
| 截图 OCR | 已接入本地 OCR / MinerU，支持按滚动次数、最近 n 天或桌面端当前页采集，保留逐页原文与调试截图 |
| 数据库采集 | 已接入桌面、CLI 和每日任务；支持文字、@提及、表情、跨群转发及已验证的附件元数据；仍处于试用阶段 |
| 数据库会话选择 | 已保存账号、会话列表和选择，下次启动恢复；唯一同名群自动匹配，重名会话手动选择 |
| 故障抽取 | 已接入群筛选规则、群/人员分类映射、附件解析及公司模型 / DeepSeek 官方模型 |
| eplat 上传 | 已实现查询初始化、唯一匹配、回写 `recId` 和更新；有可用记录时继续上传，部分处理失败交人工复核 |
| 桌面与分发 | 已有结果编辑、未匹配重传、16 列 CSV、批量删除运行记录、每日定时及 Windows 文件夹打包 |

尚待完成：第二台 Windows x64 分发验收、更多消息类型及状态语义验证，以及当前回归失败的排查与测试对齐。上传结果仍需核对各阶段报告，最新离线检查范围与结果见文末。

## 准备

- 截图 OCR 采集前，在 Windows 上打开并登录宝武微聊；数据库方式需本机已有可读的账号数据。
- 抽取有内容的消息时需配置所选模型的密钥。使用本地 OCR 时启动 `ocr_url` 服务；使用 MinerU 时配置 Token。数据库纯文字采集不调用 OCR。
- 在项目目录安装依赖并激活虚拟环境：

```powershell
uv sync --extra documents
.\.venv\Scripts\Activate.ps1
```

项目要求 Python 3.12+。可以运行 `python main.py --help` 查看完整参数。

源码运行数据库功能时，改用 `uv sync --python 3.14 --extra documents --extra database`，并准备本地 ICU 扩展；构建要求见下文“Windows 文件夹分发”和 [数据库扩展说明](database_capture/native/README.md)。已打包版本自带这些组件。旧版 `.doc` / `.xls` / `.ppt` 附件还需安装 LibreOffice，并在设置中指定程序路径。

## 可视化客户端

运行 `python desktop.py`。在“设置”中保存采集方式、OCR / 模型参数和凭证，再到“采集任务”填写群方案及采集范围。也可选择“处理已有原文”复用已有采集结果。

在“提取规则”页选中群方案后点击“设为启动默认”，下次启动会自动选中该方案并填入采集群名。OCR 采集天数留空时按滚动次数翻页，客户端会跳过进群后的初始向下置底；填写采集天数时仍从最新消息开始。勾选“只提取当前页”时忽略滚动次数和日期范围，仅识别进群后的当前页面。

### 数据库采集（试用）

“设置”中将“聊天采集方式”改为“数据库”，填写数据根目录（包含 `setting.json` 和 `Profiles`；留空使用 `Documents/baowuPC_data`）。首次点击“检查数据库”列出账号，选择账号后再次检查并保存设置。成功匹配的账号、会话列表与当前选择保存在用户数据目录的 `database-match.json`，下次启动直接恢复，无需重新匹配；更换数据根目录或账号后需重新检查，也可主动检查刷新会话。选择群方案或修改任务群名时，自动选择唯一同名数据库会话；同名会话显示完整三元组，需要手动选择，选择结果会记住。群方案名称仍用于筛选规则，会话三元组用于定位实际数据，两者分别保存。启动时优先使用启动默认方案，未设置时继承上次任务群名。任务仍需填写正整数采集天数，实际采集时继续校验账号材料和数据库快照。

数据库方式支持指定会话、Asia/Shanghai 日期窗口、`content_type=0/2` 文字（含 @提及和表情）、`content_type=4` 跨群转发聊天，以及已验证的 `content_type=14/15/48` 文件附件元数据。转发正文保留逐条原发送人、原时间及嵌套聊天；转发图片等媒体显示未解析占位并进入问题报告。已验证的“该消息已被撤回”记录不录入原文或计数。附件只从所选缓存/附件目录匹配，要求名称与精确字节数一致；缺失文件进入人工映射。数据库里只有附件元数据时，不会下载或恢复文件。文字不需要 OCR 服务或 MinerU Token；扫描附件实际调用 OCR 时才检查凭证。数据库方式禁用截图标定、滚动次数和当前页选项。

读取源文件时不写库、不 checkpoint、不删除 WAL，也不退出客户端。能够拒写锁定时锁定读取，否则对全套源文件连续读取两次并核对身份及内容；稳定重复读取不等于跨库原子快照。快照元数据保留读取方式及 WAL 校验结果，在线稳定读取本身不再触发待复核。页头逻辑页数与 WAL 提交页数允许相差 8 页以内，原始页数始终保留，超出时记录复核提示；WAL 尾帧停止原因保存在元数据中，不再单独生成复核项。消息原始 `flag` 保留在 JSONL 中，非零值不再单独阻止采集或上传。

自动上传优先：只要已提取出记录，就继续查询、匹配和上传；数据库源复核项不再检查放行名单，个别消息解码失败、附件解析失败或部分解析、信息抽取失败均不阻止其他已提取记录上传。当前任务的 `needs_review` 根据这些处理失败计数及未匹配、匹配冲突生成；仅有未支持类型、富文本分段或页数差异等来源提示时，`source_metadata.json` 仍可能为 `needs_review`，任务最终状态不一定同步。请同时核对源报告、附件/抽取报告和 `upload/` 内各阶段报告，匹配后的原因与措施正文仍需人工复核。

字段校验、唯一匹配、初始化确认和取消检查仍在各阶段执行；写入报告会记录 `unknown` / `in_flight` 等状态，重传前必须核对。当前任务状态汇总未覆盖所有写入异常，界面显示成功不代表每条记录均已确认写入。若整个采集或处理因源文件变化、账号材料不匹配或快照校验错误终止，未能产出可用结果，则无法进入自动上传。

每个库及其 sidecar 合计上限为 256 MiB。宝武微聊 `3.4.0.3346` 与对应 `WeComCore.dll` 哈希是已验证基线，版本或哈希不同、无法识别客户端时仍先尝试匹配数据库；匹配失败后才补充版本差异或无法确认版本的提示，并保留具体失败原因。本机验证为 Python `3.14.7` / SQLite `3.50.4`；第二台 Windows x64 验收尚未完成。系统 ICU 与客户端 ICU 77 可能不同，每套部署首次仍需点击“检查数据库”执行合成与真实快照检查。

每次采集保留 `source.messages.jsonl`、展示用 `result.md` 和 `source_metadata.json`。运行目录继续保存完整结构化原文及元数据；“查看原文”显示 Markdown，附件复核重跑使用 JSONL，保留消息 ID、发送者和秒级时间。独立重处理示例：

```powershell
python -m processing "采集目录/source.messages.jsonl" --source-format database_jsonl --source-metadata "采集目录/source_metadata.json" --chat-id "群方案名称"
```

CLI 采集示例（账号目录和会话三元组使用检查结果中的实际值）：

```powershell
python main.py --capture-method database --database-root "C:/Users/你的用户名/Documents/baowuPC_data" --database-profile "账号目录" --conversation-key "123,2,0" --capture-days 7 --group "群方案名称"
```

每日任务会保存采集方式、根目录、账号、会话及天数；修改全局设置不会改变已保存的任务来源，需关闭再开启定时开关。数据库检查失败时当次失败，不切换 OCR，不自动重试。日期窗口为第 n 天上海时间 00:00 至任务开始时刻，起点纳入、终点排除。

在“结果复核”页的“运行记录”中，按住 Ctrl 多选或 Shift 连选，也可点击“全选”。点击“删除选中记录”并确认后，永久删除所选记录的本地运行结果、复核文件和上传报告，同时清理关联索引。删除不影响来源文件或已上传到平台的数据；任务运行期间不可删除。

### Windows 文件夹分发

构建机安装 Visual C++ x64 与 Windows SDK 10.0.26100.0，执行以下命令构建包含数据库功能的分发包：

```powershell
uv sync --python 3.14 --extra documents --extra database
uv pip install --python .\.venv\Scripts\python.exe pyinstaller==6.22.3
.\build.ps1
```

将 `dist\BaowuChatCatch-windows-x64.zip` 发给使用者。对方完整解压后双击 `BaowuChatCatch.exe`，不需要安装 Python 或依赖；具体首次配置见压缩包内的 `使用说明.txt`。程序会把个人配置和结果保存在 `%LOCALAPPDATA%\BaowuChatCatch`，因此升级只需替换解压后的程序文件夹。

构建脚本先从仓库内的官方 SQLite 源码和字符边界适配生成 ICU 扩展，校验来源哈希并执行合成测试，再将 DLL、身份清单与时区数据打包。运行机不需要编译器。扩展加载受包内路径和哈希限制，加载后关闭；缺少数据库组件时仍可使用 OCR。分发包不含客户端 DLL、账号库、探针目录或验收正文。WAL 规则参考 [SQLite 文件格式](https://www.sqlite.org/fileformat.html#walformat)。

点击“运行（提取并写入）”后，客户端依次完成提取和 eplat 上传。上传前需在设置中保存 eplat 凭证及真实响应配置。当前查询接口会返回全部数据，暂不校验查询总数；接口更新后再补充总数校验。若有未匹配记录，结果页会标记并选中对应行；人工修改保存后点击“上传/重传未匹配”，只重传先前未匹配的记录。提交状态未知或已有写入未确认时不会重传，请先核对上传报告。

采集时会显示一个小型任务进度窗，优先放在微聊窗口外侧；空间不足时放在已标定的消息区之外。任务结束后自动关闭。

每日自动采集：在“设置”页填写“每日运行时间（HH:MM）”并保存，然后在“采集任务”页选择“采集微聊”、填写群名及采集参数，开启“每日定时运行”。开关会保存当前群名、采集范围、附件目录及数据库来源参数，定时任务按这些参数执行提取和 eplat 写入；修改采集参数后请关闭再开启开关。任务页显示下次运行时间，开关、时间和采集参数在重启后保留。时间使用电脑本地时间，需保持本程序打开、电脑唤醒；OCR 另需桌面解锁且已登录微聊，数据库方式不操作聊天窗口。程序关闭期间错过的时间不补跑。每天仅触发一次，已有任务运行时跳过当天，失败时不自动重试；详情记录在任务日志中。

在“结果复核”页选择运行记录，点击“导出 CSV”即可生成 Excel 可打开的表格；有已保存的复核结果时优先导出 `fault.reviewed.json`，否则使用 `fault.json`。表格按 16 列输出，日期为 `mm.dd`、时间为 `hh:mm`，持续时间按分钟 ÷ 60 保留两位小数，跨日按完整日期计算。“落户”使用 `shutdownType` 对应的分类名称，“专业分类”使用 `specialtyProp`，前一列“原因分析”使用 `shutdownReason`，“闭环情况”将 `closedMark` 的 0/1 显示为否/是。“机组停机”对匹配成功的记录填“是”，未匹配或匹配冲突的留空，包含重传后的匹配结果；没有匹配报告时使用已回写的非空 `recId` 判断。工场、故障属性和末尾的“原因分析”列留空。CSV 使用 UTF-8 BOM 编码，并保留文本中的逗号、引号和换行。

首次运行会在 `%LOCALAPPDATA%\BaowuChatCatch` 创建配置、规则、采集和处理目录；升级程序文件夹不会覆盖这些文件。凭证写入该目录的 `secrets.json`，例如：

```json
{"llm_api_key":"填入公司模型密钥","deepseek_api_key":"填入 DeepSeek 官方密钥","mineru_token":"填入 MinerU Token","eplat_client_id":"填入客户端 ID","eplat_client_secret":"填入客户端密钥"}
```

本地 OCR 不需要 `mineru_token`；只有调用 eplat 上传时才需要两个 `eplat_` 字段。凭证文件是本机明文 JSON，请勿放入代码仓库或分享。

设置页的“抽取模型”可选 `deepseek-flash`，使用 DeepSeek 官方 API（`https://api.deepseek.com`）及独立的 `deepseek_api_key`；原有 `deepseek-v4` 和 `Qwen3.6-35B` 仍使用公司私有 API 及 `llm_api_key`。也可分别通过 `DEEPSEEK_API_KEY`、`LLM_API_KEY` 环境变量提供密钥。选择官方模型后，抽取所用聊天内容会发送至 DeepSeek 官方服务。

## OCR 首次标定

首次使用或调整窗口大小后，先运行：

```powershell
python main.py --calibrate
```

按提示将鼠标依次移到**消息区左上角**、**消息区右下角**，每次回到终端按回车。坐标相对窗口左上角，保存到 `config.json` 的 `message_area`。采集时应保持与标定时相同的窗口大小，消息区不要包含输入框和固定工具栏。

## 命令行采集与处理

以下示例使用截图 OCR；数据库命令见上文。`main.py` 只生成抽取结果，eplat 上传使用独立命令。

```powershell
python main.py --group "冷轧设备状态管理团队" --up 3
```

`--up` 和 `--down` 表示滚轮总次数，可同时使用；未设置 `capture_days` 且都不指定时默认向上滚动 5 次。进群后程序默认在消息区快速向下滚动 5 秒，再识别首屏，之后每滚动 `wheel_steps_per_capture` 次截屏并 OCR 一次。初始化滚动不计入 `--down` 次数；可用 `initial_scroll_to_latest` 关闭，或用 `initial_scroll_seconds` 调整持续时间。最后不足一组也会识别。相邻页面几乎相同时，会停止该方向的滚动。OCR 返回空文本时会停止，以免漏采。

需要按日期采集时，使用 `--capture-days 1`（只采集今天），或在 `config.json` 中设置 `"capture_days": 1`；也可设置为任意正整数 `n`（含今天及之前 `n-1` 天，没有上限）。设置后，向上翻页不再受 `--up` 次数限制；程序在 OCR 页面出现窗口前日期时结束采集。如果同一页同时出现窗口内和窗口前日期，只合并保存窗口内的消息，不保留第 n+1 天及更早的消息，然后结束；如果该页只有窗口前日期，就不合并该页。混合页中首个日期标记之前无法确定日期的内容也会排除。日期判断依据 OCR 识别到的独立日期/时间行。`capture_days` 为 `null` 或不填写时，继续按滚动次数结束。

常用选项：

| 选项 | 作用 |
| --- | --- |
| `--capture-method ocr` | 临时选择采集方式；可选 `ocr`、`database`，未指定时使用配置 |
| `--capture-days N` | 含今天的最近 N 天；数据库方式必须为正整数 |
| `--database-root 路径` / `--database-profile 账号目录` | 数据库根目录及 `Profiles` 内的账号目录名 |
| `--conversation-key "RID,conversationtype,fw_id"` | 数据库会话三元组，使用检查结果中的实际值 |
| `--model deepseek-v4` | 选择模型；默认 `deepseek-v4`，也可用 `Qwen3.6-35B` 或官方 `deepseek-flash` |
| `--max-chars N` | 覆盖模型的单块字符上限 |
| `--ocr-provider local` | 临时覆盖 JSON 中的 OCR 服务类型；可选 `local`、`mineru` |
| `--config 其他配置.json` | 读取另一份配置文件 |
| `--no-cache` | 不扫描企业微信附件缓存 |
| `--attachments-dir 路径` | 增加附件查找目录，可重复指定 |
| `--mapping 映射.json` | 手工指定附件卡片对应的本地文件 |
| `--rules 规则.json` | 按当前群加载筛选规则；默认用户目录的 `group_rules.json` |
| `--classification 映射.json` | 群分类与人员专业映射；默认用户目录的 `classification_rules.json` |
| `--secrets 凭证.json` | 凭证 JSON；默认用户目录的 `secrets.json` |
| `--cache-root 路径` | 自行选择本机账号的附件缓存根目录，不带末尾月份 |
| `--data-dir 路径` | 指定处理结果目录，默认用户目录的 `data/` |

更多后处理选项（如 `--months`、`--all-sections`、`--soffice`）可通过 `python main.py --help` 查看。

### 输出

默认均位于 `%LOCALAPPDATA%\BaowuChatCatch` 下：

- `captures/<采集ID>/result.md`：本次 OCR 合并后的聊天原文；翻页时逐页更新。后处理失败时已采集的原文仍保留。
- 数据库采集在同一采集目录保存 `source.messages.jsonl`（完整结构化源）、`source_metadata.json`（来源与问题报告）和 `result.md`（展示预览）。
- `data/runs/<run_id>/fault.json`：本次结构化故障结果；同目录保存 `source.md`、`rules_snapshot.json`、`classification_mapping.json`、`filter_decisions.jsonl` 及消息、附件、分块和汇总文件。使用 `--data-dir` 时输出到指定目录。
- `data/runs/<run_id>/fault.reviewed.json`：人工保存的复核结果；`task.json` 保存任务状态，桌面自动上传报告位于同目录的 `upload/`。
- 数据库运行目录还归档完整的 `source.messages.jsonl` 和 `source_metadata.json`；复核重跑使用它们，不使用筛选后的 `messages.jsonl`。
- `data/latest_run.txt`：最近一次处理的 `run_id`。
- `data/confirmed_mappings.json`：人工确认过且文件内容未变的附件映射，供下次处理复用。
- `database-match.json`：桌面数据库账号与会话选择缓存；`logs/` 保存桌面日志。

### 故障分类映射

用户目录的 `classification_rules.json` 控制两个输出字段；首次创建为空映射，可从旧配置手工导入：

- `shutdown_type_by_group`：群名到输出字段 `shutdownType` 的映射。例如“冷轧设备状态管理团队”固定为 `"2"`；未配置的群由模型判断。
- `specialty_prop_by_sender`：消息发送人到输出字段 `specialtyProp` 的映射。单一专业写字符串（如 `"魏明": "E"`），两个可能专业写数组（如 `"王丹": ["M", "E"]`）。数组中的具体值由模型结合消息内容选择；无法判断时为 `null`。未配置的发送人由模型判断。

编辑 JSON 后，下次抽取会读取新规则。群名和发送人需与聊天记录中的名称一致；专业代码统一使用半角大写字母。

若只需对已有的 `result.md` 重新处理，不必重新截图。请使用原采集日期，以便解析“今天 / 昨天”等时间：

```powershell
python -m processing "采集目录/result.md" --chat-id "群方案名称" --capture-date 2026-10-28
```

数据库源重处理需同时传入 `--source-format database_jsonl` 和 `--source-metadata`，示例见上文。更多参数运行 `python -m processing --help`。

## 配置与 OCR

用户目录的 `config.json` 保存窗口位置、滚动节奏和 OCR 设置；首次创建时默认使用本地 OCR。

| 字段 | 作用 |
| --- | --- |
| `capture_method` | `ocr`（默认）或 `database` |
| `database_root` / `database_profile` | 数据根目录与账号目录名；根目录留空使用 `Documents/baowuPC_data` |
| `message_area` | 消息区相对窗口的矩形坐标，由 `--calibrate` 写入 |
| `ocr_provider` | `local` 或 `mineru` |
| `mineru_model_version` | MinerU 解析模型：`vlm`（默认）或 `pipeline`；设置页可切换，截图和附件 OCR 共用 |
| `ocr_url` | 本地 OCR 接口地址，默认 `http://127.0.0.1:8005/file_parse` |
| `ocr_timeout` | 每次 OCR 请求的超时秒数，默认 120 |
| `load_delay` | 进群后等待聊天加载的秒数 |
| `initial_scroll_to_latest` | 进群后快速向下滚动到最新消息，默认 `true`；设为 `false` 关闭 |
| `initial_scroll_seconds` | 初始化快速滚动的持续秒数，默认 `5.0`；聊天记录较长时可增大 |
| `scroll_delay_min` / `scroll_delay_max` | 每组滚动后等待截图的秒数范围 |
| `wheel_delta` | 单次滚轮增量，首次创建配置默认 120；仓库的 `defaults/config.json` 示例为 240 |
| `wheel_steps_per_capture` | 每张截图前滚动的次数，默认 3 |
| `capture_days` | 采集天数，`1` 为今天，`n` 为含今天的最近 n 天；`null` 按滚动次数采集 |

例如临时改用本地 OCR：

```powershell
python main.py --group "宝武某某群" --up 3 --ocr-provider local
```

MinerU 会上传截图及需要 OCR 的附件；本地服务使用 `ocr_url`。页面合并只删除可靠的连续重叠文字；若滚动过远导致页面没有重叠，OCR 无法补回未截图的消息，可降低 `wheel_steps_per_capture` 或增加滚动后等待时间。

## 调试

```powershell
# 只截图：不调用 OCR，不更新 result.md，也不运行后处理
python main.py --group "宝武某某群" --up 3 --debug 1

# 完整流程：额外保存截图、OCR 原文和合并文本
python main.py --group "宝武某某群" --up 3 --debug 2
```

`--debug 1` 不进行 OCR 日期判断，仍按 `--up` / `--down` 指定的次数截图；未指定时向上滚动 5 次。

单独写 `--debug` 等同于 `--debug 2`。调试文件位于本次 `captures/<采集ID>/screenshots/`。诊断输入操作可运行 `python -m ocr.diagnose_input "群名" --help`。

## 代码与验证

### eplat 顺序处理

`python -m api_upload` 在一次运行中依次查询候选并初始化副表、按机组先匹配开始时间 ±60 秒（零命中再匹配结束时间 ±60 秒）、备份并回写唯一匹配的 recId，最后用新记录完整替换副表的提取字段（空值清除旧值）。开始和结束时间只要有一端有效即可参与匹配，两端都缺失则不匹配。初始化时已有 recId 仍跳过。默认读取 `fault.json`，为每条记录用 `lineCode` 和 `date` 字段对应当天 `000000–235959` 逐条查询；取回后按双方都有值的开始或结束时间 ±60 分钟保留候选，并按 `recId` 去重。成对输入 `--start` / `--end` 时改为一次手工范围查询。默认运行完整链路；`--until prepare` 或 `--until match` 可在指定阶段完成后停止。完整替换还要求 eplat 更新接口使用文档中的固定 `SET` SQL。

命令会直接调用到达阶段所需的 API，没有本地候选替代或预览模式。实际调用前需设置凭证并根据真实响应填写解析路径配置。命令示例和中间文件说明见 [eplat API 使用说明第 15 节](docs/eplat-api-usage.md#15-已实现的统一命令)。离线检查：`python -m unittest tests.test_api_upload -v`。

### 代码目录

- `desktop.py`：桌面采集、规则编辑、结果复核、CSV 导出及定时调度
- `task_runner.py`：桌面后台任务与采集、处理、上传串联
- `main.py`：两种采集方式共用的 CLI 入口，采集后完成故障抽取
- `app_paths.py`：用户目录、首次初始化及凭证读写
- `ocr/capture.py`、`ocr/client.py`、`ocr/merge.py`：窗口采集、OCR 客户端和页面去重
- `ocr/config.py`、`ocr/human.py`：配置读写和输入动作
- `database_capture/`：数据库只读快照、WAL 校验、消息解码及结构化源输出
- `processing/`：聊天与附件解析、分块、模型抽取和结果保存
- `api_upload/`：eplat 查询初始化、停机匹配、回写及更新报告
- `tools/build_database_icu.py`、`build.ps1`：本地数据库扩展构建及 Windows 分发打包
- `tests/`：离线回归测试
