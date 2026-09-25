# dsh-skill-blender-modeling

> **Blender 建模与装配流程 skill**（DSH / Claude 风格的 `skills/<name>/SKILL.md` 目录）
> 把「捏一个物体」和「**多个 agent 造一台机器**」都写成可执行流程 + 可跑的判据。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## 这是什么

一份**给 AI agent 读的建模作业指导书**：先按决策树选形，再走流水线，每一步都要有画面或数字证据。
与常见的「Blender 入门教程」不同，它按**实测反馈**写成 —— 每一条都是踩过之后才加进去的：

- **载体不匹配就换路子**：分面装甲板禁止上 SubSurf（会把棱边圆成肥皂）；编辑模式 `bpy.ops.mesh.*`
  在 `blender -b` 下 context 很脆 → 改 bmesh；
- **装配质量靠一条硬规则**：拼件必须互相插入 **5–15mm**（面贴面必见缝）；
- **判据必须有前提**：法线朝内会让所有"穿模 / 内含"判据整体反号（实测把贴合件误报成深入 150mm）；
- **验收不能靠肉眼**：形体要「同机位对照图 + 数值 bbox」两条证据。

## 内容（SKILL.md）

| 章节 | 一句话 |
|---|---|
| §0 工具与通道语义 | `blender_rt_*` 直连链怎么用、什么时候走 headless、写租约怎么拿 |
| **§0.5 多 Agent 分工范式** | 冻结 `spec.py`（尺寸+接口表+命名）· 写作用域协议 · 接口表四要素 · 深嵌+装配后复检 · 数值门 |
| **§0.6 参考图形体还原** | 三步标定（找标尺 → `px_per_mm` → 量特征）· 双证据验收 · 读图三分法 |
| §1–§2 六步总纲 / 决策树 | 冻结规格 → 场景准备 → Blockout → 精修 → 修改器栈 → 清理 → 交接 |
| §3 配方（10 个） | 基本体 · **分面装甲板** · 光滑硬表面栈 · 编辑模式 · **bmesh 等价写法** · 布尔 · 镜像 · Array · **参数化精确节距** · **装配审计门** · **双证据验收** |
| §4 六条坑 | 长/宽/薄轴语义 · **嵌入 5–15mm** · 收尖压中线 · **headless 慎用 bpy.ops** · **法线前提** · **预览光照** |
| §5 修改器栈 + 症状表 | Mirror → Array → Solidify → Bevel → SubSurf →（Boolean）；症状 → 修法 |
| §6 数值门清单 | 0 非流形 / 0 开边 / 0 孤立点 / 0 悬空 / 0 穿模 / 嵌入 5–15mm / 形体双证据 |
| §9 更新对照 | 每条反馈落在哪一节 |

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

本技能里的 `blender_rt_*` / `blender_viewport` 工具来自配套插件（v0.9.4 起）：

- **https://github.com/sixtysevenlf/dsh-blender-plugin**

插件负责"通道"（看视口 / 跑 Python / 无头批处理 / 作业层 / 事务回滚 / 装配审计 / 一键拉起 Blender），
本技能负责"流程与判据"（选形、配方、坑、数值门）。两个一起用才完整。

## 相对 v1 的改动（2026-09-25）

v1 是"单人单线捏一个物体"的手册；v2 加上了大件 / 多 agent 真正需要的东西：

| v1 | v2 |
|---|---|
| 无并行建模章节 | **+ §0.5 多 Agent 分工范式**（冻结规格 / 写作用域 / 接口表 / 数值门） |
| 无参考图章节 | **+ §0.6 参考图驱动的形体还原**（比例标定 + 双证据） |
| 配方 2 = Bevel+SubSurf | **拆 2a 分面 / 2b 光滑**（分面件上 SubSurf = 毁形） |
| 配方 3 = 编辑模式 ops | 保留 GOI 版 + **新增 3b bmesh 版**（headless 稳） |
| 配方 6 = Array 沿曲线 | 保留装饰用途 + **新增 6b 参数化精确节距**（履带 83 节 / 0.176） |
| 三条坑 | **六条坑**（+ headless bpy.ops / 法线前提 / 预览光照） |
| 清理清单 | **数值门清单**（10 条判据 + 对应工具） |

## 许可与来源

MIT。派生自两个 MIT 项目（决策树 / 配方 / 坑表 / 六步总纲等），工具调用已改写为本机 `blender_rt_*` 直连链：

- [RobLe3/cc-blender-skill](https://github.com/RobLe3/cc-blender-skill)
- [arjun988/blender-skills](https://github.com/arjun988/blender-skills)

完整归属与本地改动见 `NOTICE.md`；`"references/modeling-overview.md"` 为上游英文原文收录。
