# API 缺口审计与逐条裁定（HDD 侧）

> **触发**：上级 2026-09-27 授权 —— 「该补的全补；API 解冻并补充**实际实现里该进 API 却没进的**；
> 权责拍板一切按**就近原则**和**实际化**处理；**SSD 这边别动**，把 **HDD 这边先收拾好**」。
> **落笔**：统筹。**输入**：`新文件系统设计稿.txt`（v0.6，1008 行）、
> `fs-api-design/{API-设计文档.md, nxfs.h, index.d.ts, CHANGELOG.md, README.md}`、
> `NXFS开发包/**`（11 个开发包 + `10-公共约定`）、`nxfs-impl/**` 实现侧公开面。
> **不是**靠回忆或推测：下面每一条都给 `文件:行` 或可复现的普查口径。

---

## 0. 方法与两条裁定原则

**审计做三件事**（都是机械可复现的）：

1. **规范点名普查**：把 `NXFS开发包/**` 里出现的每一个 `NXFS_*` 标识符，逐个查 `nxfs.h` 在不在
   ⇒ 差集 = "规范点名了、API 没有"的项（下面是原始输出）。
2. **实现面普查**：把 11 个包**自有头里实际导出的公开函数**全列出来（**281 个**），
   与 `nxfs.h` 的 **60 个函数表成员 + 58 个操作码 + 24 个能力位**求差
   ⇒ 差集 = "实现里有、API 没有"的项。
3. **跨包同类事实普查**：找"同一件事在不同包里各起一个名字、各写一遍"的族
   （v1.0.1 的 `NXFS_LEDGER_MAGIC` 上收、v1.1.0 的副本地址上收，都属这一类）。

**两条裁定原则**（上级给的，本文件把它们的**可操作定义**写下来）：

| 原则 | 可操作定义 | 判定后果 |
| :--- | :--- | :--- |
| **就近原则** | 规则/事实的**所有者** = 离它最近的那一层（碰数据的那一层）。跨包消费者**不得**重写规则，只能调所有者的公开面 | 规则**留在所有者的头文件里**；API 只在"所有者之外也要用"时才上收 |
| **实际化** | 以"**实际存在的外部消费者**"为准，不以"将来可能有人要"为准。外部消费者 = 绑定层（N-API/WASM）、OS 适配层（`A0` Windows / POSIX）、卷与分区工具（`D0`）、引导层（`B0`） | 有真实外部消费者 ⇒ **进 API**；只有核心内部包互相用 ⇒ **就近留包内**（登记，不上收） |

**范围口径**：**只做 HDD 侧**（设计稿 §一~§十四）。
`§十五 NAND 可选缓存`（已整体冻结、归二期）与 `§十六 SSD 优化方案`（暂缓、内部探索）
**明确排除**：能力位继续只预留、不置位、不定义 API。`C0-NAND可选缓存` 开发包不动。

---

## 1. 原始普查输出（可复现）

### 1.1 规范点名了、`nxfs.h` 里没有的 `NXFS_*`（去噪后）

