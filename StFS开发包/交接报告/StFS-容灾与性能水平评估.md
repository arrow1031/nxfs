# StFS 容灾与性能水平评估（独立评估 · fs-eval）

评估人：独立评估方（fs-eval），只读审计，不改一字节。评估日期：见文件 mtime。

> **标签纪律**：每条判断标 `【已实现事实】`（代码/测试里可指认）／`【理论推断】`（给出推导）
> ／`【缺证据】`（代码无法判定或本环境无法验证）。**注释不是证据**——凡引用注释处，
> 均同时给出可执行代码或测试作为独立支撑；仅有注释的结论一律降级为 `【缺证据】`。

> **引用约定（2026-10-08 起）**：源码引用**以符号名为主**（形如 `stfs_journal_layout.h:STFS_JOURNAL_ENTRY_TYPE_*`），
> 行号仅作辅助且**随实现漂移**；文中残留的 `:NNN` 为历史引文，一律以符号名为准。

---

## 1. 评估基准与范围

### 1.1 基准

- 工件树：`F:\Workspace\nxfs-impl-wt-rename`，分支 `dev/onsite-112`，**HEAD = `a955755`**。
- 工作区非干净：`verify/carrier-evidence/{B,C}.serial.log`、`matrix.json`、`matrix.txt`
  为闸门产物（脏但非源码），本评估不依赖它们。 `【已实现事实】`

### 1.2 B8「统一放宽」在飞状态

B8 把文件层地址域 u32→u64、文件头 32→40 B、格式版本统一为 1。已落盘的步骤（按提交）：
`df01404`（Step A）、`b97026c`（Step B）、`39e4698`（Step C）、`2f0712c`（30 层格式版本→1）、
`84e1d43`、`622f4f0`、`a955755`（HEAD）。 `【已实现事实】`

- 先前的 B7 已落地：`stfs_cluster_t` u32→u64；计数/长度/簇内偏移/派生下标仍 u32；
  日志父条目 9→15 B（B9 `e3afefa` 前为 13 B）；镜像索引条目 8→12 B；总账 CRC 覆盖区间 84→100；主总账 88→104 B。
  `【已实现事实】`（`stfs_journal_layout.h` 的 `stfs_journal_parent_entry_t` / `stfs_journal_index_entry_t`；`stfs_storage_layout.h:STFS_STATIC_ASSERT`）
- **受 B8 影响的结论**：容量上限（u32 计数仍限制单文件字节数）、每操作字节数（文件头
  32→40 B 使每文件元数据抬升）、镜像/索引条目的每项字节数。本文凡涉及者均已在条目内标注。
  `【理论推断】`

### 1.3 本次评估覆盖 / 未覆盖

- **已评估**：pkg 30（存储抽象/簇头 CRC/坏道）、pkg 40（元数据引擎/COW/总账/日志探针）、
  pkg 50（日志体系/恢复/检查点/恢复分片）、pkg 70（分配/回收）、pkg 80（翻译层/热缓存环）、
  pkg D0（分区表镜像）；加上跨包的屏障/同步调用点全域检索。 `【已实现事实】`
- **未评估 / 不可判定**：真实硬件断电试验、真实介质坏道、性能基准实测、pkg 60/90/A0/B0
  的端到端语义。**因此本文的性能部分只给「理论级别」（复杂度与 IO 计数），
  不给实测吞吐/延迟。** `【缺证据】`
- 评估手段限定为 `git log/show/diff` 读、源码与测试读、`grep`/`Select-String`；**未构建、
  未跑闸门、未改文件、未提交**。

---

## 2. 容灾（数据可靠性 / 崩溃一致性）逐项评估

等级口径：**L0 无 · L1 起步（有检查无恢复）· L2 可用（可检测+可回退，无冗余）·
L3 工业级（冗余+自愈）· L4 高可靠（多副本+校验+在线重构）**。

### 2.1 ① 崩溃一致性（掉电 / 进程被杀）

- `【已实现事实】` **提交点明确**：COW 引擎按
  A（主体表）→ A1b（分片链）→ A2a（分配）→ A3（换指针）→ A3b（重物化）→ B（构建镜像对象）
  → A2b（分配）→ **C（写新对象，不可见 `cow.c:stfs_meta_object_write_prepared()`）** → **D（总账提交 = 提交点 `ledger.c:stfs_meta_ledger_commit()`）**
  → E（释放旧 `cow.c:stfs_meta_object_free()`）→ F（链提交 `subject.c:stfs_meta_subject_chain_commit()`）推进；旧数据只在提交点之后释放。
- `【已实现事实】` **总账双副本 + 交替 ext 槽**：`stfs_meta_ledger_commit`（`ledger.c:stfs_meta_ledger_commit()`）
  取 `slot = ctx->ext_slot ^ 1u`（`ledger.c:stfs_meta_ledger_commit()`），`led_write_copy`（`ledger.c:led_write_copy()`）
  **先写 ext 槽**（`lba+1+slot`）**后写 v1 前缀**（`lba`），v1 前缀 = 提交点；
  代码注释明确此顺序不可反转（`ledger.c:led_write_copy()`）。交替槽的存在使「写 ext 途中掉电」
  只会污染**非当前**槽。
- `【已实现事实】` **撕裂写判定**：`led_ext_crosscheck`（`ledger.c:led_ext_crosscheck()`）要求
  `ext.txn_id == v1.txn_id`，否则判 `CORRUPT` 并注明「半写窗口」；`led_parse_v1`
  （`ledger.c:led_parse_v1()`）与 `led_parse_ext`（`ledger.c:led_parse_ext()`）各自做 magic、
  `struct_size`、版本域、CRC32C 校验；`stfs_meta_ledger_read_copy`（`ledger.c:stfs_meta_ledger_read_copy()`）
  在两个 ext 槽中挑**能过交叉校验**的那个，否则 `ext_valid=0` + `corrupt_code`，
  **不假装表指针可用**。
- `【已实现事实】` **恢复分片**：pkg 50 有区域级恢复分片（`recover_shard.c`，固定容量登记表、
  去重队列、`STFS_ERR_NO_SPACE` 兜底）；`test_shard_partial.c` 覆盖「某物理区域整片不可读
  ⇒ 只丢该片」。
- `【缺证据】` **提交链上没有屏障**：`stfs_meta_txn_commit`（`txn.c:stfs_meta_txn_commit()`）**仅在 `if (sync != 0u)`
  时**调用 `stfs_meta_volume_flush(volume, sync, NULL)`（`txn.c:stfs_meta_txn_commit()`）；`cow.c:stfs_meta_edit_commit()`
  的 C→D→E→F 之间**没有任何 flush/barrier**。全域检索显示 `stfs_st_flush`/`barrier`
  只出现在 `30/storage.c`、`05/assembly.c:ea_volume_flush()`、`40/ledger.c:stfs_meta_volume_flush()`、`40/meta.h:STFS_META_OPEN_NO_COMMIT_BARRIER`、
  测试与 `tools/*`；**pkg 50 的全部 `.c` 中没有任何 flush/barrier/sync 调用**。
  ⇒ 结论：**顺序假设依赖设备与驱动的写序，而非显式屏障**；`storage.h:stfs_st_host_fns_t.barrier` 明确
  「barrier 为 NULL 时可见地降级」，`storage.c:stfs_st_flush()` 仅当宿主提供 barrier 才置
  `device_barrier=1`。因此「ext 先、v1 后」在**无屏障的宿主**上是否真有序，代码无法保证。
- **本项等级：L2**。可检测（CRC+交叉校验+双副本回退）且可回退（选另一副本），
  但**持久化由调用方 opt-in**，提交链内部无屏障 ⇒ 不满足 L3 「崩溃后必然一致」。

### 2.2 ② 写原子性与「如实失败」

- `【已实现事实】` **不伪造成功**：`led_write_sector`（`ledger.c:led_write_sector()`）在两阶段写语义下
  把 `QUICK_RETRY_DONE`／`EVENT_LOG_EXHAUSTED`／`APPEND_UNCERTAIN` 一律**按已写处理并标 gap**，
  **绝不对同一笔写重发**。
- `【已实现事实】` **不确定态如实上抛**：`txn.c:stfs_meta_txn_commit()` 的 UNCERTAIN 返回
  `STFS_ERR_INTEGRITY_UNCERTAIN` 而非可重试错误；`txn.c:stfs_meta_txn_commit()` 支持幂等重提交；
  `txn.c:stfs_meta_txn_commit()` 把三类失败分开处置并触发镜像重同步，**绝不出现半新半旧**。
- `【已实现事实】` **副总账失败不掩埋**：`ledger.c:stfs_meta_ledger_commit()` 副总账写失败只置
  `secondary_behind=1`／`ledger_degraded=1`，主副本已提交的事实照旧成立。
- `【已实现事实】` **读侧不把 0 当有效数据**：`st_repair_read_error`（`storage.c:st_repair_read_error()`）
  在重读失败后 `memset 0` 再写回以触发固件重映射，并**故意不补写簇头 CRC**
  （`storage.c:st_repair_read_error()` 注释「避免把 0 伪装成有效数据」）。`storage.c:stfs_st_cluster_write_headed()` 的
  读改写路径在载荷不可验证且非全 0 时**拒绝写**；`storage.c:stfs_st_cluster_write_headed()` 在 gap≠0 时置
  `integrity_gap`。
