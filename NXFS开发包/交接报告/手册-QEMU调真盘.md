# 手册：**QEMU 模拟器调真盘**（Disk 0 · ST1000DM003-1SB10C · 2026-09-29 用户批准）

> 用户批准："**能试的先模拟器调真盘试：我们的文件系统是模块化的，不怕这种部分在升级开发的测试**" ✓
> 目的：让**客机（另一个 OS）**直接操作真盘上的 NXFS 卷 ⇒ 拿"**OS 看到它是一块**"的直接证据 ✓，
> 同时**宿主侧不装任何驱动** ✓（QEMU raw 透传不算驱动 ✓）。

## 0. 红线（先读）
- **只允许 Disk 0 = `ST1000DM003-1SB10C` / SN `W9A097HQ` / 931.51 GiB / 0 分区 / 非 boot 非 system** ✓。
- **绝不能碰**：Disk 1 `TOSHIBA MG10ADA800E`（F: 工作盘）· Disk 6 `SAMSUNG MZVLQ512HALU`（C: 宿主系统）·
  Disk 5 `faspeed P8-128G PLUS`（同容量伪装盘）· Disk 2/3/4 ✓。
- **一次只跑一个 guest** ✓；不装宿主驱动 ✓；不碰 BattlEye/BEDaisy ✓；只写 `F:\Workspace\**` 与客机内部 ✓。
- 透传**本身也是一次"上真盘"** ⇒ **闸门必须在关外先走完** ✓。

## 1. 宿主侧：先过闸（**在透传之前**）
```powershell
$M='F:\Workspace\nxfs-impl\tools\build\d0_mkfs.exe'
# ① 零写核身（必须 ALLOW 且序列号 == W9A097HQ）
& $M --identify-only --disk 0 --expect-serial W9A097HQ
# ② 只读总账校验（必须 D0_MKFS_OK，零写）
& $M --partition --disk 0 --expect-serial W9A097HQ --verify-only
```
**任一不中 ⇒ 立即停止，禁止透传** ✓（并要求 `Get-Disk -Number 0` 的 Model/Size/Partitions 同步复核 ✓）。

## 2. 启动客机（raw 透传 + **把身份带进客机**）
> 关键：`serial=W9A097HQ` ⇒ 客机内的护栏（`--disk N --expect-serial`）**仍能认这块盘** ✓；
> 否则客机看到的只是无名 virtio 盘 ✗，护栏就失去意义 ✓。

```powershell
# x86_64 Alpine 客机（iso 在 F:\Workspace\vm\iso\alpine-virt-3.24.2-x86_64.iso）
qemu-system-x86_64 -m 2048 -smp 4 `
  -drive if=none,id=d0,file=\\.\PhysicalDrive0,format=raw,cache=none `
  -device virtio-blk-pci,drive=d0,serial=W9A097HQ `
  -cdrom F:\Workspace\vm\iso\alpine-virt-3.24.2-x86_64.iso `
  -serial tcp:127.0.0.1:45512,server,nowait -nographic
```
⇒ 客机内该盘通常为 `/dev/vda`（**先用 `lsblk`/`cat /sys/block/vda/device/serial` 核对序列号** ✓）。

## 3. 客机内：真实使用（与宿主同一套工具）
```sh
# 网络与取件（宿主另开：python F:\Workspace\vm\share-srv.py，绑 192.168.29.1:8080）
ip link set eth0 up; udhcpc -i eth0 -q -n
wget -qO /tmp/d0_mkfs http://10.0.2.2:8080/d0_mkfs
wget -qO /tmp/nxfs_use http://10.0.2.2:8080/nxfs_use
chmod +x /tmp/d0_mkfs /tmp/nxfs_use
# 只读核身（客机侧护栏：盘号 + 序列号）
/tmp/d0_mkfs --identify-only --disk 0 --expect-serial W9A097HQ
# 只读总账
/tmp/d0_mkfs --partition --disk 0 --expect-serial W9A097HQ --verify-only
# 文件级真实使用（小文件）
/tmp/nxfs_use --disk 0 --expect-serial W9A097HQ --start-lba 2048 --count-sectors 1953523120 --cluster 4096 probe
/tmp/nxfs_use ... mkdir /guest
/tmp/nxfs_use ... putpat /guest/f 2000 --seed 7
/tmp/nxfs_use ... get /guest/f --seed 7
/tmp/nxfs_use ... rm /guest/f
# 大文件（用客机能拿到的真结构文件；66 MB 起，再试 6.4 GB）
```
**判定**：客机侧 `mount/ledger/get` 全 OK ✓、逐字节 + CRC/SHA-256 对拍 MATCH ✓、`ledger_result: OK` ✓。

## 4. 回归与留证
1. 宿主侧**再跑一次零写校验** ✓（`--verify-only` ✓）⇒ 证明**客机写过的卷，宿主仍认** ✓。
2. 记录：客机 `uname -a`、`lsblk`、序列号核对、每条命令的输出、耗时 ✓。
3. 失败即停：留下串口日志尾部 + 盘上状态判定（必要时按《真盘首用-runbook》§ 回滚：`Clear-Disk -Number 0 -RemoveData`，**并再次核对序列号** ✓）。

