---
name: blender-modeling
description: Blender 建模流程与装配教程——先按决策树选形（分面硬表面 / 光滑硬表面 / 有机 / 重复阵列 / 布尔），再走流水线（冻结规格 → 场景准备 → Blockout → 粗形精修 → 修改器栈 → 清理 → 交接），配 22 个编号配方（含 2a/2b/3b/6b 变体；基本体 / 分面装甲板 / Bevel+SubSurf 光滑栈 / bmesh 编辑 / 布尔开孔 / 镜像 / Array 沿曲线 / 参数化销轴弦长节距阵列 / 装配审计门 / 固定机位对照验收 / 程序化雕刻 / 网格修复 / 打印检查 / 沿路径扫掠 / 人形素体 / 头型 / 车辆与参考图放样 / 照片校正 / 折线族拟合 / 特征线层），覆盖六条必踩的坑（长轴·宽轴·薄轴语义、可视拼接用隐藏交叠并按项目 spec 复核而非固定倍数/不由可视脚本裁定结构连接、收尖压中线再合并、headless 慎用 bpy.ops、法线方向是一切穿模判据的前提、预览装置默认光照要看得清板缝），并含多 Agent 并行建模的分工范式（冻结 spec.py / 写作用域协议 / 接口表 / 隐藏交叠+复检 / 数值门装配审计 / 归属标签与可并行性判据）与参考图驱动的形体还原（三视图标定 + 双证据验收 + IoU 只是单视轮廓必要条件 / 单应只校正一个平面 / 跨深度误差按 Δz/(D+Δz) 算），以及交接前必跑清单与可复制的验收回执、参数块+生成脚本（generator_*）、本机 blender_rt_* 直连通道语义。用于「做一个 / 建一个 / 捏一个 3D 物体」「加个立方体 / 球 / 圆柱」「挤出 / 内插 / 倒角这个面」「加个修改器」「挖个洞」「做个剑 / 椅子 / 门」等几何创建与网格编辑请求，也用于「多个 agent 分工造一台机器」「按参考图还原外形 / 量比例」「装配体检：查穿模 / 浮空 / 按 spec 复核隐藏交叠量」等大件建模与装配任务。
license: MIT（融合自 RobLe3/cc-blender-skill 与 arjun988/blender-skills，见 NOTICE.md）
---

# Blender 建模流程

> **v2 · 2026-09-25**：按《M1A1 分件建模》实测反馈扩写（500 对象 / 7 个 agent / 全程 headless）。
> 新增 **§0.5 多 Agent 并行建模的分工范式**、**§0.6 参考图驱动的形体还原**、坑四/坑五/坑六，
> 修正配方 2 / 3 / 6 的「载体不匹配」（分面硬表面不要上 SubSurf、headless 慎用 bpy.ops、本配方口径的 `FIT_CURVE` Array 做不出精确闭环节距），
> 并把 §6 清理清单升级成**数值门清单**。逐条对照见 §9。
>
> **v3 · 2026-09-26**：按《实测驱动的通道 + Skill 优化方案》的 **§4 Skill 侧**做定点增补（流程骨架不动）：
> §0.5.5 增**交接前必跑清单 + 可复制的验收回执**（P0-4）· §0.6.3 增**读图纪律（420/900 与 hash）**（P0-5）·
> §0.6.1 扩成**三视图联立标定**（P1-3，脚本见 `references/multiview-calibration.md`）·
> §0.5.7 增**可并行性判据 + 归属标签 + 单 builder 回滚**（P1-5）· §3.0 增**参数块 + 生成脚本**（P1-4）·
> §4.0 增**六条坑分流 + 写代码前 checklist**（P2-3）。逐条对照见 §10。

先定形状，再走流程，改一步看一眼。**不要一口气写完整个模型再回来看**——每一步都要有画面或数字证据。

## 0. 本机工具与通道语义（先读，这决定你怎么写代码）

| 要做的事 | 用哪个工具 | 关键语义 |
|---|---|---|
| **拉起 Blender（GUI 会话）** | `blender_viewport(op="launch")` | **v0.9.4 起 agent 能自己点亮 GUI**：写 boot 脚本（GUI 里 enable addon → 起 socket server）→ detached spawn blender.exe → **轮询 9876 端口**（唯一可信判据）→ 顺手 doctor；幂等（已在监听只回 already，不会堆出第二个 Blender）。参数：wait_ms / file / exe / addon_module / addon_file / dry_run |
| 迭代建模、改一步看一眼 | `blender_rt_do` | **常驻 GUI 会话 + 持久内核 K**：整步约 105ms，变量**跨调用保留**；可选立刻回一帧 |
| 批处理 / 重活（**第一路径**） | `blender_rt_headless` | **每次都是全新 Blender 进程**（blender -b）：变量**不保留**，每次重新 import；shots 参数可一次出多视角；引擎默认 EEVEE + 光追 |
| 跑很久的活 | `blender_rt_job` | 作业层：op=start / **wait** / status / collect；日志落 `jobs/<id>/`；超时不丢结果（回 promoted + jobId） |
| 反复跑同一个脚本 | `blender_rt_worker` | 热无头会话：冷启动 + EEVEE 着色器编译只付一次。**单例串行** → 多 agent 并行时**不要共用**（会互相串场景） |
| 看画面 / 看动没动 | `blender_rt_see` / `blender_rt_watch` | see 回一帧（给 from/look_at 走自定义视角，带 coverage_estimate）；watch 在时间窗内采样 ≤6 帧 + 逐帧 hash |
| 回退到改动前 | `blender_rt_txn` | mark/revert 是对象级、毫秒；**拓扑改动**（布尔 / 删面 / 新建对象）必须 snapshot/restore |
| **装配体检（数值门）** | `blender_rt_plan(op="audit_*")` | audit_mesh · audit_scene · audit_connectivity · audit_gate · **audit_interference** · audit_overlap · audit_measure · audit_drift · audit_duplicates · audit_snap_floaters（见 §6） |
| **对照图 / 逐部件配色** | `blender_rt_plan(op="qc_render_views")` | 按 AABB 自动取景 + 临时三点光 + 逐张 md5 + coverage_estimate；`mode="parts_color"` 出逐部件配色图（图例写 parts_color_meta.txt） |
| **交付导出** | `blender_rt_plan(op="deliver_*")` | 单位盒归一化 + 多组 OBJ/MTL + manifest md5 |
| **参数化配方（生成器）** | `blender_rt_plan(op="generator_*")` | `generator_save/run/list/get/diff`：配方 = **参数块 + 生成脚本**（§3.0）。**改参不用改代码**；`generator_run` 在**全新无头进程**里复现并给三态 verdict（supported / refuted / unresolved）；源码一改，上次回执即 `stale` |
| 查场景 / 对象 / 引擎 | `blender_rt_cmd` `rt_commands` `rt_perf` | cmd 传 get_scene_info / get_object_info；commands 列全部 addon 命令；perf status 看引擎与采样 |

**改一步看一眼的正确节奏**：调 `blender_rt_do` 并在同一次调用里 see=true → 看图 → 不对就改 → 再看。局部修正只看问题视角加受影响的邻视角，不做每次全场景重建。

> 通道差异是本节最容易出错的地方：老式 MCP 教程会让你"每次调用重新 import 一切"——那对 `blender_rt_headless` 成立，对 `blender_rt_do` **不必要**（内核 K 和已导入的名字都还在）。但**依赖上一次调用留下的变量本身是脆的**：跨会话、后端重启、走 job 之后都会丢。稳妥写法是在**当次调用内**重新取一遍对象引用（`bpy.data.objects['GEO-x']`），不要假设 `obj` 这个变量还在。

**写通道租约（多 agent 必读）**：写操作自动带 holder 租约；别的会话持有时返回 **409 leased**，只读 op 豁免。
`blender_viewport(op="who")` 先看谁在写 → `op="lease"` 拿 → 真要抢再加 `force=true`。
**多 agent 并行的正确姿势是 headless**（每条调用独立进程，天然并行），不是抢 GUI 租约。

> **独立验证子代理只读 `references/验收清单.md`**（约 4.2k 字符 ≈ 1.4–1.8k tokens，实测估算）：别让验证者加载本 SKILL.md 全文
> （约 81k 字符 / 约 125 KB ≈ 27–34k tokens）。分派判据：**数值门归脚本跑**（`audit_*` 支持 `summary_only` / `top_k` 瘦身；
> 实测 40 对象场景 9,176 → 1,148 字符且 `clean`/`totals` 判定不变），**只有主观项**（像不像 / 构图 / 文案）
> 才需要陌生视角。反 Goodhart 的"换一条计算通路"写在清单里；单次验证固定开销 ≈27–34k → ≈8.6–9.0k。

## 0.5 多 Agent 并行建模的分工范式（★ 做大件必读）

**派子代理前（必做）**：子代理看不到你读过的目录 —— 把 `blender_rt_plan(op="catalog", args={handoff:true})` 的输出
**原样粘进它的提示词**（≈2.5 KB：15 个工具 + 12 类建模的第一步/禁止自造 + 三条硬规则）。
不粘的实测后果：子代理直接写 Python 自造放样、自写判据，插件的 vehicle_*/shape_* 全白给。

> 单人单线捏一个物体，配方够用；**造一台机器**（几百个对象、多个 builder）时，决定成败的不是捏形状，而是**分工协议**。
> M1A1 实测：500 对象 / 7 个 agent，**装配质量几乎全靠这一章**。

### 0.5.1 第一步永远是冻结规格（`lib/spec.py`）

在任何人动手之前，Lead 先写 `lib/spec.py` 并**冻结**（尺寸 + 接口表 + 命名 + 材质映射 = 唯一真源）：

```python
# lib/spec.py —— 唯一真源。builder 只读，不许各自定义常量。
TOTAL = {"length": 9.83, "width": 3.66, "height": 2.88}      # m（口径写清：含炮管 / 含裙板）
WHEEL_R, WHEEL_W, WHEELBASE = 0.33, 0.20, 4.60
DECK_Z, HULL_BOTTOM_Z = 1.62, 0.42
PARTS = {                                                     # 命名 → 归属 → 写作用域
    "GEO-hull":    {"owner": "hull-dev",    "file": "lib/hull.py"},
    "GEO-turret":  {"owner": "turret-dev",  "file": "lib/turret.py"},
    "GEO-rungear": {"owner": "rungear-dev", "file": "lib/rungear.py"},
}
INTERFACES = [                                                # 接口表：拼件之间的"合同"
    {"a": "GEO-turret_ring", "b": "GEO-hull_deck_ring", "normal": (0, 0, 1),
     "plane_z": DECK_Z, "overlap_mm": 12, "tol_mm": 2},
]
```

**没有冻结规格就分件 = 装配必崩**：炮塔和车体各按各的理解建，接口对不上是必然，不是运气问题。

### 0.5.2 写作用域协议（谁写哪个文件）

- 每个 builder **只写**自己那条线：`lib/<part>.py` + `work/<part>_test.py` + `qc/<part>_*`。
- `lib/spec.py` **只读**：要改规格 → 找 Lead 改，改完**通知全员重新量**（旧数字全部作废）。
- 装配层 `lib/assemble.py` 归属**单独一个人**（Lead 或专职 assembler）；builder 不许碰。
- 共享文件系统**没有锁**：两个人同时写一个文件，后写的覆盖前面的。守界就是纪律。

### 0.5.3 接口表怎么定（拼件的"合同"）

每条接口写清五件事：**名义面**（plane + normal + **真实接口的局部坐标系**）、**本 spec 明文写出的隐藏交叠量 `overlap_mm` 及其依据**、**公差**、**朝向语义**（谁的局部轴朝哪）、**接口特征尺寸**（轴径 / 板厚 / 最小壁厚）。
这一条把"贴合面只能靠猜"变成"按本 spec 明文的交叠量拼、装配后按 spec 复检"。**spec 没写就回去问规格**（坑二）：可视脚本不发明深度倍数或下限，也不给结构强度结论。

### 0.5.4 并行期无法互测 = 固有代价，用"隐藏交叠 + 装配后复检"消化

并行时邻件还没建出来，你**物理上不可能**知道真实贴合面。正确做法：

1. 按接口表的**名义面做隐藏交叠 `overlap_mm`**（**该接口自己的值，来自本 spec 的明文**；藏在较大件内部，**不改变可见轮廓、不捅穿薄壳**）；
2. 交付时写清"我按名义面 X 做了 N mm 的隐藏交叠（spec 出处：…），**实际面未验证**"；
3. 装配层跑 `audit_interference` / `audit_measure` 一次性收敛所有偏差。

**不要为了"看起来贴合"去猜邻件的实际形状** —— 那只会让偏差集中到装配时爆发。

### 0.5.5 单件自证 → 装配门（用数字代替肉眼）

**门的清单只有一份：§6 的十道门表**（含命令与字段）；本节只管**多 agent 交接**特有的三件事：
**单件自证与装配门别混**、**回执要贴原文**、**判据要"红过"才算数**。

- 单件自证 ⇒ `audit_mesh`（门①–④）；装配口径 ⇒ `audit_gate`（门⑥）。
- 命令原样：见 **§6.1**（三条必跑）；字段判据：见 **§6** 表。
- 跨文件批处理：`audit_mesh` / `audit_gate` / `audit_measure` 收 `args={"file": "D:/.../parts/hull.blend"}`，
  `audit_interference` 收 `file_a` / `file_b`。

**验收回执模板**（复制进交付清单；数字必须来自回执**原文**，不许转述、不许手写）：

```text
【交付回执】<件名 / agent>   <ISO 时间>
audit_mesh   objects=[...]  state=pass  verdict=pass  clean=true
  totals: boundary_edges=0  nonmanifold_edges=0  degenerate_faces=0
          loose_verts=0  loose_edges=0  self_intersections=0  normals_inverted=0  normals_unknown=0
  objects[i]: closed=true  normals_state=pass  normals_outward=true
              normals_shells=[{shell:0, faces=…, state:pass, signed_volume:<值>, nested:false}]
              # ★ 逐壳判：别拿"总 signed_volume>0"当充分条件（大正向壳 + 小反向壳会抵消成正数）
audit_scene  state=pass  self_intersections_analyzed=true     # ★ 必须显式 args={"self_intersect": true}
audit_gate   scope=COL_<agent>  state=pass  ship_ok=true
  verdict_line="..."
audit_interference  a=... b=...  verdict=refuted(无干涉)  volume_mm3=0   # 有干涉写 supported + volume_mm3
audit_measure  items[].size_mm 对 spec 偏差 ≤ ±3mm
               pairs[<a>|<b>].axis_overlap_mm.<接口法线轴> = <实测> ≥ <本 spec 明文写的 overlap_mm>
               轴选择：接口法线与世界轴一致且轴对齐盒适用 ⇒ 可作粗筛；旋转接口须沿接口局部坐标量真实表面
               依据：本 spec 的明文（示例数字只是写明它的那份示例 spec 的取值，不是通则）
               同对件 audit_connectivity 未报浮空：<是>
.blend md5=<md5>          # WSL: md5sum <file> ； Windows: Get-FileHash -Algorithm MD5 <file>
fingerprint.digest=<12 位>
```

> ⚠ 交叠那一行**只写本 spec 的值，不写通则**：老模板写 `pairs[].axis_overlap_mm ∈ [5, 15]` 是错的；
> v4/v5 换上的"`1–2×` 接口特征尺寸"**同样是规范性固定倍数，已被 v6 撤回**（见 §13）。
> 数字（例：3mm 板搭接 1.5mm）只在**写明它的那份示例 spec 里**才算数，换项目换尺度要重新由设计定。
> 现在必须写**"接口局部坐标下的量测轴 + 本 spec 的 `overlap_mm` + spec 出处"**，并附连通门未报浮空的确认（坑二 ③）。

> ⚠ **三态别读成二态（v0.9.6）**：`audit_*` 顶层现在给 `state`（= `verdict`）三态 ——
> `pass` / `fail`（确有缺陷）/ `degraded`（**没查完或判不出来**）。`clean` 收紧为"**查完了而且没缺陷**"：
> 自交检查被跳过或抛异常、`normals_state=unknown`、（**显式设了上限时**）超面数上限的逐壳分析，都会给 `clean=false` + `state=degraded`；默认不设上限、全跑。
> ⇒ **`clean=false` 有两种成因**（确有缺陷 / 没查完），必须看 `state` 与 `reason` 才下结论；
> `degraded` 的处置是**补查**（见下），不是"勉强算过"。

**三条硬纪律（都是实测踩出来的）**：

1. **单件自证用 `audit_mesh`，`audit_gate` 是装配口径**：一个完全干净的孤立立方体 `audit_mesh state=pass clean=true`，但跑 `audit_gate` 会 `state="fail"` —— 因为"整个模型就是唯一分量、谁也不接"（实测）。要跑 gate 就给它**≥2 件应当相接**的集合。
2. **判据要"红过"才算数**：拿一个已知三缺陷件（开边 / 浮块 / 穿模各一）跑一遍，三门必须红。实测：开边立方体给 `boundary_edges=4, nonmanifold_edges=4, closed=false, clean=false, state=fail`；5cm 浮块被 `audit_connectivity` 记成 `floaters` 且 `audit_gate state="fail"`；重叠 0.5m 的两块给 `audit_interference verdict="supported", volume_mm3≈4.97e8`。
3. **`ok=false` / `analyzed=false` / `state="degraded"` / `clean=false` 都不是通过**：0 个 mesh、文件打不开、全部隐藏 ⇒ `ok=false`（静默空产出必须失败）；`analyzed=false` 不能读成"已连通 / 无重叠"；`degraded` 与"`clean=false` 但 `state=pass`"这类组合不存在 —— 见到 `degraded` 就去补查（`audit_scene` 传 `self_intersect=true`；`audit_mesh` 看 `reason`/`normals_state`/`hint`），补完再下结论。

**怎么验**：交付清单里能贴出上面那张回执的**原始字段值**；三缺陷件能复现"三门红 → 修好后全绿"。

**装配层最后统一 recalc 法线**（`bpy.ops.mesh.normals_make_consistent(inside=False)` 或 bmesh 等价）——
见坑五：法线是**所有**穿模 / 内含判据的前提，装配层必须兜底一次。

### 0.5.6 分工反模式（实测踩过的）

- ❌ 两个 builder 写同一个文件 → 静默覆盖。
- ❌ 7 个 builder 共用 `blender_rt_worker` → 单例串行 + 场景互相污染。
- ❌ 每个 builder 各自定义尺寸常量 → 装配对不上；常量必须在 `spec.py`。
- ❌ 用"看起来对"验收 → 必须给数字（bbox / 间隙 / 面数 / 按 spec 的隐藏交叠量）。
- ❌ builder 自己跑装配 / 导出 → 那是装配层的事。
- ❌ 拿单张成品图当形体证据 → 必须固定机位对照图（§0.6）。

### 0.5.7 可并行性判据与归属标签（P1-5）

**先判"能不能并行"，再谈"怎么分"** —— 三个 builder 同时改同一个东西时，任何协议都救不回来。

| 判据 | 能不能并行 | 依据 / 处置 |
|---|---|---|
| 两个 builder 写**同一个文件**（`.py` / `.blend`） | ❌ 不可并行 | 文件系统没有锁，后写的覆盖前面的（§0.5.2）→ 拆文件，或串行 |
| 两个 builder 改**同一个对象**（哪怕改不同属性） | ❌ 不可并行 | bpy 是单场景单主线程；对象级回滚也按对象记账，两个 owner 会互相打回 |
| 两个 builder 改**同一份材质 / 同一个 node group** | ❌ 不可并行 | 材质是共享 datablock，改一次全体变 → 一人一份材质副本，装配期再合并 |
| 共享**只读**输入（`spec.py`、接口表、参考图） | ✅ 可并行 | 只读不产生写冲突 |
| 各自的 `lib/<part>.py` + 各自的 `COL_<agent>` | ✅ 可并行 | 默认推荐形态 |
| 各自的 headless 进程 / `rt_job` 作业 | ✅ 可并行 | 每次 spawn 独立进程（实测 512² 渲染 2 并发各只慢 ~10%） |