- `【已实现事实】` **只读闸门**：`stfs_meta_write_allowed`（`ledger.c`）作为写入口的统一
  判据。
- **本项等级：L3**（就「诚实失败」这一维度而言）。这是当前实现最扎实的一项：
  失败被显式分类、不被静默吞掉、不允许假绿。

### 2.3 ③ 元数据完整性校验

- `【已实现事实】` **CRC 覆盖面**：主总账 v1 前缀 CRC32C 覆盖 `[0, offsetof(crc32c))`
  （`ledger.c:led_parse_v1()`，B7 后为 100 B）；ext 槽 CRC32C（`ledger.c:led_parse_ext()`）；
  位图头部 CRC 覆盖 `[0, 68)`（`storage.h:stfs_st_bitmap_header_crc()`、`stfs_storage_layout.h:stfs_storage_bitmap_header_t.crc32c`）；
  位图分片 CRC 覆盖 `[0, 560)`（`stfs_storage_layout.h:stfs_storage_bitmap_shard_t.crc32c`）；**每个数据簇的整簇载荷
  CRC32C**（`stfs_storage_layout.h:stfs_storage_cluster_head_t`，8 B 簇头 = `{crc32c, flags}`，
  CRC 覆盖 `[8, cluster_size)`）；文件块 v2 头 16 B 自带自校验 CRC（`meta.h:STFS_META_FILE_BLOCK_V2_OFF_CRC`）；
  日志头记录 CRC 覆盖 `[0, head_bytes−4)`（`stfs_journal_layout.h:STFS_JOURNAL_HEAD_CRC_BYTES`／`STFS_JOURNAL_HEAD_CRC_TRAILER_BYTES`，头部布局注释）。
  CRC 本体统一复用 pkg 20 `stfs_sp_crc32c`（软件/硬件两路已由 AC-20.13 交叉验证），
  **不另写多项式**（`storage.h` §14 注释（复用包 20 的 `stfs_sp_crc32c()`）、`stfs_storage_layout.h:stfs_storage_cluster_head_t` §2 注释）。 `【已实现事实】`
- `【已实现事实】` **结构性闸门不靠 CRC 兜底**：`struct_size != sizeof` ⇒
  `STFS_ERR_NOT_SUPPORTED`（`ledger.c:led_parse_v1()`）；版本域、`boot_entry_count ≤ 4`、
  `cluster_size ∈ {4096,16384,65536}` 均在解析期拒绝；ext 的 `format_flags`/`reserved`
  必须为 0（`ledger.c:led_parse_ext()`）；指针与计数的一致性、容量与边界字段的一致性由
  `led_ext_crosscheck`（`ledger.c:led_ext_crosscheck()`）与 `journal_layout.c:jr_prefix_consistent()`
  （「容量必须等于由边界字段推出的值，不许自说自话」）分别钉住。
- `【已实现事实】` **退役与不兼容对象**：v1 整表路径
  `STFS_META_MIRROR_SUBJECT` 在 `cow.c:edit_build_mirror_objects()` 直接返回 `STFS_ERR_NOT_SUPPORTED`
  （已退役，不静默走老路）。
- `【缺证据】` **pkg 40 不在宿主分配通道上校验 `struct_size`**；`led_parse_v1`
  对 CRC 计算两次（`ledger.c:led_parse_v1()` 内先 `led.crc32c != stfs_sp_crc32c(...)`、后又 `crc = stfs_sp_crc32c(...)` 再比一次）——前者是冗余不是缺陷，但说明该路径
  未经「一次计算」审计。
- **本项等级：L3**。元数据层 CRC + magic + struct_size + 版本 + 交叉一致性齐备，
  且覆盖到**数据簇载荷**（见 2.5）。

### 2.4 ④ 介质故障冗余（副本 / RAID / 擦除码 / 镜像）

- `【已实现事实】` **盘上真实冗余只有两处**：
  ① 总账主 + 副副本，交替 ext 槽，带 `stfs_meta_ledger_repair`（`ledger.c:stfs_meta_ledger_repair()`）
  与 `stfs_meta_ledger_verify`（`ledger.c:stfs_meta_ledger_verify()`，含 `primary_ok`/`secondary_ok`/
  `txn_id_match`/`copies_differ` 与最高到 `STFS_CHECK_FULL` 的等级）；
  ② **分区表镜像** `d0_gpt_verify_mirror`（`D0-卷与分区管理\d0_part_table.h:d0_gpt_verify_mirror()`）。
- `【已实现事实】` **`MIRROR_*` 不是副本**。`meta_internal.h:STFS_META_MIRROR_*` 定义
  `STFS_META_MIRROR_NONE/SUBJECT/SNAPSHOT/SUBVOL/SUBJECT_ROOT/SUBJECT_SHARD`，
  `meta_internal.h:stfs_meta_edit_object_t.mirror` 是 `uint32_t mirror`；`edit_build_mirror_objects`
  （`cow.c:edit_build_mirror_objects()`）用 `switch (o->mirror)` 决定**该编辑对象是哪个源对象的落盘像**：
  `SUBJECT_ROOT`（构建主体表根索引对象）、`SUBJECT_SHARD`（
  1 簇数据分片）、`SNAPSHOT`、`SUBVOL`、
  `SUBJECT`（已退役）。
  ⇒ **`MIRROR_*` 是 COW 载荷内的「对象种类判别符」，不是可独立校验的第二份拷贝，
  也不是临时缓冲。** `MIRROR_BYTES(4096)` 指的是主体表**落盘像的缓冲区字节数**，
  与冗余无关。 `【已实现事实】`
- `【已实现事实】` 主体表**没有**第二份拷贝：主体表是「根索引对象 + 脏分片」的**单一**逻辑
  结构（`cow.c:edit_table_payload_len()`/`cow.c:edit_build_mirror_objects()`），不存在两份内容相同、可互校的表。
- **本项等级：L1**。文件数据与元数据**均无盘内冗余**；RAID/擦除码完全不在本文件系统内。
  这与人类设定的「介质冗余交给低层 RAID 卡」一致——即本 FS **有意**不做介质冗余，
  现状与设计口径互相吻合（见 §6.4 分层契约）。
  ⇒ 因此：**任何单份介质损坏（除总账双副本与 GPT 镜像外）都不可修复，只能检测。**

### 2.5 ⑤ 静默损坏检测与修复

- `【已实现事实】` **读路径逐簇校验**：`stfs_st_sector_read`→`stfs_st_cluster_verify`
  （`storage.c:st_load_cluster()`）；`verify == STFS_VERIFY_NONE` 时**跳过** CRC；
  失败时填 `bad_lba`/`bad_cluster`/`crc_expected`/`crc_actual`、`stats.crc_failures++`、
  返回 `STFS_ERR_CORRUPT`（`storage.c:st_load_cluster()`）。文件层读取
  （`file.c:file_block_read_hdr()`、`file.c:file_reader_read()`、`file.c:file_map_load()`（定义在后段，前段是先声明））
  **默认 `STFS_VERIFY_CRC32C`**。
- `【已实现事实】` **写路径每次落盘都封 CRC**：`storage.c:stfs_st_cluster_write_headed()` 在扇区写前
  `stfs_st_cluster_seal`；`storage.c:st_write_cluster_span()`、`storage.c:stfs_st_cluster_init()` 亦封。⇒ 校验覆盖率 = 全部
  经本层写出的数据簇。
- `【已实现事实】` **修复只到「触发固件重映射」这一级**：`st_repair_read_error`
  （`storage.c:st_repair_read_error()`）重读 → 仍失败则清零并写回。它**不重建内容**，因为
  **没有第二份数据可依据**（见 2.4）。⇒ **检测 ≠ 修复**，此处必须如实分开说。
- `【缺证据】` **没有在线 scrub / 自愈守护**：`stfs_meta_fs_check`（`ledger.c:stfs_meta_fs_check()`）
  存在，但其扫描范围（是否遍历数据簇、命中不一致后做什么）**未能确认**；
  `30-存储抽象层\storage_badsector.c`、`storage_smart.c` 的存在说明有坏道/SMART 通道，
  但**是否构成定期巡检与自愈回路，本次未能取得可指认证据**。故本项判为
  **「读触发式检测强、主动巡检与修复弱」**。
- **本项等级：L2**。检测覆盖到数据载荷（这是 L2 的支柱）；修复能力 L1（无冗余可依）。

### 2.6 ⑥ 误删除与回滚

- `【已实现事实】` 存在快照与子卷的对象种类（`STFS_META_MIRROR_SNAPSHOT`、
  `STFS_META_MIRROR_SUBVOL`），COW 机制本身在提交前保留旧对象
  （`cow.c:stfs_meta_object_free()` 之后才释放），⇒ **在提交点之前具备天然的「未生效即回滚」**。
- `【已实现事实】` **回收受授权闸门约束**：`cow.c:stfs_meta_edit_commit()` 的回收授权仅对
  「挂载已授权」的会话开放；自耗回收记录避免循环回收。
