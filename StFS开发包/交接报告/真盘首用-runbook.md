> # ⚠️ 机械盘版硬闸（2026-09-28 追加，**优先于下文所有 INTEL/盘 0 的说法**）
>
> 目标已改为**新插入的测试机械盘**。**下文凡出现「只允许盘 0 / Model 含 INTEL / 119.24GiB」一律作废**（那是旧目标 INTEL SSD 的口径）。
> 改为**盘无关**三道闸：
> 1. --disk N（N 由操作者**显式**给出，禁止自动挑盘）；
> 2. --expect-serial <SN>：工具须打印该盘 Model/Serial/SizeGB/Parts/IsBoot/IsSystem 并与 <SN> 比对，不一致 ⇒ 中止（退出码 4）；
> 3. **IsBoot==true 或 IsSystem==true 一律拒**（保护宿主 C:）；扇区 0 魔数 StFS-DISK0-PROBE **读回逐字节比对**，不 MATCH ⇒ 中止。
>
> 识别流程（**上盘后**）：Get-Disk 重新采集 → 与《上盘前盘位基线》**做差**找新增盘号 → 把 型号/序列号/容量/分区数 报操作者确认 → 才允许 --disk N --expect-serial <SN>。
> 若做差落在基线已有盘号 ⇒ **停止**（识别失败，不得写入）。

# 真盘首用 runbook —— StFS 首次落**新插入的测试机械盘**（真设备）

> 适用：把 StFS 卷首次写进**真物理盘**——本次目标是**用户新插入的一块 15 年老机械硬盘**
> （型号 / 序列号 / 容量 / 盘号**上盘后才知道**，所以本手册**不写死任何一个**）。
> 作者/依据：`dev/onsite-93` 建立实现面；`dev/onsite-98` **把护栏改为盘无关**
> （去掉「硬编码仅盘 0 / `Model` 含 `INTEL` / `Size ≈ 119.24 GiB」三条旧身份判据，
> 换成「`--disk N` 显式声明 + `--expect-serial` 硬闸 + `IsBoot`/`IsSystem` 拒」）。
> 护栏实现：`stoa-impl/tools/stfs_physdisk.{h,c}` + `stoa-impl/tools/d0_mkfs.c`
> （用法与决策记录见 `tools/README.md` **D5**/**D6**；自测 `tools/selftest_physdisk.ps1`）。
> 前置阅读：本文件头「机械盘版硬闸」；`上盘前盘位基线.md`（做差的参照）；
> `续作启动包.md` §6（真盘护栏）§7（已知坑）；`里程碑就绪检查表.md` §7.3（实机红线）。
>
> **本 runbook 的所有命令都只操作 `F:\Workspace\**` 与「**操作者显式声明的那一块盘**」。**
> **真盘写入必须提权、必须由人执行**；实现侧的等价验证在文件镜像上完成（见 §5）。

---

## 0. 红线（先读，再看命令）

| # | 红线 | 为什么 |
| :-- | :--- | :--- |
| R1 | **盘号必须由操作者显式给出**（`--disk N`），且**必须**配 `--expect-serial <SN>`。**没有自动挑盘**；N 若来自「做差」之外的任何来源 ⇒ 停手 | 盘号会因换线而**漂移**；容量会**撞车**（本机 Disk 5 `faspeed P8-128G PLUS` 与旧目标盘**同为 119.24 GB**）。唯一不漂移的是**序列号** |
| R2 | **`IsBoot==true` 或 `IsSystem==true` 的盘一律不碰**（工具会拒，操作者也别再试） | 宿主 C:（`SAMSUNG`，Boot+System）**绝不能碰**；工作盘 F:（`TOSHIBA`）**绝不能碰** |
| R3 | **绝不装任何宿主驱动** | 本路径只用 `CreateFile("\\.\PhysicalDriveN")`；不碰 BattlEye/BEDaisy |
| R4 | **一次只跑一个 guest / 一个构建** | 并发会出假红与端口冲突（`gate_all.py` 头注释） |
| R5 | **不加 `--zero`** | `--zero` 会把**整个分区**写零；本 runbook 的路径只写卷结构 |
| R6 | 中途失败**必须**走 §3 的状态判定 | 「绝不留下半成品而不自知」 |

工具内置的护栏（**缺一不可**，任一不中即中止，不做"尽力而为"）：

1. **盘号必须显式声明**：`--disk N` 是**唯一**入口；未声明 / 越界（`>64`）在**打开设备之前**就拒（退出码 **3**）。
2. **序列号硬闸**：`--expect-serial <SN>` **必填**（只有零写的 `--identify-only` 可省）；
   工具打印该盘 **Model / Serial / SizeGB / Parts / IsBoot / IsSystem** 的**实际值**，与 `<SN>`
   比对**不一致 / 读不到 / 没给** ⇒ 中止（退出码 **4**）。
3. **宿主保护**：`IsBoot == true`（宿主 OS 卷 `%SystemDrive%` 所在盘）或 `IsSystem == true`
   （ESP / 活动分区所在盘）**一律拒**；两者**读不到也拒**（fail closed）。
   另加**毒盘黑名单**（按 Model，不依赖盘号）：`SAMSUNG` / `TOSHIBA` / `faspeed`。
4. **分区数 == 0**：两个来源（`IOCTL_DISK_GET_DRIVE_LAYOUT_EX` 与扇区 0 字节级解析）取**更严者**。
5. **魔数打点**：写前向扇区 0 写 `StFS-DISK0-PROBE` 并**读回逐字节比对**，不 MATCH 即中止
   （**只有 `--probe-only` 与真正的写入路径**会打点；`--identify-only` / `--verify-only` 零写）。

核身与写入**分两次打开**：判据不中时**连一个可写句柄都不存在**。

**退出码约定**：`0` 成功 ｜ `1` 介质/校验失败 ｜ `2` 参数错误 ｜ `3` 盘号未显式声明 / 越界 ｜
`4` 核身失败或判据不中（含**序列号硬闸不中**、`IsBoot`/`IsSystem` 拒）。

---

## 1. 前置检查（**只读**，不要求提权）

> 全部只读：`Get-Disk` / `Get-Partition` / `--identify-only` 都不写任何字节。
> 若 `Get-Disk` 报权限错，就在提权窗口里跑（仍只读）。
> 下面 PowerShell 片段**一律 ASCII 文案**（§7 的坑：PS 5.1 按 GBK 读无 BOM 的 UTF-8 `.ps1` ⇒ 中文会乱/报错）。

### 1.1 上盘后识别：与《上盘前盘位基线》**做差**（先看全，再动手）

```powershell
Get-Disk | Sort-Object Number |
  Format-Table Number, FriendlyName, SerialNumber,
               @{n='SizeGiB';e={[math]::Round($_.Size/1GB,2)}},
               PartitionStyle, IsBoot, IsSystem, OperationalStatus -AutoSize