**归属约定**（写进 `spec.py` 旁边的公共工具，别只写在脑子里）：每个 builder 一个 `COL_<agent>` 集合 + 每个对象一个自定义属性 `dsh.owner=<agent>`。

```python
# lib/own.py —— 所有 builder 共用；建对象时立即登记归属
import bpy

def own(obj, agent):
    obj["dsh.owner"] = agent                      # 对象自定义属性：可被筛出来，装配期可追责
    col = bpy.data.collections.get("COL_" + agent) or bpy.data.collections.new("COL_" + agent)
    if col.name not in bpy.context.scene.collection.children:
        bpy.context.scene.collection.children.link(col)
    for c in list(obj.users_collection):          # 只留在自己的集合里（避免同一对象被两个集合"共有"）
        c.objects.unlink(obj)
    col.objects.link(obj)
    return obj

def owned(agent):                                 # audit_* 不收 owner 参数 —— 自己筛成名字列表再传
    return [o.name for o in bpy.data.objects if o.get("dsh.owner") == agent]
```

装配前逐集合过门：`blender_rt_plan(op="audit_gate", args={"scope": "COL_hull-dev"})`；
要按 owner 精确取件：`blender_rt_plan(op="audit_mesh", args={"objects": owned("hull-dev")})`。

**只回滚一个 builder**（对象级 mark/revert；毫秒级，实测只动被标记的对象）：

```text
blender_rt_txn(op="mark",   label="hull-iter7", objects="GEO-hull,GEO-hull_plate_01")   # ← objects 是**逗号分隔字符串**
... 改这一批对象 ...
blender_rt_txn(op="revert", label="hull-iter7")     # 只把这批打回 mark 时刻
```

实测回执：`{"ok":true,"reverted":"hull-iter7","changed":["PROBE_t1"],"changed_count":1,"missing_objects":[],"blocked":[]}`
—— **没被标记的对象一动不动**（另一个对象的位移保持改后的值）。
边界（回执里 `boundary` 字段的原文口径）：只覆盖 **transform / 材质槽 / 可见性 / 修改器开关**，
**不含拓扑、UV、顶点位置**；布尔 / 删面 / 合并这类改动先用 `blender_rt_txn(op="snapshot", label=...)` 留文件级回退点。

**怎么验**：mark 两个对象 → 改三个对象 → revert ⇒ 回执 `changed_count=2` 且第三个对象的值**没变**；
`blender_rt_txn(op="marks")` 仍能列到这条 mark（除非显式 `op="drop"`）。

## 0.6 参考图驱动的形体还原（"像不像"的关键）

> 实测：炮塔外形不是"捏"出来的，是**从参考图量出来的**。"裙板下沿 / 甲板高 = 48.2% vs 规格 48.4%"这种对拍才是形体验收。

### 0.6.1 三视图标定（正视 / 侧视 / 顶视；先判"能不能按比例量"，再判"能不能联立"）

单视图标定的死穴：**只要这张图带一点透视，"一个 px_per_mm"就是假的** —— 物体不同深度的比例不同，你量得越远越不准。

> ★ **纠正一条常被写错的口径**：正交三视**并不**"必须共享同一个 `px_per_mm`"。
> **正交只保证"每个视图内部"px 与 mm 呈线性**（视内残差≈0）；三张图完全可以各自有不同的
> **出图比例 / 分辨率 / 裁剪**，却都是严格正交。
> 实测（Blender 5.2.2，`world_to_camera_view` 直接量）：三个正交相机在 `ortho_scale` 4/8/2 m 下，
> **视差探针全为 0 px**（确实是正交），而 `px_per_mm` = **0.48 / 0.24 / 0.96，互差 300%**。
> ⇒ "三视共享同一比例"是一条**额外假设**（三视出自同一张工程图 / 同一次渲染），要**单独声明并验证**，
> **不能从正交性推出来**。把 `px_per_mm` 不一致读成"透视"，会让你去改本来没问题的取点。

**可运行脚本：`references/multiview-calibration.md`（`multiview_calib.py`，纯标准库，自带 `--selftest`）。**

**① 找标尺、逐视独立最小二乘**：每视取 **≥3 个已知特征**（轴距 / 轮径 / 总长 / 炮管长），量出它在本视**图像横轴方向**的毫米长 `u_mm` 与像素长 `px`，各视**各自**解

```text
px = cx + s · u          # s = 「该视自己」的 px_per_mm；cx = u=0 落在哪一列
主点偏移 = cx − 图宽/2    # 理想情况接近 0；偏得离谱说明取点或裁剪有问题
```

> ★ **两点解出来的直线必然过这两点（残差恒 0）** —— 想判"这张图正不正交"，**每视至少 3 点**。

**② 跨视互检要换算成毫米再比**（不是比 `s`）：再给**至少一条共享尺寸**（在两个及以上视图里都能量到的真实尺寸，如全高 / 总长），
把它的像素量测用**各视自己的 `s`** 换算成毫米，看两视给出的毫米值是否吻合。这一步与出图比例无关，所以它才是真正的互相印证。

**③ 判据（三条各管一件事，别混）**：

| 判据 | 阈值 | 它判的是什么 | 超限的处置 |
|---|---|---|---|
| 视内残差 | **> 2%** | 该视**内部**线不线性 ⇒ **真的有透视 / 非线性**，或特征点量错 | 该视不可按比例量 |
| 共享尺寸互差（**换算成 mm 后**） | **> 3%** | 两视对**同一条真实尺寸**对不上 ⇒ 两视互不相容 | 不要联立；逐视换算并标 `unresolved` |
| 三视 `px_per_mm` 互差 | **> 1%** | 三视**不共享同一出图尺度** | **≠ 透视** —— 逐视各自标定即可；只有另有"同尺度"依据时才合并 |

**没有共享尺寸 ⇒ 三视之间无法互相证明**，结论只能是 `unresolved`（不要默认它们一致）。

**④ 只有"同尺度"成立时才联立**：取按 `sxx` 反方差加权的**联合 `px_per_mm`**，把三视的像素量测统一换算成毫米写进 `spec.py`，**附来源**（图名 + 像素坐标 + 标尺）。
自证口径：自己用正交相机渲的图有真值 —— `px_per_mm = 渲染宽度px / ortho_scale_mm`，拿它跟联合值对拍（目标误差 ≤1%）。
⚠ 这条只在**同一个正交设置**下成立：换 `ortho_scale` 或换分辨率，`px_per_mm` 就跟着变（见上面的实测）。

**⑤ 三视不同尺度 / 单视标定怎么量**（判了"不能联立"之后不许再假装三视是一家）：

1. 每视只用**自己的标尺**定 `s`，只换算**与本视标尺同深度、同方向**的尺寸；
2. **跨深度的尺寸：误差不是固定 ±5%** —— 透视下比例按 `1/z` 变化，误差上界是可算的：

```text
err ≈ Δz / (D + Δz)      # D = 相机到近处标尺的距离，Δz = 两点深度差
```

实测（D = 8 m，Blender 5.2.2 投影量测，与解析式逐行吻合）：Δz = 0.5 m → **−5.88%**；Δz = 1 m → **−11.1%**；Δz = 3 m → **−27.3%**。
⇒ **只跨 0.5 m 就已经超出 ±5%**。先算这个上界；**估不出 `D` 或 `Δz` 就记 `unresolved`**，不要顺手写"±5% 应该够"。
想要真按比例量 ⇒ 让被测特征与标尺**同深度**，或对参考图做**真正的相机标定**（内参 + 外参）。

**⑥ 照片先校正，但单应只校正"一个平面"**：四点单应对**恰好落在取点那个物理平面**上的点严格成立，
离开该平面的点误差随深度迅速放大。实测（在 z=0 平面拟合单应，1 m ≈ 190 px）：平面上误差 **0 px**；
离平面 0.05 m → **52 mm**；0.2 m → **211 mm**；0.5 m → **543 mm**。
⇒ **平面参考**（图纸翻拍 / 标定板 / 贴平的贴纸）完全正确；**立体物**（车 / 人 / 建筑）只有取点那个平面准，
离它越远的特征（轮眉外扩 / 翼子板弧度 / A 柱前倾）误差越大 —— 四点必须是**同一平面上的真实矩形**，
且只取靠近该平面的特征量测；要整个立体物都准就得走相机标定，不是单应。细节见 `references/multiview-calibration.md` §6。

**怎么验**：`python3 multiview_calib.py --selftest` 四个合成用例必须分别给出
"A 联立 / B 视内残差红 / C 分尺度逐视可用 / D 跨视互检红"（末行 `SELFTEST PASS`）；
核心回归是 **C**：三视 `px_per_mm` 差 100% 而视内残差≈0 时，判据必须输出"不共享比例、**逐视标定可用**"，
而**不是**误判透视。

### 0.6.2 双证据验收（缺一不可）

- **数值**：`audit_measure` 出 bbox / 逐轴间隙，与 `spec.py` 对拍。
- **画面**：`qc_render_views` 用**与参考图相同的机位**出图，叠着看。

单张"好看的成品图"**不能**验收形体 —— 前裙甲上翘就是这么漏掉的：只有固定机位对照图 + bbox 数字才抓得出来。

#### IoU 是什么、不是什么（★ 别把它当"整体像不像"）

`qc_render_views(ref_path=…)` / `img_diff` 给的 **IoU 是"该视图下轮廓（前景掩膜）的交并比"**，
是一个 **2D、单视、只看轮廓** 的指标。它：

**能**判：该视图下**外轮廓重合度**（包络对不对、长度高度对不对）。
**不能**判（全都是它的盲区）：

| 盲区 | 后果 |
|---|---|
| **深度/厚度** | 侧视 IoU 0.9 的车可能薄了一半 —— 轮廓一样，纵深全错 |
| **内部特征**（板缝 / 格栅 / 灯 / 面罩 / 特征线位置） | 轮廓内一片平也照样 0.9+ |
| **曲面张力 / 面感** | IoU 高但"看着不像"正是这一类（Recipe 18 硬规则⑤） |
| **其他视图** | 侧视 0.9 对顶视一无所知 |
| **体积/质量分布** | 轮廓相同、前后配重相反，IoU 不变 |

⇒ **IoU 是"必要不充分"条件**。纪律：

1. **每个视图分开算**，**逐视各自达标**（侧 / 前 / 顶都要，别用一个数代表整体）；
2. **IoU 达标只能推翻"轮廓错"，不能证明"像"** —— 还要叠上：`clear_check` 的成对间隙、
   `audit_measure` 的逐轴尺寸、以及**内部特征线**的对照（`parts_color` 图 + 剖面差）；
3. **不要把 IoU 当优化目标去凑**：把轮廓填满就能刷高 IoU（比如把车窗全部堵上）。
   一旦发现"IoU 涨了但图更难看了"，就是这个 Goodhart 陷阱 —— 回去看特征线；
4. **IoU 低一点但该有的特征都在**，通常比"IoU 更高但特征被抹平"更接近参考图。先判"哪一层错"
   （包络错 → 改包络；轮廓错 → 改站表；面感错 → 改特征线/圆角，见 `references/参考图通用还原SOP.md` §4）。

### 0.6.3 读图纪律（决定你烧多少 token、以及会不会"看了等于没看"）

**① 结论三分**：读图结论必须三分：【**实测看到的**】【**推断的**】【**我无法判断的**】。比例、遮挡关系、法线朝向这三类最容易"看着像就下结论"。

**② 尺寸纪律（P0-5）**：两种场合，两个尺寸，不要混用。

| 场合 | `max_size` | 理由（实测） |
|---|---|---|
| 迭代期"改一步看一眼" | **420** | 420px 单帧 **79,428 B**（PNG，附件约 17 KB）；560px 单帧 116,870 B —— 迭代要看几十次，这里省的是大头 |
| 对照验收 / 出报告 | **900** + **固定机位** | 要和参考图叠着看、要能看出 1–2mm 板缝；一张就够，别连拍 |

`blender_rt_see` 给 `from/look_at/lens` 就是自定义视角（不动物体、不动用户视口）；**验收机位写进 `spec.py`**，每次都照抄 —— 机位一变，"像不像"就没法对拍。
用 `blender_rt_plan(op="qc_render_views")` 出对照图时同理：`views` 数组与 `res` 固定，回执/JSONL 逐张给 `hash`（= 该 PNG 的 md5）。

**③ hash 纪律（最重要的一条）**：每张帧回执都带 `hash`（PNG 字节的 md5 前 8 位）。
**hash 与上一次相同 = 画面逐字节没变** ⇒ **不要再看图、也不要再判读**，直接去看数值（bbox / 审计字段）。
实测：同一个视口连发 5 次，回执 hash 全等（本机 420px 连测 5 次全是 `c2b9b1d1`）；这不是"图被缓存了"，是画面真的没变。

> v0.9.4 起插件对"同一通道 + 同一 hash"的帧**默认不重复附图**，文本会写「**与上一张完全相同**（第 N 次）⇒ 未重复附图；要重发传 `force:true`」。
> 若你的回执**仍然带了图**，说明当前进程跑的是旧版插件 —— 纪律不变：**自己按 hash 判断，同一个 hash 不重复判读**。

**怎么验**：一次迭代里"取图次数 ≤ 实际改动次数"（每个新 hash 最多看一次）；验收期每张对照图的 `max_size=900` 且机位与 `spec.py` 里那组数逐字一致。

## 0.7 建模类型 → 工具指路（★ 动手前先查这张表）

**先查类型，再查能力族**：`blender_rt_plan(op="catalog", args={classes:true})` 一次拿全部 12 类
（第一步 / 算子链 / 禁止自造 / 验收门）；单类 `args={class:"recon"}`。本表是同一份数据的人读版。

| 建模类型 | 第一步 | 算子链 | 禁止自造 | 验收门 |
|---|---|---|---|---|
| 参考图还原（有外轮廓的物体） | `shape_plan(object_class=…)` | `img_rectify → shape_sections → shape_loft`（要改数字用 `shape_fit`）`→ crease_lines/inset_lines → shape_regions/shape_panels → qc_render_views(ref_path=)` | 别自造放样、别手算站表（自造 = 一张连续光滑面，没棱没缝） | 比例门 + 三视图 IoU + 板缝/特征线计数 > 0 |
| 分面硬表面（装甲/军模/板件） | Recipe 2a | 基本体/板件 + 布尔 → `bevel`（2–3 段、保硬边）→ `audit_mesh` | 别用 SubSurf 抹平棱；别靠缩放假装板缝 | 锐边/法线 + 板缝可见 + 无浮块 |
| 光滑硬表面（钣金/圆润道具） | Recipe 2b | 基本体 → Bevel(2) → SubSurf(2) 或 `crease_lines(radius_mm=…)` → `audit_mesh` | 别手推顶点做曲率；别用 shade smooth 假装圆角 | 无褶皱/自交 + 特征线 > 0 |
| 有机 / 角色 / 道具 | `sculpt_scan` | `sculpt_setup(VOXEL) → sculpt_apply(brush…) → sculpt_filter/remesh → audit_connectivity` | 别用布尔拼体积；无头别试 `bpy.ops.sculpt.brush_stroke` | 连通 = 1 + 包络 + 法线朝外 |
| 重复阵列（履带/链节/散热片） | `generator_save` 或 Recipe 6b | `generator_run(args=…)` 或 Array+Curve → `audit_interference` | 别复制粘贴硬编码件（改参要改码） | 件数/节距/间隙门 + 零互穿 |
| 布尔开孔（轮眉/散热孔/减重孔） | `destructive_guard` | 布尔 → `fix_repair` → `audit_interference` | 别在未判别的连接上布尔 | 零互穿 + 零面积面 = 0 |
| 人形素体 / 头型 | `human_*`（Recipe 15/16） | 素体/头型 → `sculpt_*` 细修 → `audit_mesh` | 别自造人体比例表 | 比例表 + 连通 = 1 |
| 管路 / 线缆 / 轨道 / 护栏 | `sweep_analyze` | `sweep_build` → `audit_connectivity` | 别手接路径段 | 弯折半径 > 型材半宽 |
| 旋转体（瓶/罐/轮毂/花瓶/喷口） | `shape_plan(object_class="rotational")` | `shape_revolve(parts=[…])` → `print_report` → `material_*` | 别放样 —— 轴对称走放样是错的路径 | 回转轴向偏差 + 壁厚 |
| 板件/型材/翼型（有弯度或不对称） | `shape_plan(object_class="aircraft"/"hull")` | `shape_sections → section_outline`（或超椭圆 + 特征线）`→ shape_loft` | 别用左右镜像表达弯度（翼型必选 `section_outline`） | 剖面偏差 + IoU + 特征线 |
| 装配 / 机构 / 铰接 | `gate_plan(preset="assembly")` | `mate_check/fit_help → motion_joints → motion_measure → motion_export_urdf`；总验收 `gate_run(spec_path=…)` | 别自写验收判据 | `gate_run` 的 verdict 三态（degraded ≠ 通过） |
| 打印可行性 / 交付 | `print_report` | `uv_* → material_* → material_bake → deliver_export → deliver_verify` | 别手写导出、别只交 .blend | manifest md5 + `resolution_mm` 警告 + UV 零面积面 = 0 |

**三条硬规则**（都是实测踩出来的，不是洁癖）：

1. **有参考图就必须先用 `shape_plan` 拿 `section_path`** —— 截面通路选错（该 `section_outline` 却用镜像对称）是**系统性失真**，不是细节问题；
2. **车壳/外壳类禁止自造放样** —— 自造只能出一张连续光滑面；板缝与棱线必须靠 `vehicle_panels` + `crease_lines` / `inset_lines`（Recipe 22）；
3. **验收一律用机械门**（§6 数值门 + `gate_run` 三态）—— "看起来像"不算通过，`degraded` 也不算通过。

### 0.71 调用约定 · 离线逃生口（S9 补）

**API 两种调用形态（返回类型不同，别混）**：

```python
api = K.dsh_audit_api            # 或 K.dsh_kit.kapi("audit")（找不到会列出已加载模块）
r1 = api("scene", {...})         # → **dict**（已解析；文档/skill 示例都是这种）
r2 = api["dispatch"]("scene", json.dumps({...}))   # → **JSON 字符串**（引擎契约；要自己 json.loads）
```

- 你自己的 `lib/*.py` 里想用 `K`：K 只注入「被执行脚本的 globals」⇒ **import 进来的模块有自己的 globals**。两种官方写法：
  `import sys; K = sys.modules["dsh_rt_kernel"]`（一行）或 `K.dsh_kit.install_kernel()`（把 K 与 kapi 装进调用方 globals）。
- `preload` 用 **runtime/<name>.py 的文件名**（如 `preload="vehicle,audit"`）；写错会报错并列出可用模块名。

**离线逃生口（后端挂了也能干活）**：

```bash
blender -b --factory-startup --python runtime/offline_bootstrap.py -- vehicle audit qc
```

或脚本里 `import offline_bootstrap as ob; ns = ob.boot(["vehicle","audit"]); ns["kapi"]("vehicle")("spec", {...})`。
契约与在线路径**完全一致**（同一个 `K.dsh_<name>_api`）。

**`audit_gate` / `audit_connectivity` 的口径（装配必读）**：默认判据是**「单体船」**（`micro_gap_mm` 默认 0.3 mm）——
222 个独立零件的**正常**装配必然被报成"可见浮块"。多零件请二选一：① 调 `micro_gap_mm`（带设计间隙的装配 1–2 mm）；
② 走 `gate_plan(preset="assembly")` → `gate_run`，别拿单体口径当装配门。

## 1. 六步总纲