- `【已实现事实】` 簇状态机 `USED --批量回收--> DELAYED --释放条件满足--> FREE --分配--> USED`
  （`stfs_block_alloc.h:stfs_cluster_state`）；回收**按 extent 批量**，`free.c:stfs_reclaim_extents()`（文件头注释同述）明确
  「批次数 == extent 数 ≪ 簇数」，并可由 trace 计数证伪（`test_reserve_grow.c:case_ac_70_7()`
  有负对照「逐簇实现必红」）；预留回收要求**时间 ∧ 使用率双条件同时满足**，
  只满足其一则**一个簇都不回收**并给出原因码（`stfs_block_alloc.h:stfs_reserve_reclaim_reason_t`）。
- `【缺证据】` **用户可见的「误删恢复」路径**（只读挂载回滚、快照挂载取回、误删回收站）
  本次未取得端到端证据；快照/子卷的**读回**语义在 pkg 60/90 侧，未评估。
  `reclaim`/`discard` 与 TRIM 的对接亦未取得证据。
- **本项等级：L2**。机制骨架（快照/子卷/COW/延迟回收/授权闸门/双条件预留回收）齐备，
  但**「人误删后如何拿回」的可用性未证实** ⇒ 不给 L3。

### 2.7 ⑦ 降级与安全兜底

- `【已实现事实】` **只读降级**：`stfs_meta_write_allowed`（`ledger.c`）为写入口统一判据；
  `stfs_journal_open` 在总账主副本坏且调用方未显式同意降级时返回
  `STFS_ERR_LEDGER_PRIMARY_CORRUPT` 并**仍填好恢复报告**（`journal.h:stfs_journal_open()`、
  `recover.c:stfs_journal_ledger_probe()`），降级需显式 `consent_degrade`（`recover.c:stfs_journal_ledger_probe()`）；
  两份总账皆不可信 ⇒ `STFS_ERR_LEDGER_SECONDARY_CORRUPT`（终局）；无根日志区 ⇒
  `STFS_ERR_RECOVERY_REQUIRED`（`recover.c:stfs_journal_open()`）。
- `【已实现事实】` **非法输入拒绝**：日志区分配任何一条不满足即 `STFS_ERR_NO_SPACE`，
  「**绝不返回一个『差不多』的位置**」（`journal.h:stfs_journal_plan_alloc()`）；区划划分要求恰好覆盖
  数据区且每片首簇在数据区内，否则 `STFS_ERR_INVALID_ARG`
  （`test_zone_plan.c:zp_test_zone_plan_consume()`）；无屏障原语 ⇒ 可见降级而非假装有屏障
  （`storage.h:stfs_st_host_fns_t.barrier`）。
- `【已实现事实】` **边界溢出拒绝**：`test_reserve_grow.c:case_ac_70_9()` 钉住「949+1 == 950 允许；
  950+1 == 951 拒绝 ⇒ 最多用到 95%」；`test_main.c:test_extra_boundaries()` 钉住日志容量用满
  ⇒ `STFS_ERR_JOURNAL_FULL`（**不偷偷扩容**）。
- `【缺证据】` **已有已知缺陷**（先于本次评估、非本次引入）：向 `volume_open` 传非法设备名
  **会崩溃**（`F:\Workspace\.tools\C5-写路径计划.md` §4「候选缺口」、§M5 侦察「已知坑」、§M5 已落地「如实记录不阻断三点」）。
  这与「非法输入必须被拒」的口径相冲突，应记为已知风险。
- **本项等级：L3**。拒绝语义在代码与测试两侧都有独立支撑，是本实现第二扎实的一项。

### 2.8 总体容灾等级

**总体：L2（可用级）**，且内部不均衡：

- 强项（已达 L3）：**诚实失败**（§2.2）、**校验覆盖面**（§2.3、§2.5 的检测侧）、
  **非法输入拒绝与只读降级**（§2.7）。
- 弱项（L1）：**介质冗余**（§2.4，且为设计选择）、**修复能力**（§2.5 的修复侧）、
  **主动巡检/自愈**（§2.5）。

**升到 L3 所需**（按依赖序）：
1. 提交链**显式屏障**：在 `cow.c` C→D 之间、`ledger.c` ext→v1 之间引入可见的
   flush/barrier（或明确声明「无屏障宿主只提供 best-effort 一致性」并写入契约）。
2. **在线 scrub + 自愈回路**：定期遍历数据簇对 CRC，并对**有副本的结构**
   （总账、GPT 镜像）自动重建。
3. **至少一类元数据的可重建冗余**：现仅总账有双副本；主体表/索引表无第二份。
   ⇒ 这正是人类提出的「纠错辅助数据」所指向的缺口（见 §6、§7）。

---

## 3. 性能水平（理论级别）

> 本节只给**复杂度与 IO 计数**，并由此推导量级上限。**没有实测数据** ⇒ 不给实测吞吐/延迟。
> `【缺证据】`

### 3.1 复杂度与每操作 IO 计数

| 操作 | 复杂度 / IO 计数 | 证据 |
|---|---|---|
| 挂载 | 读 v1 前缀 + 扫 2 个 ext 槽取交叉校验通过者，再读表根；常数次（≈4–8 次 IO），不随卷大小增长 | `ledger.c:stfs_meta_ledger_read_copy()`、`ledger.c:stfs_meta_ledger_load()` |
| 目录定位 | **O(深度)** 次头记录读（`head_reads = depth + 1`），**不扫描日志、不依赖日志规模** | `journal.h:stfs_journal_locate()`、`journal.h:stfs_journal_location_t.head_reads` |
| 枚举 | 主体表「根索引 + 分片」直取，非全表扫描 | `cow.c:edit_table_payload_len()`、`cow.c:edit_build_mirror_objects()` |
| 创建/删除（元数据事务） | COW 十阶段，随机 IO 量级 ≈ 每主体表项 1 次对象写 + 分配记账 + **总账提交 2 笔扇区写（ext 槽 + v1 前缀）**；`【理论推断】` K ≈ 4–8 次随机 IO | `cow.c:edit_alloc_objects()`、`cow.c:stfs_meta_edit_commit()`；`ledger.c:led_write_copy()` |
| 读 | 逐簇 1 次簇读 + 整簇 CRC 校验；块地址由 v2 位置映射 O(1) 取得（v1 哈希链读路径已删除） | `storage.c:st_load_cluster()`；`file.c:file_map_load()`、`file.c`（R1-v1 退役注释：`file_chain_validate()` 已删除） |
| 写 | COW：受影响簇读改写（`storage.c:stfs_st_cluster_write_headed()`）+ 新簇写；**每 N 字节新数据 ≈ N 字节数据 + 元数据对象** | `storage.c:stfs_st_cluster_write_headed()` |
| 元数据分配 | 每簇 **2 bit** 状态码（空闲／已用·非本会话／本会话已发出／本会话保留），bitmap 放**调用方提供的 scratch**，本包不自分配 | `stfs_block_alloc.h:STFS_BLK_ALLOC_SCRATCH_BYTES`（含 `STFS_BLK_LOCK(scratch_is_two_bits_per_cluster, …)`）、`stfs_block_alloc.h:stfs_blk_alloc_options_t.scratch` |
| 分配查找 | **next-fit**：自 `cursor` 起顺序找一段连续 count 簇并回绕 ⇒ 平均 O(1)、最坏 O(总簇数)；候选段做**逐簇**坏道裁决（`stfs_st_cluster_is_bad`）⇒ 每次分配 O(count) 次位图查询 | `alloc_run.c:blk_pick_run()`；`alloc_state.c:blk_bad_pass()`；`stfs_block_alloc.h`「批 70-F」权属注释、`stfs_block_alloc.h:stfs_blk_bad_query_fn`（R2 坏道判据） |
| 回收 | **按 extent 批量**，批次数 == extent 数 ≪ 簇数（有负对照防退化） | `free.c` 文件头注释、`free.c:stfs_reclaim_extents()`；`test_reserve_grow.c:case_ac_70_7()` |
| 日志追加 | **每条记录 = 1 次头簇读 + 1 次头簇写**（走溢出段再 +2 次簇 IO） ⇒ **O(M) 条记录 = O(M) 次簇 IO，无批处理** | `journal.h:stfs_journal_entry_append()` |

### 3.2 同步点计数

- `【已实现事实】` **提交内部 0 个屏障**：`stfs_meta_txn_commit` 只在 `sync != 0` 时调
  `stfs_meta_volume_flush`（`txn.c:stfs_meta_txn_commit()`）；`cow.c`、`ledger.c` 的提交路径内部
  无 flush/barrier（全域检索：pkg 50 的 `.c` 中零调用）。
- `【理论推断】` ⇒ 量级结论要分两种口径：
  (a) **`sync=0`（默认口径）**：吞吐**不受屏障限制**，但持久性由调用方承担
  ⇒ 「快但不保证」；
  (b) **每事务 `sync=1`**：每笔元数据事务引入 1 个设备屏障 ⇒ 吞吐被
  `1 / (屏障延迟 × K)` 压住，与主流日志型 FS 的 fsync 口径同构。
  **当前实现的默认（a）不能与主流 FS 的默认（b）作同等比较**，这是量级比较的前提。

### 3.3 量级推导（K × 4 KiB 随机 IO）

