# 开工简报 · B0 EFI 只读驱动（**新流**，环节3 第一包）

> **这份简报是自足的**：读完 §2 的"最小阅读集"即可开工，**不必**读设计稿全文或其他包。
> 与 `50` 简报同构；协同细则只在实现仓 `nxfs-impl/COORDINATION.md`，本简报只给摘要与指针。
>
> **为什么现在开 `B0`**：虚拟机测试的两级台阶已落地——L1（宿主侧真实镜像端到端 **29/0**）
> 与 **L2-pre（真固件前置冒烟 15/0）**。L2-pre 已把"**包 20 那段链**"（主总账 → 引导文件表
> → 内核文件）在**真 OVMF × 真 `EFI_BLOCK_IO` × 真镜像**下钉死，于是**完整 L2 的唯一剩余
> 阻塞就是 `B0` 自己**：`EFI_SIMPLE_FILE_SYSTEM_PROTOCOL` 实现 + 三阶段状态机。

---

## 1. 你是谁、现在到哪一步

| 项 | 值 |
| :--- | :--- |
| 你的包 | `B0-EFI只读驱动`（规范：`NXFS开发包/B0-EFI只读驱动/开发包.md`） |
| 落盘位置 | 实现仓 `nxfs-impl/B0-EFI只读驱动/`（**先建目录**，你自己的交付面） |
| 环节 | **一期 · 环节3**（集成与优化）的第一包；`80`/`A0`/`90` 未开工 |
| 前置现状 | `20`/`30`/`40`/`50` 已冻结；L1、L2-pre 已跑通（见下） |
| 你要交付的第一件事 | **在 QEMU+OVMF 下作为真 EFI 驱动挂上 NXFS 卷，并读出内核文件** |

**已经是既成事实的三件事（拿来就用，不要重造）**：