```
0 冻结规格   → spec.py（尺寸 + 接口表 + 命名）★ 多 agent 时这一步是硬前置（§0.5）
1 场景准备   → 集合分层、比例参照、命名规范
2 Blockout   → 基本体摆大形（可随时推翻）
3 粗形精修   → 编辑模式 / 修改器，把比例做对
4 修改器栈   → 非破坏性地加细节（倒角 / 细分 / 布尔…）
5 清理       → 合并重复点、法线、流形、松散几何（数值门见 §6）
6 交接       → 交给 UV / 材质 / 渲染，或导出（装配层统一 recalc + 审计）
               ★ 交接前**必跑** §0.5.5 的验收门清单，并贴出验收回执（boundary/nonmanifold/loose/self-intersections/closed + .blend md5）
```

要点：
- **比例在低模阶段解决**。加细分只让表面更光滑，**不会**改善造型是否像。
- **非破坏优先**：布尔保持 live 到导出前；拓扑定型再 apply 镜像。
- **Blockout 与成品分开**：轮廓没被认可前不要删 blockout。
- 每一步都留下可对照的固定相机画面，别只存最后一张好看的图。

集合结构建议（大场景时）：

```
COL_Project
├── COL_Reference      # 比例参照、正交参考图
├── COL_Blockout       # 临时大形
├── COL_Geo            # 最终几何
├── COL_Collision      # 代理网格
└── COL_Instances      # 实例化用的空物体 / 集合
```

命名：立即给名字，**永远不要留 Cube.001**；本技能统一用 `GEO-` 前缀（如 GEO-blade、GEO-guard）。
**多 agent 时命名就是接口**：名字一旦进 spec.py 就不能改（改名字 = 改所有人的引用）。

## 2. 决策树：什么形状走什么路子

```
要做什么几何？
├── 硬表面 · 分面装甲 / 军模 / 机械板件（有明确棱边、板缝）
│   → 平面板 + 小倒角（Bevel 1–3mm, 1–2 段, ANGLE 30°）＋**平坦着色** → 配方 2a
│   ✗ 不要上 SubSurf：它会把装甲棱边全部圆掉，一眼就从"军模"变"塑料玩具"
├── 硬表面 · 光滑曲面（汽车钣金 / 消费电子 / 圆润道具）
│   → Cube + Bevel + SubSurf 栈 → 配方 2b
├── 有机（角色 / 生物 / 植物——只做粗形，或要真雕出形体）
│   → Ico Sphere（统一三角面，适合雕刻与体素重构）
│   → 要"雕出形体"：走**程序化雕刻**（体素定底模 → 位移笔刷 → 遮罩限定 → 滤镜平滑）→ 配方 11
│   ✗ 手势笔触（bpy.ops.sculpt.brush_stroke）在 Blender 5.2 上 Python 构造不出 stroke 集合
│     （RNA 拒绝 dict；元素无法实例化）—— 别去试，用配方 11 的位移笔刷
├── 建筑 / 重复（栅栏、柱子、瓷砖）
│   → Plane/Cube + Array（+ Curve 沿路径）→ 配方 6
├── **等距重复 / 必须精确闭合**（履带、链节、拉链、齿圈）
│   → **参数化路径求解**（用**销轴弦长**口径，不是等弧长）→ 配方 6b
│     （本配方口径下的 Array 只能"按曲线装满"，给不了精确弦长节距 + 闭环；直线等距阵列仍可用 Array）
├── 管状（管道、柱子、瓶子）
│   → Cylinder，或 Curve + bevel_object
├── 在已有几何上开孔 / 切
│   → Boolean DIFFERENCE → 配方 4
└── 只要快速布景（大形阶段）
    → 多个基本体直接摆 → 配方 7
```

基本体怎么选：**Cube** 硬表面主力；**Ico Sphere** 有机／雕刻底模（三角面最均匀）；**UV Sphere** 要贴图的圆物（自带好 UV）；**Cylinder** 机械件；**Cone / Torus** 尖角与环。

## 3. 配方（可直接跑）

> 下面代码块是**在 Blender 里执行的 Python**。交互式用 `blender_rt_do(code=…)` 跑；批处理用 `blender_rt_headless(script=…)`（那时记得在开头 import bpy，且**变量不跨调用保留**）。
> ⚠ **headless 下 bpy.ops 的 context 很脆**（详见坑四）：批处理脚本优先用 **bmesh / data API**，配方里同时给了两套写法。

### 3.0 配方 = 参数块 + 生成脚本（P1-4，接插件已有的 `generator_*`）

高频复用的 5 条配方（**2a 分型面 / 4 布尔 / 5 镜像 / 6b 节距阵列 / 9 装配门**）不要每次复制粘贴改数字：
把数字拎成 **参数块**，把配方变成 **生成脚本**，用插件已有的生成器注册表管版本与复现。**不另造体系。**

| 动作 | 真实调用（回执字段照抄） | 关键回执 |
|---|---|---|
| 注册 | `blender_rt_plan(op="generator_save", args={"name": "rib_array", "code": CODE, "params": {...}, "note": "..."})` | `path` / `sidecar` / `hash`；写 `<root>/<name>.py`（`root` 默认 `~/dsh_generators`，`dir` 可覆盖） |
| 复现（不改参） | `blender_rt_plan(op="generator_run", args={"name": "rib_array", "expect": {"min_objects": 5}})` | `verdict` 三态 + `receipt{PARAMS, objects, names, tris, bbox}` |
| 复现（**改参**） | `blender_rt_plan(op="generator_run", args={"name": "rib_array", "args": {"n": 12, "pitch": 0.02}, "expect": {"min_objects": 12}})` | 同上；`args` 这一层**就是**生成器的 `PARAMS` |
| 看版本 | `generator_list` / `generator_get` / `generator_diff` | `stale=true` = 源码在最后一次运行后被改过 ⇒ **那次回执不能再当证据** |

**改参复现（首选工具直调）**：

```python
# ★ 直接调工具：args 键就是生成器 PARAMS（嵌套一层）
blender_rt_plan(op="generator_run",
                args={"name": "rib_array", "args": {"n": 12, "pitch": 0.02},
                      "expect": {"min_objects": 12}})
# 回执：verdict 三态 + receipt{PARAMS, objects, names, tris, bbox}；核对 receipt.PARAMS 与 args 一致
```

```python
# 备选（仍在，适合"同一段 Python 里连跑多次 / 要读中间量"）：走内核
import json
r = K.dsh_generator_api["run"](name="rib_array", args={"n": 12, "pitch": 0.02},
                               expect={"min_objects": 12})
r = json.loads(r) if isinstance(r, str) else r
print(r["verdict"], r["receipt"]["objects"], r["ms"])        # supported 12 808
```

> ⚠ **旧版迁移说明（v0.9.5 及更早）**：那时的 dispatcher 把 payload 里的 `args` 键**无条件摊平**成顶层 kwarg
> （`{"name": ..., "args": {"n": 12}}` → `generator_run(name=..., n=12)`）⇒ 报
> `generator_run() got an unexpected keyword argument 'n'`（实测原话），而且 PARAMS 会变成空 `{}`。
> **v0.9.6 已修**：`op="run"` 时 `args` 是真参数、原样传下去（规则写在 `generator.py::generator_dispatch` 的 docstring 里）。
> 在旧插件上就照上面那段**内核写法**，或升级插件 —— 旧写法在新插件上依然可用，不会失效。
> `K.dsh_generator_api` 要先由一次 `blender_rt_plan(op="generator_*")` 调用装进内核；headless 里用 `preload="generator"`。

**脚本契约**：生成脚本里读 **`PARAMS`**（dict）；建议最后 `print("DSH_RECEIPT " + json.dumps({...}))`（`json` 要自己在脚本里 `import`；实测可用）；不打印也行 —— 运行器会自己扫场景出回执。
**生成脚本必须"从零建件"**（程序即形状）：`clean_scene` 默认开，脚本里**不能**用 `bpy.data.objects['GEO-x']` 去捞现成对象 ——
它是**全新无头进程**，只有 `PARAMS` + bpy/bmesh/mathutils，**没有 `K`、也没有 `audit_*` 模块**（实测：脚本里引用 `K` 直接 `NameError: name 'K' is not defined`，verdict=`refuted`）。
需要 audit 的配方（如 9）走 `blender_rt_headless(script_file=..., preload="audit")`，生成器只用来管版本与 diff。

**三态 verdict**：`supported`=预期满足 / `refuted`=预期不满足（差异列在 `why` 里）/ `unresolved`=**没给 `expect`，只证明能跑通、没证明跑对**。
（附带一条实测坑：Blender 的 `--python` 脚本抛异常时**退出码仍是 0**，判定必须看"回执 + stderr 异常标记"，只看 `exit` 会漏判。）

**5 条配方的参数块**（照抄进脚本开头；各配方末尾给出对应写法）：

| 配方 | 参数块 |
|---|---|
| 2a 分型面 | `{"bevel_width": 0.002, "bevel_segments": 1, "angle_deg": 30}` |
| 4 布尔 | `{"cutter_radius": 0.3, "cutter_depth": 3.0, "solver": "EXACT", "apply": False}` |
| 5 镜像 | `{"axis": "X", "use_clip": True, "merge_threshold": 0.001}` |
| 6b 销轴弦长节距阵列 | `{"pitch": 0.176, "m": 6, "n_straight": 6}`（节数由 `2*(m+n_straight)` 导出） |
| 9 装配门 | `{"scope": "COL_Geo", "envelope": [...], "pairs": [["GEO-rungear", "GEO-hull"]]}` |

**怎么验**：同一生成器填 3 组参数 ⇒ 3 次 `generator_run` 全 `verdict=supported` 且 `receipt.objects` 与参数一致（实测 `n=5/8/12` ⇒ `objects=5/8/12`、`tris=60/96/144`）；**原样再跑一次** ⇒ `cached=true, ms=0`。

### Recipe 1 — 加基本体并立即命名

```python
import bpy
bpy.ops.mesh.primitive_cube_add(size=2.0, location=(0, 0, 1))
obj = bpy.context.active_object
obj.name = 'GEO-base_box'
print(f"created:{obj.name} verts:{len(obj.data.vertices)}")
```

把 primitive_cube_add 换成 _plane_ / _uv_sphere_ / _ico_sphere_ / _cylinder_ / _cone_ / _torus_ / _monkey_，参数各按形状给（radius、depth、vertices、segments、subdivisions）。
（这几种 primitive_*_add 在 headless 下是**安全**的：它们不依赖编辑模式 context。脆的是 `mesh.*` / `object.mode_set` 那一类。）

### Recipe 2a — 分面硬表面（装甲板 / 军模 / 机械板件）★ 先看这条

```python
import bpy
obj = bpy.data.objects['GEO-armor_plate']

bevel = obj.modifiers.new('Bevel', type='BEVEL')
bevel.width = 0.002          # 2mm：只是把"刀口"磨掉，棱边仍然清晰
bevel.segments = 1           # 分面板件通常 1 段就够（要更圆用 2）
bevel.limit_method = 'ANGLE'
bevel.angle_limit = 0.523599  # 30°：只倒比这更尖的边

# ✗ 不要加 SubSurf —— 分面装甲板加细分会把棱边圆成"肥皂"
for p in obj.data.polygons:
    p.use_smooth = False      # 平坦着色：板面之间要能看出转折
print(f"faceted:{obj.name} polys:{len(obj.data.polygons)}")
```

判据：**边缘高光是一条细线**（倒角）**而不是一道渐变**（细分）；相邻板面之间的转折在渲染里应当**看得见**。
参考图里能数出板缝的模型，一律走这条。

> **参数块 + 生成脚本（§3.0）**：把三个数字拎出来，前半段补一个"从零建 cube"，就是可直接注册的生成脚本。
> ```python
> PARAMS = {"bevel_width": 0.002, "bevel_segments": 1, "angle_deg": 30}
> # 脚本开头从零建件（生成器是全新无头进程，不许 bpy.data.objects['x'] 捞现成对象）：
> #   bpy.ops.mesh.primitive_cube_add(size=1.0); obj = bpy.context.object; obj.name = "GEO-armor_plate"
> #   bevel.width = PARAMS["bevel_width"]; bevel.segments = PARAMS["bevel_segments"]
> #   bevel.angle_limit = math.radians(PARAMS["angle_deg"])
> #   blender_rt_plan(op="generator_save", args={"name": "faceted_plate", "code": CODE, "params": PARAMS})
> #   blender_rt_plan(op="generator_run",  args={"name": "faceted_plate", "expect": {"min_objects": 1}})
> ```
> 判据：只改 `PARAMS` 不改代码 ⇒ `receipt.PARAMS` 与 `tris` 跟着变、`verdict=supported`。

### Recipe 2b — 光滑硬表面（汽车钣金 / 圆润道具）

```python
import bpy
obj = bpy.data.objects['GEO-base_box']

# 1. Bevel：先把硬边倒圆
bevel = obj.modifiers.new('Bevel', type='BEVEL')
bevel.width = 0.02            # 2cm 圆角
bevel.segments = 3
bevel.limit_method = 'ANGLE'  # 只倒比阈值更尖的边
bevel.angle_limit = 0.523599  # 30 度（弧度）

# 2. Subdivision 必须在 Bevel 之后
subsurf = obj.modifiers.new('SubSurf', type='SUBSURF')
subsurf.levels = 2
subsurf.render_levels = 3

# 3. 平滑着色
bpy.context.view_layer.objects.active = obj
bpy.ops.object.shade_smooth()
print(f"hardsurface:{obj.name} polys:{len(obj.data.polygons)}")
```

**顺序反了就会出收窄伪影**（最常见的业余错误）。
**载体判据**：只有"表面本来就连续"的件才配这一栈；分面件（配方 2a）上 SubSurf = 毁形。

### Recipe 3 — 编辑模式：挤出 / 内插 / 环切（GUI 可用；headless 见 3b）

```python
import bpy
obj = bpy.data.objects['GEO-base_box']
bpy.context.view_layer.objects.active = obj
bpy.ops.object.mode_set(mode='EDIT')

bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.mesh.extrude_region_move(TRANSFORM_OT_translate={'value': (0, 0, 1.0)})
bpy.ops.mesh.inset(thickness=0.1, depth=0)
bpy.ops.mesh.loopcut_slide(
    MESH_OT_loopcut={'number_cuts': 1, 'edge_index': 0},
    TRANSFORM_OT_edge_slide={'value': 0.0},
)
bpy.ops.object.mode_set(mode='OBJECT')
print(f"edited:{obj.name} verts:{len(obj.data.vertices)}")
```

### Recipe 3b — 同一件事的 bmesh 写法（★ headless / 批处理用这条）

```python
import bmesh, bpy
obj = bpy.data.objects['GEO-base_box']
bm = bmesh.new()
bm.from_mesh(obj.data)

# 1) 找到朝上的那个面（比"选中顶面"更稳：不依赖选择状态与 context）
up = [f for f in bm.faces if f.normal.z > 0.9]
face = up[0]

# 2) 挤出 + 平移（等价 extrude_region_move）
res = bmesh.ops.extrude_face_region(bm, geom=[face])
new_verts = [g for g in res['geom'] if isinstance(g, bmesh.types.BMVert)]
bmesh.ops.translate(bm, verts=new_verts, vec=(0.0, 0.0, 1.0))

# 3) 内插（等价 mesh.inset）
top = [f for f in bm.faces if f.normal.z > 0.9 and f.calc_center_median().z > 1.0]
bmesh.ops.inset_individual(bm, faces=top, thickness=0.1, depth=0.0)

bm.to_mesh(obj.data)
bm.free()
obj.data.update()
print(f"bmesh-edited:{obj.name} verts:{len(obj.data.vertices)}")
```

**为什么值得改写法**：`bpy.ops.mesh.*` 依赖"当前编辑模式 + 活动对象 + 选择集 + 区域 context"，
在 `blender -b` 下这些经常是空的，报错信息还很难懂（"context is incorrect"）。
bmesh 直接操作数据层，**没有 context 依赖**，批处理里稳得多。必须用 ops 时用 `temp_override` 补 context。

### Recipe 4 — 布尔开孔

```python
import bpy
target = bpy.data.objects['GEO-base_box']
cutter = bpy.data.objects.get('GEO-cutter')
if cutter is None:
    bpy.ops.mesh.primitive_cylinder_add(radius=0.3, depth=3.0, location=(0, 0, 1))
    cutter = bpy.context.active_object
    cutter.name = 'GEO-cutter'

mod = target.modifiers.new('Boolean', type='BOOLEAN')
mod.operation = 'DIFFERENCE'
mod.object = cutter
mod.solver = 'EXACT'

bpy.context.view_layer.objects.active = target
bpy.ops.object.modifier_apply(modifier=mod.name)   # 想保持非破坏就先别 apply
cutter.hide_viewport = True
cutter.hide_render = True
print(f"booleaned:{target.name}")
```

> 实测：军模里布尔**用得不多但很关键**（炮塔前下方 undercut、炮口开孔）。用法是"**外科手术**"，
> 不是主力建模手段 —— 大面积造型靠板件拼装（§0.5），布尔只负责那几个真开孔的地方。
>
> **参数块 + 生成脚本（§3.0）**：`PARAMS = {"cutter_radius": 0.3, "cutter_depth": 3.0, "solver": "EXACT", "apply": False}`。
> 生成脚本自带 target 与 cutter 的**两段建件**（`primitive_cube_add` + `primitive_cylinder_add`），
> `mod.solver = PARAMS["solver"]`，`apply=False` 时保持非破坏（导出前再 apply）。
> 判据：三组 `cutter_radius` ⇒ `receipt.tris` 单调变化且 `verdict=supported`；`solver` 换 `FAST` 时若孔壁出问题，`audit_mesh` 的 `nonmanifold_edges` 会红。

### Recipe 5 — 镜像（只做一半）

```python
import bpy
obj = bpy.data.objects['GEO-character_half']
mod = obj.modifiers.new('Mirror', type='MIRROR')
mod.use_axis[0] = True     # 沿 X 镜像
mod.use_clip = True        # 轴上顶点吸附
mod.use_mirror_merge = True
mod.merge_threshold = 0.001
print(f"mirrored:{obj.name}")
```

**Mirror 放在栈的最前面**（在 Bevel / SubSurf 之前）。

> **参数块 + 生成脚本（§3.0）**：`PARAMS = {"axis": "X", "use_clip": True, "merge_threshold": 0.001}`。
> `use_axis[0/1/2] = (PARAMS["axis"] == "X"/"Y"/"Z")`；生成脚本前半段从零建"半个件"（`primitive_cube_add` + 删掉一半）。
> 判据：`merge_threshold` 调大 ⇒ 中缝的点被焊上 —— **但 `audit_mesh` 读的是未求值的 `ob.data`**（实测：带未 apply 的 Mirror 的半个件报 `verts=4 / boundary_edges=4`，就是基础网格），
> 所以**先把 Mirror apply 掉**再查 `boundary_edges`：>0 → 0 就是"焊上了"的机器判据。
> （反过来说：装配级的 `audit_connectivity` / `audit_gate` / `audit_measure` / `audit_interference` 走**求值网格**，Boolean 没 apply 也算数 —— 两套口径别混。）

### Recipe 6 — Array 沿曲线（链子、栅栏、珠子）

```python
import bpy
bpy.ops.mesh.primitive_cube_add(size=0.2, location=(0, 0, 0))
unit = bpy.context.active_object
unit.name = 'GEO-bead'

path = bpy.data.objects.get('GEO-path')
if path is None:
    bpy.ops.curve.primitive_bezier_curve_add()
    path = bpy.context.active_object
    path.name = 'GEO-path'

arr = unit.modifiers.new('Array', type='ARRAY')
arr.fit_type = 'FIT_CURVE'
arr.curve = path
arr.relative_offset_displace = (1.0, 0, 0)

crv = unit.modifiers.new('Curve', type='CURVE')
crv.object = path
crv.deform_axis = 'POS_X'
print(f"arrayed:{unit.name}")
```