| 缺口 | 出处 | 现状 / 性质 |
| :--- | :--- | :--- |
| `NXFS_DISK_FORMAT_VERSION` | `20-兼容层与安全路径/nxfs_disk_layout.h:48`（= `1u`） | **真缺口**：磁盘格式版本，API 无 |
| `NXFS_STORAGE_FORMAT_VERSION` | `30-存储抽象层/nxfs_storage_layout.h:55`（= `2u`） | **真缺口**（且与上面**同名不同物、各自命名**） |
| `NXFS_META_FORMAT_VERSION` | `40-元数据引擎/nxfs_meta_layout.h:55`（= `1u`） | **真缺口** |
| `NXFS_JOURNAL_FORMAT_VERSION` | `50-日志体系与恢复/nxfs_journal_layout.h:82`（= `1u`） | **真缺口** |
| `NXFS_LAYOUT_LEDGER_PRIMARY_LBA` | `20/nxfs_disk_layout.h:82` | **半缺口**：v1.1.0 已把 `secondary_lba` 等**字段**上收，但**派生名**仍在包内；`30` 另有 `NXFS_STORAGE_SECONDARY_LBA(total_sectors)`（`storage.c:274` 仍**现场推导**）、fixtures 里还有 `NXFS_FX_SECONDARY_LBA` |
| `NXFS_ST_SECTOR_BYTES` / `NXFS_STORAGE_CLUSTER_HEAD_BYTES` / `NXFS_FX_VOLUME_SECTORS` | `30/nxfs_storage_layout.h:58,87`；`verify/fixtures/nxfs_fixture.h:34` | **就近留包内**（扇区/簇头/夹具尺寸是各层几何，`D0` 已按"所有者 = 30"引用） |
| `NXFS_META_OPEN_RESCAN_BITMAP` | `40/meta.h:60` | **就近留 40**（打开标志） |
| `NXFS_XF_ERR_NOT_UNIQUE` / `_ORDER` / `_REGION_FULL` | `60` 开发包 | **就近留 60**（跨文件区自己的状态码；外部只见 `NXFS_ERR_*` 通用码 + `NXFS_VOLATTR_HAS_CROSSFILE_REGION`） |
| `NXFS_BLK_ERR_IN_PLACE_ONLY` / `NXFS_BLK_LOCK` | `70` 开发包 | **就近留 70**（块分配器自己的状态码） |
| `NXFS_EFI_*`（`NXFS_EFI_SUCCESS`/`TIMEOUT`/`MEDIA_CHANGED`/`EVT_TIMER`/`TPL_CALLBACK`/`TIMER_RELATIVE`） | `20/safepath_uefi_types.h` | **就近留 20**（UEFI 平台面，不是 FS 契约）；`13/13` 已由字面量断言锁住（尾巴批 W5） |
| `NXFS_STORAGE_ZONE_` / `NXFS_FILECLASS_` / `NXFS_PROGRAM_FOLDER_` / `NXFS_JOURNAL_` / `NXFS_JOURNAL_H` / `NXFS_TEST_DEVICE_H` | 各开发包 | **误报（前缀片段/文件名）**：`NXFS_STORAGE_ZONE_*` 在 30 头里、`NXFS_PROGRAM_FOLDER_INHERIT/EXPLICIT` 在 `nxfs.h` 里、`nxfs_file_class_t` 在 `nxfs.h` 里 |

### 1.2 实现里有、API 没有的**功能面**（差集摘要）

| 实现侧（所有者） | 面 | 外部消费者存在吗 | 裁定 |
| :--- | :--- | :--- | :--- |
| `D0` 的格式化入口 + `40` 的 `nxfs_meta_ledger_format` | **卷创建/格式化** | **有，且是硬需求**：`AC-A0.8` 要求"系统能经驱动的分区管理模块做原生管理（创建/移除/调整/**格式化**）"；`D0` 整包为此存在 | **进 API**（新操作码 + 尾追加成员）→ §2.2 |
| 四包各一份的格式版本 | **磁盘格式版本** | **有**：`A0`/`D0` 必须能判"这个卷的格式是不是我认得的版本"，否则只能盲挂 | **进 API** → §2.1 |
| `80` 的 `nxfs_tl_cache_stage/commit/stage_abort`（提交闸门） | 写路径暂存 | 无（调用方走 `file_write` + `txn_commit`） | **不上收**（就近留 `80`；登记） |
| `80` 的 `nxfs_tl_ring_*`（槽位生产/消费/ack） | 数据面槽位操作 | 无：API 的 §7 **刻意只声明 ring 的**内存布局**（`nxfs_ring_slot_t` / 槽位状态机 / `NXFS_RING_SLOT_SIZE`），由绑定层直接操作槽位** | **不上收**（登记：这是**有意**的分工，不是缺口） |
| `80` 的 `nxfs_tl_hot_*`（热路径直读） | 读路径加速 | 无（`file_read` 的 `read_options.bypass_cache` 已在 API 里） | **不上收**（登记） |
| `90` 的 `nxfs_prog_detect` / `prog_place_plan` / `region_select` / `mark_*` | 程序文件夹识别与放置策略 | 无（`dir_set_program_folder` 操作 + `NXFS_PROGRAM_FOLDER_*` 常量已在 API；策略由核心内部执行） | **不上收**（就近留 `90`；登记） |
| `30` 的 `nxfs_st_zone_plan`（区域划分）、`nxfs_st_cluster_is_bad` | 几何与坏簇 | 无（`50` 消费并核验；`bad_sector_report` / `NXFS_CAP_BAD_SECTOR_BITMAP` 已在 API） | **不上收**（就近留 `30`；登记） |
| `70` 的 `nxfs_alloc_*` / `nxfs_grow_eval` / `nxfs_reserve_*` / `nxfs_delayed_release` / `nxfs_delete_mark` | 块分配与回收引擎 | 无（`NXFS_OP_DEFRAG` / `COMPACT` / `RECLAIM` 操作已在 API；`reserved_bytes` 字段已在上收的块结构里） | **不上收**（就近留 `70`；登记） |
| `20` 的 `nxfs_sp_*` / `nxfs_layout_*` | L0/L1 原语 | **有**：`nxfs_safepath_v1_t`（7 个成员）与 `nxfs_safe_*` 结构**已在 API** | **已在 API**（无缺口） |
| `A0` 的 `nxfs_win_*` | OS 映射 | `A0` **就是**适配层本身 | **不上收**（是 API 的消费者，不是 API 的一部分） |
| `B0` 的 `nxfs_b0_*` | EFI 驱动 | `boot_stage`/`boot_handover`/`boot_table_read`/safepath **已在 API**；B0 的厂商协议面是 **EFI 专用**（SFSP/驱动模型） | **不上收**（登记） |