- 设单盘随机 IOPS 量级为 **R**：HDD ≈ 10² ，SATA SSD ≈ 10⁴–10⁵，NVMe ≈ 10⁵–10⁶。
- 设一笔元数据操作需 **K** 次随机 IO：由 §3.1，**K ≈ 4–8**（其中 2 次为总账扇区写）。
  `【理论推断】`
- ⇒ 元数据操作吞吐上限 ≈ **R / K**，单操作延迟 ≈ **K / R**：
  - HDD（R≈150）：吞吐 ≈ **19–38 笔/秒**，延迟 ≈ **27–53 ms/笔**；
  - SATA SSD（R≈5×10⁴）：吞吐 ≈ **0.6–1.3 万笔/秒**，延迟 ≈ **0.08–0.16 ms/笔**。
  `【理论推断】`
- 数据吞吐上限：每 N 字节写入 ≈ N 字节数据 + 元数据对象（COW 重建）
  ⇒ 顺序大写在 (a) 口径下受**介质带宽**限制而非 IOPS 限制，量级与主流 FS 同档；
  小随机写则退化为 `R / K`。 `【理论推断】`

### 3.4 并发与缓存

- `【已实现事实】` **全树没有任何互斥原语**：`pthread_mutex`/自旋锁/`InterlockedCompareExchange`
  在全树 `.c` 中**零命中**；唯一的原子操作在 `80-翻译层与热缓存\ring.c:TL_STATE_*`
  （`__atomic_store_n(..., RELEASE)` / `__atomic_load_n(..., ACQUIRE)`），
  另有 `verify/carriers/wm_probe.c`（测试载体）。
  ⇒ **一致性模型是单线程提交 + 一个无锁单写者环**；**没有多核扩展性**，
  并发度 = 1（除非上层包自行串行化）。 `【已实现事实】`
- `【缺证据】` **批处理 / 预取 / 回写**：pkg 80（翻译层与热缓存）的预取与回写策略
  本次未取得可指认证据；`ring.c` 的环容量与淘汰策略亦未评估。
- **本项等级：理论 L2**。复杂度设计（O(深度) 定位、按 extent 回收、常数次挂载）
  是好的一档；但**无批处理、无屏障默认、无并发**压制了实际可达水平。

### 3.5 与主流文件系统的量级差

- `【理论推断】` 在**同为每事务同步**的口径下：主流日志型 FS（ext4 data=ordered / XFS）
  一笔元数据改动通常 1–2 次日志写 + 1 次屏障；StFS 的 COW 重建 + 总账提交为
  **K ≈ 4–8** ⇒ 比主流**慢数倍（同一数量级内）**，**不构成数量级差距**。
- `【理论推断】` 但在**无屏障默认**下 StFS「看起来更快」，这是**口径不等价**，
  不构成性能优势的结论。
- `【理论推断】` 真正的数量级风险点有两个：**① 日志逐条追加 = 每 15 B 记录
  一次整簇读改写（写放大 ≈ 4096/15 ≈ 273×）**（B9 `e3afefa` 前为 13 B / ≈315×）；**② 每文件日志区 ≥1 头簇 + ≥1 隔离簇
  （4 KiB 簇下 ≥8 KiB）**，小文件密集时空间开销按**簇**而非按字节增长（见 §7.3）。

### 3.6 两大性能瓶颈（结论）

1. **COW 元数据重建无批处理**：每笔元数据事务重建对象（`cow.c:stfs_meta_edit_commit()`），
   且提交链内无写合并/回写聚合 ⇒ 写放大由 K 决定，K 只靠 O(1) 查找与按 extent
   回收部分抵消。
2. **日志逐条读改写头簇**：`journal.h:stfs_journal_entry_append()` 明确「追加一条 = 1 次头读 + 1 次头写」
   ⇒ 记录数 M 的代价是 O(M) 次簇 IO，且每条 15 B 却整簇搬 4 KiB（B9 `e3afefa` 前为 13 B）。
   **这对人类提议的「每更新追加一条 CRC」方案是决定性成本**（见 §7.4）。

**性能总体等级：理论 L2**（设计复杂度好、同步与批处理弱、无并发）。

---

## 4. 结论一句话

StFS 当前是一个**「检测与诚实失败做得比修复好」的文件系统**：元数据与数据簇的 CRC
覆盖、双总账交替提交、非法输入拒绝与只读降级都已在代码与测试里坐实（容灾 L2，
其中诚实失败/校验覆盖/拒绝语义达 L3），但**盘内几乎没有介质冗余**（除总账双副本与
GPT 镜像外）、**提交链内部无屏障**、**无在线 scrub/自愈**，性能上复杂度设计属良好一档
而**无批处理、无屏障默认、无并发**（理论 L2），与主流 FS 为**同一数量级内的数倍差距**
而非数量级差距。

---

## 5. 不确定性清单（全部 `【缺证据】`）

1. **无实测**：本环境未做真实断电/掉电/坏道试验，亦无基准数据 ⇒ §3 全部为理论推导。
2. **无屏障宿主的真实写序**：`cow.c` C→D、`ledger.c` ext→v1 的顺序在无 barrier 宿主上
   是否成立，代码无法保证；交替 ext 槽能覆盖「撕裂」但**不排除「重排」**。
   ⇒ 应用真机 + 断电注入验证。
3. **`stfs_meta_fs_check`（`ledger.c:stfs_meta_fs_check()`）的扫描范围与动作未确认**：是否遍历数据簇、
   命中不一致后是报告还是修复，未能取得可指认证据。
4. **pkg 60/90/A0/B0 未评估**：快照/子卷读回、误删恢复的用户可见路径、TRIM/discard
   对接均未证实。
5. **pkg 80 的预取/回写策略未确认**；`ring.c` 环容量与淘汰策略未评估。
6. **已知缺陷**：非法设备名传入 `volume_open` **会崩溃**（`C5-写路径计划.md:80,114,145`），
   与「非法输入必须被拒」冲突。
7. **B8 在飞**：本文基准为 `a955755`；若 B8 后续步骤（`.tools/widen-step*.py`）继续落地，
   **容量上限与「每操作字节数」的结论会变**（文件头 40 B、条目 12 B 等已定，
   但计数域是否一并放宽会直接改变单文件/单卷上限）。
8. **pkg 40 不在宿主分配通道校验 `struct_size`**；`led_parse_v1` 对 CRC 算两次
   （`ledger.c:led_parse_v1()` 内两次 CRC 比较）。
9. **格式版本策略**：已裁定「永远 = 1，无 V1/V2/V3 兼容」⇒ 任何盘上格式改动都
   **没有版本判别分支可依靠**，只能靠 `struct_size`/自描述字段（如
   `prefix_bytes`、`entry_stride`）区分。这对 §7 的成本评估是关键约束。

---

## 6. 纠错辅助数据：必要性与现状盘点

> 人类设计口径：系统内的容灾**够用即可**；错误恢复与坏道容忍**依托详细日志**，
> 超出磁盘自身 ECC 的部分不追求；**介质冗余交给低层 RAID 卡**；目标是
> **在逻辑层把数据从 RAID「危险状态」中抢救出来**，如同抢救坏道数据。
> 本节回答「是否需要在日志/结构里加『日志所对应文件的纠错辅助数据』」的现状盘点。

### 6.1 现状盘点

**① 哪些结构有 CRC、是否覆盖文件数据块**

- 有 CRC 的结构：主总账 v1 前缀（`ledger.c:led_parse_v1()`）、总账 ext 槽（`ledger.c:led_parse_ext()`）、
  位图头部（`storage.h:stfs_st_bitmap_header_crc()` / `stfs_storage_layout.h:stfs_storage_bitmap_header_t.crc32c`）、位图分片（`stfs_storage_layout.h:stfs_storage_bitmap_shard_t.crc32c`）、
  日志头记录（`stfs_journal_layout.h:STFS_JOURNAL_HEAD_CRC_BYTES`）、日志父条目 1 B `check`
  （`stfs_journal_layout.h:stfs_journal_parent_entry_t.check`，仅 CRC32C 低 8 位）、文件块 v2 头 16 B 自校验
  （`meta.h:STFS_META_FILE_BLOCK_V2_OFF_CRC`）、**每个数据簇的整簇载荷**（`stfs_storage_layout.h:stfs_storage_cluster_head_t`）。
- **文件数据块：已覆盖。** 数据簇 CRC32C 由 pkg 30 在**每次写**时封（`storage.c:stfs_st_cluster_write_headed()`、
  `storage.c:st_write_cluster_span()`、`storage.c:stfs_st_cluster_init()`），**每次读**时校验（`storage.c:st_load_cluster()`），文件层读取默认
  `STFS_VERIFY_CRC32C`（`file.c:file_block_read_hdr()`、`file.c:file_reader_read()`、`file.c:file_map_load()`（定义在后段））。
  `meta.h:stfs_meta_object_load_reported()` 亦声明「数据簇的簇头 CRC 由包 30 校验」。
  **⇒ 关键结论：文件数据块的完整性检测今天已经具备，且不在日志里，而在簇头里。**
  `【已实现事实】`

**② `MIRROR_*` 是独立可校验第二副本，还是缓冲/其它用途**