> ⚠ **载体判据**：Array 适合"装饰性重复"（珠子、栅栏、铆钉）。**本配方（`FIT_CURVE` + 曲线变形）**给不了
> "精确弦长节距 + 闭环"—— `FIT_CURVE` 只保证"沿曲线装满"，首尾接缝与每节间距都不受你控制。
> 直线等距阵列不是它的短板（`FIXED_COUNT` + 精确 offset 就能定节距）；**曲线上的精确闭环节距**才是，走 6b。

### Recipe 6b — 参数化**闭合**路径阵列（★ 履带 / 链节 / 齿圈）

> ★ **先分清两种"节距"** —— 这是这条配方唯一容易搞错的地方：
>
> | 口径 | 定义 | 直段上 | 圆弧上 |
> |---|---|---|---|
> | **等弧长节距**（arc-length） | 把路径**弧长**均分 N 份 | 弦长 = 弧长 = 步长 | **弦长 < 步长**（弦永远比弧短） |
> | **销轴弦长节距**（机械真值） | 相邻销轴中心的**直线距离**恒为 `p`（节是刚体，销距是规格） | = `p` | 需 `p = 2R·sin(θ/2)`，即弧长 `R·θ > p` |
>
> **链条/履带必须用弦长口径**：节是刚体，销孔位置由"销距"定死，不是由弧长定。
> 用等弧长均分，弧段上销距会**系统性偏短**，到直/弧交界处累积误差直接装不上。
> 实测（`p=176mm`、每半圆 6 节、R 由弦长反解）：
>
> | | 数值 |
> |---|---|
> | 等弧长口径的弧上弦长 | **175.021mm**（规格应为 176）→ 每节 **−0.979mm** |
> | 一条半圆累积 | **−5.874mm**；两条弧共 **−11.748mm** |
> | 半圆需要多少步 | **6.0343 步**（非整数！销会落在直/弧交界**内部**） |
> | 弦长口径实测 `max|弦长−p|` | **3.3e-06 mm**（浮点噪声级）+ 闭合 |

```python
import bpy, bmesh, math
from mathutils import Vector

# ── 参数块（销轴弦长口径：给 pitch 和**整数**节数，半径/直段自己长出来）
PITCH  = 0.176     # m，销轴弦长节距（规格值）
M      = 6         # 每个半圆上的节数（整数！）
N_STR  = 6         # 每条直段上的节数（整数！）
#   ⇒ 节数 N = 2*(M + N_STR)（本形必然为偶数）；半径与直段由 pitch 反解：
#      R = pitch / (2*sin(pi/(2M)))      A = N_STR * pitch
TEMPLATE = "GEO-track_link"                 # 单节模板对象名
BODY_W, THICK = 0.12, 0.02                  # 链板宽 / 厚（只为"看得见"，与节距无关）

def build_link_template(name, pitch, body_w, thick):
    """从零造**可视单节**（矩形链板），省掉"得先有个模板对象"的前置。
    ★ **模板契约**（下面的循环就是按这五条摆件的，换几何也必须守）：
      ① **原点 = 本节起点销轴中心**（不是链板中点）：循环里 `ob.location` 直接落在路径点上；
      ② **局部 +X = 指向下一销**：循环用 `to_track_quat('X','Z')` 把 +X 转到下一销方向；
      ③ **下一销轴在局部 (+pitch, 0, 0)**：两销轴距**必须正好 = PITCH** ⇒ 模板长度是"由节距导出"的参数；
      ④ **销轴方向 = 局部 +Z**（`to_track_quat('X','Z')` 的 up 轴）；链板厚度沿 Z 居中；
      ⑤ 模板的 **x/y 会被忽略、只有 z 被沿用**（循环里 `ob.location = (x, y, tpl.location.z)`）
         ⇒ 模板放 z=0 就等于"路径平面 z=0"。
    要更"像链节"就只换本函数内部几何（圆头 / 倒角 / 销孔 / 带凸台的销，见下面那条提示），
    **契约①–⑤不变** ⇒ 循环与全部验收判据都不用改。"""
    me = bpy.data.meshes.new(name)
    bm = bmesh.new()
    bmesh.ops.create_cube(bm, size=1.0)                    # 单位立方 → 重映射到局部契约
    for v in bm.verts:
        v.co.x = (v.co.x + 0.5) * float(pitch)              # x ∈ [0, pitch]：0 = 本节销轴
        v.co.y *= float(body_w)                             # y ∈ [−body_w/2, +body_w/2]
        v.co.z *= float(thick)                              # z ∈ [−thick/2, +thick/2]
    bm.to_mesh(me); bm.free(); me.update()
    ob = bpy.data.objects.new(name, me)
    bpy.context.scene.collection.objects.link(ob)
    return ob

def stadium_pins(p, m, n):
    """赛道形闭合路径：上直段(+X) → 右半圆 → 下直段(-X) → 左半圆。
    全程按**弦长**步进 ⇒ 相邻销心距离恒为 p，且首尾闭合。"""
    R = p / (2.0 * math.sin(math.pi / (2.0 * m)))
    A = n * p
    hx, th = A / 2.0, math.pi / m
    pts = []
    for i in range(n):                                        # 上直段
        pts.append((-hx + i * p, R))
    for i in range(m):                                        # 右半圆 90° → -90°
        a = math.pi / 2 - i * th
        pts.append((hx + R * math.cos(a), R * math.sin(a)))
    for i in range(n):                                        # 下直段
        pts.append((hx - i * p, -R))
    for i in range(m):                                        # 左半圆 -90° → -270°
        a = -math.pi / 2 - i * th
        pts.append((-hx + R * math.cos(a), R * math.sin(a)))
    return pts

pts = stadium_pins(PITCH, M, N_STR)
N = len(pts)                                                  # = 2*(M+N_STR)

tpl = build_link_template(TEMPLATE, PITCH, BODY_W, THICK)     # ★ 从零建单节（不再要求场景里先有模板）
tpl.location = (0.0, 0.0, 0.0)                                # z = 路径平面
for i, (x, y) in enumerate(pts):
    ob = tpl.copy()                                           # 或 linked duplicate 省内存
    bpy.context.collection.objects.link(ob)
    ob.location = (x, y, tpl.location.z)
    d = Vector((pts[(i + 1) % N][0] - x, pts[(i + 1) % N][1] - y, 0.0))
    ob.rotation_euler = d.to_track_quat('X', 'Z').to_euler()   # 局部 +X 指向下一销
    ob.name = "%s_%03d" % (TEMPLATE, i)
bpy.data.objects.remove(tpl, do_unlink=True)                  # ★ 模板不算件，否则对象数 = N+1

# ── 脚本判据（不许靠看）
made = [bpy.data.objects["%s_%03d" % (TEMPLATE, i)] for i in range(N)]
ch = [(made[i].location - made[(i + 1) % N].location).length for i in range(N)]
xs  = [o.location.x for o in made]; ys = [o.location.y for o in made]
R = PITCH / (2.0 * math.sin(math.pi / (2.0 * M))); A = N_STR * PITCH
print("N=%d  max|chord-p|=%.6fmm  closure=%.6fmm" %
      (N, max(abs(c - PITCH) for c in ch) * 1000, ch[-1] * 1000))
print("bbox_x=%.6f (应 = A+2R = %.6f)  周长=2A+2*pi*R=%.6f  N*p=%.6f" %
      (max(xs) - min(xs), A + 2 * R, 2 * A + 2 * math.pi * R, N * PITCH))
```

**验收（脚本量，不许靠看）**（本机实测值见括号）：
本节第 1–3 条 + 第 5 条由**脚本**判；第 4 条是**装配门**，口径见下面的实测（★ 它**不是**"只有 1 条连通分量"）。

1. `max|相邻销心弦长 − PITCH| ≤ 0.001mm`（实测 **3.3e-06mm**）；
2. **闭合**：末节→首节的弦长也 = PITCH（实测 **175.999998mm**）；
3. **对象数 = N**，模板必须删掉（实测 `N=24`，场景里恰好 **24** 个 `GEO-track_link_*`）；
4. **不散架**用 `audit_connectivity` 的 `floaters` / `gate.state` 判（★ 实测口径见下）；
5. **几何尺寸对的是这三条**（★ 别拿 bbox 当周长）：

   | 量 | 正确关系 | 实测（PITCH=0.176, M=N_STR=6） |
   |---|---|---|
   | bbox **最长边** | **`A + 2R`**（赛道形：直段 + 一个直径） | **1.736012 m** |
   | 路径**周长**（弧长） | `2A + 2πR` | **4.248320 m** |
   | `N × PITCH` | **≠ 周长**（弦长总长 < 弧长总长） | 4.224000 m（差 **24.32mm**） |

   ⚠ 老版本写的是"`receipt.bbox` 的长度轴 ≈ `n*pitch`" —— **错 680mm**（它给的 1.056m 只是单条直段长，
   既漏了两个半圆，又把 bbox 当成了周长）。周长与 bbox 是**两个不同的量**，别互相替代。

**★ 链节装配口径（v5 实测，别按直觉读）**：24 节各是**独立对象**、只端面贴合、不合并顶点，所以

| 跑什么 | 实测 | 正确读法 |
|---|---|---|
| `audit_mesh(objects=24 个节, self_intersect=true)` | `state=pass`、`clean=true`，每节 `closed=true`、`normals_state=pass` | 单件自证过（门①–④） |
| `audit_connectivity` | `floaters=[]`、各分量 `attached=true` + `mesh_confirmed=true`、`gate.state=pass`；**分量数 = 14**（两条直段各 1 块 + 12 个弧上单节） | **"不散架"要看 `floaters`/`gate.state`，不要看"分量数=1"**（贴而未焊 ⇒ 天然是多分量）；⚠ 顶层**没有** `state`，门口径在 `gate.state` |
| `audit_overlap(a=链节0, b=链节1)` | `pair_count = 1`（`真交线段` 120mm，即两节端面**共用的那条边**） | 贴合接触**会被记成 1 对** —— `pair_count=0` 不是"没贴合"、`pair_count=1` 也不是"穿模"；判贴合/穿透要用 `contact_probe` 与三角级体积 |
| `audit_interference(a=链节0, b=链节1)` | `verdict="unresolved"`、`volume_mm3=0.0`（20 万采样点里 0 点落在双方内部；统计上界 6.33mm³ > 可执行下限 1.0mm³ ⇒ 判不了） | **不是 supported（没抓到互穿）、也不是 refuted（没证到无互穿）**；链节的"设计接触"要在门里**白名单化**（§坑二 ③），别拿它当缺陷 |

> ★ **模板契约（照抄这五条才不会"跑不起来 / 装错方向"）**：明细写在 `build_link_template()` 的 docstring 里，
> 最容易搞错的是 **①原点在起点销轴（不是链板中点）、③下一销轴在局部 `(+pitch, 0, 0)`** ——
> 原点若放到链板中点，`ob.location` 就不再落在路径点上，`max|弦长−PITCH|` 立刻变红，闭合处还会错半个节距。
> 想更像真链节：把 `build_link_template()` 内部换成"圆头链板 + 两个销孔 + 一端销凸台"即可，
> **只要守住①–⑤**，循环与所有判据都不用动。


> ⚠ **本配方只解"节距 + 闭合"**：同一模板复制 N 次，相邻节在接缝处是**端面贴合**（零体积接触）——
> **实测**：`audit_interference` 给 `verdict="unresolved"`/`volume_mm3=0.0`（抓不到互穿、也证不到无互穿），
> `audit_overlap` 会给 `pair_count=1`（两节端面**共用的那条边**，120mm 真交线段）。⇒ 这类"设计接触"
> 要在门里**白名单化**（§坑二 ③），**不要**拿 `pair_count` 当"有没有穿模"的开关。
> 真链节的"内/外板交替 + 销的归属 + 销孔让位"**不在本配方范围内**；要过干涉门就按坑二用**两套模板交替**
> （内板/外板各一套，`pts` 隔点取用），或把销单列成件按接口表给 `overlap_mm`。

> **参数块 + 生成脚本（§3.0，这条是样板）**：把 `PITCH / M / N_STR` 拎成 `PARAMS`，
> 脚本前半段就是上面的 `build_link_template()`（从零建单节），后半段是循环：
> ```python
> # 注册：blender_rt_plan(op="generator_save", args={"name": "track_array", "code": CODE,
> #         "params": {"pitch": 0.176, "m": 6, "n_straight": 6}})
> # 改参复现（首选工具直调：args 这一层就是 PARAMS；旧插件回退内核写法，见 §3.0）：
> blender_rt_plan(op="generator_run", args={"name": "track_array",
>     "args": {"pitch": 0.176, "m": 6, "n_straight": 6}, "expect": {"min_objects": 24}})
> ```
> 判据：`receipt.objects == 2*(PARAMS["m"] + PARAMS["n_straight"])`（模板不算件），
> 且每组参数的 `max|弦长-pitch| ≤ 0.001mm`、`bbox 最长边 ≈ A + 2R`、`周长 ≈ 2A + 2πR`。
> ⚠ 旧文本里的 `{"n": 83, "pitch": 0.176}` **不是这个闭合形能产生的组合**（本形节数必为偶数），
> 那组数是从项目里抄来的占位；要 83 节就换非对称路径，或改 `M/N_STR` 并重新核对弦长与闭合。

### Recipe 7 — Blockout 布景

```python
import bpy
bpy.ops.mesh.primitive_plane_add(size=10)
bpy.context.active_object.name = 'GEO-floor'
bpy.ops.mesh.primitive_cube_add(size=1.5, location=(0, 0, 0.75))
bpy.context.active_object.name = 'GEO-subject'
bpy.ops.mesh.primitive_cylinder_add(radius=0.5, depth=2, location=(2, 1.5, 1))
bpy.context.active_object.name = 'GEO-prop_pillar'
print('blockout:done')
```

### Recipe 8 — 曲线转网格 / 布尔之后的清理（装配层也跑这一条）

```python
import bpy
obj = bpy.data.objects['GEO-target']
bpy.context.view_layer.objects.active = obj
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.mesh.remove_doubles(threshold=0.0001)
bpy.ops.mesh.normals_make_consistent(inside=False)   # ★ 法线一致化：穿模判据的前提（见坑五）
bpy.ops.object.mode_set(mode='OBJECT')
bpy.ops.object.shade_smooth()
print(f"cleanup:{obj.name} verts:{len(obj.data.vertices)}")
```

> headless 里 `mode_set` / `mesh.*` 可能因 context 失败 —— 等价 bmesh 写法：
> `bmesh.ops.remove_doubles(bm, verts=bm.verts, dist=1e-4)` + `bmesh.ops.recalc_face_normals(bm, faces=bm.faces)`。

### Recipe 9 — 装配审计门（把 §6 的数值门一把跑完）

```python
# 方式 A（推荐，模型侧）：工具层直接调，回执是结构化 JSON
#   blender_rt_plan(op="audit_gate",      args={"scope": "COL_Geo"})          # 出厂门：连通 + 包络
#   blender_rt_plan(op="audit_interference", args={"a": "GEO-rungear", "b": "GEO-hull"})
#   blender_rt_plan(op="audit_measure",   args={"neighbors": True})           # bbox + 逐轴间隙/重叠

# 方式 B（headless 批处理里）：preload="audit" 之后直接调 python API
#   K.dsh_audit_api("mesh", {"objects": ["GEO-hull"]})
#   K.dsh_audit_api("connectivity", {"scope": "COL_Geo"})
#   K.dsh_audit_api("gate", {"scope": "COL_Geo", "envelope": ENVELOPE})
```

判据全在 §6。**装配完成后、导出之前，必跑一次**；每个 builder 交件前也要跑单件那几条。

> **参数块 + 生成脚本（§3.0；9 的特殊之处：注册走生成器、执行走 headless）**：门清单与阈值就是参数块：
> ```python
> AUDIT = {"scope": "COL_Geo",
>          "envelope": [x0, y0, z0, x1, y1, z1],
>          "pairs": [["GEO-rungear", "GEO-hull"], ["GEO-turret_ring", "GEO-hull_deck_ring"]]}
> # 注册 + 查改没改过（回执过期制）：
> #   blender_rt_plan(op="generator_save", args={"name": "audit_gate_run", "code": CODE, "params": AUDIT})
> #   blender_rt_plan(op="generator_diff", args={"name": "audit_gate_run"})     # changed=true ⇒ 上次回执作废
> # 执行（★ 不走 generator_run）：脚本里要调 audit 模块，只有 headless 的 preload 通道给得到
> #   blender_rt_headless(script_file="<generator_get 回执里的 path>", preload="audit", engine="none")
> ```
> ⚠ 原因（实测）：`generator_run` 的子进程里只有 `PARAMS` + bpy，**没有 `K`、也没有 `audit_*`** —— 脚本里引用 `K` 直接 `NameError`，verdict=`refuted`。
> 所以这条配方是「生成器管版本与 diff，headless + `preload="audit"` 管执行」。
> 判据：脚本改一行 ⇒ `generator_diff` 回 `changed=true`；拿三缺陷件按这份清单跑 ⇒ 三门红（§0.5.5）。

### Recipe 10 — 形体验收：固定机位对照图 + 数值 bbox

```python
# ① 数值：与 spec.py 对拍（这是"像不像"的硬证据）
#   blender_rt_plan(op="audit_measure", args={"objects": ["GEO-turret"], "neighbors": False})
#   → 回 sizeof/bbox；逐个与 spec.py 的期望值比，偏差写进报告（军模级 ±3mm）

# ② 画面：用**与参考图相同的机位**出图（多视角 harness 自带取景与三点光）
#   blender_rt_plan(op="qc_render_views", args={
#       "views": [{"name": "front", "from": [0, -12, 1.6], "look_at": [0, 0, 1.6], "lens": 50}],
#       "res": [1280, 720], "samples": 64, "outdir": "D:/DSH/blender/out/qc"})
#   → 逐张 md5 + coverage_estimate；主体占画面 <5% 会给 subject_too_small 警告
```

**只用其中一条都不够**：数值抓得出"前裙甲上翘 6°"，但看不出"焊缝难看"；画面看得出"像不像"，但看不出 3mm 的偏差。

### Recipe 11 — 程序化雕刻（体素底模 → 位移笔刷 → 遮罩 → 滤镜）

> **能雕什么**：`draw / inflate / pinch / flatten / smooth / crease` 六支位移笔刷（numpy 实现，**无头也能跑**），
> 加拓扑准备（体素重构 / multires / subdiv / dyntopo）、遮罩（sphere/box 直接写 `.sculpt_mask`）、
> GUI 滤镜（`sculpt.mesh_filter`）。**不能**注入手势笔触（见决策树里的 API 限制）。
> 实测成本：体素重构 1986→12148 顶点 84ms；单笔 1986 顶点、257 顶点受影响 6ms。