---

## 2. 逐条裁定（本批要落的）

### 2.1 裁定 A：**磁盘格式版本族进 API**（就近原则 + 实际化）

**事实（硬）**：磁盘格式版本在**四个包里各定义一份、各起一个名**：

```
20/nxfs_disk_layout.h:48        #define NXFS_DISK_FORMAT_VERSION     1u
30/nxfs_storage_layout.h:55     #define NXFS_STORAGE_FORMAT_VERSION  2u   /* 注释自陈"同属磁盘格式 v1 家族，各自独立演进" */
40/nxfs_meta_layout.h:55        #define NXFS_META_FORMAT_VERSION     1u
50/nxfs_journal_layout.h:82     #define NXFS_JOURNAL_FORMAT_VERSION  1u
```

而 `nxfs.h` 里**一个都没有**——`nxfs_volume_info_t`（`nxfs.h:879-897`）只有 `vol_attrs` 位，
**没有任何版本字段**。更刺眼的是：API 自己 v1.1.0 写的契约（`nxfs.h:1265-1272`）要求
"**不一致即判损坏**（**布局演进必须走 `version`**，不得静默漂移）"——**它引用的那个 `version`，API 里不存在**。

**裁定（就近原则 + 实际化）**：
- **就近**：每个版本的**值**由它那一层拥有（20/30/40/50 各自演进），API **不接管**具体数值逻辑；
- **实际化**：但"**一套对外可读的版本号 + 一个能一次问全的查询面**"必须由 API 单一来源，
  因为外部消费者（`A0` 的驱动、`D0` 的工具）**必须**能判"这卷我认不认得"。
- **落地**：① 在 `nxfs.h` 定义**版本族常量**（`NXFS_FORMAT_VERSION_DISK/STORAGE/META/JOURNAL`），
  各包把自己的宏**别名**到它并加静态断言（值保持一致，不许再各写一个数）；
  ② `nxfs_volume_info_t` **尾部追加**四个版本字段（MINOR，`struct_size` 保护旧调用方）。

### 2.2 裁定 B：**卷格式化进 API**（实际化，有 AC 背书）

**事实（硬）**：`nxfs.h` 的 **58 个操作码里没有任何创建/格式化操作**（只有 `DIR_CREATE` /
`SNAPSHOT_CREATE` / `SUBVOL_CREATE`）；而 `AC-A0.8` 明确要求系统能经驱动的分区管理模块
做**创建 / 移除 / 调整 / 格式化**，`D0-卷与分区管理` 整包就是为此存在，`40` 也已有
`nxfs_meta_ledger_format()`。

**裁定**：
- **格式化** = **文件系统**的操作（"把这个卷按 NXFS 格式初始化"）⇒ **进 API**：
  新操作码 `NXFS_OP_VOLUME_FORMAT` + `nxfs_api_v1_t` **尾部追加** `volume_format` 成员 +
  新的 `nxfs_volume_format_options_t` / `nxfs_volume_format_result_t`。
