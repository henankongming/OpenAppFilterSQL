# AI 持续协作记录：统一使用 SQLite 存储

> 本文是后续 AI 编程会话的项目上下文与工作日志。每次继续开发前，先阅读本文和相关源码；每完成一项改造，更新“进度记录”“当前状态”和“下一步计划”。不要把计划写成已完成的功能。

## 1. 分支目的

- 仓库：`henankongming/OpenAppFilterSQL`
- 工作分支：`feature/unified-sqlite-storage`
- 基线分支：`master`
- 目标：重新设计插件的数据持久化架构，将所有需要持久化的业务数据统一保存到 SQLite，减少大量按设备、按日期生成的 JSON 文件和目录，改善历史数据清理、查询和维护。
- 本分支是面向重新设计的开发分支，不以兼容旧版本数据为目标。**不需要导入旧 JSON、不需要支持旧 JSON 读取，也不需要为旧数据迁移保留兼容层。** 新版从空数据库初始化，采用新的 schema 和新的数据库读写路径。

## 2. 用户观察与动机

用户在路由器上运行插件时观察到：
- 目录按 MAC 设备拆分，层级较深。
- 可见文件包括 `hourly_2026-09-16.json`、`client.json` 等。
- 运行约一个月后，相关目录总大小约 300 MB。
- 希望长期持续改造插件，并在不同 AI 对话之间保留明确的设计目的、现状和待办。

300 MB 是用户现场观察值，不是基准测试结果。该现象是重新设计持久化架构的动机之一。

## 3. 当前代码现状（基于初步源码检查）

主要文件：
- `open-app-filter/src/fwx_user.c`
- `open-app-filter/src/fwx_user.h`
- `open-app-filter/src/main.c`
- `open-app-filter/files/fwx.config`

### 已有 SQLite

`fwx_user.c` 已经实现 SQLite 访问记录数据库：
- 表：`app_visit_record`
- 字段：`mac`、`record_date`、`appid`、`start_time`、`end_time`、`duration`、`action`
- 已创建按日期和按 MAC + 日期查询的索引。
- 相关函数包括 `open_client_visit_db()`、`save_visit_record_to_db()`、`init_client_visit_db()` 以及数据库清理相关函数。

注意：`save_client_visit_data_to_file()` 这个函数名具有误导性；其主体已经把访问记录写入 SQLite，而不是简单地写一个 JSON 文件。不要仅凭函数名判断存储格式。

### 仍写入 JSON 的持久化业务数据

初步源码检查确认至少包括：
- 客户端每小时统计：`<client-data>/<MAC>/stats/hourly_YYYY-MM-DD.json`，由 `save_daily_stats_to_file()` 写入。
- 客户端每日热门应用统计：`<client-data>/<MAC>/stats/top_apps_YYYY-MM-DD.json`，由 `save_daily_top_apps_stats_to_file()` 写入。
- 客户端信息/状态备份：每个客户端备份目录下的 `client.json`，由 `save_client_backup_to_file()` 相关逻辑写入。
- 全局应用类型统计、全局流量统计等也存在文件持久化接口，例如 `save_global_traffic_stats_to_file()`；需要继续逐一追踪实际读写路径。
- 头文件中有历史统计从文件加载的接口，如 `load_history_stat_from_file()` 和 `load_history_top_apps_stat_from_file()`。迁移时必须同步调整读写路径，而不是只改保存函数。

### 当前内存数据与临时文件

- 在线客户端对象、实时统计和会话状态有一部分保存在内存结构中；需要判定哪些数据需要在进程重启后恢复，哪些只是实时缓存。
- `/tmp/visit_list`、`/tmp/dev_list` 等接口文件，以及应用识别数据、配置、日志、升级临时文件等，必须逐项分类。**“所有数据进 SQL”应优先理解为所有需要持久化的业务数据进入 SQLite，不等于把配置文件、日志、IPC 文件、临时文件一股脑塞进业务数据库。**
- 默认 UCI 配置中，`history_data_path` 和 `base_data_path` 指向 `/tmp/oaf`；运行设备的实际配置可能不同，不能假定所有用户都使用默认路径。

## 4. 总体设计目标

1. 为所有需要持久化的业务数据建立统一 SQLite 存储层。
2. 统一管理设备资料/备份、访问记录、小时统计、每日热门应用、全局应用类型统计、全局流量统计及其他经源码审计确认的持久化业务数据。
3. 为常用查询建立合理索引，避免按 MAC × 日期创建海量小文件。
4. 将历史保留期限和数据库容量控制设计为明确、可配置、可验证的策略；删除旧数据后评估是否需要 `VACUUM`，避免频繁全库整理影响路由器性能。
5. 提供清晰的数据库 schema 版本与升级机制，支持新版数据库结构的后续迭代；不实现旧 JSON 导入或旧 JSON 读取兼容。
6. 尽量保留现有 API/前端响应结构，降低 LuCI 或其他调用方受到的影响。
7. 在低内存、低闪存、闪存写入寿命有限的 OpenWrt 设备上控制事务、索引、缓存和写入频率。
8. 对迁移失败、数据库损坏、磁盘空间不足、进程中断等情况提供日志和恢复路径。
9. 不实现旧 JSON 数据导入、旧 JSON 读取兼容或旧数据迁移流程。新版按新数据库结构运行；如需重新安装或升级部署，应在发布说明中明确旧历史数据不会自动导入。