```python
# ① 体检 + 定底模（体素尺寸给 "auto" 会按 40M cell 预算反解）
#   blender_rt_plan(op="sculpt_scan",   args={"objects": ["GEO-head"]})
#   blender_rt_plan(op="sculpt_setup",  args={"object": "GEO-head", "mode": "VOXEL",
#                                             "voxel_size": "auto", "symmetry": ["x"]})
#   → 回执 verts_before/verts_after；voxel_size 太小会直接报错并给可用值（不会卡死 Blender）
#
# ② 雕（每条笔触独立；symmetry 会各自镜像成独立笔触，不会串成一条假路径）
#   blender_rt_plan(op="sculpt_apply", args={"object": "GEO-head", "strokes": [
#       {"brush": "draw",   "points": [[0,0,1.0],[0.12,0,1.06]], "radius": 0.45, "strength": 0.7},
#       {"brush": "crease", "points": [[0.7,0,0.7]], "radius": 0.15, "strength": 0.9, "symmetry": ["x"]},
#       {"brush": "smooth", "points": [[0,0,1.1]], "radius": 0.30, "strength": 0.5}]})
#   → 每条回 affected_verts / max_delta / ms；max_delta=0 说明笔刷没落到网格上（先检查 points 与 radius）
#
# ③ 限定作用域（先遮罩再雕 = 与 destructive_guard 同一套纪律）
#   blender_rt_plan(op="sculpt_mask", args={"objects": ["GEO-head"], "mode": "sphere",
#                                           "center": [0,0,1], "radius": 0.6})
#   → 之后 sculpt_apply 默认 use_mask=true：被遮的地方不动
#
# ④ 整块平滑（GUI 滤镜；无头会明确报"没有 VIEW_3D"）
#   blender_rt_plan(op="sculpt_filter", args={"objects": ["GEO-head"], "type": "SMOOTH",
#                                             "strength": 0.35, "iterations": 2})
```

**验收**（照 §0.6.2 双证据）：`audit_mesh` 前后对比（顶点/nonmanifold/degenerate）+
`qc_render_views` 出同机位图给眼睛看。**改前先 `blender_rt_txn(op="snapshot")`** ——
对象级 mark/revert 不含拓扑改动，雕完就回不去了。

### Recipe 12 — 网格修复 + UV（audit 只诊断，fix 负责治）

```python
# ① 先诊断（只读）→ 拿到 boundary/nonmanifold/degenerate/loose/self_intersections
#   blender_rt_plan(op="audit_mesh", args={"objects": ["GEO-part"]})
# ② 治：默认只做安全三件（合并重复点 / 消零面积面 / 删孤立点）+ 重算法线
#   blender_rt_plan(op="fix_repair", args={"objects": ["GEO-part"],
#       "actions": ["merge_doubles","dissolve_degenerate","delete_loose","recalc_normals"]})
#   → 回执带 before/after 缺陷计数；**没修动就是 ok=false**（不许把"跑过"当"修好"）
#   → 开放薄壳（设计上就该开口）会报 open_shell=true，但不算残留缺陷；fill_holes 会封死它，慎用
#   ⚠ 这里的 delete_loose 只清**孤立的点/边**（它一个面都不删）——**内部面/埋面不归它管**，见 §5.1
# ③ 复检 + 减面（可选）
#   blender_rt_plan(op="fix_decimate", args={"objects": ["GEO-part"], "target_tris": 2000})
# ④ UV（交付带贴图的 OBJ 必须有）
#   blender_rt_plan(op="uv_smart_project", args={"objects": ["GEO-part"], "angle_limit": 66})
#   blender_rt_plan(op="uv_unwrap",       args={"objects": ["GEO-part"], "method": "SLIM"})   # 有机件更省拉伸
#   blender_rt_plan(op="uv_pack",         args={"objects": ["GEO-part"], "margin": 0.002})
#   blender_rt_plan(op="uv_stats",        args={"objects": ["GEO-part"]})   # 判据：有 UV 层 且 零面积 UV 面 = 0
```

⚠ UV 算子会进 EDIT 模式并全选面（模块在 finally 里恢复原对象/模式/选择）；枚举值（unwrap method、
pack shape_method）不在代码里写死，而是从算子 RNA 读，传错会回"允许列表"。

### Recipe 13 — 制造检查：薄壁与悬垂（只读，交付门用）

```python
#   blender_rt_plan(op="print_report", args={"objects": ["GEO-part"], "min_mm": 1.2, "max_angle_deg": 45})
#   → 壁厚：从每个采样面中心沿 -法线打射线，命中距离 = 局部厚度；回 min/p05/median/max
#   → 悬垂：面法线与"正下方"夹角 ≤ max_angle_deg 的面积占比（45° ≈ FDM 常见上限）
#   ⚠ 回执里的 resolution_mm 是本网格的测量分辨率（本地面尺寸）：min_mm 低于它时会明确警告
#     "结论不可信" —— 体素重构出的 60mm 面测不了 5mm 壁厚，先细化或降 voxel_size
#   正例（实测）：200×200×5mm 薄板（面 4mm）@min_mm=10 → min_mm=5、thin_samples=5000、无警告
```

### Recipe 14 — 沿路径扫掠：管路 / 线缆 / 轨道

```python
# ① 先算：路径弯折半径 vs 型材半宽（太紧扫出来必然自交）
#   blender_rt_plan(op="sweep_analyze", args={"path": [[0,0,0],[0.6,0,0],[0.6,0,0.6]],
#                                             "profile": {"type":"circle","radius":0.05}})
#   → 回 min_radius / required_min_radius / min_radius_at（第几个点）
# ② 再建：平行移动标架（rotation-minimizing frame），拐弯处不扭转
#   blender_rt_plan(op="sweep_build", args={"name": "GEO-hose",
#       "path": [[0,0,0],[0.6,0,0],[0.6,0,0.6],[1.2,0,0.6]],
#       "profile": {"type":"circle","radius":0.05,"segments":12}, "cap": True})
#   → 弯折过紧默认**拒绝**（先算后建，与 destructive_guard 同一套纪律）；force=true 才硬做
#   profile 支持 circle / rect{width,height} / points[[u,v]…]（自定义型材）
```

### Recipe 15 — 人形素体（零素材）：比例由数字定，几何由 Skin+Subsurf 出

```python
# ① 有参考图就先量比例（像素比，零下载）
#   blender_rt_plan(op="human_spec", args={"image": "D:/ref/front.png"})
#   → bbox / height_px / head_w_px / shoulder_px + profile（每行前景宽度）
# ② 建素体：只填数字（身高 mm / 头身比 / 肩宽 / 姿势）
#   blender_rt_plan(op="human_base", args={"height_mm":1750,"heads":7.5,"pose":"A","name":"GEO-human"})
#   → 回 heads_measured（几何量出来的）/ head_mm / shoulder_over_head / marker_count（19 个关节空物体）
# ③ 验收：与 spec 比（这些字段门里能直接判）
#   blender_rt_plan(op="human_measure", args={"name":"GEO-human"})   → heads / shoulder_over_head
# ④ 挂载与摆姿：<name>_joints 集合里是 pelvis/chest/shoulder_l…foot_r
#    → 供 motion_joints 与硬表面挂点；装甲件直接插到这些点上
```

**硬规则**：① 素体=假人级（比例对、形体简），别指望写实解剖；② 关节标记是**唯一**可靠的挂载基准，别靠肉眼对齐；
③ 门里判比例用 `heads` / `shoulder_over_head`（实测 7.546 / 1.51 对 7.5 / 1.5）。

### Recipe 16 — 人脸 / 头型（零素材）：参数化头 + 五官定位 + 「遮住」策略

```python
# ① 9 点 landmark（像素坐标，自己标或外部检测器给）→ 头型 + 五官定位标记
#   blender_rt_plan(op="human_head", args={"head_mm":233, "name":"GEO-head",
#       "landmarks":{"top":[0,0],"chin":[0,400],"face_l":[-150,200],"face_r":[150,200],
#                    "eye_l":[-70,200],"eye_r":[70,200],"nose_base":[0,290]}})
#   → landmark_fracs（眼线 0.50 / 鼻底 0.725 / 嘴线 0.825）+ 6 个标记 crown/chin/eye_l/eye_r/nose_base/mouth
# ② 比例判定（与经典比例比）
#   blender_rt_plan(op="face_ratios", args={"landmarks":{...}})   → failed / worst_rel_err
# ③ 轮廓相似度（与参考图）
#   blender_rt_plan(op="img_diff", args={"a":"D:/out/front.png","b":"D:/ref/front.png"}) → iou
```

**硬规则**：① **本技能这 22 条配方 + 零素材**（没有参考图 / 没有雕刻素材 / 不做相机标定）**做不出写实人脸** ——
这条路线的边界在"比例 + 粗形"，毛孔 / 皱纹 / 眼睑这类细节没有证据来源，只能靠猜；所以高细节部位用
**面罩 / 头盔 / 护目镜**遮住，把硬表面顶上去（MK1 那类装甲的正确解）。
有参考照片时先按 Recipe 20 校正再量（但注意单应只校正取点那个平面），或换用外部素材/真正的雕刻素材库 —— 那是本技能之外的输入；
② 眼睛鼻子嘴的零件挂到 `*_M_*` 标记上，别靠估位置；③ 比例容差按项目钉进 spec（风格化角色要重钉，经典比例只是参考）。

### Recipe 17 — 车辆外壳：参考图 → 剖面数字 → 放样 → 正交视图 IoU 迭代

```python
# ① 比例先过门（不过就别建壳 —— 人眼对车比例极敏感）
#   blender_rt_plan(op="vehicle_package", args={"spec":{"type":"sports","length_mm":4300,"width_mm":1850,
#       "height_mm":1250,"wheelbase_mm":2600,"front_axle_from_front_mm":900,"wheel_r_mm":340,
#       "wheel_w_mm":245,"track_mm":1580}})
#   → rows[]：WBR / 轴距比 / 高长比 / 轮距比 / 前后悬 / 离地间隙，逐项 pass|fail
# ② 建壳（参数同 cargen 的纵向剖面；轮眉布尔扣出 ⇒ 轮与壳零互穿是构造保证）
#   blender_rt_plan(op="vehicle_base", args={"spec":{...上面那组...}, "name":"GEO-car"})
#   → wheels[]（x/z/r）+ 标记 front_axle / rear_axle / nose / tail + stations（轮缘处已加密）
# ③ 验收两条腿（都要跑）
#   blender_rt_plan(op="clear_check", args={"pairs":[{"id":"wheel_fl","a":["GEO-car_wheel_fl"],
#       "b":["GEO-car"],"min_mm":8}]})                    → gap_mm / interpenetrating
#   blender_rt_plan(op="qc_render_views", args={..., "views":["right"], "ortho":true,
#       "ref_path":"D:/ref/side.png"})                    → IoU（这就是「像不像」的数值）
# ④ 不像就改剖面参数（hood_len / windshield_angle / roof_len / rear_window_angle…），改完回 ③
```

**硬规则**：① **先过比例门再建壳**；② 轮眉用**布尔扣**（解析轮眉在轮缘是断崖，粗站位下弦会切进轮子——实测 58 → 20 对面相交）；
③ 站位必须在**轮缘加密**；④ 「像不像」的**数值只算一条腿**：侧视正交渲染 vs 参考图的 IoU 能判**外轮廓**，
但**判不了深度/内部特征/面感**（见 §0.6.2 的 IoU 盲区表）—— 必须**多个视图各自算**，再叠 `clear_check` 间隙 +
`audit_measure` 尺寸 + 特征线对照，别用单视 IoU 代表整体；⑤ 细节件（灯/格栅/后视镜/玻璃）用硬表面单独做再挂。

### Recipe 18 — 跑车/复杂外壳：三视图 → 站表 → 参数化放样 → 缝 → IoU 迭代

> 适用面很广：**凡是有明确外轮廓、能用正交视图描述的物体**都走这条（船体/飞机/头盔/鞋/家电/枪械/机器人外壳）。
> 不适用的只有两类：没有明确外轮廓的（布料/毛发/流体）、靠内部结构定义的（多孔晶格/拓扑优化件）。

```python
# ① 包络：先过比例门（不过门别往下做）
#   blender_rt_plan(op="vehicle_package", args={"spec":{"type":"sports","length_mm":4300,"height_mm":1250,
#       "wheelbase_mm":2600,"wheel_r_mm":340,"track_mm":1580,"width_mm":1850}})
# ② 站表：侧视图必需；俯视图给平面收放、前视图给截面性格（腰线/侧倾/下裙）
#   blender_rt_plan(op="vehicle_sections", args={"side":"D:/ref/side.png","top":"D:/ref/top.png",
#       "front":"D:/ref/front.png","mm_per_px":3.2,"stations":41,"wheel_r_px":106})
#   → stations[{x_mm, z_top_mm, z_bottom_mm, half_w_mm}] + package + section_params + wheels
#   （轮径无法从填充轮廓反推 ⇒ 给 wheel_r_px 或已知规格；轮轴 x 由轮廓底部低洼段自动定位）
# ③ 放样：站表定轮廓，截面参数定性格
#   blender_rt_plan(op="vehicle_loft", args={"stations": <上一步的 stations>, "name":"GEO-shell",
#       "n_top":5, "n_bot":3, "tumblehome_mm":120, "beltline_frac":0.55,
#       "shoulder_inset_mm":25, "sill_tuck_mm":60, "flare_mm":30, "subsurf":1, "crease_shoulder":0.7})
#   → 回 size_mm / crease_edges / flare.at_x_mm（轮拱外扩落在底边被抬起的站位）
# ④ 缝：按特征线切分件（缝是真几何，可用 clearance 逐对量）
#   blender_rt_plan(op="vehicle_panels", args={"object_name":"GEO-shell","cuts_mm":[1500,3000],"gap_mm":4})
#   blender_rt_plan(op="clear_check", args={"pairs":[{"id":"hood_door","a":["GEO-shell"],
#       "b":["GEO-shell.001"],"min_mm":0}]})                       → gap_mm ≈ 4 / interpenetrating=false
# ⑤ 验收：正交渲染 vs 原图算 IoU（**侧/前/顶各自分开算**，且 IoU 只判轮廓，不判整体像不像）
#   blender_rt_plan(op="qc_render_views", args={..., "views":["right"], "ortho":true, "ref_path":"D:/ref/side.png"})
#   blender_rt_plan(op="img_diff", args={"a":"render_side.png","b":"D:/ref/side.png"})   → iou
#   ⚠ 单视 IoU 是必要不充分条件：深度、内部特征（板缝/格栅/灯）、曲面张力它全都看不见，
#     且"把轮廓填满"就能刷高它。配套必须再加 clear_check 间隙 + audit_measure 尺寸 + 特征线对照（§0.6.2）
```

**硬规则**：① 照片参考**先校正**，但注意**单应只校正取点那个平面** —— 对立体物来说，离该平面越远的特征误差越大
（实测：平面上 0 px，离平面 0.2 m → 211 mm，0.5 m → **543 mm**）；车轮/翼子板这类凸出件若不在取点平面上，
它的量测值不可信，要单独处理或做真正的相机标定（§0.6.1 ⑥）；② 没有俯视图时 `half_w_mm` 缺失，
车身会偏窄（实测 611mm vs 应 1850mm）——**平面收放必须来自俯视图**；③ 轮径不可从轮廓反推；
④ 折痕用 Blender 4+ 的 `crease_edge` attribute（`e.crease_weight` 已不存在）；⑤ 最后 20% 面感靠折痕位置与曲面张力微调。

### Recipe 19 — 外壳还原三件套：前视截面 / 多区域 / IoU 拟合闭环

```python
# ① 三视图一次量完（前视图现在会给**截面轮廓** section_shape，不只是三个数）
#   blender_rt_plan(op="vehicle_sections", args={"side":"D:/ref/side.png","front":"D:/ref/front.png",
#       "top":"D:/ref/top.png","mm_per_px":3.2,"wheel_r_px":106})
#   → stations[] + section_shape[{z_frac,hw_frac}] + package + wheels + warnings
#   ⚠ 没有俯视图时 half_w 走「前视最大宽 × 平面收放假设」（half_width_source 会写明）
# ② 精确复刻：站表直放样（构造上等于轮廓）
#   blender_rt_plan(op="vehicle_loft", args={"stations": <stations>, "section_shape": <section_shape>})
# ③ 要「可改参数的 spec」：站表 → IoU 拟合闭环（纯 2D，不用渲染）
#   blender_rt_plan(op="vehicle_fit", args={"stations": <stations>, "iterations": 240, "target_iou": 0.9})
#   → iou_start / iou / deltas / spec（直接喂 vehicle_base / vehicle_loft）
#   ⚠ 它拟合的是 15 参数形状族：轮廓不在族内时 IoU 会停在 ~0.67 —— 要精确就回 ②
#   ⚠ IoU 在这里只是**拟合目标**（2D 轮廓），不是"像不像"的验收标准：把轮廓填满就能刷高它。
#     拟合完必须回 ④ 的多视 + 特征线复核，别把 target_iou 达标当交付（§0.6.2 IoU 盲区）
# ④ 多体量：按 x 区间切块，每块一个独立体量（卡车/装甲/科幻）
#   blender_rt_plan(op="vehicle_regions", args={"stations": <stations>, "name":"GEO-truck",
#       "regions":[{"name":"cab","x0_mm":0,"x1_mm":2200},{"name":"bed","x0_mm":2250,"x1_mm":6000}]})
```

**分工**：`vehicle_loft(stations)` = 贴参考图；`vehicle_fit` = 出可复用的参数 spec；`vehicle_regions` = 多体量装配。

### Recipe 20 — 照片参考：先校正再量（`img_rectify`）

```python
# ① 斜视照片必须先校正：四角点按「左上→右上→右下→左下」给（对应一个**真实矩形**的四角）
#   blender_rt_plan(op="img_rectify", args={"path":"D:/ref/car_photo.jpg",
#       "quad":[[x1,y1],[x2,y2],[x3,y3],[x4,y4]], "out_w":1600, "out_h":600,
#       "known_w_mm":4300, "out":"D:/out/car_side_rect.png"})
#   → mm_per_px（给了 known_w_mm 就有）+ 单应矩阵；校正后的图再量才准
# ② 之后一切照旧：量站表 → 放样 / 拟合
#   blender_rt_plan(op="vehicle_sections", args={"side":"D:/out/car_side_rect.png","mm_per_px":<上一步>, ...})
```

**硬规则**：① 四点必须是**同一平面上的矩形**（车侧面的整体外框 / 地面标定框），不是随便四个点；
② 斜视越强，取点误差被放得越大 —— 先用 `img_scan` 看清轮廓再取点；③ 校正只解决几何，不解决分辨率与光照；
④ ★ **单应只校正"那四点所在的平面"**：平面上误差为 0，离开平面误差随深度放大
（实测 1 m ≈ 190 px 的机位下：离平面 0.05 m → 52 mm，0.2 m → 211 mm，**0.5 m → 543 mm**）。
所以：**平面参考**（图纸 / 标定板 / 贴平的贴纸）放心用；**立体物**（车/人/建筑）只有取点平面准 ——
轮眉外扩、翼子板弧度、A 柱前倾这些离平面远的特征**量出来是错的**，要么只量靠近该平面的尺寸，
要么改走真正的相机标定（内参 + 外参）。别把"整张照片都校正过了"当成"整个物体都准"。

### Recipe 21 — 形状不在参数族里？换折线族（K 扫描挑控制点数）

```python
# 方箱 / 皮卡 / 装甲这类造型，cargen 的 15 参数族拟合会卡在 ~0.67
#   blender_rt_plan(op="vehicle_fit", args={"stations": <站表>, "family":"polyline",
#       "control_points":[6,10,16], "iterations":200})
#   → curve[{control_points, iou_init, iou}]（挑 K）+ best.profile_pts（闭合剖面：上+下轮廓控制点）
#   → 把 best.profile_pts 喂 vehicle_spec/vehicle_base 就能复现；K 越小 spec 越简单
```

**实测**：cargen 族 0.667 · 折线 K=6 **0.754** · K=10 **0.781** · K=16 **0.821**。

### Recipe 22 — 特征线层：让「面感」也可参数化

```python
# 放样时一起给特征线（crease = 硬折线；inset = 折面/凹槽）
#   blender_rt_plan(op="vehicle_loft", args={"stations": <站表>, "section_shape": <截面轮廓>,
#       "crease_lines":[{"frac":0.55,"weight":0.8},                    # 腰线（按各环比例，推荐）
#                       {"z_mm":560,"x_mm":[400,3900],"weight":0.85},  # 绝对高度纵向线
#                       {"x_mm":1500,"weight":1.0}],                   # 横向分缝（自动插站）
#       "inset_lines":[{"z_mm":470,"band_mm":60,"inset_mm":12,"x_mm":[500,3800]}]})
#   → crease_lines[{spec, edges}] + inset_hits + stations_inserted
```