- **分区**（创建/移除/调整分区表项）**不进 API**：它是**存储栈/工具**的事，不是 FS 契约；
  就近归 `D0` + 工具；API 只提供**分区表标识**（`NXFS_MBR_TYPE_BYTE` / `NXFS_GPT_TYPE_GUID`，已在 API）。
- **L4 安全路径不参与**：格式化需要分配与策略 ⇒ 与"快照/去重/压缩/RAID/跨文件区分发/块级整理"同类，
  在 L4 入口上同样**不属 L4 域**（`NXFS_ERR_SAFEPATH_FORBIDDEN`）。

### 2.3 裁定 C：两条**语义勘误**（PATCH 性质，随本次 MINOR 一并发布）

| # | 项 | 现状 | 裁定 |
| :-: | :--- | :--- | :--- |
| C1 | `nxfs.h:1412` 说 `volume_gen`「使**所有**缓存整体失效一次」，而同文件 `:447` 说缓存有效性按**主体** `(kind,a,b)+gen`、"**不是**按卷/全局事务号" | 两条不能同真 | **按后者**（设计文档 §13.3 已给同一结论：条目有效性看**主体世代**，卷级基准点 `mounted_at_seq` 只用于"整体重整"）⇒ **改写 `:1412` 措辞**，明确 `volume_gen` 是 **VOLUME 主体**的世代，与 `mounted_at_seq` 的分工写清 |
| C2 | 交接后 `authority` 取哪个枚举值，`nxfs.h` 未定（`B0` 实现自行裁定为：阶段二 ⇒ `HANDOVER_PENDING(3)`、阶段三 ⇒ `LOCAL(0)`） | 实现已落地并冻结，规范侧无出处 | **追认该映射**并写进 `nxfs.h`（设计文档 §9.4「权威不对称层级」+ §11.2「阶段转移是所有权交接」本就要求"窗口内仍为源侧"）|

### 2.4 裁定 D：§18 的两条**推断项**转正（文档侧，无 ABI 影响）

`API-设计文档.md` §13.1/§13.2 的并发模型与崩溃一致性等级**原标"（推断）"**、§18.1 列"需你确认"。
现实化依据：`80` 批 80-B 已按"单生产者/单消费者 ring、`COMMITTED` 才可见"落地并逐条验收；
`50`/`40` 的事务与检查点语义已落地。⇒ **追认为规范**（去掉"推断"字样、§18.1 改为"已追认"），
依据栏引用实现侧的 AC 与用例。**SSD/NAND 排除**写进 §18.2 的口径行。

---

## 3. 明确**不补**的项（登记，免得后来人再问一遍）

| 项 | 为什么不补 |
| :--- | :--- |
| SSD 语义化放置 / ZNS / Zone TRIM（设计稿 §十六） | 上级裁定"SSD 这边别动"；设计稿本身标注"暂缓，内部探索，**不发布**"。能力位 `NXFS_CAP_SSD_SEMANTIC_PLACEMENT` **只预留、不置位** |
| NAND 可选缓存（设计稿 §十五 / `C0` 开发包） | **二期整体冻结**；三个能力位（`NXFS_CAP_NAND_CACHE` / `NXFS_VOLATTR_HAS_NAND_CACHE` / `NXFS_FEATURE_NAND_CACHE`）只预留不置位，探测恒返回"不可用" |
| NTFS ADS / 稀疏文件 / 配额 / send-recv / SMB-CIFS 锁语义 | 设计稿**未定义**（§18.2），没有"实际化"依据 ⇒ 保持"能力位预留 + API 不定义"；**触发条件**：出现真实消费者或设计稿补章 |
| 块大小进 API | 尾巴批 W3 已裁定"不落"，并给三条触发条件；本次**不推翻**。**新增观测点**：大虚拟环境（大虚拟盘 + 真 Windows 挂载）若使触发条件 ③ 成立，再按同一流程复议 |
| `30` 区域划分 / `90` 识别与放置 / `70` 分配引擎 / `80` 暂存闸门与槽位操作 / `60`/`70` 私有状态码 | **就近原则**：它们的外部消费者为零（只有核心内部包互相用）⇒ 留在所有者头里；本批**登记**，不上收 |

---

## 4. 本批（API v1.2.0 MINOR）的改动清单