## 5. 建议的实施顺序

### 阶段 A：盘点和设计（尚未完成）

- [ ] 全仓搜索 JSON 文件创建、读取、更新、删除和目录遍历逻辑（已对主要 C 模块、内核侧源码和规则脚本进行第二轮筛查；仍需最终核对剩余源码、Lua 控制器与所有调用方）。
- [x] 核心文件首轮确认 SQLite 打开、建表、读写、清理逻辑及数据库路径；全仓连接生命周期盘点仍待完成。
- [x] 对核心文件中已发现的数据类型标记业务含义、写入时机、读取入口、保留策略及重启恢复行为（见下文首轮盘点）；全仓复核仍待完成。
- [x] 确认核心文件中 `hourly_*.json`、`top_apps_*.json`、`client.json`、`online_offline_*.json`、全局流量文件及访问记录 DB 的主要调用链；其他源码调用方仍待全仓复核。
- [ ] 设计表结构、主键/唯一约束、索引、schema 版本和迁移状态表。
- [ ] 明确 UCI 中历史路径和大小配置在新数据库方案中的兼容方式。

### 阶段 B：重新设计统一存储层（尚未开始）

- [ ] 抽取/整理 SQLite 初始化、事务、预编译语句、错误处理和 schema 迁移。
- [ ] 先迁移客户端资料/备份，定义重启恢复行为。
- [ ] 迁移小时统计和每日热门应用统计，并改造所有历史读取入口。
- [ ] 迁移全局应用类型、全局流量及其他盘点出的 JSON 持久化数据。
- [ ] 决定运行态数据哪些只保留内存，哪些需要落库；为需要落库的数据设置合理写入节流和批处理。

### 阶段 C：新版数据库初始化与运行验证（尚未开始）

- [ ] 实现新版空数据库初始化、schema 版本记录和必要的索引/约束。
- [ ] 验证首次启动、重复启动、进程重启及数据库不存在时的行为。
- [ ] 验证数据库初始化失败、文件系统只读、空间不足和数据库损坏时的错误处理。
- [ ] 明确新版发布/部署说明：不自动导入旧 JSON 历史数据，不支持旧 JSON 历史读取。

### 阶段 D：清理、容量与验证（尚未开始）

- [ ] 将清理策略统一到数据库记录和索引上，不再依赖文件名解析来清理 JSON 历史数据。
- [ ] 验证保留天数边界、跨日/跨月、时区、无数据日期和设备删除场景。
- [ ] 测试 SQLite 文件大小、查询性能、写入频率、闪存写入量和内存占用。
- [ ] 测试进程重启、路由器重启、数据库被占用、空间不足、数据库异常等情况。
- [ ] 新设计中直接移除已被 SQLite 取代的 JSON 业务数据读写逻辑，不增加旧 JSON 兼容层；保留确有必要的 IPC、配置、日志和临时文件。

## 6. 设计约束与注意事项

- SQLite 不是自动更省空间；应以真实数据和设备测试衡量收益。
- 不要把整个数据库容量限制简单等同于删除任意比例记录。需要明确保留策略、事务边界及清理后的空间回收行为。
- 不需要实现旧 JSON 导入、旧 JSON 读取兼容、旧数据迁移或为旧 JSON 保留双写逻辑。
- 对时间字段统一单位和语义，建议明确 UTC 时间戳或设备本地日界线的定义，避免日期归档/清理错位。
- 对统计类数据应明确“累计值”还是“按小时/按天聚合值”，避免重复归档导致双计数。
- 为数据表设置合适的唯一约束，确保重试和重复归档幂等。
- 不要在每秒轮询路径中无节制地打开数据库、执行全表查询或 `VACUUM`。
- 数据库事务应覆盖同一批次的相关写入；所有语句都应检查错误并正确释放 statement/connection。
- 保持 GPL 许可和项目现有代码风格。
- 不宣称已测试、已迁移或已优化，除非有实际构建、测试或运行证据。

## 7. 当前已完成改造

截至本记录首次创建：
- [x] 创建工作分支 `feature/unified-sqlite-storage`，从 `master` 分出。
- [x] 添加本文档，记录项目目标、现有存储方式、迁移范围和后续工作步骤。
- [ ] 尚未改动业务代码。
- [ ] 尚未新增数据库表或 schema 版本。
- [ ] 尚未完成新数据库架构和业务代码重构。
- [x] 用户明确：这是重新设计，不需要迁移旧数据，也不需要支持旧 JSON 读取。
- [ ] 尚未进行构建、单元测试、设备测试或性能测试。

后续每个完成项都应附上提交 SHA、涉及文件和验证结果；未验证的工作应继续标记为未完成。

## 8. 每次 AI 会话的工作协议

开始新会话时：
1. 阅读本文，确认当前分支和最新提交。
2. 查看 `git status` / 分支 diff（如果能访问工作区），并阅读本次要修改的源码上下文。
3. 先复述本次准备解决的具体任务和影响范围；遇到数据语义不明确时先调查，不凭函数名猜测。
4. 一次聚焦一个可验证的重构步骤，避免大范围一次性替换；不为旧 JSON 数据设计兼容层。
5. 修改后执行可用的静态检查、构建或测试，并准确记录哪些测试实际运行、哪些未运行。
6. 更新本文中的进度、当前状态、决策和下一步，并与代码改动一起提交。

结束会话前更新以下模板：