**⑤ 圆角特征线（真半径）**：条目加 `radius_mm` + `segments` ⇒ 走 Bevel 权重 + Bevel 修改器（正圆过渡），
而不是用折痕权重近似半径。

```python
#   "crease_lines":[{"frac":0.55,"radius_mm":6.0,"segments":3},    # 腰线 R6 圆角（3 段圆弧）
#                   {"x_mm":1500,"radius_mm":4.0,"segments":2}]    # 横向分缝 R4
```

`radius_mm` 缺省 = 硬折线（SubD 折痕）；`segments` 越大越圆（1–8）。实测硬折线 762 顶点 → R6 圆角 1244 顶点。

**硬规则**：① **腰线用 `frac`**（按各环自身高度比例）—— 用 `z_mm` 只在几何确实有那个高度处中标，数量会偏少；
② 横向特征线（`x_mm` 给数值）要求该位置有站位，`vehicle_loft` 会自动插一个；
③ 折痕是 SubD 权重，**必须配 `subsurf>=1`** 才看得出效果；④ 凹陷 `inset_mm` 与 `band_mm` 决定折面强度与宽度。

### Recipe 23 — 多步链一条命令跑完（pipe_run + @工件）

```python
# 以前：4 次工具调用，还要把上一步的 JSON 粘回下一步（vehicle_sections 的 stations 有 5,656 字符）
# 现在：一次调用，产出用 @名字 传递
#   blender_rt_plan(op="pipe_run", args={"steps":[
#       {"api":"vehicle", "op":"sections", "args":{"side":"D:/out/car_side.png", "front":"D:/out/car_front.png", "mm_per_px":10.0}, "out":"@sec"},
#       {"api":"vehicle", "op":"loft", "args":{"stations":"@sec.stations", "section_shape":"@sec.section_shape", "name":"GEO-shell"}, "out":"@shell"},
#       {"api":"vehicle", "op":"panels", "args":{"object_name":"@shell.object", "cuts_mm":[1500,3000], "gap_mm":4}, "out":"@panels"},
#       {"api":"clearance", "op":"check", "args":{"pairs":[{"id":"seam", "a":["@panels.parts.0"], "b":["@panels.parts.1"], "min_mm":0}]}}
#   ]})
#   → 逐步回执 ok/ms/keys/stored；失败默认即停
# 工件也能手工存：pipe_put(name="my_nums", value={...}) → 后续步骤写 "@my_nums"
# 查看/清理：pipe_list / pipe_get(name, path="a.b") / pipe_clear
```

**硬规则**：① `@引用` 只在 `pipe_run` 内解析（单发 op 不会自动解引用）；② 引用不存在会当场报错并列出现有工件名；
③ 工件活在当前 Blender 会话；④ 长任务仍走 `rt_job`/headless，`pipe_run` 是同步串行。

## 4. 六条必踩的坑

### 4.0 六条坑怎么分流（P2-3：能机械判定的交给门，不能的写进 checklist）

| 坑 | 能不能机械判定 | 落在哪 |
|---|---|---|
| 坑五 **法线方向** | ✅ | **验收门**：`audit_mesh` 的 `normals_state`（`pass`/`fail`/`unknown`/`n/a`）+ `normals_shells[]`（逐壳）；⚠ 别只看总 `signed_volume`（大正向壳 + 小反向壳会抵消成正数），`closed=false` 时 `normals_outward=null`（**不是"没问题"**），`unknown` ⇒ 至少 `degraded`（别的缺陷同时在就是 `fail`），**任何情况都不给 pass** |
| 坑二 **浮块 / 悬空** | ✅ | **验收门**：`audit_connectivity` 的 `floaters[].{span_fraction, gap_mm, mesh_confirmed}` + **`gate.state`**（顶层没有 `state`）、`audit_snap_floaters`（默认只报告）、`audit_gate` 的 `state` |
| 坑二 **穿模 / 隐藏交叠** | ✅ | **验收门**：`audit_interference`（三态 + `volume_mm3`；`unresolved` = 判不了）、`audit_overlap`（`pair_count`；**贴合会给 ≥1**）、`audit_measure` 的 `pairs[].axis_overlap_mm`（**逐轴 AABB：仅当接口法线与世界轴一致且轴对齐盒对该几何适用时才作粗筛**；旋转接口沿接口局部坐标量真实表面，见坑二 ③） |
| 坑一 长轴 / 宽轴 / 薄轴语义 | ❌ 语义，机器判不了 | **写代码前 checklist** |
| 坑三 收尖压中线再合并 | ❌ | **写代码前 checklist** |
| 坑四 headless 慎用 bpy.ops | ❌ | **写代码前 checklist** |
| 坑六 预览光照 / 材质 / 机位 | ⚠ 半（参数进 `lib/preview.py`；出图后看 `coverage_estimate` 与 `warning.frame_looks_empty`） | 装置默认值 + 门⑨ |

**写代码前 checklist（不可机械判定的三条，动手前逐条念一遍）**：

1. **长轴 / 宽轴 / 薄轴**：这个件哪个轴是长？哪个轴是"看得出形状"的宽面？相机在哪一侧？—— 先在注释里写下 `(长, 宽, 薄) = (局部?, 局部?, 局部?)` 再动手（坑一）。
2. **收尖**：要收尖的那一圈顶点，是不是**先压到中线（x=y=0）再 `remove_doubles`**？只按比例缩放一定得到平头（坑三）。
3. **headless**：这段脚本里有没有 `object.mode_set` / `mesh.*` / `loopcut` / `transform.*` / `uv.*`？有 ⇒ 改 bmesh 或 data API（坑四表 + 配方 3b/8）。

**怎么验**：checklist 三条各自能在代码里指出对应的一行（轴注释 / 压中线那两行 / 没有 ops 调用）；
机械判定的三条则必须出现在交付回执里（§0.5.5）。

### 坑一：细长物体的"长轴 / 宽轴 / 薄轴"语义

任何细长或不对称的东西（剑刃、刀、瓶、木板、骨头、螺丝刀头），三个轴的**含义不同**：

- **长轴** = 长度（剑刃 78cm）
- **宽轴** = 宽面（剑刃 4.5cm，是"能看出是把剑"的那个面）
- **薄轴** = 截面厚度（剑刃 0.8cm，也就是刃口方向）

**永远让宽轴朝向英雄镜头的相机。** 从薄轴看过去的剑，渲染出来就是一根细棍。

| 物体 | 局部 X | 局部 Y | 局部 Z |
|---|---|---|---|
| 剑刃 | 薄 0.8cm | 宽 4.5cm | 长 78cm（竖直） |
| 刀刃 | 薄 | 宽 | 长（水平） |
| 木板 | 薄 | 宽 | 长 |
| 瓶子 | 对称（半径） | 对称（半径） | 长（高度） |

建完记得**旋转物体**让宽面朝向相机（相机在 -Y 方向时，把刀刃绕 Z 转 90 度，局部 Y 宽轴就朝向世界 X，宽面可见）。

### 坑二：拼件接缝——可视拼接与真实连接是两回事（★ 全场最高价值的一条）

多部件拼装（剑 = 刃 + 护手 + 柄 + 柄头；坦克 = 车体 + 炮塔 + 悬挂 + 裙板）时，**面贴面**即使数值上接触，渲染也会有可见的缝；不同形状之间（圆柱柄插进方块护手）更明显。
展示模型的做法是**隐藏交叠**：让两件在不可见处互相压进去一点，缝就没了。

> ★ **纠正一条被写死的通则**：老版本把"拼件互相插入 **5–15mm**"当硬规则；v4/v5 把它换成"`1–2×` 接口特征尺寸"，
> 但**那仍是规范性的固定倍数/下限**。**v6 撤回全部这类通则**（见 §13）：本技能不再给任何通用倍数、百分比或下限。

**① 展示层（本技能负责的那部分）：隐藏交叠 + 按 spec 复核**

- 拼接可以在**不改变可见轮廓、不捅穿薄壳**的前提下选择**隐藏交叠**（把交叠埋在较大件内部或不可见面之间）；
- **交叠量由本项目的 spec 明文规定**（`spec.py` 接口表逐条写自己的值）；**没有通则**，spec 没写就回去问规格，别自己发明一个数；
- 装配后按 spec 复核三件事：交叠量（`audit_measure`）、浮空（`audit_connectivity`）、真互穿（`audit_interference`）。

**② 真实连接不由可视脚本裁定**

一件东西**能不能承载、抗扭、抗弯、装不装得上**，由**设计配合（配合与公差）、适用标准、承载工况、制造工艺（壁厚 / 拔模 / 打印方向）与运动间隙**共同规定 —— 这些都是要人给的设计输入。
**本技能的几何脚本只回答"叠没叠、浮没浮、穿没穿"，不能给结构强度结论**：`volume_mm3` / `pair_count` / `axis_overlap_mm` 都不是强度或连接合格证。

**③ 怎么用 `audit_measure` 判交叠（★ 它有口径坑）**

`pairs[].axis_overlap_mm` 是**逐轴 AABB 重叠**，回执自带 `note: "bbox 级（不是三角级）"`。

- **只有"接口法线与某根世界轴一致、且轴对齐盒这个近似对这种几何适用"时，逐轴 AABB 才能当粗筛**；此时读**接口法线那一根轴**的值 —— 不是取最大、也不是求和；
- **旋转 / 斜置接口不能这么读**：接口法线不与世界轴对齐时，"挑一根世界轴"量到的是**盒子的重叠**，不是接口处的深度。必须**沿真实接口的局部坐标**去**量测真实表面**（在接口局部坐标系里取截面 / 量真实三角面），或改用真三角级证据；
- 读法示例（示例只演示"怎么读 AABB"，**不构成任何深度标准**）：Ø6 轴入毂若接口轴与 z 一致、回执给 `{x:6, y:6, z:6}`，z 的 6 是盒级重叠；3mm 薄板搭接给 `{x:180, y:200, z:1.5}` —— x/y 那 180/200 是**板本身有多大**，跟交叠无关；只看单个数或取最大值必然得出错误结论；
- **AABB 相交 ≠ 真的有交叠**：全内含的 10mm 浮块同样给 `overlap_bbox=true`、三轴各 10mm。所以交叠复核必须**与浮空门并跑**（`audit_connectivity` / `audit_gate`）；
- 要**真三角级**穿透证据用 `audit_interference` 的 `volume_mm3` / `audit_overlap` 的 `pair_count`；
  注意 `audit_overlap` 是"真三角-三角相交"，但**共面且互相覆盖的面不报**（插件回执原文），共面贴合要看 `contact_probe`。

**④ 圆润件开平滑着色**：`bpy.data.polygons[].use_smooth = True`（headless 稳）或 GUI 里 `bpy.ops.object.shade_smooth()`。平面着色的圆柱能看出每一段棱面；方块类硬表面保持平面着色（Blender 5.x 已移除 `Mesh.use_auto_smooth`，用修改器式平滑或逐面标记）。

> 实测（500 对象 / 7 agent）：**装配质量几乎全靠这一条**。悬挂臂穿模、驱动轮盖悬空、挡泥板浮空 193mm —— 都是靠"隐藏交叠 + 装配后按 spec 复检"抓出来并收敛的。
> 并行建模时把它写进**团队协议 + 接口表**（§0.5.3，每条接口写自己的 `overlap_mm` 与它在 spec 里的出处）。

想要**真正无缝**（高质量渲染）时，把同材质的件 Boolean Union 成一个网格——但仅限材质相同。

### 坑三：收尖不能只靠缩放

把顶部顶点缩到接近 0 会得到"凿子一样的平头"。正确做法是**把顶点压到中线再合并**：

```python
import bpy, bmesh
obj = bpy.data.objects['GEO-blade']
bpy.context.view_layer.objects.active = obj
bpy.ops.object.mode_set(mode='EDIT')

bm = bmesh.from_edit_mesh(obj.data)
bm.verts.ensure_lookup_table()
max_z = max(v.co.z for v in bm.verts)
top_verts = [v for v in bm.verts if abs(v.co.z - max_z) < 0.001]
for v in top_verts:
    v.co.x = 0.0
    v.co.y = 0.0
bmesh.update_edit_mesh(obj.data)

bpy.ops.mesh.select_all(action='DESELECT')
for v in top_verts:
    v.select = True
bmesh.update_edit_mesh(obj.data)
bpy.ops.mesh.remove_doubles(threshold=0.001)   # 少了这步就是退化拓扑的"假尖"
bpy.ops.object.mode_set(mode='OBJECT')
print(f"tapered:{obj.name}")
```

### 坑四：headless 下慎用 bpy.ops（★ 能省掉很多"不明原因失败"）

`bpy.ops` 的很多算子是**context 依赖**的：要求"当前是编辑模式 + 活动对象是它 + 有选择集 + 有合适的区域（VIEW_3D）"。
在 `blender -b`（无窗口、无区域）里，这些前置条件经常不成立，于是你拿到的是
`RuntimeError: Operator bpy.ops.mesh.xxx.poll() failed, context is incorrect` 这种**看不出原因**的失败。

| 类别 | headless 下的表现 | 建议 |
|---|---|---|
| `primitive_*_add`（加基本体） | ✅ 稳 | 直接用 |
| `object.modifier_apply` / `modifier_add` | ✅ 基本稳（给 `view_layer.objects.active` 就好） | 直接用，注意 apply 前把对象设为 active |
| `object.mode_set` + `mesh.*`（挤出 / 内插 / 环切 / remove_doubles…） | ⚠ 常见 poll 失败 | **改 bmesh**（配方 3b / 8） |
| `mesh.loopcut_slide`、`transform.*`、`uv.*` | ⚠ 依赖区域 / 模态输入 | 不要用；用 bmesh 或直接改 data |
| `object.shade_smooth` / `object.select_all` | ⚠ 依赖 view_layer | 用 data API（见下） |

替代写法（**推荐**）：

```python
# 平坦/平滑着色：直接改面属性，不碰 ops
for p in obj.data.polygons:
    p.use_smooth = False        # 或 True

# 法线一致化：bmesh 版（等价 normals_make_consistent）
import bmesh
bm = bmesh.new(); bm.from_mesh(obj.data)
bmesh.ops.recalc_face_normals(bm, faces=bm.faces)
bm.to_mesh(obj.data); bm.free(); obj.data.update()
```

必须用 ops 时：用 `bpy.context.temp_override(...)` 补齐 context，或者**改走 GUI 通道**（`blender_rt_do`，那里有真的窗口与区域）。
判据：**同一段脚本在 headless 里报 poll/context 错、在 GUI 里正常 → 就是这个坑**，不要去怀疑几何数据。

### 坑五：法线方向是一切"穿模 / 内含"判据的前提（★ 会给出反向结论）

任何形如 `(p − loc) · n < 0 → "在内部"` 的判据（点在内侧、件插进主体多深、面是否朝外），
**都假设法线朝外**。法线朝内时结论会**整体反号**。

实测：车体侧板外表面法线朝内 → 审计把"贴在外壁上的支架"误报成"深入车体 150mm"。
这类错误最坑的地方是**它看起来像几何错了**，于是你去改几何，越改越乱。

**纪律**：

1. 做任何穿模 / 内含判定**之前**，先验法线：
```python
import bmesh, mathutils
def normal_health(obj, probe_point=None):
    """返回 (是否一致朝外, 有符号体积)。有符号体积 < 0 = 整体法线朝内。"""
    bm = bmesh.new(); bm.from_mesh(obj.data)
    bmesh.ops.triangulate(bm, faces=bm.faces)
    vol = 0.0
    for f in bm.faces:
        a, b, c = [v.co for v in f.verts]
        vol += a.dot(b.cross(c)) / 6.0
    bm.free()
    return (vol > 0), vol
```
   （闭合网格的有符号体积 < 0 ⇒ 整体朝内 ⇒ 先 recalc 再判。）
   ⚠ **这只是"单体单壳"的快检**：一个对象里可以有多块互不相连的壳，`bm.calc_volume(signed=True)` 把它们**相加**——
   大正向壳（+8）+ 分离的小反向壳（−1）总和仍是 +7 > 0 ⇒ 反向的那块被吞掉。多壳件交给 `audit_mesh`
   （v0.9.6 起**逐连通壳**判：`normals_state` ∈ `pass`/`fail`/`unknown`/`n/a`，明细在 `normals_shells[]`，含逐壳 `state`/`signed_volume`/`nested`）。
2. 装配层**统一兜底**一次 recalc（Recipe 8 的那一行），别指望每个 builder 都记得。
3. 判据写在报告里时要带**法线前提**："在法线朝外的前提下，X 深入 Y 12mm"。
4. 双向确认：同一对件用 `audit_interference` 与 `audit_measure` 各跑一遍，结论相反就先查法线。
5. **`unknown` 不许当 pass**（v0.9.6 的逐壳判据）：

   | `normals_state` | 含义 | 处置 |
   |---|---|---|
   | `pass` | 每个壳都"可确证是独立实体"且朝外；合法嵌套空腔标 `nested` 豁免（负体积不误报） | 记入回执 |
   | `fail` | 至少一个**非嵌套**壳闭合、绕向一致、`signed_volume < 0` | 先 recalc 再复检 |
   | `unknown` | 判不出来：壳的 AABB 相交但无法确证嵌套/空腔（互穿、部分重叠、重合副本）、闭合/绕向/体积不可判、或（显式设了上限时）面数超过逐壳分析上限 | **不给 pass**：无其它缺陷时 `state=degraded`，同时有缺陷（如互穿被抓成自交）时 `state=fail` ⇒ **补查**（`normals_reason` 给原因；大网格可用 env `DSH_SHELL_MAX_FACES` 放宽），**不要**下"没问题"结论 |
   | `n/a` | 网格未闭合（`closed=false`），"整体朝向"无意义 | 先闭合，或对有意的开口件只按口部法线单独核对 |

### 坑六：给 agent 用的预览装置，默认必须看得清几何突变

实测（三个组员都抱怨过）：白材质 + 白背景的预览图**"白对白"**，板缝、焊缝、分块全看不见，
panel 级根本没法 QC —— 最后是组员自己改成"**灰材质 + 单侧强太阳**"才看清。

**默认就该这么配**（写进你的 `lib/preview.py`，而不是每次临时调）：

- 材质：中性灰（`Base Color ≈ 0.18`、`Roughness ≈ 0.6`、`Metallic = 0`），**不要纯白**；
- 光照：**单侧强太阳**（斜 45°，`angle ≈ 1–3°` 的硬阴影）+ 很弱的环境光 —— 硬阴影能把 1–2mm 的台阶照出来；
- 背景：中灰或渐变，别用纯白 / 纯黑（前者吃掉高光，后者吃掉暗部）；
- 零件级 QC：用 `mode="parts_color"` 逐部件配色 + 图例，比"整体一个灰"更容易发现"少了一块 / 装反了"；
- **每次至少两个机位**：一个"英雄机位"（像不像）+ 一个"斜 45° 检查位"（缝、浮空、穿模）。

判据：**如果一张图看不出板缝在哪，这张图就不能作为 QC 证据**。

## 5. 修改器栈序与症状表

> ★ **纠正一条被写死的"固定链"**：老版本让人"背下来"这一串 ——
> `Mirror → Array → Solidify → Bevel → Subdivision Surface →（需要时 Boolean）`。
> **不存在这样一条通用顺序。** 修改器栈是一个**按目标函数排序的求值链**：同一个基础网格，
> 只换两三条的顺序，**拓扑和尺寸都会变**。实测（Blender 5.2.2，同一个 1m 立方体）：