| # | 文件 | 改动 |
| :-: | :--- | :--- |
| 1 | `nxfs.h` | 版本族常量 `NXFS_FORMAT_VERSION_{DISK,STORAGE,META,JOURNAL}`；新操作码 `NXFS_OP_VOLUME_FORMAT`；`nxfs_volume_info_t` **尾部追加**四个版本字段；新增 `nxfs_volume_format_options_t` / `nxfs_volume_format_result_t`；`nxfs_api_v1_t` **尾部追加** `volume_format`；C1/C2 两处措辞；`NXFS_API_VERSION` → `"1.2.0"`、`NXFS_ABI_MINOR` → `3u`；新增静态断言 |
| 2 | `index.d.ts` | 同步上述常量 / 结构体 / 成员 / 版本 |
| 3 | `API-设计文档.md` | §13.1/§13.2 去"推断"；§18 重写为"已裁定 / 已排除 / 触发条件"；新增"磁盘格式版本族"与"卷格式化"两节 |
| 4 | `CHANGELOG.md` | 新增 `v1.2.0 — MINOR` 条目（性质、逐项、契约、验证项数） |
| 5 | `verify/consistency_check.py` | 新增一致性检查（版本族常量存在且与实现侧同名同值；`volume_format` 在 C 与 d.ts 双侧存在；新字段在布局表中） |
| 6 | `README.md` | 版本、结构体/操作码/接口计数按实测更新；把"不在当前范围"里 SSD 那行改为"**明确排除（上级裁定）**" |
| 7 | **实现侧**（另一批） | 四包把版本宏**别名**到 API 常量 + 静态断言；`D0` 的格式化入口按新操作码接线 |

**验证口径（必须全绿才算完）**：`verify/verify_layout.c`（C11 布局与契约断言）→
`verify/verify_align.c`（C11 全量对齐）→ `zig c++ -std=c++17`（可编译性）→
`verify/ts`（`tsc --strict`）→ `verify/consistency_check.py`（三方一致性，项数按实测更新）
→ 推 API 仓 + **打 tag `v1.2.0`** → 父仓 `submodule-bump`（`nxfs-impl` 与本超项目两处指针）。

---

## 5. **空 API**：API 声明了、但没有任何块实现（本批第二面）

> 上级第二句话：「不只是求差，**API 设计时有了但是对应块没实现的空 API 也得在对应块补好**」。
> 这一面的普查口径 = **`nxfs.h` 声明的每一个对外面，在 `nxfs-impl/**` 里有没有实现落点**。

### 5.1 事实（硬）：**整张 `nxfs_api_v1_t` 是空 API**

| 证据 | 原文 |
| :--- | :--- |
| `40-元数据引擎/meta.h:545` | 「9. API（**本包实现面**；`nxfs_api_v1_t` 的**装配由集成层负责**）」 |
| `50-日志体系与恢复/README.md:385` | 「（**装配层 `80`/`A0` 将来**按 `nxfs_api_v1_t` 接线）」 |
| 全仓检索 | `nxfs_query_interface` 只在 **API 仓自身**（`nxfs.h` / `index.d.ts` / `verify/*`）出现；`nxfs_library_info` 同。`nxfs-impl/**` 的 11 个包里**没有任何一处**实现这两个导出符号，也**没有任何一处**填过 `nxfs_api_v1_t` |

⇒ 结论：**60 个函数表成员 + 2 个导出符号 + 58 个操作码的"信封"层，当前实现度为零。**
核心包各自提供的是"**本包实现面**"（签名比顶层 ABI 多一个卷句柄等，见 `40/README.md:304`），
**把实现面装配成对外 API 表的那一层从来没有人写**。这不是"某几个操作漏了"，是**整整一层缺席**。

### 5.2 逐条落点普查（按操作码/成员，`_<名>` 精确匹配实现仓）