### 进度记录模板

- 日期：
- 目标：
- 改动文件：
- 具体改动：
- 新 schema/行为变化：
- 测试命令及结果：
- 未解决问题/风险：
- 提交 SHA：
- 下一步：

## 9. 下一步建议（下一次开发优先做）

**先完成阶段 A 的持久化数据盘点与新 schema 设计，不要立即重写所有存储代码。** 建议从以下函数及其调用方开始，逐项追踪文件路径和读写逻辑：
- `save_daily_stats_to_file()` / `load_history_stat_from_file()`
- `save_daily_top_apps_stats_to_file()` / `load_history_top_apps_stat_from_file()`
- `save_client_backup_to_file()` / `load_client_backup_from_file()`
- `save_global_traffic_stats_to_file()` 及全局统计对应读取函数
- `cleanup_old_record_files()`、`cleanup_expired_hourly_stats()`
- `build_client_visit_db_path()`、`open_client_visit_db()`、`check_and_cleanup_history_data_by_size()`

第一项代码任务应输出一份“持久化数据清单 + 运行时读写调用链 + 目标 schema 草案”，经核对后开始按新架构重构。旧 JSON 只作为理解旧代码行为的参考，不作为新版输入源。


## 10. 首轮持久化盘点与 Schema 草案（2026-10-15）

> 范围说明：本轮直接检查了工作分支中的 `open-app-filter/src/fwx_user.c`、`fwx_user.h`、`main.c` 和 `files/fwx.config`。这是核心存储路径的首轮源码审计，不代表已完成全仓所有文件的最终核对。下面的 schema 是设计草案，不是已经实现的数据库结构。

### 10.1 已确认的数据类型和调用链

| 数据类型 | 当前写入/来源 | 当前读取/恢复 | 当前清理或生命周期 | 新版处理建议 |
| --- | --- | --- | --- | --- |
| 应用访问记录 | `check_and_archive_all_clients()` 将在线访问移入历史列表后调用 `save_visit_record_to_db()`；归档时 `save_client_visit_data_to_file()` 在事务内按 MAC + 日期先删后插入 `app_visit_record` | SQLite 查询函数使用同一 `client.db`；具体 API 调用方仍需全仓核对 | `delete_expired_visit_records()`、按日期/MAC 范围删除、按最旧记录批量删除；超限时可能执行 `VACUUM` | 保留记录语义；将初始化、事务和清理纳入统一存储层 |
| 客户端资料/状态备份 | `save_client_backup_to_file()`；`save_all_client_backup_to_files()` 被定时同步和日归档调用；路径为 `client_backup/<MAC>/base_info/client.json` | `main()` 在 `init_client_visit_db()` 后调用 `load_client_backup_from_files()`；加载 MAC、IP/IPv6、hostname、nickname、过期/在线时间、标记字段和嵌套统计 | 当前没有独立数据库清理；启动时扫描备份目录 | 用 `client_profile` 保存持久化资料；区分可恢复的累计字段与必须重置的实时字段 |
| 客户端小时统计 | `get_today_stat()` 在日期切换时保存；`check_and_archive_all_clients()` 对昨日统计调用 `save_daily_stats_to_file()`；写入 `hourly_YYYY-MM-DD.json` | `load_history_stat_from_file()` 当前直接返回 NULL；在该函数定义处未见实际文件读取实现 | `cleanup_expired_files_by_days()`、`cleanup_old_record_files()` 和按范围删除会按文件名日期删文件 | 拆成每天/每小时一行，保存流量、在线/活跃时长，并用子表保存每小时热门应用 |
| 客户端每日热门应用 | `check_and_archive_all_clients()` 调用 `save_daily_top_apps_stats_to_file()`；写入 `top_apps_YYYY-MM-DD.json` | `load_history_top_apps_stat_from_file()` 当前直接返回 NULL | 与客户端统计共用文件清理路径 | 用 `client_daily_top_app` 记录日期、应用、时长和排名 |
| 客户端在线/离线事件 | `archive_and_save_online_offline_records()` 调用 `save_online_offline_records_to_file()`；写入 `online_offline_YYYY-MM-DD.json` | 本轮在核心文件中未发现对应 JSON 读取函数；调用方/API 需继续核对 | 按日期文件名清理 | 用事件表存储类型、时间戳、持续时长 |
| 全局小时流量统计 | `check_and_archive_all_clients()` 检测日期变化后调用 `save_global_traffic_stats_to_file()`；写入 `global/stats/traffic_YYYY-MM-DD.json` | `get_global_traffic_stats()` 只返回当前进程内的 `g_global_hourly_traffic`；本轮未发现历史文件加载函数 | 按日期文件名清理 | 用 `global_traffic_hourly` 按日期和小时存储上/下行字节 |
| 全局应用类型统计 | `update_global_app_type_stats()` 更新内存中的日累计数组和小时事件链表；小时数据通过 `cleanup_expired_hourly_stats()` 清除过期事件 | `get_global_daily_app_type_stats()` / `get_global_hourly_app_type_stats()` 读取内存状态 | 日切换重置每日数组；小时记录仅保留近 3600 秒 | 当前证据表明这是运行态聚合，不应直接假定必须持久化；需根据 API/产品需求确认是否保存每日汇总 |
| 在线客户端、实时速率、当前会话 | 来源于客户端对象、内核接口、DHCP/无线状态等；`main.c` 周期刷新客户端列表和统计 | 进程内存；备份恢复后会通过 `reset_client_runtime_fields()` 清空部分实时字段 | 进程生命周期；在线会话部分字段会随客户端备份写出 | 默认留在内存，只有明确要求跨重启恢复的会话累计字段才入库 |
| IPC/诊断文件 | `dump_client_list()` 写 `OAF_DEV_LIST_FILE`，`dump_client_visit_list()` 写 `OAF_VISIT_LIST_FILE`；同时读取 `/proc/net/af_visit`、`/proc/net/af_client`、`/tmp/dhcp.leases` | 供其他进程/诊断读取，或作为系统输入 | 临时/运行态文件 | 不纳入业务数据库；继续保留文件/内核接口语义 |