| 顺序 | verts / faces | bbox 最长边 |
|---|---|---|
| Bevel(0.2/2 段) → SubSurf(2) | **866 / 864** | **0.9904 m** |
| SubSurf(2) → Bevel(0.2/2 段) | **98 / 96** | **0.8395 m** |
| Solidify(0.03) → Bevel(0.05/2 段)（平面） | **56 / 54** | — |
| Bevel(0.05/2 段) → Solidify(0.03)（平面） | **8 / 6** | — |

两边的结果**都对**，取决于你要的是"圆角被细分保留"还是"细分后的低模再被倒圆"；
`bbox` 差 **15%** 说明这不只是面数问题。⇒ 顺序靠**目标**决定，不靠背。

**怎么定顺序**：从目标反推，逐条问"这一步要吃上一步的什么"。

| 目标 | 顺序（相对） | 为什么（设计意图） |
|---|---|---|
| 镜像件要浑然一体 | **Mirror 在前**，Bevel / SubSurf 在后 | 让镜像缝先焊成连续面，再统一倒角/细分；反过来缝处会被当成自由边处理 |
| 重复阵列的每个单元各自带倒角 | **Bevel 在前**，Array 在后 | 后级只复制前级的结果；反了则倒角作用在"整排"的接缝上 |
| 薄壳（钣金/壳体）要边缘圆润 | **Solidify 在前**，Bevel 在后 | 先有厚度才有"实心边"可倒；实测反着放倒角基本被吃掉（54 面 → 6 面） |
| 光滑曲面但要保棱 | **Bevel 在前**，SubSurf 在后 | 倒角提供支撑边，细分才不会把棱圆掉（配方 2b） |
| 分面硬表面（装甲板/军模） | **不要 SubSurf** | 细分会把棱边全部圆成"肥皂"→ 走配方 2a（小倒角 + 平坦着色） |
| 布尔开孔 | Boolean **尽量靠后**、导出前再 apply | 保持非破坏；布尔后的 n-gon 还会影响后续细分 |
| 拓扑定型后再镜像 | **apply Mirror 在改拓扑之前** | 拓扑一变，镜像缝的对齐就不保证 |

**顺序对不对，用回执判**（不是靠背）：`evaluated_get(depsgraph).to_mesh()` 取求值后的
`verts / faces / bbox`，与你要的目标对拍；再加 §6 的数值门。**改了顺序就重跑一遍门。**

补充规则：布尔保持 live 到导出前；拓扑定型再 apply 镜像；**不要叠多层 subsurf 而没有支撑边**；应用缩放（Apply Scale）之后再布尔 / 导出。

| 症状 | 修法 |
|---|---|
| 一看就是默认方块 | 加 Bevel（0.02m / 3 段）+ SubSurf（**仅限光滑件**） |
| **装甲棱边被圆掉、像塑料玩具** | 这是分面件误用了 SubSurf → 删掉 SubSurf，改走配方 2a（小倒角 + 平坦着色） |
| 圆形上出现夹点 | Bevel 要在 SubSurf **之前** |
| 渲染出黑面 | 重算法线（坑五）；先查有符号体积判断是不是整体朝内 |
| 布尔产生 n-gon | apply 布尔后进编辑模式改回四边面，再 SubSurf |
| 对称破掉 | 用 Mirror 修改器，不要复制再翻转 |
| **网格里有看不见的内部面 / 埋在实体里的面** | ★ **不是 `Delete Loose`**（它只删"不属于任何面的点/边"，一个面都不删）→ 见下方专条 |
| 细分后发皱 / 塌陷 | 缺支撑边：加环切固定结构，别继续加细分层级 |
| 阵列首尾对不上（履带 / 链节） | 本配方口径的 `FIT_CURVE` 阵列给不了精确闭环节距 → 配方 6b 参数化**销轴弦长**路径 |
| headless 里 ops 报 context/poll 错 | 坑四：改 bmesh / data API，或用 temp_override |

### 5.1 「删除内部面」的正确做法（★ 老版本这里教错了）

老版本的症状表写"网格里有看不见的内部面 → Clean Up → **Delete Loose**"。**这是错的**：
`Delete Loose` 只删除**不属于任何面的顶点与边**，**一个面都不会删**。

**实测（Blender 5.2.2）**，两种内部面场景，Blender 自己打印的是一行 `Info`：

| 场景 | 内部面 | 跑 `delete_loose` 后 |
|---|---|---|
| 立方体里塞一张独立浮动四边形 | 1 张 | `Info: Removed: 482 vertices, 992 edges, **0 faces**` → 那张面还在 |
| 两个重叠立方体 join（每个壳各有一面埋进对方） | 2 张 | `Info: Removed: 0 vertices, 0 edges, **0 faces**` → 面数仍是 12 |

顺带否掉第二个常见误解：`bpy.ops.mesh.select_interior_faces()` **也不解决**这个问题 ——
上面那个 join 出来的重叠对，它选中 **0** 张面（它对"每条边都有 >2 个面用户"那类才有效）。

**正确做法（按成因分）**：

```python
import bpy, bmesh
from mathutils.bvhtree import BVHTree

def buried_faces(obj, eps=1e-5):
    """埋在实体内部的面：从面中心沿**外法线**打射线，命中 = 该面被别的几何挡住了。
    （clean cube -> 0；两个重叠立方体 join -> 恰好 2）"""
    bm = bmesh.new(); bm.from_mesh(obj.data)
    bvh = BVHTree.FromBMesh(bm)
    out = []
    for f in bm.faces:
        c = f.calc_center_median() + f.normal * eps
        loc, nor, idx, dist = bvh.ray_cast(c, f.normal, 1e6)
        if idx is not None:
            out.append(f)
    bm.free()
    return out
```

| 成因 | 正确处置 |
|---|---|
| **join / 复制出来的重叠壳体**（最常见） | **Boolean UNION**（或 `mesh.intersect` 的 Boolean）把两壳并成一个实体；**别用"选中再删"** —— 实测删掉埋面后留下 **8 条开边**（壳被捅开了），还得再补面 |
| Boolean 求解残留的碎片 | 换 `solver='EXACT'` 重跑，或 `fix_repair`（`merge_doubles` + `dissolve_degenerate` + `delete_loose` + `recalc_normals`）—— 注意这里的 `delete_loose` 干的是它**真正**该干的活（清孤立点/边） |
| 建模时手滑多做的内壁 | 用上面的 `buried_faces()` 找出来删，**删完必须补面或改走布尔**，并复检 `audit_mesh` 的 `boundary_edges` |
| 起因不明 | 先 `audit_mesh` 看 `self_intersections` / `nonmanifold_edges` / `boundary_edges` —— **`self_intersections` 才是"互相穿进对方"的机器判据** |

**判据**：删内部面之后 `audit_mesh` 必须仍是 `clean=true`（`boundary_edges=0` / `nonmanifold_edges=0`）。
只盯着"面数变少了"而不管开边，等于把穿模换成了破面。

## 6. 数值门清单（交付 / 交接前逐条过）★**本节是唯一的权威门表**，其余章节一律引用它

> v1 这一节只有"清理清单"，实测证明**肉眼验收不可靠** —— 这里给每条判据配上可命令与字段。
> 单件交付跑 ①–④，装配完成后跑 ①–⑧。**缺字段 = 没跑**。
>
> ★ **三态口径（v0.9.6，比"红/绿"多一态）**：`audit_mesh` / `audit_scene` 顶层给 `state`（等价字段 `verdict`）= `pass` / `fail` / `degraded`。
> `clean` 收紧为"**查完了而且没缺陷**"⇒ `degraded` 时 `clean=false`。**`clean=false` 有两种成因**（确有缺陷 / 没查完），
> 必须看 `state` + `reason`；`degraded` 的处置是**补查**（不是放过，也不是当失败）。

| # | 门 | 判据 | 怎么跑（命令 → 看的字段） |
|---|---|---|---|
| ① | 重复点 / 松散几何 | 0 孤立点、0 松散边 | `audit_mesh` → `totals.loose_verts=0` **且** `totals.loose_edges=0`（+ `remove_doubles`） |
| ② | 流形 | 0 非流形边（有意开口除外） | `audit_mesh` / `audit_scene` → `totals.nonmanifold_edges=0`；⚠ `audit_scene` 默认**不跑自交** ⇒ `state=degraded`，要当门用必须传 `self_intersect=true` |
| ③ | 开边 | 0 开边（闭合件口径） | `audit_mesh` → `totals.boundary_edges=0`、`objects[i].closed=true` |
| ④ | 法线 | **逐连通壳**一致朝外（不是"总 `signed_volume > 0`"） | `audit_mesh` → `objects[i].normals_state`（`pass`/`fail`/`unknown`/`n/a`）+ `normals_shells[]`（逐壳 `state`/`signed_volume`/`nested`）；⚠ `closed=false` ⇒ `normals_outward=null`（开放网格没有"整体朝向"这回事）；`normals_state=unknown` ⇒ **不给 pass**（无其它缺陷时 `state=degraded`，有缺陷时 `state=fail`，看 `normals_reason`） |
| ⑤ | 尺寸 | 关键 bbox 与 `spec.py` 偏差 ≤ 公差（军模级 ±3mm） | `audit_measure` → `items[].size_mm` |
| ⑥ | 浮空 | 0 悬空件（可见浮块 = 最长 bbox 边 ≥ 全模型 1%） | `audit_connectivity` → `gate.state=pass` **且** `floaters=[]`（⚠ 顶层**没有** `state`；**别拿"分量数=1"当判据** —— 贴而未焊天然是多分量）/ `audit_snap_floaters`（默认只报告）/ `audit_gate` 的 `state` |
| ⑦ | 穿模 | 无异常互穿；允许的隐藏交叠白名单化 | `audit_interference` → `verdict` + `volume_mm3`（⚠ `unresolved` = **判不了**，不是"无干涉"；贴合接触常给 `unresolved`）+ `audit_overlap` → `pair_count`（⚠ 贴合会 ≥1；`analyzed=false` ≠ 无重叠） |
| ⑧ | 隐藏交叠 | 逐接口交叠量对齐**本 spec 明文**的 `overlap_mm`（坑二：**无通用倍数 / 百分比 / 下限**；不改可见轮廓、不捅穿薄壳） | `audit_measure` → `pairs[].axis_overlap_mm`（**仅接口法线与世界轴一致且轴对齐盒适用时作粗筛**；旋转接口沿接口局部坐标量真实表面）+ `audit_connectivity`（同一对件若同时报浮空，"交叠"不成立） |
| ⑨ | 形体 | 固定机位对照图 + 数值 bbox（两条都要） | `qc_render_views`（固定机位 + `max_size=900`）+ `audit_measure`；**IoU 只判轮廓**，盲区见 §0.6.2 |
| ⑩ | 面数 | 对三角面预算 | `audit_scene` / `audit_gate` 的 `tris`（只看数，不看状态） |
| — | 证据留档 | 源一变即作废 | `fingerprint.digest` + `.blend` 的 md5 |

**MUST**：非破坏到导出阶段；对象一建就命名（GEO- 前缀）；blockout 与成品分离；布尔 / 导出前应用变换；
**多 agent 时先冻结 spec.py 与接口表**；每个件交付带数值证据。
**MUST NOT**：轮廓没认可就删 blockout；过早 apply 所有修改器；留默认名；没有比例参照就开始建模；跳过清理；
**用单张成品图验收形体**；**在法线未验证的前提下下穿模结论**；**拿单视 IoU 当"整体像不像"**。

### 6.1 交接前必跑（P0-4）：三条命令 + 一张回执

照抄命令；**判据与字段一律查 §6 那张表**（本节不重复门表）。

```python
# ① 单件 / 集合的网格健康（boundary / nonmanifold / degenerate / loose / self-intersections / normals）
blender_rt_plan(op="audit_mesh", args={"scope": "COL_Geo"})
# ② 出厂门：连通 + 包络（★ 装配口径；只有单件时请只用 ①）
blender_rt_plan(op="audit_gate", args={"scope": "COL_Geo", "envelope": [x0, y0, z0, x1, y1, z1]})
# ③ 穿模 / 干涉（逐对；设计上允许的隐藏交叠要白名单化）
blender_rt_plan(op="audit_interference", args={"a": "GEO-rungear", "b": "GEO-hull"})
# ④ 全场景初筛（★ 交付验收必须显式开自交检查：默认不跑 ⇒ state=degraded，拿不到 pass）
blender_rt_plan(op="audit_scene", args={"summary_only": True, "top_k": 10, "self_intersect": True})
```

> ★ **`audit_scene` 的自交开关（v0.9.6 实测）**：它按性能契约**默认 `self_intersect=false`**（便宜初筛），
> 这时无论场景多干净，回执都是 `state=degraded` + `clean=false` + `warn: "DEGRADED ≠ 通过"` +
> `hint: "要完整结论请用 self_intersect=true 重跑"` —— **"跳过检查"不许读成"没问题"**。
> ⇒ **交付/验收口径一律传 `self_intersect=true`**（代价按面数走）；只看"哪几个最脏"的日常初筛可以不开，
> 但那时**不能**把回执当门。

**验收回执模板 + 三条硬纪律见 §0.5.5**（含 `.blend` md5 与 `fingerprint.digest` 的写法）。

**三条口径（实测，别混）**：
1. `audit_mesh` 读**未求值的 `ob.data`** —— 未 apply 的修改器不算数（带 Mirror 的半个件报基础网格的 `verts=4 / boundary_edges=4`）；
   装配级的 `audit_connectivity` / `audit_gate` / `audit_measure` / `audit_interference` 走**求值网格**（Boolean 未 Apply 也算数）。要拿 `audit_mesh` 判"改完没有"，先 apply 掉相关修改器。
2. `ok=false` / `analyzed=false` / `state="degraded"` **都不是通过** —— 那是"没跑"或"没分析完"，不能读成"没问题"。
3. **`clean=false` 有两种成因**：`state=fail`（确有缺陷，看 `totals` 里非 0 的计数字段）或 `state=degraded`（没查完/判不出来：自交被跳过或抛异常、`normals_state=unknown`、逐壳分析超面数上限）。拿到 `clean=false` **先读 `state`**，再决定"改几何"还是"补查"。

## 7. 深水参考

需要超出配方的精度时读 `references/modeling-overview.md`（英文原文，含：bpy.ops.mesh 与 bmesh 两套 API 的选择规则、8 种基本体的专业选择、编辑模式工具箱逐条说明、bmesh 精度 API、硬表面 bevel-weight 打法、布尔工具箱、关键修改器、完整 bpy.ops.mesh 速查，以及作者自己标注的未完成项）。

什么时候去读：配方不匹配（需要 bmesh 级精度或自定义操作）；拓扑要求比平时严（动画可用、游戏 LOD）；性能敏感（foreach_set、批量操作）；要用到配方里没列的算子。

**`references/multiview-calibration.md`**（本技能自带）：三视图联立标定的可运行脚本（纯标准库，`--selftest` 自检）+ 判据表 + 取点规则 + "怎么知道标定是对的"的自证口径 —— 走 §0.6.1 时读它。

## 8. 来源与许可

本技能由以下两个 MIT 许可项目的**建模流程部分**融合改写而成，工具调用一律换成本机 `blender_rt_*` 直连链：

- **RobLe3/cc-blender-skill**（`blender-modeling` 及其 `references/overview.md`）——决策树、配方、坑表、深水参考
- **arjun988/blender-skills**（`blender-modeler`）——六步总纲、集合结构、修改器规则、清理清单、MUST / MUST NOT

完整归属与许可见同目录 `NOTICE.md`。上游原版面向 MCP（mcp__blender__*），本版已改写为本机直连工具链。

## 9. 本次更新对照（v2 · 2026-09-25）

来源：一次 500 对象 / 7 agent 的军模项目复盘（skill 命中率约 1/3，失效的全是"载体不匹配"）。

| 反馈 | 落在哪 |
|---|---|
| 真正救场的只有"拼件互相插入 5–15mm" | 坑二升级为"全场最高价值"，并写进 §0.5.3 接口表（**v4 已修**成"深度不是常数"，**v6 再修**：连"`1–2×` 特征尺寸"这类固定倍数也撤回 —— 见 §13） |
| skill 缺"多 Agent 并行建模的分工范式" | **新增 §0.5**（冻结规格 / 写作用域 / 接口表 / 深嵌+复检 / 数值门 / 反模式；**v6 起**"深嵌"改称"隐藏交叠 + 按 spec 复核"） |
| skill 缺"参考图驱动的形体还原" | **新增 §0.6**（三步标定 / 双证据验收 / 读图三分法） |
| skill 缺"headless 下慎用 bpy.ops" | **新增坑四** + 配方 3b（bmesh 等价写法）+ 配方 2a/8 的替代写法 |
| 形体不能靠单张成品图验收 | **新增配方 10** + §6 门⑨ |
| 配方 2（Bevel+SubSurf）完全没用（分面装甲被圆掉） | **拆成 2a 分面 / 2b 光滑**，决策树同步修正 |
| 配方 3（编辑模式 ops）没用（headless context 脆） | 配方 3 保留给 GUI，**新增 3b bmesh 版** |
| 配方 6（Array 沿曲线）没用（**该项目的载体**要精确闭环节距，`FIT_CURVE` 给不了） | 保留装饰性用途，**新增 6b 参数化路径**（当时给的 83 节 / 0.176 节距；**v4 已修**：原实现是恒原点占位、跑不出形状，且误用等弧长口径 —— 见 §11） |
| 第 6 节清理清单被扩展成自动门 | 升级为 **§6 数值门清单**（10 条判据 + 工具） |
| 预览图"白对白"、panel 级没法 QC（组员自己改灰材质+单侧强太阳） | **新增坑六**（预览装置的默认光照 / 材质 / 机位纪律） |
| 法线朝内导致穿模判据整体反号（误报深入 150mm） | **新增坑五**（判据前先验法线 + 装配层统一 recalc） |
| 插件侧：headless/job 回执"收不回来"、agent 点不亮 GUI | 插件 v0.9.4 修返回通道 + 加 `blender_viewport(op="launch")`（见插件 `CHANGELOG.md`） |

## 10. 本次更新对照（v3 · 2026-09-26 · 实测驱动）

来源：《实测驱动的通道 + Skill 优化方案-2026-09-26》的 **§4 Skill 侧**（P0-4 / P0-5 / P1-3 / P1-5 / P1-4 / P2-3）。
**只增补方法论层，流程骨架（六步总纲 / 决策树 / 配方编号 / 修改器栈 / 症状表）保持 v2 原样。**

| 条目 | 改了什么 | 落在哪 |
|---|---|---|
| P0-4 验收门必跑清单 | "交接前必跑"写成清单 + **可复制的验收回执模板**（含 `.blend` md5）+ 三条硬纪律（单件用 `audit_mesh`、装配才用 `audit_gate`；判据要"红过"才算数；`ok=false` ≠ 通过） | §0.5.5「交接前必跑」、§1 交接步、**§6.1** |
| P0-5 读图纪律 | 迭代 **420px** / 验收 **900px + 固定机位**；**只在新 hash 时看图**（回执带 hash，相同 hash = 画面没变） | §0.6.3 |
| P1-3 多视角联合标定 | §0.6.1 扩成**三视联立标定**（逐视最小二乘 + 主点偏移 + 三视互检）；判据 **残差 >2% 或两两比例互差 >3% ⇒ 判"非正交/透视"，退回单视**；配可运行脚本（**v4 已修**：比例互差**不能**判透视，见 §11 #2；脚本判据与 selftest 已重写） | §0.6.1 + **`references/multiview-calibration.md`**（新建） |
| P1-5 归属 + 可并行性 | **可并行性判据表**（共享对象 / 共享文件 / 共享材质 ⇒ 不可并行）+ `COL_<agent>` 集合与 `dsh.owner` 属性 + `rt_txn mark(objects=...)/revert` **只回滚一个 builder** | §0.5.7 |
| P1-4 参数化配方 | 5 条高频配方（2a / 4 / 5 / 6b / 9）升级为「**参数块 + 生成脚本**」，接插件**已有**的 `generator_save/run/list/get/diff`；写明"`generator_run` 子进程没有 `K`/`audit_*`"这条实测坑（**v5 已修**：改参不再"必须走内核"，直接调工具即可 —— 见 §12 #1） | §3.0 + 五条配方末尾 |
| P2-3 六条坑分流 | 法线 / 浮块 / 穿模 接**验收门**；长轴语义 / 收尖压中线 / headless 慎用 ops 写进「**写代码前 checklist**」 | §4.0 |

