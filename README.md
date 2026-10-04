# StFS

模块化文件系统——面向非移动硬盘，CoW 事务 + 三层日志 + 跨文件区 + 文件夹块自包含。
核心追求：写入不卡顿、长期不碎片化、断电可恢复、程序运行近原生延迟、绝不蓝屏。

> 当前版本：设计稿 v0.6.4 ｜ **一期（机械盘侧）进行中** ｜ 二期（NAND）部分冻结 ｜
> **API 已冻结**（基线 tag `v1.0.1`；当前 **`v1.1.0` / ABI 1.2**，尾部追加的 MINOR）
> ｜ 环节1（只读最小可跑）已达成并合练 ｜ **环节2 部分完成**（正在对系统驱动侧进行开发，已实现作为windows内嵌驱动时可读可写，驱动正在与Windows侧api做对齐。由于模块开发时便与linux侧api对齐，所以linux侧驱动放到windows侧之后执行开发）

---

## 仓库结构

本仓是 **meta 仓**：放设计稿与开发包规范（含各包验收标准）。代码与对外接口各自独立成仓，以子模块形式挂入。

| 条目 | 说明 |
| :--- | :--- |
| [`新文件系统设计稿.txt`](新文件系统设计稿.txt) | 总设计规划书 v0.6.1（含修订记录与两处事实性修正） |
| [`StFS开发包/`](StFS开发包/README.md) | 按块开发包规范：11 块 + 公共约定，每块含 `AC-*` 验收标准，可独立开工 |
| [`StFS开发执行表.txt`](StFS开发执行表.txt) | 项目 → 模型调用对照表 |
| [`fs-api-design/`](fs-api-design/) | **子模块** → 对外 API 规范（`stfs.h` 为 C ABI 单一事实源，编译期断言锁定） |
| [`stoa-impl/`](stoa-impl/) | **子模块** → 各包的**实现代码**与构建/验收脚本（包 20 / 30 / 40 已冻结；包 50 环节2 首步已落地并首次冻结） |

## 总开发期

| 期 | 内容 | 状态 |
| :--- | :--- | :--- |
| **一期** | 机械盘侧全部开发（环节1 只读最小可跑 → 环节2 写入路径 → 环节3 集成与优化） | 🚧 进行中 |
| **二期** | NAND 相关全部功能（设计稿 §十五） | ⏸️ 部分冻结，SSD基本优化已完成 |

## 相关仓库

三个仓分工：**规范回答"对外暴露什么"，开发包回答"要满足什么"，实现回答"怎么做到"**。

- **[stoa-api-design](https://github.com/arrow1031/stoa-api-design)**（私有）— 对外 C ABI 规范子项，子模块挂在 `fs-api-design/`；
  权威物 `stfs.h`（编译期断言锁定的单一事实源）。
- **[stoa-impl](https://github.com/arrow1031/stoa-impl)**（私有）— 实现侧子项，子模块挂在 `stoa-impl/`；
  每个实现包一个目录（编号与 `StFS开发包/` 一一对应），各带 `build.py` 验收脚本；
  同时子模块化 `stoa-api-design` 以取得 `stfs.h`。

两者的更新方式相同：**先在子仓提交并推送，再在父仓提交子模块指针（gitlink）变更**。

## 本地开发

指导文件（本地，均不在仓库内或已在仓库内标注）：

| 文件 | 用途 |
| :--- | :--- |
| [`StFS开发包/README.md`](StFS开发包/README.md) | 开发入口：包清单、构建顺序、环节划分 |
| [`StFS开发包/10-公共约定.md`](StFS开发包/10-公共约定.md) | **全局必读**：常量、错误码、不变量、仓库分工与本地工具 |
| [`StFS开发执行表.txt`](StFS开发执行表.txt) | 模型路由（含上下文纪律，控制额度消耗） |
| `gh\ghctl.py` + [`gh\README.md`](gh/README.md) | **GitHub 操作统一入口**（仓库 / secret / deploy key / CI 日志 / 子模块 bump）。`gh\` 不在任何仓库内 |

```powershell
python gh\ghctl.py status                                   # 账号 / 额度 / 三仓状态
python gh\ghctl.py runs arrow1031/stoa-impl --steps          # CI 结论
python gh\ghctl.py submodule-bump stoa-impl --commit --push  # 子仓推送后更新父仓指针
python stoa-impl\verify\run_all.py                          # 全部实现包验收（含跨包依赖封口前置）
```

> **跨包依赖封口**（`10-公共约定.md` §14）：一个源只有一个所有者包，别包的源走
> `link_only`，**不许复制副本**；聚合构建必须按去重集合编译。机器强制
> `stoa-impl\verify\check_deps.py`，已作为 `run_all.py` 的前置阶段（CI 每次都会跑）。

## 本仓库不含

- 任何密钥 / 凭据（API key、GitHub token 均在仓库外，已被白名单忽略）
- 工具链、构建产物、会话临时文件