### 10.2 已核实的路径和运行时行为

- SQLite 数据库当前路径由 `build_client_visit_db_path()` 生成：`<history_data_root>/client.db`；`history_data_root` 通过 UCI 的 `history_data_path`、`base_data_path`、`terminal_data_path` 回退选择，默认回退到 `/tmp/fwx`。示例配置 `files/fwx.config` 将 `history_data_path` 和 `base_data_path` 设为 `/tmp/oaf`。
- 当前访问记录表为 `app_visit_record(mac, record_date, appid, start_time, end_time, duration, action)`，索引为 `record_date` 和 `(mac, record_date)`。建表发生在 `open_client_visit_db()`，并非独立的一次性 schema 初始化。
- `main()` 启动顺序包括初始化客户端列表、加载有效访问时长配置、`init_client_visit_db()`、加载客户端备份。主循环每秒运行；周期计数触发归档、清理、备份同步。需要注意周期计数并不等于准确墙钟时间。
- `load_history_stat_from_file()` 与 `load_history_top_apps_stat_from_file()` 在当前分支源码中是空实现（返回 NULL）；不能把它们视为已实现的历史 JSON 读取路径。
- `save_client_visit_data_to_file()` 名称虽含 `to_file`，主体实际对 SQLite 的 `app_visit_record` 执行事务性替换写入。
- 目前存在两种清理模型：按 `record_time` 天数清理业务记录/文件，以及按 `history_data_size` 限制访问记录数据库文件大小。新设计应将保留天数与容量策略分开定义，避免通过任意比例删除来处理容量上限。
- 统计日期由 `localtime()` / `mktime()` 计算，为设备本地日界线语义。Schema 应明确将日期保存为设备本地日的起始 Unix 秒值（或等价的 ISO 本地日期），不要混用 UTC 日期和本地日期。

### 10.3 统一 SQLite Schema 草案（待源码调用方核对）

以下是逻辑表设计，不是本次提交已创建的 SQL：

1. `schema_migrations(version PRIMARY KEY, applied_at)`：记录数据库 schema 版本。
2. `client_profile(mac PRIMARY KEY, ip, ipv6, hostname, nickname, expire, offline_time, online_time, mf_user_loaded, is_wireless, updated_at)`：只放需要跨进程重启恢复的客户端资料/累计状态。实时速率、当前在线标记、访问中的 URL 等易变运行态字段不在此表。
3. `app_visit_record(id PRIMARY KEY, mac, record_date, appid, start_time, end_time, duration, action)`：保留现有业务字段；索引建议为 `(record_date)`、`(mac, record_date)` 和按查询形态确认后的时间索引。是否增加唯一约束需先核实同一 MAC、应用和时间区间能否合法重复，不能直接假设唯一键。
4. `client_hourly_stat(mac, record_date, hour, up_bytes, down_bytes, online_seconds, active_seconds, PRIMARY KEY(mac, record_date, hour))`：每客户端每天每小时一行。
5. `client_hourly_top_app(mac, record_date, hour, rank, appid, PRIMARY KEY(mac, record_date, hour, rank))`：保存每小时有限数量的热门应用，避免把数组打包成 JSON。
6. `client_daily_top_app(mac, record_date, appid, rank, total_time, PRIMARY KEY(mac, record_date, appid))`：保存每日热门应用聚合结果。
7. `client_online_offline_event(id PRIMARY KEY, mac, event_type, timestamp, duration)`：保存在线/离线事件；需确认是否有其他字段或外部调用依赖。
8. `global_traffic_hourly(record_date, hour, up_bytes, down_bytes, PRIMARY KEY(record_date, hour))`：保存全局每小时流量。
9. `global_app_type_daily(record_date, app_type, duration, PRIMARY KEY(record_date, app_type))`：仅在确认全局每日应用类型统计需要跨重启保留后启用；不持久化每秒级/每次更新的小时事件链表。

所有表的日期列应采用同一约定（建议设备本地日的起始 Unix 秒值），持续时间单位为秒，流量单位为字节。写入应使用参数绑定和事务；统计聚合写入应具备幂等性。索引最终应由真实查询语句决定。

### 10.4 待核实问题（进入重构前必须完成）