- **都不是副本，也不是缓冲。** 它是 COW 编辑对象的**种类判别符**：
  `meta_internal.h:STFS_META_MIRROR_*` 定义 `NONE/SUBJECT/SNAPSHOT/SUBVOL/SUBJECT_ROOT/SUBJECT_SHARD`，
  字段为 `uint32_t mirror`（`meta_internal.h:stfs_meta_edit_object_t.mirror`），
  `edit_build_mirror_objects`（`cow.c:edit_build_mirror_objects()`）以 `switch (o->mirror)` 决定
  **本对象承载哪个源对象的落盘像**（根索引 / 脏分片 / 快照表 / 子卷表）。
  `MIRROR_BYTES(4096)` = **主体表落盘像的缓冲区字节数**（大小计算），不是冗余。
  `【已实现事实】`

**③ pkg 50 的日志是 intent log 还是 data log**

- **是 intent log（定位/状态意图），不是 data log。**
  父条目 15 B 定长（B9 `e3afefa` 前为 13 B）：`{child_cluster u64 @+0, length_clusters u32 @+8, state u8 @+12,
  type u8 @+13, check u8 @+14}`（`stfs_journal_layout.h` 的 `stfs_journal_parent_entry_t`）
  ——只记「子文件夹日志区起始簇号 / 大小 / 状态 / 类型 / 1 B 校验」，
  **不含任何被保护对象的载荷副本**。
  头前缀 80 B 记身份与边界（`folder_id`/`parent_folder_id`/`parent_slot`/`generation` 等，
  `stfs_journal_layout.h:stfs_journal_head_prefix_t`），亦无载荷。
  `50/checkpoint.c` 文件头注释第 3 条明确「重放不解释载荷」；`stfs_journal_replay_timed`
  （`checkpoint.c:stfs_journal_replay_timed()`）只做读+校验+计数。
  **⇒ 日志只重复「元数据变更的意图/位置」，不重复数据。** `【已实现事实】`

### 6.2 能力判定：元数据 vs 文件数据

| | 检测能到哪一步 | 能**修复**吗 | 依据 |
|---|---|---|---|
| **元数据** | 可**定位到具体结构**（总账/位图头/位图分片/日志头/表分片各有 CRC 与 magic/struct_size/版本/交叉校验）；可判「撕裂写（半写窗口）」 | **仅总账可修**（主/副 + `ledger.c:stfs_meta_ledger_repair()`）。**主体表/索引表不可修**——它们是单一逻辑结构（`cow.c:edit_table_payload_len()`/`cow.c:edit_build_mirror_objects()`），**没有第二份内容可依据** | `ledger.c:led_ext_crosscheck()`、`ledger.c:stfs_meta_ledger_read_copy()`、`ledger.c:stfs_meta_ledger_repair()`；`cow.c:edit_table_payload_len()`、`cow.c:edit_build_mirror_objects()` |
| **文件数据** | 可**定位到簇（LBA）**：`bad_lba`/`bad_cluster`/`crc_expected`/`crc_actual`（`storage.c:st_load_cluster()`）；由 file.c 的读上下文可归属到文件 | **不可修**。只有「重读 → 仍失败则清零并写回以触发固件重映射」（`storage.c:st_repair_read_error()`），**不重建内容**，且刻意不补写簇头 CRC 以免把 0 当有效数据（`storage.c:st_repair_read_error()`） | `storage.c:st_load_cluster()`、`storage.c:st_repair_read_error()` |

**⇒ 「可检测」≠「可修复」。修复必须有第二份（副本或奇偶/擦除码）。
当前盘上只有两处第二份：总账双副本、GPT 分区表镜像。其余一律不可修。**

### 6.3 缺口与代价

**缺什么**

1. **元数据侧可修复冗余**：主体表（根索引/分片）、日志头、位图分片都只有单份。
   要有「元数据级可修复冗余」，需为其引入**第二份或待重建的辅助数据**。
2. **危险状态抢救行为**：目前有「只读降级」（`stfs_meta_write_allowed`、
   `stfs_journal_open` 的 `STFS_ERR_LEDGER_PRIMARY_CORRUPT` + `consent_degrade`
   `recover.c:stfs_journal_ledger_probe()`），但**没有**：只读抢救挂载（尽最大可能读出、跳过坏块并记录、
   继续扫描）、元数据优先、尽力导出。`stfs_journal_recover_region`
   （`test_bad_cluster_input.c:bi_test_ac50_11_input_side()` 用它做「整片不可读 ⇒ 只丢该片」的标定）
   是最接近的构件，但它是**区域级**诊断，不是抢救导出。
3. **坏块跳过 + 记录 + 继续扫描的导出一级能力**：`st_repair_read_error`
   （`storage.c:st_repair_read_error()`）做了「跳过并继续」，但它**清零后写回**——
   对抢救场景这是**破坏性的**（会把抢救中的原始坏区内容抹成 0）。
   ⇒ 抢救模式需要一条**不改介质**的只读跳过路径。

**代价与是否改盘上格式**

- 加 CRC/辅助数据字段 ⇒ 必然动盘上结构。但**版本永远 = 1、无兼容分支**（已裁定），
  因此可行的路径只有一种：**用自描述字段承载新增** ——
  `stfs_journal_layout.h:stfs_journal_head_prefix_t.prefix_bytes`（注释明说「尾部追加后变大」）、
  `stfs_journal_layout.h:stfs_journal_head_prefix_t.entry_stride`（注释明说「自描述便于将来加宽」）、
  `stfs_journal_layout.h:stfs_journal_head_prefix_t.reserved1/reserved2`（注释明说「尾部追加用」）。
  `【已实现事实】`
- ⇒ `【理论推断】` **在版本 1 内可行**，条件是：新增只通过
  「`prefix_bytes` 变大 / `entry_stride` 变大 / 新 `type` 值」三条路，且**所有消费者
  都必须按字段读而不是按常量读**。风险点：`STFS_JOURNAL_ENTRY_TYPE_*` 目前是
  **三值域且被静态断言钉死**（`stfs_journal_layout.h:STFS_JOURNAL_ENTRY_TYPE_*` 与其同名 `STFS_STATIC_ASSERT`），要加新 type 必须同时扩展断言；
  `type` 字段本身是 `uint8_t`，空间够，但**语义断言需要同步修改**
  （`stfs_journal_layout.h:stfs_journal_entry_type_valid()` 的 `type <= TYPE_PROGRAM` 判断字面表达了三值语义）。
- 若要走「日志侧按更新追加 CRC 记录」这条路，则**必须先解决 §7.2 的轮转/淘汰缺口**
  与 §7.4 的写放大，否则记录数无上限而写入代价 O(M) 次簇 IO。

### 6.4 分层契约（建议措辞）

> **StFS 在逻辑层负责「检测 + 隔离 + 诚实失败 + 抢救」；介质冗余由低层 RAID 承担。**
>
> - **检测**：数据簇与元数据结构的 CRC 由文件系统在每次写时封、每次读时校验；
>   不一致时返回 `STFS_ERR_CORRUPT` 并给出精确的簇/LBA 与期望/实际校验值。
> - **隔离**：坏簇被逐簇裁决（`stfs_st_cluster_is_bad`，`stfs_block_alloc.h`「批 70-F」权属注释、`stfs_block_alloc.h:stfs_blk_bad_query_fn`（R2 坏道判据））
>   并在分配时避开；「查不到」是显式第三态，**不猜**。
> - **诚实失败**：不确定即报不确定（`STFS_ERR_INTEGRITY_UNCERTAIN`，`txn.c:stfs_meta_txn_commit()`），
>   绝不伪造成功；降级必须显式同意（`recover.c:stfs_journal_ledger_probe()`）。
> - **抢救**：在介质危险状态下以只读方式尽量导出数据，元数据优先，
>   坏区跳过并记录，**不写回、不覆盖**。
> - **介质冗余不在文件系统内**：单份介质损坏只能检测、不能修复；修复依赖低层 RAID
>   的副本/奇偶。**唯一例外（如实声明）**：总账主+副双副本（含 `ledger_repair`）
>   与 GPT 分区表镜像 `d0_gpt_verify_mirror`，这两处是文件系统内**已存在**的真实冗余。
>
> **⇒ 因此：若要让文件系统自身具备「从 RAID 危险状态抢救」的完整能力，
> 缺的不是「介质冗余」，而是「只读抢救模式」与「元数据可重建的辅助数据」。
> 介质冗余继续留在 RAID 层，是正确的分层。**

`【已实现事实】`（双副本与 GPT 镜像的存在）＋`【理论推断】`（其余措辞为契约建议，
不是现状描述——现状尚无抢救模式）。

---

## 7. 纠错辅助数据：现状与成本

> 人类定稿口径：每份文件的 CRC 记录**放进「该文件所属的日志」**，**不放文件本身**
> （理由：日志本身可重建 ⇒ CRC 丢失不致命）；**按更新记录**追加（完整性历史/时间线）；
> 配额 = 文件大小 × 5%，超出**从最旧丢弃**；单条记录 > 5%·size ⇒ 保底 2 条；
> 单条记录 ≥ 10%·size ⇒ 该文件不留 CRC。
> **本节只回答「放不放得下、代价多少」，并对口径中唯一未规定的一点（跨文件淘汰顺序）
> 给出代码可证的缺口。**