```

**基线盘号（做差时必须排除；见 `上盘前盘位基线.md`）**：`0,1,2,3,4,5,6`。
**判据**：新插入的机械盘应当是**新增盘号**（不在基线的 `0..6` 里），且
`PartitionStyle=RAW`、`IsBoot=False`、`IsSystem=False`、分区数 0。

> **若做差落在基线已有盘号 ⇒ 立即停止**（识别失败，**不得写入**）。
> 记下新增盘的 **Model + Serial + SizeGiB + Parts**，**当面报操作者确认**"就是这块老机械盘"。

确认后，把这两个值填进后续所有命令（**下文一律用变量**）：

```powershell
$N  = 3                                  # <-- 上盘后由本步做差得到，报操作者确认
$SN = 'WD-WCASY1234567'                  # <-- 上盘后由本步实测得到，报操作者确认（逐字符照抄）
"target: disk=$N serial=$SN"
```

### 1.2 盘无关**只读**断言（硬断言：不中即中止）

```powershell
$d  = Get-Disk -Number $N
$np = @(Get-Partition -DiskNumber $N -ErrorAction SilentlyContinue).Count
$chk = [ordered]@{
  'explicit disk number' = ($N -ge 0 -and $N -le 64)
  'not a baseline disk'  = (-not ($N -in 0,1,2,3,4,5,6))
  'Serial == expected'   = ($d.SerialNumber -eq $SN)
  'not boot'             = (-not $d.IsBoot)
  'not system'           = (-not $d.IsSystem)
  'Partitions==0'        = ($np -eq 0)
  'size >= 64MiB'        = ($d.Size -ge 67108864)
}
$chk.GetEnumerator() | ForEach-Object { '{0,-22} {1}' -f $_.Key, $(if ($_.Value) {'OK'} else {'**FAIL**'}) }
'actual: number={0} model="{1}" serial="{2}" size={3} ({4} GiB) parts={5} boot={6} system={7}' -f `
  $d.Number, $d.FriendlyName, $d.SerialNumber, $d.Size, [math]::Round($d.Size/1GB,2), $np, $d.IsBoot, $d.IsSystem
if ($chk.Values -contains $false) { throw 'GUARD FAILED: do NOT write anything to this disk' }
'GUARD PASSED (read-only precheck)'
```