- [ ] 交叉搜索仓库中其他源码、LuCI/ubus API、脚本及构建文件里的 JSON 读写、目录遍历、数据库打开和历史统计调用方（第二轮已覆盖主要 C 模块、功能更新模块、内核侧 JSON 输出和规则脚本；仍需最终全仓复核）。
- [ ] 核对所有历史查询 API 的响应结构，确认统计数据字段与当前前端依赖（已确认小时统计、每日热门、全局流量的主要字段；仍需核对剩余调用方及边界行为）。
- [ ] 确认 `online_offline_*.json` 是否有外部读取者，是否需要跨重启查询（在已扫描的 C/Lua 代码中找到归档写入与过期内存事件删除，未发现对应历史读取 API；外部调用者仍待确认）。
- [ ] 确认 `client.json` 中 `online_session` 和当天统计快照哪些字段必须恢复，避免恢复过期实时状态或重复累计（已核对序列化/反序列化字段；启动加载先重置部分运行态字段，再恢复 session 与当日聚合快照，重启后累计语义仍需产品决策）。
- [ ] 明确 `record_time`、`history_data_size` 的现有用户可见语义，以及数据库超过上限后的可预测策略。
- [ ] 确认是否需要持久化全局每日应用类型统计；小时事件链表看起来是短期内存态。
- [ ] 根据 API 查询和删除场景最终确定索引、外键策略、唯一约束与删除级联行为。

### 10.5 本轮进度记录

- 日期：2026-10-15
- 目标：完成核心存储源码首轮盘点并提出统一 SQLite schema 草案。
- 改动文件：`docs/AI_WORKLOG_SQLITE_MIGRATION.md`
- 具体改动：记录当前 SQLite 表/路径、客户端备份恢复调用链、小时统计/每日热门/在线离线/全局流量 JSON 写入、运行态分类、清理逻辑与 schema 草案。
- 新 schema/行为变化：无；仅文档设计，不修改业务代码、不创建数据库表。
- 测试命令及结果：未运行构建或测试；本轮为源码阅读和文档更新。
- 未解决问题/风险：已核对核心存储 C 文件及主要 LuCI/ubus 历史查询路径；仍需对剩余模块做最终交叉搜索。Schema 尚未定稿。
- 提交 SHA：`83a49660bb261ac33268d76394a8bfc1bae74d82`（历史查询调用链补充提交；该记录已在本次进度更新中校正）。
- 下一步：结束剩余模块的持久化路径交叉检查，基于已确认的 API 查询/删除语义修订并冻结 schema；任务 1 完成后再进入统一存储层实现。


### 10.6 LuCI / ubus 历史查询调用链补充（全仓首轮交叉核对）

本轮继续核对了 `open-app-filter/src/fwx_ubus.c`、`luci-app-oaf/luasrc/controller/oaf_app_record.lua`、`oaf_user.lua`、`oaf_dashboard.lua` 和 `oaf_internet_audit.lua`：

- 应用访问历史：LuCI `oaf_app_record.lua:get_app_history_records()` -> ubus `fwx/common` 的 `get_app_history_records` -> `fwx_api_get_app_history_records()`，直接查询 `<history_data_root>/client.db` 的 `app_visit_record`，支持 MAC、appid、起止时间和分页过滤。现有返回数据字段由 C API 组装，重构必须保持 API 响应形状。
- 客户端小时历史：`get_hourly_stats` 的历史日期分支在 `fwx_ubus.c` 中直接读取 `client_data/<MAC>/stats/hourly_YYYY-MM-DD.json`；当天分支则读内存结构。因此虽然 `load_history_stat_from_file()` 是空实现，历史 API 本身仍依赖 JSON 文件。
- 客户端每日热门应用历史：对应 API 的历史日期分支直接读取 `top_apps_YYYY-MM-DD.json`；当天分支读内存结构。迁移时需要将该 API 历史分支改为 SQL 查询，不能只改写入函数。
- 全局小时流量历史：`get_global_traffic_stats` 的历史日期分支直接读取 `global/stats/traffic_YYYY-MM-DD.json`；`get_history_traffic_stats` 按天汇总历史 JSON 文件并合并当天内存统计。两条 API 都必须迁移到 SQLite。
- 全局应用类型统计 API `get_global_app_type_stats` 目前从内存的日累计数组或近一小时事件链表生成响应，没有从历史 JSON/数据库恢复的路径。是否需要跨重启历史仍是产品语义决策。
- LuCI 的 `set_record_base` 会更新 UCI 中的 `history_data_path`、`base_data_path`、保留天数和大小配置；`fwx_ubus.c` 现有路径变更逻辑会移动旧目录及 `client.db`。新版数据库路径切换行为必须保留或明确重设计，不能遗漏。
- 记录清理 API 会按 MAC、日期范围和类型（visits/stats/all）删除数据；当前访问记录 SQL 删除与统计 JSON 文件删除混合在 `delete_client_record_files()` 中。迁移后必须保持该 API 的参数/响应契约，并把所有删除范围落实到数据库表。
- 现有清理/重置 API 还会移除客户端数据、备份和全局目录，并清空访问记录表。重构时需明确“清空历史数据”与“删除客户端资料”是否是不同操作，避免误删配置或其他运行文件。

### 10.7 其他文件型数据分类补充