### 7.1 包 50 日志的记录形态：定长？文件标识？可挂 CRC 的字段？N = ?

- **定长，是。** 父日志条目 `stfs_journal_parent_entry_t` 为 **15 B 定长**（`STFS_JOURNAL_PACKED`，
  `stfs_journal_layout.h:stfs_journal_parent_entry_t`；B9 `e3afefa` 前为 13 B）：`child_cluster u64 @+0`、`length_clusters u32 @+8`、
  `state u8 @+12`、`type u8 @+13`、`check u8 @+14`。
  宽度由头前缀的 `entry_stride` 自描述（`entry_stride /* +28 父日志条目宽度（v1 = 9；
  自描述便于将来加宽） */`，`stfs_journal_layout.h:stfs_journal_head_prefix_t.entry_stride`；B7 宽化后实测为 13，B9 再宽化为 15）。
  ⇒ 与 4 KiB 簇一致性核对：内联容量 263 = `(head_bytes − entry_area_off − CRC尾 4) / 15`
  = `(4088 − 128 − 4)/15`，测试断言 `entry_capacity == 263`（`test_main.c:test_ac50_2_head_self_verify()`），
  溢出簇每簇 272 条，`entry_area_clusters = 2` 时总数 535（`test_main.c:test_extra_boundaries()`）。
  （B9 `e3afefa` 前按 stride 13 为 `(4096 − 128 − 4)/13 = 304`、每簇 314、总数 618。）
  `【已实现事实】`
- **文件标识：条目内没有。** 身份在**区域级**而非记录级——头前缀带
  `folder_id @+8`、`parent_folder_id @+12`、`parent_slot @+16`（`stfs_journal_layout.h:stfs_journal_head_prefix_t`），
  即「哪个日志区 = 哪个文件夹/主体」。父条目只记**子区域起始簇**（`stfs_journal_layout.h:stfs_journal_parent_entry_t.child_cluster`、`INV-10`）。
  ⇒ **若 CRC 记录要能报出「是哪个文件失败」，要么依赖「记录落在该文件的区域里」
  这一区域级归属，要么必须在条目里新增文件标识字段（当前没有）。** `【已实现事实】`
- **可挂 CRC 的字段 / 扩展位**：
  - 条目内**唯一**校验位是 `check`（`stfs_journal_layout.h:stfs_journal_parent_entry_t.check`，B9 前在 `:205`、现行为 `:219`），= `CRC32C(前 14 B)` 的**低 8 位**（B9 `e3afefa` 前为前 12 B）
    ⇒ 只有 **CRC8 强度**，且它校验的是**条目自身**，不是被保护的数据。
    **条目内没有空余字节**（15 B 全部占满；B9 `e3afefa` 前为 13 B）。 ⇒ 想加**完整** CRC 必须
    **加宽 `entry_stride`**（例如 15 → 19，多出 4 B 放 CRC32C；B9 `e3afefa` 前为 13 → 17）。
  - 头前缀有 `flags @+56`（保留位，v1 必须为 0，读取时忽略未知位；`stfs_journal_layout.h:stfs_journal_head_prefix_t.flags`）、
    `reserved1 @+72`、`reserved2 @+76`（`stfs_journal_layout.h:stfs_journal_head_prefix_t.reserved1/reserved2`，注释明说「尾部追加用」）、
    `prefix_bytes @+6`（`stfs_journal_layout.h:stfs_journal_head_prefix_t.prefix_bytes`，注释明说「尾部追加后变大」）。
    ⇒ **头前缀是官方指定的尾部追加扩展点**，可挂「该区域 CRC 记录上限/计数/最旧记录游标」
    一类记账字段，而**不必加宽条目**。
  - `type`（`stfs_journal_layout.h:stfs_journal_parent_entry_t.type`）是 `uint8_t`，空间够，但当前是**三值语义且被静态断言钉死**
    （`STFS_JOURNAL_ENTRY_TYPE_UNSET/NORMAL/PROGRAM`，`stfs_journal_layout.h:STFS_JOURNAL_ENTRY_TYPE_*` 与其同名 `STFS_STATIC_ASSERT`），
    且 `stfs_journal_layout.h:stfs_journal_entry_type_valid()` 用 `type <= TYPE_PROGRAM` 做判断 ⇒ 新增 type **必须同步改断言与判断**。
  `【已实现事实】`
- **追加一条记录的实际字节数 N**：
  - **逻辑占用 N = 15 B/条**（若保持现有 `entry_stride`；B9 `e3afefa` 前为 13 B）；
    若按「完整 CRC32C」加宽 stride，则 **N = 19 B/条**（B9 `e3afefa` 前为 17 B/条）。 `【理论推断】`
  - **物理代价与 N 无关，才是重点**：`journal.h:stfs_journal_entry_append()` 明说
    「追加一条父日志条目（读改写头记录 = **1 次头读 + 1 次头写**；溢出段另加 2 次簇 IO）」。
    ⇒ **追加 1 条 15 B 记录要搬运整个头簇（4 KiB）**，写放大 ≈ `4096/15 ≈ 273×`（B9 `e3afefa` 前为 13 B / ≈315×）；
    M 条记录 ⇒ **O(M) 次簇 IO，无批处理**。`【已实现事实】`

### 7.2 日志区容量与轮转：全局上限？写满如何处理？跨文件淘汰顺序能不能做？

- **没有全局上限，只有「每区域」上限。** 头前缀有 `entry_capacity @+32` /
  `entry_count @+36`（`stfs_journal_layout.h:stfs_journal_head_prefix_t.entry_capacity/entry_count`），且**容量不是自由字段**：必须等于由
  `entry_area_off`/`entry_stride`/`head_bytes` 推出的值，否则判 `STFS_ERR_CORRUPT`
  （`journal_layout.c:jr_prefix_consistent()`「容量必须等于由边界字段推出的值，不许自说自话」）。
  ⇒ 容量在**建区时定死**，全卷没有一份「日志总量」账目。`【已实现事实】`
- **写满 = 拒绝追加，不覆盖、不轮转。** `journal.h:stfs_journal_entry_append()`：
  「**本函数不分配空间**：容量在建区时定死，用满即 `STFS_ERR_JOURNAL_FULL`」；
  测试独立钉住：填满到上限 `STFS_OK`，再追加即 `STFS_ERR_JOURNAL_FULL`
  （`test_main.c:test_extra_boundaries()`，并注明「**不偷偷扩容**：分配不是本包的事」）。
  `【已实现事实】`
- **「轮转」在代码里指『新区域的物理落位分散』，不是日志轮转/淘汰。**
  `journal.h:stfs_journal_plan_alloc()` 分工：包 70 拥有空闲池，包 50 只拥有**落位规则**；
  `journal_alloc.c:stfs_journal_plan_alloc()`「按区域轮转：从 cursor 指定的区域开始，逐个区域试」，
  受 `STFS_JOURNAL_ISOLATION_MIN_CLUSTERS 1`（`stfs_journal_layout.h:STFS_JOURNAL_ISOLATION_MIN_CLUSTERS`）约束。
  `test_alloc_seq.c:sq_test_ac50_4_plan_on_real_pool()`、`test_main.c:test_ac50_4_scatter_and_gap()` 断言的正是**落位轮转**。
  `【已实现事实】`
- 「覆盖最旧」在代码里**只有内存先例**，不在盘上：`checkpoint.c:ck_lookup()`
  固定容量状态表满时 `spare = &g_ck[0]; /* 表满：覆盖最旧的一格（固定容量，绝不分配） */`；
  `recover_shard.c:rs_note_kept()`、`layer.c:stfs_journal_window_commit()` 的「上限」是**如实回报到什么程度**的截断，
  不是淘汰。`【已实现事实】`
- **⇒ 人类口径中唯一未规定的一点（跨文件淘汰顺序）当前做不了，缺三样东西**：
  1. **全局记账缺失**：没有「所有文件 CRC 记录总量」的账目，也没有「某文件占了多少字节」
     的字段（头前缀只有 `entry_count` 条数，没有字节数，也没有「最旧记录游标」）。
  2. **删除/回收缺失**：日志只追加；**没有删除单条记录、也没有回退 `entry_count`
     的接口**（`journal.h` 的条目写面只有 read/append；无 delete/truncate）。
     ⇒ 无法「从最旧丢弃」。
  3. **裁决者缺失**：5% 配额是「按文件自身大小」算的，但**卷级预算谁来管**没有归属
     （包 50 不做空闲池，包 70 不知道 CRC 记录语义）。 ⇒ 需要一个新的
     「日志配额裁决者」，或者把预算改成「区域级」而非「文件字节级」。
  `【缺证据】`（「没有 delete 接口」这一点来自 `journal.h` 条目写面的可见接口集合；
  若存在未在本文件声明的内部删除路径，则本项结论需修正。）

### 7.3 按 N 推算：代表性文件大小各能存几条、日志占用多少

按 `N = 15 B`（B9 `e3afefa` 前为 13 B）、归约式 `rec ≥ 10%·size ⇒ 0；rec > 5%·size ⇒ 2；否则 max(2, floor(0.05·size/N))`：