**判定标准**：7 行全 `OK` 且最后打印 `GUARD PASSED (read-only precheck)` ⇒ 允许进入 §2。
任何一行 `**FAIL**` ⇒ **停手**，不许"再试一次"，先把实际值记录下来报统筹。

> **注意**：`SizeGiB` **不再有** 119.24 的窗口——本次目标是**老机械盘**，容量未知。
> 只要 `>= 64 MiB`（工具侧的下限）就过；真正的身份判据是**序列号**。
> 若 `Get-Disk -Number $N` 报错 ⇒ 盘号不存在或已掉线 ⇒ 回到 §1.1 重新做差。

### 1.3 工具侧**零写**核身（与 §1.2 互为独立观察面）

工具自己读一遍设备（`IOCTL_STORAGE_QUERY_PROPERTY` 取 Model/Serial、
`IOCTL_DISK_GET_LENGTH_INFO` 取容量、`IOCTL_DISK_GET_DRIVE_LAYOUT_EX` 取分区数与
ESP/活动分区、`%SystemDrive%` → 卷 → 盘号 派生 `IsBoot`），**一个字节都不写**：

```powershell
$TOOL = 'F:\Workspace\stoa-impl\tools\build\d0_mkfs.exe'
& $TOOL --disk $N --identify-only --expect-serial $SN
"exit=$LASTEXITCODE"
```

**必须看到**（`identify-only:` 行是**设备实测值**，方括号里是分区数的两个来源）：

```
identify-only: **零写核身**（只读打开；不打魔数、不写分区表、不格式化）
physdisk: **实际值** number=3 (known=1) model="WDC WD5000AAKS-00V1A0" (known=1) serial="WD-WCASY1234567" (known=1) size_bytes=500107862016 (known=1, 465.76 GiB) partitions=0 (known=1) [source ioctl=0, source bytes=0] IsBoot=0 (known=1) IsSystem=0 (known=1)
expect: **期望** disk=3 (显式声明，0..64) expect_serial="WD-WCASY1234567" require IsBoot=false IsSystem=false partitions=0 size>=67108864 B
guard: ALLOW —— 全部判据通过（显式盘号 + 序列号硬闸 + 非 boot/system + 未分区）
identify-only: 硬闸通过；**本步零写**（未做任何写入）
result: D0_MKFS_OK
exit=0
```

**判定标准**
- `model=...` / `serial=...` / `size_bytes=...` / `partitions=0` / `IsBoot=0` / `IsSystem=0`
  **必须与 §1.2 的实际值逐字符一致**（不一致 ⇒ 换盘了 / 盘号变了 ⇒ 停手重做 §1.1）。
- **必须**出现 `guard: ALLOW`。
- 方括号里是**两个来源**（`ioctl=` 系统盘布局 / `bytes=` 扇区 0 解析）：两者都是 `0` 最干净；
  任一为 `-1` 表示该来源不可判定（工具取**更严者** ⇒ 会拒）⇒ 停手查原因，**别绕过**。
- `IsBoot` / `IsSystem` 只要**不是 0 或读不到**，工具一律拒 ⇒ 说明这块盘承着宿主系统，**停手**。
- 若看到 `phys: 只读打开 \\.\PhysicalDrive3 失败：Win32=5` ⇒ **没提权**（本步未写任何字节，
  回 §2.0 提权；或先用 §1.2 的 `Get-Disk` 口径确认）。

> **为什么先跑 `--identify-only` 而不是 `--probe-only`**：`--probe-only` **会写 1 个扇区**
> （打魔数）；`--identify-only` **一个字节都不写**。核身阶段只用后者。

### 1.4 留证（把基线存档，回滚时用它对照）

```powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
New-Item -ItemType Directory -Force -Path 'F:\Workspace\onsite98-tmp\disk-evidence' | Out-Null
Get-Disk | Sort-Object Number | Format-List * |
  Out-File "F:\Workspace\onsite98-tmp\disk-evidence\$stamp-getdisk-before.txt" -Encoding utf8
Get-Partition -DiskNumber $N -ErrorAction SilentlyContinue |
  Out-File "F:\Workspace\onsite98-tmp\disk-evidence\$stamp-part-before.txt" -Encoding utf8
"baseline saved: $stamp (target disk=$N serial=$SN)"
```

---

## 2. 提权执行（确切命令）

### 2.0 提权确认（必须为 True）

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
 ).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

同时把 §1.1/§1.2/§1.3 的片段**在提权窗口里再跑一遍**（只读，无副作用）。

### 2.1 工具与构建（若需重建）

```powershell
$TOOL = 'F:\Workspace\stoa-impl\tools\build\d0_mkfs.exe'
& "$TOOL" 2>&1 | Select-Object -First 3   # 空参数应打印用法（含 --identify-only / --expect-serial）
# 需要重建时（**在独立 worktree 里构建，别在主树抢工作区**）：
# $PY='F:\DSH\dsh-home\dsh-runtimes\dsh-primary-runtime\dependencies\python\python.exe'
# cd F:\Workspace\stoa-impl-wt-md; & $PY tools\build_d0_mkfs.py
```

> `dev/onsite-98` 的改动在**独立 worktree**（`F:\Workspace\stoa-impl-wt-md`）里构建与自测；
> 并入 `main` 后 `F:\Workspace\stoa-impl\tools\build\d0_mkfs.exe` 与本 runbook 同源。
> 两者都可用：`$TOOL` 指哪个都行，只要 `--disk N` 的护栏打印齐全（含 `expect:` 行与 `IsBoot`/`IsSystem`）。

### 2.2 步骤 A —— **只读**核身 + 只读挂载校验（零写；空盘上 exit=1 是**预期**）

```powershell
& $TOOL --partition --verify-only --disk $N --expect-serial $SN
"exit=$LASTEXITCODE"
```

**必须看到**（`physdisk:` 两行是**设备实测值**，方括号里是分区数的两个来源）：

```
physdisk: **实际值** number=3 (known=1) model="WDC WD5000AAKS-00V1A0" (known=1) serial="WD-WCASY1234567" (known=1) size_bytes=500107862016 (known=1, 465.76 GiB) partitions=0 (known=1) [source ioctl=0, source bytes=0] IsBoot=0 (known=1) IsSystem=0 (known=1)
expect: **期望** disk=3 (显式声明，0..64) expect_serial="WD-WCASY1234567" require IsBoot=false IsSystem=false partitions=0 size>=67108864 B
guard: ALLOW —— 全部判据通过（显式盘号 + 序列号硬闸 + 非 boot/system + 未分区）
guard: 序列号硬闸通过（实测="WD-WCASY1234567" == --expect-serial）
verify-only: **只读挂载校验**（本步不写任何字节；跳过魔数打点）
phys: 读写句柄已打开 \\.\PhysicalDrive3 capacity_sectors=976773168
verify-only: 介质上没有可识别的 StFS 分区 —— 中止（未写任何字节）
result: D0_MKFS_PROBLEM
exit=1
```

**判定标准**
- `model=` / `serial=` / `size_bytes=` / `partitions=0` / `IsBoot=0` / `IsSystem=0` 六项
  **必须与 §1.2/§1.3 的实际值一致**（不一致 ⇒ 换盘了 ⇒ 停手）。
- **必须**出现 `guard: ALLOW` 与 `guard: 序列号硬闸通过`。
- `partitions=0` 的方括号里是**两个来源**：两者都是 `0` 最干净；任一为 `-1` ⇒ 工具取**更严者**会拒 ⇒ 停手。
- `capacity_sectors=<N>` 是这块盘的**自述容量**（扇区数 = 字节 ÷ 512）——**记下它**，后面 §2.4 要对照。
- 空盘上 `exit=1` + `没有可识别的 StFS 分区` 是**预期**：它同时证明了「这块盘目前没有 StFS 卷」。
- 若看到 `phys: 只读打开 \\.\PhysicalDrive3 失败：Win32=5` ⇒ **没提权**（本步未写任何字节，回 §2.0）。