- `/etc/board.json`、功能识别/升级状态文件、feature 列表缓存等属于系统能力发现、升级流程或缓存，不属于客户端历史业务统计；应保持各自文件接口，除非另有独立需求。
- UCI 配置（例如 `fwx.config`、`fwx_record`）继续作为配置来源，不迁入业务数据库。
- `/tmp/record_whitelist_state`、`/tmp/appfilter_rules_state`、`/tmp/appfilter_whitelist_state`、`/tmp/fwx_cache/user_parental_control_detail.json` 属于运行状态标记或临时缓存；本轮没有证据表明它们是需要统一保留的历史业务数据。
- `/proc/net/af_visit`、`/proc/net/af_client`、`/proc/net/af_active_app`、`/tmp/dhcp.leases` 和 `/tmp/visit_list`、`/tmp/dev_list` 是运行时输入、内核接口或 IPC/诊断接口，不纳入业务 SQLite。
- 因此“所有持久化业务数据统一 SQLite”应覆盖客户端备份、访问历史、客户端统计、在线/离线事件、全局流量历史及经需求确认的全局汇总；不等于把配置、升级产物、日志、缓存、IPC 和系统状态文件一并迁入。



### 10.8 协作进度回写（2026-10-15，任务 1 进行中）

根据本轮实际源码检查与已提交文档内容，更新当前状态：

- [x] 更新本工作日志，记录首轮核心持久化盘点与 SQLite Schema 草案。
- [x] 补充主要 LuCI / ubus 历史查询路径，确认客户端小时统计、每日热门应用、全局流量历史 API 仍有直接读取 JSON 文件的分支。
- [x] 补充路径切换、历史删除/重置 API 的兼容要求，并区分业务历史数据与 UCI、IPC、升级状态、内核接口和临时缓存。
- [ ] 完成全仓余下文件、脚本、构建配置及所有 JSON/SQLite 调用方的最终交叉核对。
- [ ] 按实际 API 响应字段核实每类统计的字段单位、缺失值、排序和分页语义。
- [ ] 明确在线/离线事件、客户端备份实时/累计字段、全局每日应用类型统计的持久化边界。
- [ ] 定稿数据库大小上限与保留天数策略，确认事务、索引、外键/级联删除和 Schema 迁移规则。
- [ ] 在上述核对完成后冻结 Schema；之后才进入存储层实现。

**当前任务状态：进行中，尚未完成。** 本轮修改仅涉及协作文档，不包含业务代码变更，不创建/迁移数据库，也未运行构建或设备测试。不得将 Schema 草案当作已实现设计，也不得把首轮搜索描述为全仓审计完成。

下一次继续时，应从未完成的核对清单开始，逐项验证并将证据、修改文件、测试命令及结果追加到本工作日志；一次只推进一个任务。


### 10.9 本次 AI 会话交接备注（2026-10-15）

本会话到此结束，后续工作交由其他 AI 继续。本会话完成的范围仅限于**源码调查、数据分类、调用链记录和 Schema 草案文档化**，没有修改业务源码。

- 已检查的核心源码：`open-app-filter/src/fwx_user.c`、`fwx_user.h`、`main.c`、`fwx_ubus.c`，以及示例配置 `open-app-filter/files/fwx.config`。
- 已检查的主要 LuCI 控制器：`oaf_app_record.lua`、`oaf_user.lua`、`oaf_dashboard.lua`、`oaf_internet_audit.lua`。
- 已记录当前 SQLite 访问记录表及数据库路径、客户端备份恢复流程、小时统计/每日热门/在线离线/全局流量的文件持久化入口、主要历史查询 API、路径切换和删除/重置 API 的约束。
- 已在本日志中整理统一 SQLite 的逻辑 Schema 草案，并标出需要进一步确认的数据语义和 API 契约。
- 未完成：全仓最终交叉核对、Schema 定稿、业务代码修改、数据库初始化/迁移实现、构建测试、设备测试及性能验证。
- 验证情况：本会话没有运行构建、单元测试或设备测试；不能据此声称功能已实现或通过验证。
- 建议后续 AI 首先通读本文第 10 节，逐项处理第 10.4、10.8 节的未完成项；确认数据字段和 API 契约后再冻结 Schema，并按阶段一次实现一个可验证的改动。不要实现旧 JSON 导入或读取兼容层，因为项目目标已明确排除该需求。

**交接状态：任务 1（持久化盘点与 Schema 设计）仍进行中；本会话无业务代码改动。**


### 10.10 第二轮跨模块存储审计（2026-10-15）

> 本轮范围：在核心用户/ubus 文件之外，筛查服务端其他 C 模块、功能更新流程、内核侧源码及规则脚本中的文件访问、JSON 序列化、目录遍历与 SQLite 调用。该轮提高了覆盖面，但尚不将其描述为全仓最终审计。

#### 新确认的非业务历史文件

- `open-app-filter/src/fwx_app_filter.c`、`fwx_mac_filter.c`、`fwx_record.c` 中的 `/tmp/appfilter_rules_state`、`/tmp/appfilter_whitelist_state`、`/tmp/macfilter_rules_state`、`/tmp/macfilter_whitelist_state`、`/tmp/record_whitelist_state` 是规则/白名单变更标记文件，内容是简单状态值，不属于历史统计数据。
- `open-app-filter/src/main.c` 写入功能升级状态与功能信息文件；`fwx_feature_online.c` 写入在线更新状态、临时下载/归档文件，并读取 `/etc/board.json` 与功能列表缓存。它们服务于系统能力发现、升级和状态展示，不应因统一业务数据库而迁入 SQLite。
- `fwx_stat.c` 的连接数来自 `/proc/sys/net/netfilter/nf_conntrack_count`，会话曲线使用固定大小的内存环形缓冲区；本轮未发现其业务历史持久化需求。
- `open-app-filter/src/fwx_utils.c` 对 `/dev/fwx` 的文件访问是设备接口；`/tmp/dhcp.leases`、`/proc/net/af_client` 等是系统输入。它们都不是业务历史存储。
- `open-app-filter/files/rule_manager.lua` 中的 `/tmp/fwx_cache/user_parental_control_detail.json` 是家长控制规则详情缓存；其他规则脚本的 JSON 多用于 UBUS 请求/响应或状态缓存。UCI 规则和白名单仍以配置包为准，不迁入历史业务库。
- 内核侧 `oaf/src/fwx_client.c` 的 JSON 构造用于运行态客户端/访问信息输出，不是独立的历史归档数据库。内核 proc 接口与用户态 IPC 输出必须保持原有协议。