| 类别 | 项 | 落点 |
| :--- | :--- | :--- |
| **有落点**（核心包已实现，名字可对上） | `volume_open/close/query/flush`、`txn_begin/commit/abort`、`snapshot_*`、`subvol_*`、`subject_query/gen_map/freeze/handover/invalidate`、`window_*`、`defrag`、`reclaim`、`bad_sector_report`、`device_health`、`space_query`、`checkpoint*`、`dir_create`、L4 的 7 项 | `40`（主）、`50`、`60`、`70`、`80`、`90`、`20`、`30`、`B0` |
| **无落点**（API 有、实现仓没有任何实现） | `fs_check`、`event_drain`、`event_ack`、`progress_query`、`batch_open`、`compact`、`file_seek`、`file_truncate`、`file_get_attr`、`file_set_attr`、`file_ioctl`、`dir_enum`、`dir_link`、`dir_rename`、`dir_remove` | **无** |
| **装配缺失**（核心有实现面，但没有对外表的接线） | 全部 60 个成员的"信封/句柄/错误码映射"层 | **无** —— 见 §5.4 |

### 5.3 归属裁定（就近原则 + 实际化）

| 面 | 就近所有者 | 依据 |
| :--- | :--- | :--- |
| **API 装配层**（`nxfs_query_interface` / `nxfs_library_info` / 61 成员接线 / 句柄透传 / 错误码映射） | **新建实现包 `05-API装配层`**（⚠️ **本行已改正**：原写 `A0-Windows集成层`，理由见 §5.6） | ① **实际化**：装配层必须与**核心同进程** —— API 的 L2/L3 契约要求热路径**零拷贝共享内存**（`nxfs.h` §7），跨进程装配等于每次调用都 IPC，与契约冲突；② **就近**：它是"核心的门面"，跨 40/20/30/50/60/70/80/90/D0 **所有核心包**，没有任何一个核心包"最近" ⇒ **没有现成所有者** ⇒ 按上级原话"**在对应块补好**"= **新建一个块**；③ **A0 承担不了**：A0 是 **OS 适配层（跨进程）**，且它自己有两条硬纪律 —— `build.py` 阶段 0 要求"**文件作用域可变对象 0 个**"（注入 IPC 客户端需要可变全局指针）、`AC-A0.1` 要求"**进程边界**、不得静态链接核心" ⇒ 放 A0 会把它的两条纪律**同时**打破 |
| **数据面（L3）装配件**：ring 槽位生产/消费/ack、热路径、提交闸门 | **`80`**（`A0` 装配层调用） | 就近：`80` 已有 `nxfs_tl_ring_*` / `nxfs_tl_cache_*` / `nxfs_tl_hot_*`；`A0` 只接线不重写 |
| **`volume_format` 的实现面** | **`D0`**（调 `40` 的 `nxfs_meta_ledger_format`） | 就近：格式化 = 卷与分区管理 |
| **`fs_check`** | **`40`**（汇总 `30` 的 `bitmap_rescan`/`space_query`、`50` 的 `tree_consistent`、`20` 的 `ledger_verify`） | 就近：一致性检查的判据分散在各层，**汇总入口**归元数据引擎（它的 `ledger_verify` 是骨架） |
| **`event_drain` / `event_ack`** | **`60`**（跨文件区/分发事件）＋ **`50`**（日志区事件）—— 按 §10 的"延后完成事件" | 就近：事件的产生者与排空者同层；`50` 负责日志区侧、`60` 负责分发侧 |
| **`progress_query`** | **`40`** | 就近：进度（分发/整理/回收/迁移）的事实源在 `40` 的 `progress` 结构（`40` 已有 `progress` 落点） |
| **`batch_open`** | **`80`**（映射由装配层交给调用方） | 就近：ring 是 `80` 的 |
| **`compact`** | **`70`**（`reclaim_extents` / `reserve_reclaim_eval` 之上加"压缩回收"口径） | 就近：块级回收在 `70` |
| **`file_seek` / `file_truncate` / `file_get_attr` / `file_set_attr` / `file_ioctl`** | **`40`**（文件对象与事务），`ioctl` 的**透传**归装配层 | 就近：文件语义在元数据引擎；`ioctl` 是设备私有通道 ⇒ 装配层只做信封透传 |
| **`dir_enum` / `dir_link` / `dir_rename` / `dir_remove`** | **`40`**（目录对象与事务）＋ **`90`**（程序标记跟随、链式继承） | 就近：目录语义在 `40`；`90` 只提供"标记怎么跟随"的策略（已有 `nxfs_prog_mark_*`） |

### 5.4 本批（"空 API 补实现"）的批次划分