### 2.3 步骤 B —— 魔数打点 + 读回比对（**第一次写**，只写 1 个扇区）

```powershell
& $TOOL --whole-disk --disk $N --expect-serial $SN --count-sectors 262144 --probe-only
"exit=$LASTEXITCODE"
```

**必须看到**：

```
probe: write lba=0 magic="StFS-DISK0-PROBE" rc=0 done=1 timed_out=0
probe: readback rc=0 done=1 timed_out=0 bytes=[535446532D4449534B302D50524F4245]
probe: readback MATCH magic="StFS-DISK0-PROBE" —— 写路径自证通过
probe-only: 核身 + 魔数打点均通过；按要求停止（**未写分区表、未格式化**）
result: D0_MKFS_OK
exit=0
```

**判定标准**
- `bytes=[535446532D4449534B302D50524F4245]` 就是 `StFS-DISK0-PROBE` 的十六进制（**逐字节相同**）。
- 必须是 `readback MATCH`；出现 `MISMATCH` / 写或读 `rc!=0` ⇒ **中止**（工具已中止，别再往下格式化）。
- 本步之后磁盘扇区 0 的前 16 字节 = 魔数、**其余为 0**（**分区表此刻不存在**，这是预期的中间态）。
  自测脚本已逐字节证明「**只动扇区 0**」（第 1~7 扇区仍全零）。

### 2.4 步骤 C —— 整盘格式化（MBR `0x7F` 分区 + StFS 卷 + 自验）

```powershell
& $TOOL --whole-disk --disk $N --expect-serial $SN `
    --start-lba 2048 --count-sectors 262144 --cluster 4096 --label StFS-MD