#### 历史统计字段与 API 契约补充

- 客户端小时历史文件 `hourly_YYYY-MM-DD.json` 的顶层字段为 `date`、`mac`、`hourly_stats`。数组包含 24 个小时对象：`hour`、`apps`（最多 `TOP_APP_PER_HOUR` 个 app，含 `appid`，可能带 `name`/`icon`）、`traffic.up_bytes`、`traffic.down_bytes`、`online_time`、`active_time`。流量单位是字节，时间字段是秒。
- 每日热门应用文件 `top_apps_YYYY-MM-DD.json` 包含 `date`、`mac`、`count`、`apps`；每项包含 `appid`、`total_time`，可能包含展示用的 `name`/`icon`。持久化时长单位为秒；应用名称与图标缺失标记可在查询时根据应用目录动态生成，不宜作为统计事实重复存储。
- 全局流量文件 `traffic_YYYY-MM-DD.json` 包含 `date` 和 `hourly_traffic`，每小时含 `hour` 及 `traffic.up_bytes/down_bytes`。历史 API `get_global_traffic_stats` 以 `date`、`is_today`、`hourly_traffic` 返回数据；`get_history_traffic_stats` 另会跨日期汇总。SQL 迁移必须保持这两个 API 的响应字段和日期范围语义。
- `get_hourly_stats`、`get_top_apps_stats` 的当天分支取内存聚合，历史分支取文件；迁移后应只替换历史数据来源，不改变返回的 `mac/date/is_today/hourly_stats/count/apps` 等字段结构。
- 应用访问记录查询使用可选 MAC、appid、起止时间和分页条件，按 `end_time DESC, duration ASC` 排序。Schema 索引需以这些实际 where/order by 组合为依据；目前不能只依赖现有的 `record_date` 索引来断言所有历史查询都已优化。
- `online_offline_YYYY-MM-DD.json` 由内存链表归档，记录字段为 `type`、`timestamp`、`duration`；归档后会删除早于当天的内存事件。已扫描的历史 API 代码未发现该 JSON 的读取入口，但尚不能排除其他进程或外部脚本读取，因此需继续确认产品是否要求历史查询/跨重启保留。
- 客户端备份当前包含 `online_session`（上下行字节、在线/活跃时长、最近应用、开始时间）、`daily_stats` 和 `daily_top_apps_stats`。启动加载会先调用 `reset_client_runtime_fields()`，再恢复这些字段。目标库应区分跨重启累计快照与瞬时在线/速率/URL 状态；是否恢复跨重启的当前会话及如何避免重启时长重复累计仍需定稿。

#### 路径配置与清理语义风险

- 当前 `base_data_path` 主要控制 `client_data`/`client_backup`，而 `history_data_path` 控制 `global` 和 `client.db`；`set_record_base` 在路径变更时分别移动两组数据，并重置路径缓存。统一数据库后必须明确数据库的唯一位置及两个现有 UCI 路径字段的行为，不可简单忽略其中一个设置。
- `record_time` 以天为单位用于过期历史清理；`history_data_size` 当前主要作用于访问记录数据库大小控制。新版应把保留期与容量上限分开定义，并为所有历史表明确哪些数据按日期删除。
- 当前 `clean_all_data` 会清空数据库中所有非 SQLite 系统表，并清理客户端/全局目录。若加入 `schema_migrations` 或其他数据库元数据表，新清理逻辑必须保留 schema 版本元数据，仅清理业务数据；客户端资料删除、历史清空和配置保留也需要区分。
- 访问记录表没有唯一约束；归档逻辑按 MAC + 日期替换数据，而单条记录查询 API 支持同一时间段内多条记录。不能在未证明业务唯一性的情况下对访问事件设置可能丢失合法数据的唯一键。
- 设备本地日界线仍由 `localtime()/mktime()` 定义。以 Unix 秒存储日期时，应使用当地日零点时间戳，不能把它当作 UTC 日期或固定 86400 秒间隔来处理跨夏令时地区；按日循环清理也需使用本地日历递增以避免边界偏差。

#### 本轮状态

- 结果：新增了非业务 JSON/文件的分类、统计 JSON 字段契约、访问查询排序、路径双根配置及全量清理与 schema 元数据之间的风险记录。
- 业务代码：未修改。
- 测试：未运行构建、单元测试或设备测试。
- 仍未完成：剩余源码/LuCI 控制器和脚本最终复核；online/offline 事件的外部读取需求；客户端会话恢复语义；全局每日应用类型统计的持久化需求；UCI 双路径映射；保留期/容量策略；最终索引/约束/级联策略。
- 下一步：继续完成剩余调用方复核与语义决策，再冻结 schema；任务 1 未完成前不开始统一存储层业务代码改造。