| 文件大小 | 5% 预算 | floor(预算/15) | 实际保留条数 | 记录字节占用 | 备注 |
|---|---|---|---|---|---|
| 1 KiB | 51 B | 3 | **3** | **45 B** | 预算 > N，按公式 |
| 4 KiB | 205 B | 13 | **13** | **195 B** | |
| 64 KiB | 3277 B | 218 | **218** | **3270 B** | |
| 1 MiB | 52429 B | 3495 | **3495** | **52425 B** | |
| 1 GiB | 53.7 MB | 3579139 | **3579139** | **53.7 MB** | 见下「能不能装下」 |

- 小文件边界（`N = 15`，B9 `e3afefa` 前为 13）：`15 ≥ 10%·size` ⇔ `size ≤ 150 B` ⇒ **不留 CRC**；
  `150 B < size < 300 B` ⇒ **保底 2 条**。⇒ 小于 300 B 的文件规则才生效，
  否则一律走 `floor` 公式。`【理论推断】`

**「所有文件都按 5% 记账」时日志总量会不会炸？**

- **按字节算：不会超过 5%。** 归约式对每个文件的上限就是其自身大小的 5%
  ⇒ 卷级上界 = 数据总量的 5%。这是**构造性上界**，不需要猜。 `【理论推断】`
- **但真正的爆点在两处，都是代码可证的**：
  1. **区域粒度爆点（决定性）**：每份文件的 CRC 记录要放在「该文件所属的日志」里，
     而**每个日志区至少 1 个头簇 + 与相邻日志区 ≥1 簇隔离带**
     （`STFS_JOURNAL_ISOLATION_MIN_CLUSTERS 1`，`stfs_journal_layout.h:STFS_JOURNAL_ISOLATION_MIN_CLUSTERS`；
     `journal.h:stfs_journal_plan_alloc()`；`test_main.c:test_ac50_4_scatter_and_gap()` 断言隔离带）。
     **4 KiB 簇下每个文件最小日志代价 = 2 簇 = 8 KiB。**
     ⇒ 100 万个 200 B 的小文件：字节预算合计仅 50 MB 级，
     但日志区合计 **≥ 8 GiB**。 **5% 的字节预算根本不是约束，簇粒度才是。**
     `【理论推断】`（推导自常量与隔离带断言；区域是否可与他文件共用未取得证据）
  2. **单区域容量爆点（当时的硬上限）**：`length_clusters` 是 **u16**
     （`stfs_journal_layout.h:stfs_journal_parent_entry_t.length_clusters`；B9 `e3afefa` 后为 **u32@+8**，该字段 B9 前在 `:202`、现行为 `:216`）⇒ **单个日志区 ≤ 65535 簇**。
     按现行 stride 15 重算，4 KiB 簇时容量 = `263 + 272×(65534)` ≈ **1.78 × 10⁷ 条**
     （B9 前按 13 B 为 `304 + 314×65534` ≈ 2.06 × 10⁷ 条）；
     ⇒ 1 GiB 文件需要的 3.58 × 10⁶ 条**装得下**（约占区域 256 MiB 中的 53.7 MB）；
     但 **文件大于约 5 GiB 时，其 5% 预算（> 268 MB）就超过单区上限（256 MiB）**，
     **该文件拿不到完整配额**。`【理论推断】`
     （注：头前缀 `log_clusters` 是 u32（`stfs_journal_layout.h:stfs_journal_head_prefix_t.log_clusters`），与父条目的 u16 不一致；
     子树可达区域受 u16 窄侧约束。这一宽窄不一致本身是隐患。）
     > **演进注（2026-10-08）**：B9 `e3afefa` 已将 `length_clusters` 由 u16 宽化为 u32
     > （父条目 13→15 B）⇒ 本条「单区 ≤ 65535 簇」「> 5 GiB 文件拿不到 5% 配额」
     > 「宽窄不一致是隐患」三条结论**已被消除**；原文与 `2.06 × 10⁷ 条` 按**历史口径**保留，
     > 上式 `1.78 × 10⁷ 条` 是现行 stride 15 下的对照重算，现行上界为 u32（`≤ 4294967295` 簇）。
- `【缺证据】` 单个日志区在**卷几何上**能否长到几十 MiB / 256 MiB，
  取决于 pkg 50 落位规则与卷几何的联合限制
  （`journal.h:stfs_journal_plan_alloc()` 的边界校验，本次未逐条核算）。

### 7.4 落地成本与判据

**需改哪些文件**

| 文件 | 改动 |
|---|---|
| `50-日志体系与恢复\stfs_journal_layout.h` | 新增记录种类：加宽 `entry_stride`（15→19；B9 `e3afefa` 前为 13）或新增条目结构；扩展 `STFS_JOURNAL_ENTRY_TYPE_*` 三值语义与静态断言（`stfs_journal_layout.h:STFS_JOURNAL_ENTRY_TYPE_*`、`stfs_journal_layout.h:STFS_STATIC_ASSERT`、`stfs_journal_layout.h:stfs_journal_entry_type_valid()`）；若用扩展位则在头前缀尾部追加记账字段（`stfs_journal_layout.h:stfs_journal_head_prefix_t.reserved1` / `.reserved2`） |
| `50-日志体系与恢复\journal_layout.c` | append/verify/parse 适配可变 stride；头记录 CRC 重算路径（`journal_layout.c:stfs_journal_head_patch_layer()`）；容量推导（`journal_layout.c:jr_prefix_consistent()`、`journal_layout.c:jr_derived_capacity()`、`journal_layout.c:stfs_journal_head_init()`） |
| `50-日志体系与恢复\journal.h` / `journal.c` | **新增删除/截断/丢弃最旧记录**的接口与实现（当前不存在）；`stfs_journal_entry_append` 的写放大治理（批处理或专用追加区） |
| `50-日志体系与恢复\checkpoint.c` | 重放**有意不解释载荷**（`checkpoint.c` 文件头注释第 3 条）⇒ CRC 记录对重放不可见；若要求「从日志重建元数据」须在此教会重放理解新记录 |
| 配额裁决者（新增职责） | 全局/跨文件淘汰顺序与账目 —— **当前无归属** |
| `50-日志体系与恢复\test\test_main.c` | 硬编码的内联容量/总容量断言随 stride 变化必须改：**现为 263 / 535**（`test_main.c:test_ac50_2_head_self_verify()` 的 `stfs_journal_head_prefix_t.entry_capacity == 263` 断言、`test_main.c:test_extra_boundaries()` 的 `stfs_journal_inline_capacity()`（=263）与 `stfs_journal_head_prefix_t.entry_capacity == 535` 断言）；B9 `e3afefa` 前为 304 / 618（B9 前旧行号 `:723`、`:1024`、`:1058`，不可对应现行文件） |

**是否改盘上格式（版本仍 = 1）**

- **可以不改版本号，但确实改盘上布局。** 依据：`stfs_journal_layout.h:stfs_journal_head_prefix_t.prefix_bytes` 与
  `stfs_journal_layout.h:stfs_journal_head_prefix_t.entry_stride` 都是自描述字段，且 `stfs_journal_layout.h:stfs_journal_head_prefix_t` 的布局冻结注释明写「布局一经冻结只能靠
  `format_version` + **尾部追加**（`prefix_bytes` 自描述增长）」；
  `stfs_journal_layout.h:stfs_journal_head_prefix_t.reserved1/reserved2` 注释即「尾部追加用」。
  ⇒ 新增只走「扩 `prefix_bytes` / 扩 `entry_stride` / 新 `type`」三条路时，
  **版本保持 1 是可行的**。`【理论推断】`
- **代价与风险**：① 静态断言与 `type <= TYPE_PROGRAM` 判断必须同步改
  （`stfs_journal_layout.h:stfs_journal_entry_type_valid()`、`STFS_JOURNAL_ENTRY_TYPE_*` 静态断言），否则「三值语义」被破坏；
  ② 所有消费者必须**按字段读**而非按 13/9 常量读 —— 而代码里已存在常量硬编码
  （`test_main.c:test_ac50_9_ledger_degrade_reported()` 的注释文字曾是 9 B 时代的「439」，与实际 304 不符；B9 `e3afefa`
  宽化后该处注释与断言已同步为 263 / 535，见 `test_main.c:test_ac50_2_head_self_verify()`、`test_main.c:test_extra_boundaries()`），
  说明**常量漂移已经发生过一次**，这是可证的风险信号。`【已实现事实】`

**三条判据的当前状态**