判据来源（方案 §0 实测）：同一画面连发 5 次 **hash 相同**；560px 单帧 **116,870 B**；本机 420px 单帧 **79,428 B**。

**怎么验**：`git diff --stat` 只应看到 `SKILL.md` / `README.md` 与新增的 `references/multiview-calibration.md`；
§3 的配方编号 1–10（含 2a/2b/3b/6b）一个没变；把新参考文档里的脚本存成 `multiview_calib.py` 跑 `--selftest` ⇒ `PASS`。

## 11. 本次更新对照（v4 · 2026-09-26 · 纠正被写死的错口径）

来源：实测反馈 —— v3 里几条被当成"通则"的写法**本身是错的**。每条都先用本机 Blender 5.2.2
headless / 独立进程量过再改（GUI 场景与插件均未改动）。

| # | 原来写的（错） | 改成 | 落在哪 | 实测证据 |
|---|---|---|---|---|
| 1 | 拼件**必须**互相插入 **5–15mm** | 嵌入深度按**接口类型与尺度**定（`1–2×` 接口特征尺寸）；5–15mm 只是"特征尺寸 5–30mm"那一档；给接口类型表 + 四条不等式 + `audit_measure` 读法 —— ⚠ **本行已被 v6 取代**：固定倍数 / 下限与 5–30mm 推导全部撤回，见 §13 | 坑二 · §0.5.3 · §0.5.4 · §0.5.5 · §6 门⑧ · §6.1 | 3mm 薄板搭接 **1.5mm 合格**；Ø6 轴 **6mm**；10mm **浮块** `overlap_bbox=true` 但那是缺陷（`audit_measure` 回执，`note: "bbox 级（不是三角级）"`） |
| 2 | 三视**必须共享同一个 px_per_mm**；"两两比例互差 >3% ⇒ 非正交/透视" | 正交**不推出**共享比例；**跨视互检换算成毫米再比**；`s` 互差只决定"能不能合并成一个联合比例" | §0.6.1 · `references/multiview-calibration.md`（脚本判据重写 + selftest 四例） | 三个正交相机 `ortho_scale` 4/8/2 m：视差探针全 **0.000px**，`px_per_mm` = 0.48/0.24/0.96 → 互差 **300%** |
| 3 | 跨深度尺寸**公差放宽到 ±5%** | 误差上界 **`err ≈ Δz/(D+Δz)`**；估不出 `D`/`Δz` 记 `unresolved` | §0.6.1 ⑤ · multiview-calibration §5 · SOP §3.2 | D=8m：Δz=0.5m → **−5.88%**；1m → **−11.11%**；3m → **−27.27%**（解析式与实测逐行吻合） |
| 4 | 照片"必须先做透视校正（求单应），否则尺寸全错" | 单应**只校正取点那个平面**；平面参考放心用，立体物只有该平面准 | §0.6.1 ⑥ · Recipe 18 硬规则① · Recipe 20 硬规则④ · SOP §3.1 | 平面内 **0.000px**；离平面 0.05m → **52mm**；0.2m → **211mm**；0.5m → **543mm** |
| 5 | 用 IoU 判「像不像」（`target_iou` 当验收） | IoU 是**单视轮廓**的**必要不充分**条件；给盲区表 + 4 条纪律 | §0.6.2 新增小节 · Recipe 17④ · Recipe 18⑤ · Recipe 19③ · SOP §4 · `验收清单.md` 主观门 | 盲区为分析性结论（深度/内部特征/面感不在轮廓里）；刷高方式=填满轮廓 |
| 6 | 修改器顺序**背下来** `Mirror→Array→Solidify→Bevel→SubSurf→(Boolean)` | **不存在固定链**；按**目标**排（给"目标 → 顺序"表）+ 用求值回执核对 | §5 · `references/modeling-overview.md` | 同一立方体：Bevel→SubSurf **866v/864f, bbox 0.9904** vs SubSurf→Bevel **98v/96f, bbox 0.8395**；平面 Solidify→Bevel **54f** vs Bevel→Solidify **6f** |
| 7 | 内部面 → Clean Up → **Delete Loose** | `Delete Loose` **一个面都不删**；`select_interior_faces` 对 join 重叠壳也选 0 → Boolean UNION 或射线法找埋面 | **§5.1 新增** · §5 症状表 · `modeling-overview.md` | 场景一 `Removed: 482 vertices, 992 edges, **0 faces**`（内嵌平面仍在）；场景二 `Removed: …, **0 faces**`（面数仍 12）；`select_interior_faces` 选中 **0** |
| 8 | Recipe 6b：`path(t)` **恒原点**占位、节距用**等弧长**、`bbox 长度轴 ≈ n*pitch`、`expect objects == n` | 改成**真正可运行**的闭合赛道形；区分**等弧长**与**销轴弦长**节距；修正 bbox（`A+2R`）与对象数（`2*(m+n_straight)`，模板须删） | Recipe 6b · §3.0 参数表 · 决策树 · §9 相关行 | 弦长口径 `max|弦长−p|` = **3.3e-06mm**、闭合弦长 **175.999998mm**、对象 **24**；等弧长：弧上弦长 **175.021mm**（每节 −0.979mm，两弧累积 **−11.748mm**，每半圆需 6.0343 步=非整数）；`bbox 最长边 = A+2R = 1.736012m`（旧写法 1.056m，**错 680mm**；`N*p = 4.2240m` ≠ 周长 `4.2483m`） |
| 9 | 配方数写"10 个" | 实际 **22 个编号**（含 2a/2b/3b/6b 变体），frontmatter `description` 同步 | frontmatter · README | 逐条数 Recipe 1…22 |

**同步范围**：§0.5.3 / §0.5.4 / §0.5.5 / §0.6.1 / §0.6.2 / §5 / §5.1 / §6 / §6.1 与
`README.md`、`references/{multiview-calibration,参考图通用还原SOP,验收清单,modeling-overview}.md` 全部对齐到新口径；
重复的表述做了合并（验收门只留一张权威表 + 引用），**流程骨架不动**（六步总纲 / 决策树 / 配方编号 / 症状表结构）。

**怎么验（v4）**：
1. `python3 multiview_calib.py --selftest` ⇒ 末行 `SELFTEST PASS`（A 联立 / B 视内残差红 / C 分尺度逐视可用 / D 跨视互检红）；
2. Recipe 6b 脚本按原样在 headless 里跑 ⇒ `N=24`、`max|chord-p| = 3.3e-06mm`、`closure = 176.000mm`、对象数 24、`bbox_x = A+2R`；
3. `grep -n "5–15mm" SKILL.md` 只剩**纠正性**表述（"不是固定 5–15mm"），没有把 5–15mm 当通则的句子（**v6 复验口径见 §13**）；
4. `grep -n "Mirror → Array → Solidify" SKILL.md` 只应命中 §5 的**纠正引用**，不应有"背下来"的用法。

## 12. 本次更新对照（v5 · 2026-09-26 · 对齐插件 v0.9.6 与当前配方边界）

来源：并行修复完成后的**技能集成**（只改技能：`SKILL.md` / `README.md` / `references/验收清单.md`；**插件一行未动**）。
每条都对着**插件 v0.9.6 的真源码/真回执**写，改完在本机 headless 复跑（见"怎么验"）。

| # | 原来写的 | 改成 | 落在哪 | 依据 |
|---|---|---|---|---|
| 1 | §3.0「改参复现**只能走 Python 内核**」+ 警告"不要写成 `op="generator_run"` + 嵌套 `args`" | **首选工具直调**：`args` 这一层就是生成器 `PARAMS`；内核写法降为**备选**；**保留旧版迁移说明**（v0.9.5 及更早把 `args` 无条件摊平 ⇒ `generator_run() got an unexpected keyword argument 'n'`、`PARAMS` 变空 `{}` ⇒ 在旧插件上照内核写法或升级） | §3.0（表格 + 两段代码 + 迁移说明）· §10 P1-4 行 · Recipe 6b 末尾 | `generator.py::generator_dispatch`（v0.9.6：`op=="run"` 时 `args` 原样传、绝不摊平）；本会话**直调实测** `verdict=supported` 且 `receipt.PARAMS == args` |
| 2 | `clean` 只有真/假两种读法 | `audit_mesh` / `audit_scene` 顶层 **`state`（= `verdict`）三态** `pass` / `fail` / `degraded`；`clean` 收紧为"**查完了而且没缺陷**" ⇒ **`clean=false` 有两种成因**（确有缺陷 / 没查完），先读 `state`+`reason`；`degraded` 的处置是**补查** | §0.5.5 回执模板 + 硬纪律 3 · §4.0 坑五行 · §6 表头三态口径 + 门②④ · §6.1 口径 3 · `验收清单.md` | `audit.py`：跳过/抛异常的自交检查、`normals_state=unknown`、超面数上限 ⇒ 逐对象 `state=degraded`；`clean = not bad and state == "pass"` |
| 3 | `audit_scene` 可以"`clean=true` 即通过" | **默认 `self_intersect=false`**（性能契约）⇒ 无论多干净都 `state=degraded` + `warn`/`hint`；**交付/验收一律传 `self_intersect=true`**；只看"哪几个最脏"的初筛才不开 | §0.5.5 回执模板 · §6.1 新增 ④ 命令 + 开关说明 · §6 门② · `验收清单.md` | `audit_scene` 源码：`elif (not self_intersect) … state = "degraded"` + `hint: "要完整结论请用 self_intersect=true 重跑"` |
| 4 | 门④"一致朝外（闭合件有符号体积 > 0）" | **逐连通壳**判：`normals_state` ∈ `pass`/`fail`/`unknown`/`n/a` + `normals_shells[]`（逐壳 `state`/`signed_volume`/`nested`）；**合法嵌套空腔**（负体积）标 `nested` 豁免；**聚合 `signed_volume > 0` 不是充分条件**（大正向壳 + 分离的小反向壳会被抵消成正数）；`unknown` **一律不给 pass**（无其它缺陷时 `state=degraded`，有缺陷时 `fail`） | §4.0 坑五行 · 坑五 body（新增五条纪律 + `unknown` 处置表）· §6 门④ · §0.5.5 回执模板 | `audit.py::_shell_normals()`（逐壳体积/闭合/绕向 + AABB 关系 + 射线奇偶确认嵌套）；`SHELL_MAX_FACES=300000` 超限 ⇒ `unknown`；实测：大正向壳(8.0) + 分离小反向壳(−0.125) 聚合 `signed_volume=7.875 > 0` 但 `normals_state=fail`、逐壳 pass/fail 两态 |
| 5 | Recipe 6b 直接 `bpy.data.objects[TEMPLATE]` —— **空场景必 `KeyError`**，且没人说模板的原点/销轴在哪 | 补 **①–⑤ 模板契约**（原点 = 起点销轴 / 局部 +X = 下一销 / 下一销在 `(+pitch,0,0)` / 销轴 = 局部 +Z / 只有 z 被沿用）+ **从零建可视单节** `build_link_template()`（纯 bmesh，无前置）+ 明确**本配方只解"节距 + 闭合"**（相邻节共面贴合；内/外板交替与销归属不在范围内） | Recipe 6b | headless 原样复跑：空场景可直接执行，`N=24`、`max\|弦长−p\| = 3.3e-06mm`、`closure = 176.000mm`、`bbox_x = A+2R`（与 v4 数字一致） |
| 6 | "Array **做不到**精确节距闭合"、"写实人脸在零素材下**做不到**"（绝对表述） | 收窄为**当前配方边界**：Array 的短板是 `FIT_CURVE` **沿曲线**的精确弦长节距 + 闭环（**直线等距阵列不受影响**）；写实人脸是"本技能 22 条配方 + 零素材"这条路的边界（有参考照片走 Recipe 20 校正；外部素材/雕刻素材属本技能之外的输入） | 决策树 · Recipe 6 载体判据 · §5 症状表 · Recipe 16 硬规则① · §9 相关行 | 无依据的绝对表述一律降到"本配方口径"；**未新增任何固定数字** |
| 7 | 自述体量 `68,823 字符 ≈ 23–29k` / 清单 `3,117 字符` | 更新为**编辑后实测的近似值**：SKILL.md **约 81k 字符 / 约 125 KB ≈ 27–34k**、`验收清单.md` **约 4.2k 字符 ≈ 1.4–1.8k** | §0 独立验证段 · `验收清单.md` 头部 | `len()` / `wc -c` 实测 |
| 8 | Recipe 6b 验收第 4 条"`audit_connectivity` 里履带是**一条连通分量**"；以及"相邻节共面贴合 ⇒ `audit_overlap` 不报" | 按实测改成**链节装配口径表**："不散架"看 `floaters=[]` + **`gate.state`**（顶层没有 `state`）；**分量数是 14 不是 1**（贴而未焊天然多分量）；相邻节 `audit_overlap` **`pair_count=1`**（端面共用的那条边，120mm 真交线段）、`audit_interference` 给 **`unresolved`**（`volume_mm3=0.0`，统计上界 6.33mm³ > 可执行下限 1.0mm³）⇒ 设计接触要白名单化，别拿 `pair_count` 当穿模开关 | Recipe 6b 验收 + "链节装配口径表" · §6 门⑥⑦ · §4.0 浮块/穿模行 · `验收清单.md` 连通与干涉两行 | headless 实测（`audit_connectivity` 14 分量 / `floaters=[]` / `gate.state=pass`；`audit_overlap pair_count=1`、`point_kind="平面求交 + 三角形裁剪（真交线段）"`；`audit_interference verdict="unresolved"`）；源码口径见 `audit.py::audit_connectivity` 的顶层键（无 `state`，有 `gate`） |

> ⚠ **本次刻意不做的事**：① 不改插件（任何 `runtime/*.py`、`src/*`）；② 不新增**无实测依据**的数字 ——
> 正交/透视公式范围（`Δz/(D+Δz)`、单应跨平面误差）**逐字保留 v4 原值**；
> **嵌入深度那段（`1–2×` 特征尺寸 / 接口类型表 / 四条不等式）已由 v6 取代并删除**（见 §13），此处不再是现行口径；
> ③ 不动流程骨架（六步总纲 / 决策树 / 配方编号 1–22 / 症状表结构）。

**怎么验（v5）**：
1. Recipe 6b 脚本（含新的 `build_link_template()`）在**空场景** headless 里跑 ⇒ 不报 `KeyError`，且
   `N=24`、`max|chord-p| ≤ 0.001mm`、`closure = 176.000mm`、场景对象数 24（模板已删）、`bbox_x = A+2R`；
2. `blender_rt_plan(op="audit_mesh", args={objects:[…]})` ⇒ 回执同时有 `state` 与 `verdict`（同值），
   `clean=true` 只在 `state="pass"` 时出现；`audit_scene` 不传 `self_intersect` ⇒ `state=degraded`，传 `true` ⇒ 可 `pass`；
   `normals_state=unknown` ⇒ 不给 pass（无其它缺陷 `degraded`、有缺陷 `fail`）；
3. `blender_rt_plan(op="generator_run", args={"name": …, "args": {…}})` ⇒ `verdict` 三态且 `receipt.PARAMS` 与 `args` 一致（直调，不经内核）；
4. Recipe 6b 的 24 节跑装配门 ⇒ `audit_mesh` 全 `pass/clean=true`；`audit_connectivity` `floaters=[]` + `gate.state=pass`（**分量数 14**，不是 1）；
   相邻节 `audit_overlap pair_count=1`（`真交线段`）、`audit_interference verdict="unresolved"`、`volume_mm3=0.0`；
5. `grep -n "只能走 Python 内核\|干净件有符号体积 > 0\|Array 做不到\|写实人脸在零素材下\|68,823\|3,117"` ⇒ **只应命中 §9–§12 的纠正引用**（规范正文里必须为零）；配方编号 1–22 与 2a/2b/3b/6b 一个不变。

## 13. 本次更新对照（v6 · 2026-09-26 · 撤回"固定倍数 / 下限"通则）

来源：Lead 复查 —— v4/v5 虽然撤掉了 `5–15mm`，却换上"`1–2×` 接口特征尺寸"这条**同样是规范性的固定倍数**，
而且与示例数字（3mm 板搭接 1.5mm 合格 vs "3mm 板最少 3mm"）自相矛盾。
本次**只做定点文本修订**（`SKILL.md` / `README.md`）：不重跑几何、不动插件、不动 `references/验收清单.md`、不新增章节骨架。

| # | 原来写的 | 改成 | 落在哪 |
|---|---|---|---|
| 1 | 存在"正确的嵌入判据"：`1–2×` / `0.5×板厚` / `≥1.0×轴径` / 四条不等式 / 接口类型建议表 / "5–15mm 只是特征尺寸 5–30mm 那一档" | **删除全部规范性的固定倍数、百分比与下限，以及 5–30mm 的推导解释**；改为：展示拼接可在**不改变可见轮廓、不捅穿薄壳**下选择**隐藏交叠**，交叠量由**本项目 spec 明文**规定、装配后按 spec 复核 | 坑二 ① · §0.5.3 · §0.5.4 · §0.5.5 回执行 · §6 门⑧ |
| 2 | 数字（Ø6→6mm、3mm 板→1.5mm、"1.5mm 合格"）读起来像通用判据 | 明确标注：**示例数字只是写明它的那份示例 spec 的取值，不是通则**；换尺度 / 材料 / 连接形式就重新由设计定 | 坑二 ③ · §0.5.5 回执行 · §11 #1 行 |
| 3 | 没写清可视脚本的能力边界 | 写明：**真实连接由设计配合、标准、承载工况、制造工艺与运动间隙规定**；几何脚本只答"叠没叠 / 浮没浮 / 穿没穿"，**不给结构强度结论** | 坑二 ② · §0.5.3 |
| 4 | `axis_overlap_mm` 只说"取接口法线那根轴" | 补前提：**只有接口法线与世界轴一致、且轴对齐盒对该几何适用时才作粗筛**；**旋转 / 斜置接口必须沿真实接口的局部坐标量测真实表面**，不能挑一根世界轴当深度 | 坑二 ③ · §4.0 分流表 · §6 门⑧ · §0.5.5 回执行 |

**历史口径**：§9 相关行、§11 #1（v4）、§12 尾注（v5）都已就地标注**被 §13 取代**，不留自相矛盾。

**怎么验（v6 · 纯文本一致性检查）**：
1. `grep -n "1–2×\|5–30mm\|0.5×板厚\|1\.0×轴径\|四条不等式\|接口类型表" SKILL.md README.md`
   ⇒ **只命中 §11 / §13 的纠正引用与"已取代"标注**；规范正文（坑二 / §0.5.3–0.5.5 / §6）为零；
2. 坑二的 `axis_overlap_mm` 段必须同时带"**仅当接口法线与世界轴一致且轴对齐盒适用**"这一前提与"**旋转接口沿接口局部坐标量真实表面**"；
3. `grep -n "embed_mm" SKILL.md README.md` ⇒ **只余本行自身**（§13 的历史引用）；规范正文与 `README.md` 为 0（字段名统一为 `overlap_mm`）；
4. `grep -n "隐藏交叠" SKILL.md README.md` ⇒ 坑二 / §0.5.3–0.5.5 / §6 门⑧ 齐备；未重跑几何、未动 `references/验收清单.md`、未动插件。