### 10.11 本次进度确认与交接（2026-10-15）

本次会话按用户要求核对并整理了现有交接记录，确保后续 AI 能从已完成事项与未完成事项的边界继续工作。此前第二轮审计结果已记录在第 10.10 节，并通过提交 `a2a90a67d51e7f7ffb2fabe7a7ff8331b951d7fe` 更新本文件。

#### 已完成（仅限调查和文档）

- [x] 完成核心存储路径首轮调查并形成逻辑 Schema 草案（第 10.1–10.3 节）。
- [x] 记录主要 LuCI / ubus 历史查询链路，以及小时统计、每日热门应用、全局流量历史 API 对 JSON 文件的依赖（第 10.6 节）。
- [x] 对其他 C 模块、功能更新、内核侧输出和规则脚本进行第二轮存储分类审计，区分业务历史数据与 UCI、IPC、系统接口、升级状态及临时缓存（第 10.7、10.10 节）。
- [x] 记录统计 JSON 的主要字段、单位、查询排序/分页、路径双根配置、清理行为与潜在 schema 风险（第 10.10 节）。
- [x] 将上述发现回写到本工作日志，供后续 AI 作为交接上下文。

#### 明确尚未完成

- [ ] 全仓最终交叉核对所有 C/Lua/脚本/构建配置中的文件读写、JSON、SQLite 连接与调用方。
- [ ] 对照实际 API 实现和前端调用，完成所有响应字段、缺失值、排序、分页、日期边界的契约核验。
- [ ] 决定 online/offline 历史事件是否需要跨重启查询或被外部消费者读取。
- [ ] 决定客户端 `online_session` 的恢复/重启累计规则，以及全局每日应用类型汇总是否需要持久化。
- [ ] 明确 `base_data_path` 与 `history_data_path` 在统一数据库后的唯一存储位置和配置行为。
- [ ] 定稿 `record_time` 保留策略、`history_data_size` 容量策略、索引/约束/外键/级联删除、事务边界及 schema 版本机制。
- [ ] 完成以上调查与决策后冻结 schema；之后才开始阶段 B 的业务代码实现。

#### 本次会话验证边界

- 本次确认了工作分支中的 `AW.md` 已包含第 10.10 节审计记录。
- 本次没有修改业务源码，没有实现 schema，也没有运行构建、单元测试、设备测试或性能测试。
- 本工作日志中的 schema 仍是设计草案，不得描述为已定稿或已实现。
- 最近一次已知文档提交：`a2a90a67d51e7f7ffb2fabe7a7ff8331b951d7fe`。本节是本次准备追加的交接记录，提交完成后应以工具返回的实际 SHA 为准。

**交接指令：从上面的未完成清单继续，先做证据驱动的剩余调用方审计与语义决策。不要重复已完成的核心盘点，也不要在任务 1 完成前提前大规模修改业务存储代码。**


### 10.12 实际代码改动：统一访问记录 DB 初始化入口（2026-10-15）

#### 改动目的

在继续盘点过程中发现，`open-app-filter/src/fwx_user.c` 的 `open_client_visit_db()` 已负责打开 `client.db`、创建 `app_visit_record` 表并维护日期/MAC 日期索引；但 `open-app-filter/src/fwx_ubus.c` 的历史访问查询函数又重复执行了一次不完整的 `CREATE TABLE IF NOT EXISTS app_visit_record`。这种重复定义会让 schema 维护分散，未来修改字段或索引时容易产生漂移。

#### 已提交代码

- [x] 将 `open_client_visit_db(sqlite3 **db)` 从 `fwx_user.c` 的文件内静态函数提升为模块内共享函数。
- [x] 在 `fwx_user.h` 声明共享函数，并显式包含 `sqlite3.h` 以保证 `sqlite3 **` 声明有类型定义。
- [x] 修改 `fwx_ubus.c` 的历史访问记录查询路径：保留“数据库文件不存在时返回空结果”的既有分支；文件存在时改用共享 DB opener，不再在查询函数中单独创建一份表结构。
- [x] 复查更新后的 `fwx_ubus.c`，确认原有重复的 `CREATE TABLE IF NOT EXISTS app_visit_record` 片段已移除，查询路径调用 `open_client_visit_db(&db)`。

#### 文件和提交

- `open-app-filter/src/fwx_user.c`：`600bf824560ec2ea7449d6ddc1266468279c29e0`
- `open-app-filter/src/fwx_user.h`：共享函数声明 `dbd94d33240b7b5fe411b69d3552d9d75663d5e0`，SQLite 头文件依赖补充 `bd9ea14ac5736005f925e066cf0f34be8b03b146`
- `open-app-filter/src/fwx_ubus.c`：`85f290ec745d9e0d3b552272f10613140b59ccfb`

#### 验证和限制

- 已通过 GitHub 文件回读确认 `fwx_ubus.c` 中重复建表语句不存在，且共享初始化函数调用存在。
- 未运行编译、静态分析、单元测试或设备测试；因此目前只确认源码文本层面的改动，不宣称构建通过。
- 这是一项范围有限的存储初始化去重，不代表统一 SQLite schema 已实现或任务 1 已完成。全仓审计、schema 定稿及其他业务 JSON 读写迁移仍未完成。

**下一步：** 继续核对剩余源码/LuCI/脚本的持久化调用方与 API 契约；等 schema 设计有证据支撑后，再推进下一项存储代码改造。