| # | 事实 | 位置 |
| :-: | :--- | :--- |
| 1 | **真固件里可跑**：`EFI_BLOCK_IO` → 包 20 包装 → `ledger_parse` / `boot_table_parse` / `kernel_read` 已在 OVMF 下跑通（15 项 / 0 失败） | `nxfs-impl/tools/vm/uefi_probe.c`、`verify/vm_smoke.py`、`tools/vm/README.md` |
| 2 | **镜像可造**：`nxfs_image format` 能产出真 NXFS 镜像（含引导表与内核文件，CRC 正确） | `nxfs-impl/tools/nxfs_image`、`tools/README.md` |
| 3 | **工具链已就位**：QEMU 11.1.0 + OVMF + zig 0.16.0，装在 `F:\Workspace\.tools\`（**不落 C 盘**） | `verify/nxfs_pkg.py` 的 `find_qemu`/`find_ovmf`（环境变量 `QEMU`/`OVMF_CODE`/`OVMF_VARS` 可覆盖） |

---

## 2. 最小阅读集

**必读**（4 份，读完即够开工）：

1. `NXFS开发包/B0-EFI只读驱动/开发包.md` —— 你的全部规范（AC-B0.1~13、三阶段、INV-6/INV-7、常见坑）。
2. `NXFS开发包/10-公共约定.md` **§12**（EFI 相关）与 **§13/§14**（API 冻结、跨包依赖权属）。
3. `nxfs-impl/tools/vm/uefi_probe.c` + `tools/vm/README.md` —— **L2-pre 探针**：它是你驱动里
   "读链"那一段的**已验证最小实现**（照抄其调用顺序与判据形态，别重新摸索）。
4. `nxfs-impl/COORDINATION.md` —— §1 分层顺序、§2 权属、§4 当前批次、§5 纪律、§7 记行。

**不要读**：设计稿全文、其他包的 `开发包.md`、`verify/joint/` 源码。

---

## 3. 第一批的交付面（建议范围）

**做**：`EFI_SIMPLE_FILE_SYSTEM_PROTOCOL` 的**只读**实现，映射到包 20：

| 协议函数 | 映射 | 注意 |
| :--- | :--- | :--- |
| `OpenVolume()` | `ledger_parse` → 根日志区偏移 | 认卷判据 = 总账魔数/版本，**不要**靠分区类型字节 |
| `Open()` | 只读路径解析（深度 ≤ `NXFS_SAFEPATH_MAX_DEPTH`） | 用 `entry.role`；`UNUSED` 条目不可读 |
| `Read()` | `kernel_read` / `sector_read` | **不截断**：缓冲不足报错（`20` 已如此） |
| `GetInfo()` | 总账容量信息 | 只读，零分配 |

**对应 AC（第一批全做）**：`AC-B0.1`（OVMF 下可枚举并读出文件）、`AC-B0.2`（总账→引导表→内核文件
完整走通，**读到正确内核字节**）、`AC-B0.3`（**篡改主总账 ⇒ 降级且 `from_secondary == 1`**）、
`AC-B0.4`（引导路径**不调用**任何用户态恢复流程——调用图检查）、`AC-B0.5`（全程无动态分配）、
`AC-B0.6`（无 `assert`，所有分支有错误返回）、`AC-B0.7`（深度超限返回 `SAFEPATH_DEPTH_EXCEEDED`）、
`AC-B0.13`（新代码量显著小于 1300 行）。

**第一批明确不做**（不是遗漏，是排期）：

| AC | 为什么先不做 |
| :--- | :--- |
| `AC-B0.8`~`AC-B0.11`（三阶段交接 / 世代递增 / 缓存作废 / 数据盘场景） | 需要**阶段二/三的对手方**（`80` 内核侧、`A0` 用户态），它们未开工。**先把阶段一做成**，交接面留接口不实现 |
| `AC-B0.12`（整理后内核文件 LBA 不变） | 依赖 `70` 的 `defrag` 与 `allow_kernel_file_move` 强制为 0，**`70` 未开工** ⇒ 本批只做"**把不变量写进注释与接口约定**"，验收待 `70`（**不许假装验过**） |

---

## 4. 边界（最容易撞车的四处）

1. **不要重写包 20 的解析**。总账/引导表/降级判据的唯一实现点是 `20`（`safepath.c` +
   `nxfs_disk_layout.h` 的 `static inline`）。你只**调用**：经 `nxfs_sp_get_safepath_v1()` 拿 vtable。
   重写会被 `verify/check_deps.py` 的 F 段判红（规则重复是机器强制的）。
2. **不要自抄 UEFI 结构体**。用 `20/safepath_uefi_types.h` 里的 `nxfs_efi_block_io_t` /
   `nxfs_efi_block_io_media_t`。**这条有血泪教训**：该 media 结构体的 `last_block` 曾被写成
   `u32`（偏移 20），而规范是 `EFI_LBA`＝`u64`（偏移 **24**）⇒ 真固件里容量恒读成 1 块 ⇒
   `kernel_read` 判 `NXFS_ERR_INVALID_ARG` ⇒ **固件读不到内核文件**；且它**只在真固件可见**，
   因为当时的断言与假 media 同源（"断言没有牙"）。详见 `tools/vm/README.md` §4 与
   包 20 `FREEZE.md` §0.1 末两行。
3. **不改别包文件**。需要 `20`/`30`/`40`/`50` 改动（例如把包装里那个 4 KB 栈缓冲改为静态）⇒
   **报统筹**走 §5.6 代改留痕 + 追认流程，**不要**自己动手。
4. **阶段一必须零恢复依赖**（`§12.4` 硬约束）：不得调用重放/检查点/跨文件区。
   只允许"固定偏移 + 固定结构体 + CRC 校验 + 副总账降级"。

**`B0` 会撞上的两个链接符号（L2-pre 已实测踩到，先记下）**：

| 符号 | 原因 | 处理 |
| :--- | :--- | :--- |
| `__chkstk` | 包 20 的 `nxfs_sp_uefi_io()` 有 `uint8_t blk[4096]` 栈缓冲 ⇒ x64 ABI 下帧 > 4 KB 会引用它 | EDK II 链接 `BaseLib`（自带实现）即可；或用 zig 直编时像探针那样给空实现 |
| `_fltused` | MSVC ABI 的"本模块用过浮点"标记 | 同上；探针里的写法可直接照搬 |

> 包 20 的阶段 4 **只编译不链接**，所以它自己的验收看不见这两条。**你必须真链接一次**。

---

## 5. 共享设施与门禁（**直接复用，勿新建**）

| 设施 | 用途 | 归属 |
| :--- | :--- | :--- |
| `verify/nxfs_pkg.py` | `PackageSpec` 描述 + 六阶段构建/验收框架（`build.py` 的公共库） | 统筹 |
| `verify/check_deps.py` | 跨包权属封口 A~F（源/`link_only`/去重/合并链接冒烟/头副本/规则重写） | 统筹 |
| `verify/vm_smoke.py` + `tools/vm/` | **真固件冒烟**（你的 AC-B0.1~B0.3 的现成判据形态与脚本骨架） | 统筹 |
| `tools/nxfs_image` | 造真镜像（`format` / `info` / `verify`） | 统筹 |
| `verify/fixtures/` | 共享夹具（**不要**自建第二份） | 统筹 |

```powershell
# 每批必跑（**串行**，并发构建必出假红：WinError 5/145、zig UnableToWriteArchive）
python nxfs-impl\B0-EFI只读驱动\build.py            # 你自己的六阶段
python nxfs-impl\verify\check_deps.py --link-smoke   # 权属封口（含自检）
python nxfs-impl\verify\run_all.py                   # 全仓聚合（前置封口 + 逐包 + 合练 + L1）
python nxfs-impl\verify\vm_smoke.py                  # 真固件（你的 AC 的最终判据）
```

> **口令**：`B0` 的包描述（`build.py` 的 `PackageSpec`）里 `headers` 必须含
> `safepath.h`、`nxfs_disk_layout.h`、`safepath_uefi_types.h`；`link_only` 放包 `20` 的实现源
> （参考 `50/build.py` 的写法）。**别把包 20 的源塞进 `sources`**——那是把依赖说成自有，
> A 段会红。

---

## 6. 协同规则（摘要；细则在 `COORDINATION.md`）

1. **一包一主**：`B0-EFI只读驱动/**` 的唯一修改者是你。共享设施（`verify/**`、`tools/**`、
   `COORDINATION.md`）的唯一修改者是统筹。
2. **记行**：任何批次落地后在 `COORDINATION.md` §7 追加一行（动机 / 触及文件 / 影响面 / 门禁结果）。
3. **冻结纪律**：交付面完成后写 `B0-EFI只读驱动/FREEZE.md`，**§0 冻结次序必须是第一节**；
   **不写"当前指针"哈希**（以父仓 gitlink 为准）；后续冻结**只追加**。
4. **真话三条**：① 一个不会失败的自检等于没有自检（**必须有牙的负对照**）；
   ② 要求了却没做，不许静默跳过；③ 不写"行为零变化"除非能证明（数字未变 **且** 语义未变）。
5. **推送**：默认 `git push` 在本机会失败（schannel/凭据助手）；用 `COORDINATION.md` §6 的
   显式 URL + `openssl` + `credential.helper=` 形式，并用 `ls-remote` 核对远端。

---

## 7. 需要上报上级（统筹 / API 流）的情形——**不要自行决定**

| 情形 | 找谁 |
| :--- | :--- |
| 需要改 `20`/`30`/`40`/`50` 的任何文件（含那个 4 KB 栈缓冲） | **统筹**（走 §5.6 代改留痕 + 追认） |
| 需要 `nxfs.h` 新增/修改结构体或函数（阶段交接 API 可能要） | **API 流**（ABI 冻结中；MINOR 追加可谈，MAJOR 不可） |
| 需要写整盘分区表 / "按偏移打开"设备 | **统筹**（属 **L3/实机** 前置，刻意缓做） |
| AC 无法在本批验收（如 `AC-B0.12` 依赖未开工的 `70`） | **统筹**（记入"未具备"清单，**不许假装验过**，也不许静默跳过） |
| 发现别的包有缺陷（像 L2-pre 抓到 20 的 media 布局那样） | **统筹**（由归属方改；你只出证据与最小复现） |

---

## 8. 你的"第一批完成"长什么样

**判据（缺一不可）**：

1. `nxfs-impl/B0-EFI只读驱动/` 下有：实现源、`build.py`（六阶段）、`test/` 或验收脚本、`README.md`、`FREEZE.md`。
2. `python nxfs-impl\verify\vm_smoke.py` 或你自己的固件验收脚本，在 **QEMU+OVMF** 下：
   - ✅ 固件加载你的驱动 → **枚举/读出内核文件**（`AC-B0.1`）；
   - ✅ 读到**正确内核字节**（与引导表 `crc32c` 一致，`AC-B0.2`）；
   - ✅ **负对照有牙**：篡改主总账一字节后重跑 ⇒ 必须看到 `from_secondary == 1` **且链路仍通**（`AC-B0.3`）；
   - ✅ 标记式判据（`OK` / `FAIL <原因>`），**QEMU 退出码不参与判定**（固件自己会退出，码不可靠）。
3. `check_deps.py --link-smoke`、`run_all.py` 全绿；`COORDINATION.md` §7 有你的记行；
   包内 `FREEZE.md` §0 冻结次序齐全（`AC-B0.8`~`B0.12` 明确记为"未具备 + 依赖谁"）。
4. 上游 CI（Linux runner）在你推送后 success。

> **一句话**：第一批不是"写完一个驱动"，而是**在真固件里把内核文件读出来，并且证明降级路径
> 也真的会响**——`B0` 之后再无"包 20 那段链"的不确定性，剩下的只是阶段二/三的交接。