## 5. 与"混合版本测试"的配合（模块化红利 ✓，用户已批）
- **兼容矩阵**：旧读者（`14588fe` 构建）vs 新读者 × poke `ext.version=2` / `format_flags` bit0 / 对象头 `version=2`（每次重算 CRC）
  ⇒ 判"**旧读者分类拒收**" ✓（拒收门现成：`ledger.c:395/401`、`cow.c:39/85` ✓）。
- **模块混搭**：新 40 + 旧 50/70、新 20/30 + 旧 40 ⇒ 各跑包内 + L1 ✓ ⇒ 证明"部分升级不互相认错" ✓。
- 这些**不跑 `gate_all`** ✓（演练轨 ✓），于是可与实现轨并行而不抢资源 ✓。

---

## 6. 【追加】VMware **Windows 客机**也要做（用户 2026-09-29 批准）

> 用户原话："**不止 QEMU，VMware 这边的 Windows 也做能做的**" ✓
> 价值：Windows 是 NXFS 的目标宿主 ⇒ 这一路能跑**只有 Windows 才存在**的代码路径 ✓（QEMU/Alpine 跑不到 ✓）。

### 6.1 客机与通道（现成件）
- 客机：F:\Workspace\vm\win11\win11.vmdk（VMware · UEFI · SecureBoot 关 · 硬盘挂 **SATA 0:0** ✓）
- 启动/操作：F:\Workspace\vm\vmctl.ps1（-Action run -Guest win11 ✓）
- 宿主**脚本化完全体**：WinRM
  `powershell
   = New-Object System.Management.Automation.PSCredential('nxfs', (ConvertTo-SecureString '<pw>' -AsPlainText -Force))
   = New-PSSession -ComputerName 127.0.0.1 -Port 15985 -Credential  -Authentication Basic
  Copy-Item -ToSession  -Path <本机文件> -Destination C:\nxfs\
  Invoke-Command -Session  -ScriptBlock { ... }
  `

### 6.2 在 Windows 客机里做什么（**按价值排序**）
1. **Windows 目标二进制的真实运行**：zig cc -target x86_64-windows-gnu 构建的 d0_mkfs.exe / 
xfs_use.exe ✓ ⇒
   在客机里跑**完整流水线**（--whole-disk --image ✓ → --verify ✓ → 
xfs_use 的 mkdir/put/get/ls/rm ✓）。
   **判据**：与宿主/Linux 客机的结论**逐项一致** ✓（这构成**第三个载体**的证据 ✓）。
2. **只有 Windows 才有的路径**（QEMU 路跑不到 ✗）：
   - \\.\PhysicalDriveN 的 CreateFile + Overlapped 硬超时路径 ✓（**无提权时**应**明确中止**而不是崩溃 ✓）
   - IOCTL_STORAGE_QUERY_PROPERTY 的 **StorageDeviceTrimProperty** 与 IOCTL_STORAGE_MANAGE_DATA_SET_ATTRIBUTES ✓
     ⇒ 这是 S2（TRIM）在 Windows 侧**唯一能被真实触发**的地方 ✓（**在客机的虚拟盘上**试 ✓ —— 宿主真盘是 HDD 不支持 UNMAP ✗）
   - A0 集成层的宿主面 ✓
3. **.vmdk 当作"真盘形态"**：在客机里对一块**附加的 vmdk**（而非内存盘 ✓）跑格式化 + 文件级使用 + 只读回验 ✓ ⇒
   验证"**非 4096 对齐 / 真实块设备**"上的行为 ✓（比文件镜像更接近真盘 ✓）。
4. **不做**：在 Windows 客机里**挂载** NXFS 卷 ✗（一期无 Windows 驱动 ✓，属明示排除 ✓）；不装宿主/客机驱动 ✓。

### 6.3 纪律（与 QEMU 路相同）
- **一次只跑一个 guest** ✓（QEMU 与 VMware 不得同时 ✓）
- 客机内**只写客机磁盘** ✓；不碰宿主真盘 ✗（除非走 §1 三道闸 + §2 透传 ✓）
- 结果与日志留证：uname/systeminfo、盘号、序列号、每条命令输出、耗时 ✓
- 失败即停 + 留串口/日志尾部 + 判定盘上状态 ✓

### 6.4 三载体与混合版本的组合（本手册的完整矩阵）
| 载体 | 工具形态 | 主要证明 |
| :--- | :--- | :--- |
| 宿主 Windows | 宿主构建 | 基线 + 真盘（Disk 0）✓ |
| **QEMU Alpine x86_64/aarch64** | musl 静态 | **别的 OS 也能正确操作该卷** ✓ + 弱内存序 ✓ |
| **VMware Win11** | **windows-gnu** | **Windows 侧独有路径**（PhysicalDrive / TRIM IOCTL / A0）✓ |
| 混合版本 | 新旧模块/新旧读者 | **部分升级不互相认错** ✓（模块化红利 ✓） |