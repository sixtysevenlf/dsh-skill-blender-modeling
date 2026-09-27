# dsh-skill-blender-modeling

> **Blender 建模与装配流程 skill**（DSH / Claude 风格的 `skills/<name>/SKILL.md` 目录）· **v1.0**
> 把「捏一个物体」和「**多个 agent 造一台机器**」都写成可执行流程 + 可跑的判据。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## 这是什么

一份**给 AI agent 读的建模作业指导书**：先按决策树选形，再走流水线，每一步都要有画面或数字证据。
与常见的「Blender 入门教程」不同，它按**实测反馈**写成 —— 下面每一条都是踩过之后才加进去的：

- **载体不匹配就换路子**：分面装甲板禁止上 SubSurf（会把棱边圆成肥皂）；编辑模式 `bpy.ops.mesh.*`
  在 `blender -b` 下 context 很脆 → 改 bmesh；
- **展示拼接 ≠ 真实连接**：面贴面必见缝，所以展示模型用**隐藏交叠**（不改可见轮廓、不捅穿薄壳），交叠量由
  **本项目 spec 明文**定、装配后按 spec 复核 —— **没有通用倍数或下限**；能不能承载由设计配合 / 标准 /
  承载工况 / 制造工艺 / 运动间隙规定，几何脚本不给强度结论；
- **判据必须有前提**：法线朝内会让所有"穿模 / 内含"判据整体反号（实测把贴合件误报成深入 150mm）；
- **指标必须有适用范围**：IoU 只是**单视轮廓**的必要条件（看不见深度/内部特征/面感）；单应只校正
  **取点那一个平面**（离平面 0.5m 实测误差 543mm）；跨深度误差不是固定 ±5% 而是 `Δz/(D+Δz)`；
- **验收不能靠肉眼**：形体要「同机位对照图 + 数值 bbox」两条证据；判定走机械门（§6 十道门 / `gate_run` 三态），`degraded` 不算通过。

## v1.0 版内容一览

- **两条大件主线**：§0.5 多 Agent 并行建模的分工范式（冻结 `lib/spec.py` / 写作用域协议 / 接口表 /
  隐藏交叠 + 装配后复检 / 归属标签与可并行性判据）、§0.6 参考图驱动的形体还原（三视图标定 / 双证据验收 / 读图纪律）。
- **§0.7 建模类型 → 工具指路**：12 类建模类型的「第一步 / 算子链 / 禁止自造 / 验收门」+ 三条硬规则；
  §0.71 收调用约定（`kapi` 的 dict 与 JSON 两种形态）与离线逃生口（`offline_bootstrap.py`）。
- **配方 Recipe 1–23**（`### Recipe` 小节共 26 条：2 拆 2a / 2b，另含 3b、6b 变体）：基本体 → 分面 / 光滑硬表面 →
  bmesh 编辑 → 布尔 → 镜像 / Array → 参数化销轴弦长闭合阵列 → 装配审计门 → 形体验收 → 程序化雕刻 → 网格修复 + UV →
  打印检查 → 沿路径扫掠 → 人形素体 / 头型 → 车辆与参考图放样 → 照片校正 → 折线族拟合 → 特征线层 → 多步链一条命令跑完（`pipe_run`）。
- **五条高频配方参数化**（2a / 4 / 5 / 6b / 9）= 参数块 + 生成脚本，直接接 `generator_save/run/list/get/diff`（§3.0）。
- **六条必踩的坑**（§4）：长轴·宽轴·薄轴语义、可视拼接与真实连接的区别、收尖压中线、headless 慎用 `bpy.ops`、
  法线前提、预览光照；§4.0 按「能不能机械判定」分流到验收门或**写代码前 checklist**。
- **数值门清单**（§6：十道门 ①–⑩ + 证据留档）+ 交接前必跑与可复制的验收回执（§0.5.5 / §6.1）；
  判定是三态 `pass` / `fail` / `degraded` —— `clean=false` 先读 `state`，`degraded` 不是通过。
- **容易写错的"通则"在本版里的现行口径**：不存在固定修改器链（§5）；`Delete Loose` 一个面都不删，
  内部面走 Boolean UNION / 射线法（§5.1）；正交**不推出**三视共享 `px_per_mm`、跨深度误差 `≈ Δz/(D+Δz)`（§0.6.1）；
  单应只校正取点那个平面；`audit_scene` 默认不跑自交 ⇒ 交付验收必须显式传 `self_intersect:true`。

## 内容（SKILL.md）