"exit=$LASTEXITCODE"
```

> `--count-sectors 262144` = **128 MiB 分区**，即本项目等价验证跑通的那一组参数（首用取小的，写量小、回滚快）。
> 想用满整盘见 §2.6。**机械盘写 128 MiB 是秒级**（对比 `--zero` 整盘写零是小时级，见 R5）。

**必须看到**（与文件镜像等价实测**逐字同形**，只有容量数字不同）：

```
guard: ALLOW —— 全部判据通过（显式盘号 + 序列号硬闸 + 非 boot/system + 未分区）
guard: 序列号硬闸通过（实测="WD-WCASY1234567" == --expect-serial）
image: PhysicalDrive sectors=976773168 (476940.0 MiB) cluster=4096 mode=whole-disk
media: PHYSICAL(真盘整设备，\\.\PhysicalDrive) disk=3 magic="StFS-DISK0-PROBE"
probe: readback MATCH magic="StFS-DISK0-PROBE" —— 写路径自证通过
mbr: slot=0 type=127 start_lba=2048 count_sectors=262144 disk_sectors=976773168
layout: primary_ledger_lba=65536 primary_boot_table_lba=66048 secondary_ledger_lba=131072 boot_table_backup_lba=131584 usable=1
format: D0 rc=0 40 rc=0 volume_sectors=262144
verify: 40 ledger rc=0 valid=1 ext_valid=1 version=1 cluster=4096 boot_entries=0 root_log=2 checkpoint=1 secondary_lba=131072 boot_table_lba=66048 boot_table_backup_lba=131584
verify: 40 boot_table rc=0 entry_count=0 kernel_entries=0 bootable=0
result: D0_MKFS_OK
exit=0
```

**判定标准（必须全中）**
1. `guard: ALLOW`（再一次核身）与 `guard: 序列号硬闸通过`。
2. `probe: readback MATCH`（写路径自证）。
3. `mbr: slot=0 type=127`（类型字节取自 `stfs.h` 单一事实源）且 `start_lba=2048 count_sectors=262144`
   （与命令行一致）。
4. `format: D0 rc=0 40 rc=0`。
5. `verify: 40 ledger rc=0 valid=1 ext_valid=1` 且 `verify: 40 boot_table rc=0`。
6. 最后一行 `result: D0_MKFS_OK`，且 `exit=0`。
- `disk_sectors` 应与 §2.2 的 `capacity_sectors` **一致**（都是这块盘的自述容量）。若明显不同 ⇒ **停手**。
- `boot_entries=0` / `entry_count=0` 是**空卷**的正常值（工具只产卷结构，不建根日志区）。

### 2.5 步骤 D —— 随后**只读挂载校验**（零写；独立复核）

```powershell
& $TOOL --partition --verify-only --disk $N --expect-serial $SN
"exit=$LASTEXITCODE"
```

**必须看到**（`physdisk:`/`expect:`/`guard:` 行同 §2.2，此处不重复）：

```
verify-only: **只读挂载校验**（本步不写任何字节；跳过魔数打点）
phys: 读写句柄已打开 \\.\PhysicalDrive3 capacity_sectors=976773168
part: slot=0 start_lba=2048 count_sectors=262144
verify-only: 盘上真实分区 start_lba=2048 count_sectors=262144（不采信命令行 --start-lba）
verify: 40 ledger rc=0 valid=1 ext_valid=1 version=1 cluster=4096 boot_entries=0 root_log=2 checkpoint=1 secondary_lba=131072 boot_table_lba=66048 boot_table_backup_lba=131584
verify: 40 boot_table rc=0 entry_count=0 kernel_entries=0 bootable=0
result: D0_MKFS_OK
exit=0
```

**判定标准**：`verify-only` 从**盘上真实 MBR** 找到 `start_lba=2048`、总账 `rc=0 valid=1 ext_valid=1`、
`result: D0_MKFS_OK` 且 `exit=0` ⇒ 真盘上的 StFS 卷**只读可挂载**，首用成功。
（本步与写路径**互相独立**：一个从写侧自报，一个从读侧复核。）

### 2.6 可选 —— 用满整盘

```powershell
# count = 盘容量 − start（工具内取），对齐由包 20/40 判据把关（不合法就报错，绝不自动取整）
& $TOOL --whole-disk --disk $N --expect-serial $SN --start-lba 2048 --count-sectors 0 --label StFS-MD-FULL
```
**必须看到** `count: --count-sectors 0 ⇒ 用满剩余 <N> 扇区 (<x> GiB)` 且后续判定同 §2.4。
若格式化报对齐/几何类错误（`STFS_ERR_ALIGN_INVALID` / 几何推导失败），改用显式对齐值：
`$C = [math]::Floor((<capacity_sectors>-2048)/2048)*2048` 然后 `--count-sectors $C`
（`<capacity_sectors>` 取 §2.2 打印的实际值）。

---

## 3. 失败与回滚

### 3.1 先判定盘上状态（**只读**，三步足够定性）

```powershell
Update-HostStorageCache                       # 让 OS 重新读盘（只读）
Get-Disk -Number $N | Format-List Number,FriendlyName,SerialNumber,Size,PartitionStyle,IsBoot,IsSystem
Get-Partition -DiskNumber $N -ErrorAction SilentlyContinue |
  Format-Table PartitionNumber,Offset,Size,Type -AutoSize
& $TOOL --partition --verify-only --disk $N --expect-serial $SN     # 需提权；零写
```

对照表（`PartitionStyle` × `验证结果` ⇒ 结论）：

| `Get-Disk $N` 的 PartitionStyle | `--verify-only` | 结论 | 处置 |
| :-- | :-- | :--- | :--- |
| `RAW` + 0 分区 | `没有可识别的 StFS 分区` (exit 1) | **未动过**，或只打了魔数（扇区 0 无 55AA） | 可直接重跑 §2.3 起 |
| `RAW` + 0 分区 | — | 已 `clean` 干净 | 可直接重跑 §2.3 起 |
| `MBR` + 1 分区(type `0x7F`) | `ledger rc=-20/valid=0` | **半成品**：分区表写了、卷没写完/写坏 | 走 §3.3 回滚，再重跑 |
| `MBR` + 1 分区(type `0x7F`) | `valid=1 result: D0_MKFS_OK` | 完成 | 无需回滚 |
| 其它形态（多分区/非 `0x7F`） | 任意 | **不是本 runbook 造的盘** ⇒ 停手 | **不要** `clean`，先报统筹 |

要看**扇区 0 原始字节**（区分"只打了魔数"和"什么都没写"）时，用下面这段**只读**探针
（hosted Python；**不要用 Add-Type / 内联 C#** —— §7 记过：PowerShell 内联编译会撞 AMSI `0xC0000005`）：

```powershell
@'
import os, sys
n = sys.argv[1]
h = os.open(r'\\.\PhysicalDrive' + n, os.O_RDONLY | os.O_BINARY)   # 只读句柄
b = os.pread(h, 512, 0)                                            # 512 对齐读扇区 0
os.close(h)
print('magic@0  =', repr(b[0:16]))
print('mbr_sig  =', b[510:512].hex(), '(55aa = MBR 签名存在)')
print('type@450 =', hex(b[446+4]), '(0x7f = StFS 分区类型字节)')
print('part_lba =', int.from_bytes(b[446+8:446+12], 'little'),
      'sectors =', int.from_bytes(b[446+12:446+16], 'little'))