| 批 | 内容 | 归属 | 验收口径 |
| :--- | :--- | :--- | :--- |
| **E1** | **API 装配层骨架**：新建包 `05-API装配层` —— `nxfs_query_interface` / `nxfs_library_info` / 61 成员全部非空（先按"逐成员接线，未具备的如实返回 `NXFS_ERR_NOT_SUPPORTED`"落地，再逐条填空）＋句柄**透传**（不做句柄表，避免引入可变状态）＋错误码映射 | **新建包 `05-API装配层`** | 宿主测试从 `nxfs_query_interface(1,3,&t)` 起，**逐成员断言非空**；对已具备的核心能力跑**端到端小链**（挂载→建目录→写文件→事务提交→读回→查询）；**有牙负对照**：故意让一个成员为空 / 让它返回 `NXFS_OK` 却不干活 ⇒ 判红 |
| **E2** | **L2 空缺操作补实现**：`fs_check` / `progress_query` / `compact` / `file_seek` / `file_truncate` / `file_get_attr` / `file_set_attr` / `file_ioctl` / `dir_enum` / `dir_link` / `dir_rename` / `dir_remove` / `event_drain` / `event_ack` | `40`（主）、`70`、`60`、`50`、`90` | 每项一个 `AC` 级用例（正面 + 一条有牙负对照）；窗口外单独登记 |
| **E3** | **数据面（L3）装配**：`batch_open` + ring 槽位操作对接到 API 表 | `80`（`E0` 接线） | 与 `AC-80.7`/`.9`/`.10` 共用器具；新增"经 API 表调用"的路径 |
| **E4** | **`volume_format`**（与 §2.2 同批） | `D0` | 造一个真镜像文件（红线内）+ 独立解析原始字节核验 |

> **纪律不变**：每批走 `dev/onsite-*` 分支 + 统筹 `--no-ff` 合并；每批 `gate_all` 全绿 + CI 成功；
> 判据只增不减；大虚拟环境（Windows/Linux 客户机）建好后，**E1 的端到端小链要能在客户机里跑通**。

### 5.5 本批进展（2026-09-27）

| 项 | 状态 |
| :--- | :--- |
| **① 缺口 → API** | ✅ **已发布 `API v1.2.0`**（tag `v1.2.0` = `17d7559`，`main` = `d009f5e`）：磁盘格式版本族上收 + `NXFS_OP_VOLUME_FORMAT` 卷格式化 + 勘误 C1/C2 + §18 重写（**SSD 明确排除**）。API 仓 **CI #8 success**；`consistency_check` **183/0**、C11 布局（50 结构体 / 18 断言）、全量对齐、C++17、`tsc --strict` 全绿。两处 gitlink 已上收（`nxfs-impl` = `69661d1`、超项目 = `3c154c4`） |
| **② 空 API → 机器可跟踪** | ✅ **铁则 §17 落成机器判据**：`verify/api_coverage.py`（四条判据 + 5 例自检）+ `verify/api-coverage.md`（A 段 **128** 行 + B 段 **14** 条跨包边）+ `verify/api-coverage-baseline.txt`（缺口基线 **127**）；`gate_all` **30 → 34 步**。⇒ **缺口从此可跟踪、只减不增**，且任何"新增对外面不登记 / 归属对不上 / 跨包边无主"都会**当场判红** |
| **② 空 API → 补实现（E1~E4）** | ⏳ **待做**：E1 装配层（**新建包 `05-API装配层`**，见 §5.6）→ E2 L2 空缺操作（`40`/`70`/`60`/`50`/`90`）→ E3 数据面装配（`80`）→ E4 `volume_format` 实现面（`D0`）。**棘轮基线 127 会随每批下降** |
| **③ 权责拍板** | ⏳ 进行中：就近原则/实际化已写进 `10-公共约定.md` §17.2；**装配层归属已改正**（§5.6） |
| **④ 大虚拟环境** | 🔄 **进行中**：WHPX **已启用并实测可用**（`HypervisorPlatform=Enabled` + `hypervisorlaunchtype=Auto` + `-accel whpx` 起得来）；Windows 11 23H2 客户机**正在装**（`F:\Workspace\vm\win11\win11.qcow2`，128 GB 稀疏；宿主窗口可交互 —— 上级亲自在点）。**实测结论**：① q35 默认会塞一个**空 CD** 且 OVMF **不认 `-boot order=d`** ⇒ 首次引导需从 EFI Shell 手动 `fs0:` + `\efi\boot\bootx64.efi`（已做成 `vmctl.ps1 -Action bootcd`）；② `bootindex` 只能挂在 `-device` 上（写进 `-drive` 会被当成块格式选项而报错）；③ `-nodefaults` 会连默认 VGA 一起去掉（`screendump` 报"no console"）⇒ 必须显式 `-vga std`；④ **`vvfat` 自报为固定盘**（`wmic logicaldisk` 里 DriveType=3）⇒ **Setup 不读它的 `autounattend.xml`**（Setup 只在**可移动**介质上找），故本机装 Win11 需用 Shift+F10 打 `LabConfig` 绕过 + 手动点完 GUI；⑤ QEMU **sendkey 的 `%` 必须映射成 `shift-5`**（漏了会让 `for %v` 变成 `for v`，cmd 报"此时不应有 v"）；⑥ `-WindowStyle Hidden` 会让 GTK 窗口 `MainWindowHandle=0`（人看不到），要给人看必须 `-WindowStyle Normal` |