| 章节 | 一句话 |
|---|---|
| §0 工具与通道语义 | `blender_rt_*` 直连链怎么用、什么时候走 headless、写租约怎么拿、独立验证者为什么只读验收清单 |
| **§0.5 多 Agent 分工范式** | 冻结 `spec.py`（尺寸+接口表+命名）· 写作用域协议 · 接口表**五要素**（含逐接口 `overlap_mm`，来自 spec 明文）· 隐藏交叠+装配后复检 · 数值门 · **交接前必跑清单 + 可复制的验收回执** · **可并行性判据 + `COL_<agent>`/`dsh.owner` 归属 + 单 builder 回滚** |
| **§0.6 参考图形体还原** | **三视图标定**（逐视独立最小二乘；跨视互检**换算成毫米再比**，因为正交**不**推出三视共享 `px_per_mm` —— 实测差 300% 全是正交）+ **IoU 只是单视轮廓的必要条件** + 双证据验收 + **读图纪律（420/900 与 hash）**；脚本见 `references/multiview-calibration.md` |
| **§0.7 建模类型 → 工具指路** | **先查类型再查能力族**：12 类的第一步 / 算子链 / 禁止自造 / 验收门（`catalog classes`）+ 三条硬规则；**§0.71** 调用约定与离线逃生口 |
| §1–§2 六步总纲 / 决策树 | 冻结规格 → 场景准备 → Blockout → 精修 → 修改器栈 → 清理 → 交接；按形状选路子 |
| §3 配方（Recipe 1–23；含 2a/2b/3b/6b，共 26 条小节） | 基本体 · **分面装甲板** · 光滑硬表面栈 · 编辑模式 · **bmesh 等价写法** · 布尔 · 镜像 · Array · **销轴弦长节距闭合阵列**（**模板契约①–⑤：原点在起点销轴 / 局部 +X 指下一销 / 下一销在 `(+pitch,0,0)` / 销轴 = 局部 +Z / 只有 z 被沿用**；自带从零建可视单节） · **装配审计门** · **双证据验收** · 程序化雕刻 · 网格修复 + UV · 打印检查 · 沿路径扫掠 · 人形素体 · 头型 · 车辆/参考图放样 · 照片校正 · 折线族拟合 · 特征线层 · **`pipe_run` 多步链**；**§3.0「参数块 + 生成脚本」**（2a/4/5/6b/9 五条接 `generator_save/run/list/get/diff`；**改参直接调工具、`args` 嵌套一层**，旧插件见迁移说明） |
| §4 六条坑 | 长/宽/薄轴语义 · **可视拼接用隐藏交叠、结构连接不由脚本裁定** · 收尖压中线 · **headless 慎用 bpy.ops** · **法线前提**（**逐连通壳**判 `normals_state` ∈ pass/fail/unknown，`nested` 空腔豁免） · **预览光照**；**§4.0 分流表**（法线/浮块/穿模 → 验收门；语义类 → 写代码前 checklist） |
| §5 修改器栈 + 症状表 | **没有固定链**（同一网格换顺序：Bevel→SubSurf 864 面 / SubSurf→Bevel 96 面，bbox 差 15%）→ 按**目标**排 + 用求值回执核对；**§5.1 删内部面的正确做法**（`Delete Loose` 一个面都不删 → Boolean UNION / 射线法） |
| §6 数值门清单 | ① 重复点/松散几何 · ② 流形 · ③ 开边 · ④ 法线（逐壳） · ⑤ 尺寸 · ⑥ 浮空 · ⑦ 穿模 · ⑧ 隐藏交叠（按 spec 明文） · ⑨ 形体（双证据） · ⑩ 面数 · + 证据留档；**三态口径**（`pass`/`fail`/`degraded`）；**§6.1 交接前必跑**（命令块 + 一张回执；`audit_scene` 交付验收必须 `self_intersect:true`） |
| §7–§8 深水参考 / 来源与许可 | `references/modeling-overview.md`（英文原文）· `references/multiview-calibration.md`（可跑的标定脚本）· `references/参考图通用还原SOP.md`（通用还原八步）· `references/验收清单.md`（独立验证者专用） |
| §9–§13 逐版修订对照 | SKILL.md 内保留的历史反馈落点（v2–v6）；本 README 不重复罗列 |

> 计数口径：配方按 §3 的 `### Recipe` 小节数（26 条）+ 编号 1–23；坑按 §4 的 `### 坑一`–`坑六`（6 条）；
> 门按 §6 表 ①–⑩（10 道，另有"证据留档"一行）。SKILL.md 的 frontmatter 与 §12 的计数已同步为 **23 条配方**
> （Recipe 23 `pipe_run` 加入后），加载器读到的技能目录也已是 23。

## 安装

```bash
git clone https://github.com/sixtysevenlf/dsh-skill-blender-modeling.git
cd dsh-skill-blender-modeling

# 方式 A（推荐）：软链，改一处两边同步
ln -s "$PWD" ~/.dsh/skills/blender-modeling

# 方式 B：直接拷贝
cp -r . ~/.dsh/skills/blender-modeling
```

> ⚠ `SKILL.md` **必须保留 YAML frontmatter**（`name` / `description` / `license`）——
> 加载器靠它建目录；frontmatter 一掉，技能会直接从技能目录里消失（踩过）。

## 配套插件

本技能里的 `blender_rt_*` / `blender_viewport` 工具来自配套插件（**v1.0.0**，能力冻结线）：

- **https://github.com/sixtysevenlf/dsh-blender-plugin**

插件负责"通道"（看视口 / 跑 Python / 无头批处理 / 作业层 / 事务回滚 / 装配审计 / 一键拉起 Blender），
本技能负责"流程与判据"（选形、配方、坑、数值门）。两个一起用才完整。

## 许可与来源

MIT（见 `LICENSE`，Copyright © 2026 sixtyseven67）。派生自两个 MIT 项目（决策树 / 配方 / 坑表 / 六步总纲等），
工具调用已改写为本机 `blender_rt_*` 直连链：

- [RobLe3/cc-blender-skill](https://github.com/RobLe3/cc-blender-skill)（决策树、配方、坑表、深水参考；
  `references/modeling-overview.md` 为上游英文原文收录）
- [arjun988/blender-skills](https://github.com/arjun988/blender-skills)（六步总纲、集合结构、修改器规则、清理清单、MUST / MUST NOT）

完整归属与本地改动见 `NOTICE.md`。`references/modeling-overview.md` 里「固定修改器顺序」与「内部面 → Delete Loose」
两处上游旧说法**已就地标注纠正**（现行口径见 §5 / §5.1）。