'@ | Set-Content -Encoding ascii 'F:\Workspace\onsite98-tmp\read_disk.py'
$PY='F:\DSH\dsh-home\dsh-runtimes\dsh-primary-runtime\dependencies\python\python.exe'
& $PY 'F:\Workspace\onsite98-tmp\read_disk.py' $N
```

判读：`magic@0 = b'StFS-DISK0-PROBE\x00...'` ⇒ 只走到步骤 B；`mbr_sig = 55aa` 且 `type@450 = 0x7f` ⇒ 走到了步骤 C；
`magic@0` 全零且无 `55aa` ⇒ 扇区 0 干净。

### 3.2 顺手取证（失败时的必做项）

```powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$ev = "F:\Workspace\onsite98-tmp\disk-evidence"
Get-Disk | Sort-Object Number | Format-List * | Out-File "$ev\$stamp-getdisk-fail.txt" -Encoding utf8
Get-Partition -DiskNumber $N -ErrorAction SilentlyContinue | Out-File "$ev\$stamp-part-fail.txt" -Encoding utf8
& $TOOL --partition --verify-only --disk $N --expect-serial $SN *>&1 |
  Out-File "$ev\$stamp-verifyonly-fail.txt" -Encoding utf8
"evidence saved: $ev\$stamp-*"
```
**失败但没取证 = 半成品不自知**（本 runbook 不接受）。

### 3.3 回滚为「0 分区 / 可重用」

> `diskpart clean` 只清**分区表与首尾各约 1 MB**（秒级），**不是**整盘擦除。
> 先 `detail disk` 看清 **Model / Serial / Size**，**确认序列号 = `$SN`** 再 `clean`。

```powershell
diskpart
```
在 diskpart 提示符下（**逐条**敲，看输出再走下一步）：

```
list disk
select disk N            <-- N = $N（操作者显式声明的那一块）
detail disk              <-- 必须显示同一块盘的 Model / Serial / 容量 / 分区数
clean
detail disk              <-- 再次确认：分区表已无、盘回到未初始化
exit
```

> ⚠️ `diskpart` **不认序列号**，只认 `select disk N` ⇒ **在 `detail disk` 输出里逐字符核对
> Model 与 Serial** 再敲 `clean`。序列号对不上 ⇒ **立即 `exit`，不要 clean**。

回滚后复核（回到 §1.1/§1.2 的标准）：

```powershell
Update-HostStorageCache
Get-Disk -Number $N | Format-List Number,FriendlyName,SerialNumber,Size,PartitionStyle,IsBoot,IsSystem
@(Get-Partition -DiskNumber $N -ErrorAction SilentlyContinue).Count      # 必须为 0
& $TOOL --partition --verify-only --disk $N --expect-serial $SN          # 必须是「没有可识别的 StFS 分区」
```

**成功判据**：`PartitionStyle = RAW`、分区数 **0**、`--verify-only` 报「没有可识别的 StFS 分区」。
此时盘回到**与首用前等价的可重用状态**（扇区 0 的魔数也会被 `clean` 一并清掉）。

### 3.4 绝不做

- ❌ `clean all`（整盘逐扇区擦：老机械盘要几十小时，且无必要）。
- ❌ 对**任何**未显式声明 / 未核对过序列号的盘做 `diskpart select disk N` / `clean` / `create partition`。
- ❌ 为了"绕过判据"去改 `stfs_physdisk.h` 的判据、给工具加 `--force`、或**省略 `--expect-serial`**。
  真需要改授权 ⇒ 走统筹（改判据 + 本 runbook + 常量，三处同时改，留痕）。
- ❌ 装任何宿主驱动 / 关闭安全软件来"让写入成功"。

---

## 4. 期望输出与判定标准（一页速查）

| 步骤 | 命令（提权） | 成功判据（**必须看到**） | 失败动作 |
| :-- | :--- | :--- | :--- |
| 1.1 识别 | `Get-Disk \| Sort Number` | 新增盘号 **不在基线 `0..6`**；`RAW` / 非 boot 非 system / 0 分区 | 停手（做差落在基线盘号 = 识别失败） |
| 1.2 只读断言 | 断言片段 | 7 行 `OK` + `GUARD PASSED` | **停手**（不许写） |
| 1.3 零写核身 | `--disk $N --identify-only --expect-serial $SN` | `guard: ALLOW` + `IsBoot=0 IsSystem=0` + `exit=0` | 看 §3.1（本步零写） |
| A 只读核身 | `--partition --verify-only --disk $N --expect-serial $SN` | `guard: ALLOW` + `序列号硬闸通过` + `没有可识别的 StFS 分区`（exit 1 预期） | 看 §3.1 |
| B 打点 | `--whole-disk --disk $N --expect-serial $SN --probe-only` | `readback MATCH` + `result: D0_MKFS_OK`（exit 0） | 中止，§3.1 |
| C 格式化 | `--whole-disk --disk $N --expect-serial $SN --start-lba 2048 --count-sectors 262144 …` | `mbr: … type=127` + `format: D0 rc=0 40 rc=0` + `valid=1 ext_valid=1` + `result: D0_MKFS_OK`（exit 0） | §3.1 → §3.3 回滚 |
| D 只读复核 | `--partition --verify-only --disk $N --expect-serial $SN` | `verify-only: 盘上真实分区 start_lba=2048` + `valid=1` + `result: D0_MKFS_OK`（exit 0） | §3.1 → §3.3 回滚 |
| 可选 满盘 | `--count-sectors 0` | `count: … 用满剩余` 且 C 的判据全中 | 用显式对齐 count 重试 |

**退出码**：0 成功 ｜ 1 介质/校验失败 ｜ 2 参数错误 ｜ 3 盘号未显式声明/越界 ｜
4 核身失败或判据不中（含序列号硬闸不中、`IsBoot`/`IsSystem` 拒）。

---

## 5. 覆盖边界（诚实标注：哪些**还没**在真盘上验证过）

| 面 | 状态 |
| :--- | :--- |
| **盘无关判据逻辑**（显式盘号 / 序列号硬闸 / `IsBoot` / `IsSystem` / 黑名单 / 容量下限 / 分区数 / 魔数读回 / MBR + 格式化 + 自验 / 只读挂载校验的**代码路径**） | **已在文件镜像 + 注入身份上等价验证**（`dev/onsite-98` 自测 `tools/selftest_physdisk.ps1` **23/23**；`verify/image_e2e.py` **72 项 0 失败**） |
| `IOCTL_STORAGE_GET_DEVICE_NUMBER` / `IOCTL_STORAGE_QUERY_PROPERTY`（Model/SN）/ `IOCTL_DISK_GET_LENGTH_INFO` / `IOCTL_DISK_GET_DRIVE_LAYOUT_EX` 取**真设备实际值** | **未验证**（需提权；实现按 Win32 语义书写，本会话只能验证到「非提权 ⇒ Win32=5 ⇒ 明确中止」） |
| **`IsBoot` / `IsSystem` 的 Windows 侧派生**（`%SystemDrive%` → `\\.\C:` → `IOCTL_STORAGE_GET_DEVICE_NUMBER` 得盘号；盘级布局里的 ESP 类型 GUID / MBR 活动分区） | **未在真设备验证**（注入身份上已验判据逻辑：`true` ⇒ 拒、**未知 ⇒ 拒**） |
| 真设备上的扇区读写语义（`\\.\PhysicalDriveN` 对齐/缓存/共享打开） | **未验证**（复用包 20 的既有实现，该实现在文件与卷设备上已被 L1/L2 覆盖） |
| 真设备 `--verify-only` / `--identify-only` 的零写性 | **未在真盘验证**（文件镜像上已用 **SHA-256 前后一致**证明；真盘侧请以 §3.1 的取证留痕佐证） |
| 实机 EFI 引导 | **已移出一期验收**（上级决定，见 `续作启动包.md` §6） |

**未验证的几项都只能在提权后由人执行本 runbook 才能闭合** —— 它们不是"已通过"，是"待真盘闭合"。