### 5.6 裁定改正：装配层**不属于 `A0`**，**新建包 `05-API装配层`**（2026-09-27）

**为什么改正**：§5.3 原先按"包名就叫集成层 + `40`/`50` 文档点名 `A0`"把装配层判给 `A0`。
落地前核对 `A0` 的**自身纪律**，发现**放不进去** —— 两条会被**同时**打破：

| `A0` 的硬纪律 | 出处 | 装配层会怎么打破它 |
| :--- | :--- | :--- |
| **文件作用域可变对象 0 个** | `A0-Windows集成层/build.py` 阶段 0 的 `file_scope_objects()`（`const` 常量表不算） | 真实部署里操作表由 **IPC 客户端**注入 ⇒ 需要**可变全局指针**；若改成"直接静态链接核心"，又与下一条冲突 |
| **进程边界 / 不得静态链接核心** | `AC-A0.1`（也正是适配层 `nxfs_win_core_ops_t` **注入式**设计的前提） | 装配层若静态链接 40/20/50/… 就把核心拉进 A0 进程，正是该 AC 要避免的 |

**改正后的裁定（就近原则 + 实际化）**：

1. **装配层与核心同进程**：`nxfs.h` §7 的 L2/L3 契约要求热路径**零拷贝共享内存**；跨进程装配 = 每次调用都 IPC ⇒ 与契约冲突。故装配层**不是** OS 适配层的一部分。
2. **没有现成所有者**：它是"核心的门面"，横跨 40/20/30/50/60/70/80/90/D0 ⇒ 按上级"**在对应块补好**"，**新建一个块**：实现包 **`05-API装配层`**（延续现有编号风格；HDD 侧）。
3. **`A0` 地位不变**：继续做 OS 映射（`nxfs_win_*`），**通过 IPC 客户端消费 API**；它的两条纪律**一个字不改**。
4. **`E0` 的新增纪律（与铁则 §17 配套）**：① 只做**接线与转译**，**不重写任何规则**（每个成员只调"就近所有者"的实现面）；② **不做句柄表** —— `nxfs_handle_t` 与核心上下文**同一化/透传**，从而**零可变全局状态**（沿用 A0 的 `const` 表纪律）；③ 每个成员**一条正面用例 + 一条有牙负对照**；④ 棘轮基线随本包落地**只减不增**。

**下一步（E1 切片，按序落地）**：

1. 建包骨架：`05-API装配层/{build.py, nxfs_api_assembly.h, assembly.c, test/test_assembly.c, README.md, FREEZE.md, verify_freeze.py}`；
2. `nxfs_query_interface` / `nxfs_library_info` + **61 个成员全非空**（已具备的直接接线；未具备的**如实**返回 `NXFS_ERR_NOT_SUPPORTED`，登记表里保持"未具备"——**不许假装**）；
3. 逐成员填空：先 `40` 的卷/文件/事务/快照/子卷/主体 → `50` 的窗口 → `30` 的健康/坏道 → `20` 的 L4 → `70`/`90` 的整理 → `D0` 的格式化 → `80` 的数据面；
4. 每填一批：`E0/build.py` 全绿 → `api_coverage` **缺口下降**（棘轮） → `gate_all` **32/32**（接入 E0 后步数增加） → CI。
