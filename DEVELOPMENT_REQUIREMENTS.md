# ATK-Pulse 开发与验收要求

本文维护行为契约和验收口径。项目介绍和支持机型见 [README](README.md)，便携版操作见 [使用说明](使用说明.txt)。

## 协议与电量

- 由 [设备注册表](protocols/device_registry.json) 限定可查询的 VID/PID 和 HID 集合，按协议核对报告长度。已启用家族为 `nordic52`、`nordic54`、`evision_nearlink` 和 `holtek_nano8k`，`nano_ff05` 仍为 `pending_capture`。目录占位或协议待核实的槽位保持零主动发包。新增 VID/PID、报文字节和字段含义须有可追溯的实测依据。查询保持只读，禁止修改固件、DPI、回报率、官方配置或代理。
- 共用接收器的型号由用户确认后写入 `device_bindings.json`。处于 `ambiguous_unbound` 时保留固件原报口径，完成绑定后才启用对应机型 LUT。Holtek 8K Nano 无线在绑定前零发包。若同时存在可查询接口，优先查询该接口。已绑定的 Holtek 无线接收器与同型号 USB 并存时，优先查询唯一通过 HID 校验的有线接口，否则沿用无线查询。
- 报文合法、百分比合法、候选电压范围分别记录。电压越界仍保留合法固件电量和 `voltage_raw`。校准来自实测，候选电压在范围内只说明可显示。
- [电压估算器](voltage_soc.py) 保留 15 点分段 LUT、25 mV 压降门限连续两帧确认、15 mV 回弹容差、无线单调防抖与快速解锁。估算以 1% 为步进，`< 3722 mV` 标注低段外推。缺电压或未标定机型回退固件原报。F1 V2 Ultimate 的满刻度要求固件报 100% 且当前与滤波电压均达到 4.300 V，否则有效电压估算封顶 99%。该门限为满电记录中最低的固件满刻度读数留出 13 mV 余量。
- `charging_state` 始终为 `unknown`，兼容字段 `charging` 保持 `None`。`link_byte == 1` 只作电压滤波器的快速解锁信号，充电状态与电量估算独立。电量告警由本次有效回复触发。

## 查询与事件

- 单工作线程、单设备互斥、一个待处理的手动请求。并发手动请求合并，按编号确认完成。常规读取预算为 1000 ms，默认分片上限为 200 ms，Holtek 使用 700 ms 读取预算。默认每 300 秒采样，失败后按 15、60、300 秒退避，接收器缺席时每 30 秒枚举。手动请求可提前触发，积压周期不补发。
- 停止与恢复以 generation 和取消事件丢弃旧结果。每次按当前接口路径打开，查询后重新枚举核对。GUI 异步等待线程退出。HIDAPI 枚举、打开、发送和关闭是同步调用，分片超时约束的是读取，驱动异常时继续等待并记录。
- `WM_POWERBROADCAST` 使用 1 秒防抖，`WM_DEVICECHANGE` 使用 4 秒防抖。进程检查记录父子关系。互斥只约束采用同一实现的程序，早期工具和官方驱动可能竞争接口。
- `query_id` 标识主机侧单次事务，重试生成新值。`request_id` 标识手动请求，合并或重试时沿用。记录开始、本机接收、完成时间及 UTC 偏移。设备实际测量时间未知，`response_freshness=unverified`。路径复核、generation 和超时只能验证软件侧流程。
- `receiver_visibility` 为 `present/absent/unknown`，`outcome` 独立描述本次结果。明确的枚举错误保持 `unknown`，底层仅返回空列表时判断受限。`last_success` 只保存最后有效读数的时间、数值、身份和查询编号。启动时读取的历史样本按历史处理。
- `packet_valid` 与 `percent_valid` 独立。`voltage_validation` 使用 `candidate_in_range/candidate_out_of_range/unknown`。读取成功仍可能保存失败，此时 `connected=true`、`outcome=success` 并带 `storage_error`。界面显示保存失败，直至后续成功落盘。

## 存储与诊断

- `battery_history.csv` 的 V0.1/V0.2 历史与 `battery_samples.v1.csv` 永久只读。新有效采样只追加到 `battery_samples.v2.csv`。保留原始字段和异常样本，估算仅在运行时推演。表头不符或尾行截断时拒绝追加，保留现场。
- V2 样本含 `schema_version=2`、`device_uid`、`vid_pid`、`model_id`、`protocol_family`、`link_mode` 和 `battery_source_mode`。按每行 `app_version` 显示来源版本。设备标识为接收器序列号摘要或路径摘要，路径摘要可能随接入方式改变。旧记录的未知字段保留，缺时区仅为显示与排序暂按本地时间解释并标记。界面默认读 50,000 条，完整导出使用 `limit=None`。
- 查询诊断和软件日志独立轮转，`query_events.v1.jsonl` 与 `app.log` 各约 1 MB 并保留 3 个备份。失败事件进诊断日志。仅排查时短时启用 `ATK_PULSE_DEBUG=1` 记录详细报文。单条异常巨大的记录仍可能超过目标大小。
- 周期 CSV 与诊断 ZIP 在本地生成，剔除用户名、主机名、绝对路径和原始序列号。导出目标须是独立文件。运行保持 100% 离线，诊断资料由用户决定是否提交。备份和回退只处理程序或源码，原始 CSV 与用户配置应保留。