| 判据 | 当前状态 | 依据 |
|---|---|---|
| ① 翻转数据块内 1 字节 ⇒ 能**定位**到块/文件 | **今天已满足**（无需日志 CRC 记录）。翻转任一数据簇载荷 ⇒ 簇头 CRC32C 失配 ⇒ 返回 `bad_lba`/`bad_cluster`/`crc_expected`/`crc_actual` + `STFS_ERR_CORRUPT`，并计入 `crc_failures`；文件层读默认校验 | `storage.c:st_load_cluster()`、`storage.c:stfs_st_cluster_write_headed()`；`file.c:file_block_read_hdr()/file_reader_read()/file_map_load()`；`stfs_storage_layout.h:stfs_storage_cluster_head_t` |
| ② 文件级 CRC 能报出**是哪个文件**失败 | **不具备**。今天只能给到**簇/LBA 级**；归属到文件靠调用方上下文（file.c 的读路径），**盘上没有持久的文件级完整性记录**。⇒ 正是日志 CRC 记录方案的价值所在 | §7.1「条目内没有文件标识」；`storage.c:st_load_cluster()` |
| ③ **从日志重建元数据**可复现（有无现成测试/探针） | **不具备，且是硬缺口。** 无重放实现：`recover.c` 文件头注释（「本批不做」三条）明确列出**未做**「日志重放到最终状态」（理由：需 70/90 的载荷语义）、「碎片区备份降级」「检查点执行」；`checkpoint.c` 文件头注释第 3 条「重放不解释载荷」；`checkpoint.c:stfs_journal_replay_timed()` 只读+校验+计数；`FREEZE.md` 的 `AC-50.10` 行为 **未具备**、`FREEZE.md`「本批不做」表「**恢复重放**」行记恢复重放未做。⇒ **既无测试也无探针可复现「从日志重建元数据」**；缺的是重放语义（谁解释载荷）与 70/90 的载荷契约 | `recover.c` 文件头注释；`checkpoint.c` 文件头注释第 3 条、`checkpoint.c:stfs_journal_replay_timed()`；`FREEZE.md` 的 `AC-50.10` 行、`FREEZE.md`「本批不做」表「**恢复重放**」行 |

**⇒ 本节结论**：
(a) 「检测」不需要日志 CRC 记录 —— 数据簇检测今天已经具备（判据①）；
(b) 「文件级归属」与「元数据可重建」才是日志 CRC 记录的真实价值（判据②③），
    但当前**日志是 intent log、无文件标识、条目 15 B 无空位（B9 `e3afefa` 前为 13 B）、无删除/淘汰接口、
    无全局配额裁决者**，且判据③连**重放**都还没实现。
(c) 因此**顺序应为**：先补「日志重放 + 载荷语义」（判据③的地基），
    再加「文件级 CRC 记录 + 跨文件淘汰 + 配额裁决」；
    否则加了记录也无从重建，只会把 §7.4 的写放大（O(M) 次簇 IO）立刻暴露。

### 7.5 块级 CRC 落点：簇头内 +8 B vs 元数据侧表 —— 成本对比与建议

**选项 A：块头内扩 +8 B（簇头 8 B → 16 B）**

- 空间：**+8 B / 簇 = 4 KiB 簇的 0.195%**；64 KiB 簇为 0.012%。
  1 GiB 文件（4 KiB 簇，262144 簇）⇒ **+2 MiB**。`【理论推断】`
- 现状契合度：**高**。簇头已是整簇载荷 CRC 的落点
  （`stfs_storage_layout.h:stfs_storage_cluster_head_t`，`sizeof == 8` 有 `stfs_storage_layout.h:STFS_STATIC_ASSERT` 锁定）；
  写时封（`storage.c:stfs_st_cluster_write_headed()`）、读时校（`storage.c:st_load_cluster()`）两条路径都已存在，扩字段是**同一条路径加宽**。
- 代价：**必须改盘上布局**；`STFS_STORAGE_CLUSTER_HEAD_BYTES` 与
  `offsetof(flags) == 4` 等静态断言（`stfs_storage_layout.h:STFS_STATIC_ASSERT`）随之改；
  所有簇头消费者受影响。但**不引入第二份需要保持一致的元数据结构**，
  符合「一条规则只有一个实现点」（`10-公共约定` §14.4 的口径，
  `stfs_journal_layout.h:STFS_JOURNAL_INLINE_CAPACITY`（锚点：说明 `STFS_JOURNAL_INLINE_CAPACITY` 的 §4 注释；**非 §14.4**）明确引用）。
- 注意：现有 CRC 覆盖 `[8, cluster_size)`，**簇头自身（含 `flags`）不被覆盖**
  ⇒ 扩到 16 B 时应明确新 CRC 是否覆盖前 12 B（否则头字段本身仍不可校验）。
  （本行讨论的是**存储簇头** `STFS_STORAGE_CLUSTER_HEAD_BYTES` 8 → 16 B 的扩字段方案，
  与 B9 `e3afefa` 的**日志父条目** 13 → 15 B 无关 ⇒ 现行口径仍为 16 B / 前 12 B，
  B9 未改簇头宽度，`sizeof(簇头) == 8` 的静态断言仍在。）
  `【已实现事实】`

**选项 B：元数据侧表（每簇一条 CRC 记录）**

- 空间：4 B/簇 ⇒ 1 GiB 文件 = 262144 × 4 = **1 MiB**，
  与选项 A 的 2 MiB **同量级**（若只存 CRC 不存偏移）。`【理论推断】`
- 代价：**要新建并维护一张覆盖全部簇的表** ⇒ 需要自己的 COW/原子提交路径、
  自己的空间分配、自己的「表与数据一致性」校验；与 §2.4 的结论叠加后，
  **这张表本身会成为新的单点**（它只有单份，不可修复）。
  且它**与 pkg 50 不碰 pkg 30 簇头的既有边界（`stfs_journal_layout.h` 文件头注释、`INV-4`）
  相冲突**——需要跨包改写契约。`【已实现事实】`
- 优点：**不动簇头**，对已有盘上几何零影响。

**建议结论**（`【理论推断】`，须人类裁定）：

1. **块级校验不要另起一张元数据侧表。** 现有簇头 CRC 已在正确的层
   （pkg 30 写时封、读时校），另起一张表会造出一个新的单点、新的跨包契约，
   且空间收益仅 ~2×。⇒ **若确需更强块级校验，应在簇头内加宽（选项 A）**，
   并明确新区段的 CRC 覆盖范围。
2. **人类口径的「文件级 CRC 历史」与块级 CRC 是两件事，不要混：**
   块级 CRC 解决「这簇坏了没」（已解决）；
   文件级 CRC 历史解决「**是哪个文件**坏了」与「**能否重建元数据**」（未解决）。
   前者不该进日志，后者才是日志 CRC 记录的目标。
3. **落地顺序（回到 §7.4 结论）**：判据③（日志重放 + 载荷语义）是前置，
   否则文件级 CRC 记录既无法重建、又立刻带来 O(M) 次簇 IO 的写放大。

---

## 8. 4Kn 与介质矩阵实测（2026-10-08 追加）

> **本节为追加内容**：§1–§7 一字未改。口径来源：本批「4Kn / 介质矩阵」实测（QEMU 矩阵 + 我们的 FS 真跑），
> 结果口径由 Lead 提供；本节只登记结果，不改写任何既有结论。

### 8.1 QEMU 8 格矩阵：8/8 全 PASS

- 矩阵 = **512e / 4Kn × HDD / SSD × 有 / 无 TRIM = 8 格** ⇒ **8/8 全 PASS**。`【已实现事实】`
- **我们的 FS 真跑**（在上述矩阵盘上真建卷、真挂载）：**MBR 8/8**、**GPT 8/8**，退出码 **`rc = 0`**。`【已实现事实】`
- 范围界定：本节补的是**介质侧格子**的实测。§5 第 4 条与 §2.6 关于 FS 侧
  `reclaim`/`discard` 与 TRIM 对接的结论**仍按原样保留**，本节不改变它们。`【缺证据】`

### 8.2 修前缺陷（如实记录）：4Kn 下表头与保护性 MBR 落在同一设备块

- **修前（缺陷）**：**4/4 个 4Kn 格**的 `EFI PART`（GPT 表头）落在 **`[512, size − 512]`** 区间
  ⇒ 表头与保护性 MBR 落在**同一个设备块内部**（4Kn 的设备块 = 4096 B，512 B 偏移仍属第 0 块）✗
  ⇒ 该设备块一旦写坏，**MBR 与 GPT 表头同损**，块级原子性被破坏。`【已实现事实】`
- **修后**：主表头落 **设备 LBA1**、备份表头落 **末设备 LBA** ✓。`【已实现事实】`
- **回归证据**：**512e 格的盘上字节一字不变**（修前 / 修后逐字节相同）✓
  ⇒ 该修复**只改 4Kn 的落点**，不动 512e 的任何字节。`【已实现事实】`

### 8.3 两份口径更正（以实测为准）

- **`nvme` 设备**：**只能出 4Kn**（给 512 / 4096 仍报 `512/512`）⇒ **512e 只能由 `scsi-hd` /
  `virtio-blk` 提供**。`【已实现事实】`
- **`rotation_rate`**：语义是 **RPM**（`1` = SSD 标记、`7200` = HDD、缺省 `0` = 旋转盘）。`【已实现事实】`

### 8.4 结论：整盘型方案 B 在盘 0 不可行

- 我们的分区表要求 **头部 34 个扇区（LBA0..33）＋ 尾部 33 个扇区**的保护带；
  4Kn 上主表头必须在 **LBA1（设备块 1）**、备份头必须在**末设备 LBA**（§8.2 修后落点）。
- ⇒ **整盘型方案 B 在盘 0 不可行**（保护性 MBR / 表头与首个数据区在 4Kn 下无法分开落块，
  §8.2 的修前缺陷即该问题的实测形态）；**方案 C 未批**（无裁定，本节不为其背书）。`【理论推断】`

---

**（本报告在只读约束下完成：未构建、未跑闸门、未修改任何源码、未提交。）**