## 图表与界面

- `research.Thresholds` 保存描述性异常阈值。连续回升为 10 个百分点，10 分钟内下降为 10 个百分点，相近电压比较窗为 20 mV，平台期比较为 30 分钟内变化 20 mV，跨空档显著回升为 15 个百分点。异常只描述变化，跨空档重置平台期起点。采样周期用于初步观察。
- `research.DEFAULTS.chart_gap_s = 450` 保留为分析默认值。界面按视图设置空档阈值，1 小时为 7.5 分钟，24 小时为 45 分钟，7 天和本周期为 3 小时，30 天和全部记录为 12 小时。超过各视图阈值、来源或已知身份变化处断线，休眠灰带宽度至少 38 px 才显示「休眠」。
- `TrendChart` 先使用 `NoPen` 填闭合面积，再用 `NoBrush` 描开放折线。长周期、分段过多或过窄时关闭面积填充，孤点画圆。保留固件对照、Y 轴自动缩放及悬停详情。
- 主窗口固定 1080×700。在 PySide6 6.12 中使用 `setWindowFlag` 禁用最大化并显式保留最小化、关闭按钮。`closeEvent` 在 `allow_quit=False` 时隐藏窗口并保留 `TaskbarBadge`。使用 `_hidden_to_tray` 标记驻留，释放 `GreenhouseCanvas` 缓存，时钟降频到 5000 ms 并延后历史绘制，轮询、告警和贴片刷新继续运行。工作集回收只在进入驻留时容错触发。
- 总览左卡默认 3.8 px 纯数据圆环，鼠标图片由用户导入本地 PNG。约 4.7% 透明度的 90° `MOUSE` 水印贴右侧内边缘。右上角只显示计量口径，低电状态放在状态槽。电压统一显示三位小数。
- QSS 自定义背景容器使用 `QFrame` 和 `WA_StyledBackground`。多列卡片水平策略用 `Ignored`，长文本换行。
- 任务栏贴片使用 Win32 透明分层窗口，保留浅深色、DPI、`TaskbarCreated`、全屏隐藏与异常降级机制。它通过 `UpdateLayeredWindow` 绘制并独立管理 Z-order。物理像素比例为 `dpi / 96.0`，外框保持逻辑 1.35 px 且无中横线，液面内缩逻辑 1 px。低电沿用用户设置的 `low_threshold`。全画布命中底独立维持 `alpha >= 1`，休眠淡化只作用于内容。仅悬停显示浅圆角底，液位达到 75% 时滚轮改用反色，浅底用 `#FFFFFF`，深底用 `#1E1E24`。

## 构建与发布

- [打包脚本](scripts/build_release.py) 生成便携目录和 ZIP。`resource_path()` 兼容源码路径和 PyInstaller 的 `sys._MEIPASS`。PyInstaller 裁剪保留 `libcrypto-3.dll` 与 `qico/qjpeg/qwebp/qwindows/qoffscreen`。开机自启区分冻结版 `ATK-Pulse.exe --minimized` 和源码版 `pythonw.exe main.py`，保持原设置命名空间。
- 遵循仓库 [许可证](LICENSE)。已经清除旧上游原型包及其调试变量，现役源码和文档继续遵守净室边界。公开的协议研究出处和项目致谢统一维护在 [README](README.md#开源许可与致谢)。
- 构建前确认目标 EXE 已退出。已验收的常规改动以 Git 版本回退，便携版以对应正式发布包回退。

## 隔离验收

[GitHub Actions CI](.github/workflows/ci.yml) 在 `windows-latest` 运行隔离验收并校验 `hardware_access: false`。合入 `main` 的 PR 必须通过 GitHub Actions 的 `Windows isolated acceptance` 检查。

从仓库根目录执行，给每轮验收使用独立输出目录。重定向 `LOCALAPPDATA` 可同时隔离旧版历史路径。运行器会设置 `ATK_BATTERY_DATA_DIR`，并生成逐项 JSON 与日志。

```powershell
$acceptance = Join-Path (Get-Location) ("analysis_reports\acceptance-" + (Get-Date -Format yyyyMMdd-HHmmss))
$env:LOCALAPPDATA = Join-Path $acceptance 'localappdata'
.\.venv\Scripts\python.exe tests\run_acceptance.py --output $acceptance
```

- 自动验收使用模拟设备和隔离设置，结果应标记 `hardware_access: false`。覆盖协议错误、超时、取消、竞争、线程投递、Windows 互斥、存储与诊断失败、CSV 兼容和异常分析。
- 便携 EXE 用 `--smoke` 离屏启动，设置 `QT_QPA_PLATFORM=offscreen`、隔离的 `ATK_BATTERY_DATA_DIR` 和独立 `ATK_SMOKE_OUT`。有中文字体渲染问题时可设 `QT_QPA_FONTDIR=C:\Windows\Fonts`。核对退出码、截图和启动日志。
- 实机验收另测活动、静置、移动后、开关鼠标、接收器拔插、快速刷新、查询中退出、系统睡眠恢复、长期运行与官方驱动共存。报告需区分已实测项和待验收项。若需观察放电，在日常充满后记录时间，约每五分钟采样，跳变只备注当时操作，到正常低电提醒即按日常方式补电。
