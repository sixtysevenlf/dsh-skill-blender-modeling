# 三视图标定（multiview calibration）

> 配套 `SKILL.md` **§0.6.1 三视图标定**。用来回答三个很具体的问题：
> 1. **这张参考图能不能按"一个 px_per_mm"换算成毫米？**
> 2. 三张视图**能不能联立**（合并成一个比例）？
> 3. 跨深度的尺寸，误差到底有多大？
>
> 纯标准库（不需要 numpy / Blender），脚本自带 `--selftest`。

## 0. ★ 先纠正一个常见误解：正交 ⇏ 三视共享同一 px_per_mm

**正交只保证"每个视图内部"像素与毫米呈线性**（该视的 `s` 恒定、视内残差≈0）。
它**不推出**三视共享同一个 `px_per_mm` —— 三张图可以各自有不同的**出图比例 / 分辨率 / 裁剪**，
却都是严格正交。

**实测（本机 Blender 5.2.2，`world_to_camera_view` 直接量，无渲染）**：

| 视图 | ortho_scale | 1000mm 标尺的像素长 | px_per_mm | 视差探针（改深度 2m 引起的 x 位移） |
|---|---|---|---|---|
| front | 4.0 m | 480 px | 0.48 | **0.000 px** |
| side | 8.0 m | 240 px | 0.24 | **0.000 px** |
| top | 2.0 m | 960 px | 0.96 | **0.000 px** |

三个视差探针全为 **0 px** ⇒ 三视**都是严格正交**；而 `px_per_mm` 互差 **300%**。
⇒ **"三视必须共享同一 px_per_mm"是错的**。共享比例是一条**额外假设**（"三视出自同一张工程图 /
同一次渲染、同分辨率、同裁剪"），必须**单独声明并验证**，不能从正交性推出来。

**推论（直接决定怎么判）**：跨视互检要**换算成毫米再比**，不能直接比 `s`。
直接比 `s` 会把"三张图缩放不同"误判成"透视"，于是去改本来没问题的取点。

## 1. 什么时候用

| 场合 | 用不用 |
|---|---|
| 有正视图 / 侧视图 / 俯视图，**且能确认出自同一张图或同一次渲染** | ✅ 联立标定，可合并联合 `px_per_mm` |
| 三视各来自不同来源（截屏 / 单独照片 / 缩放过的图） | ⚠ **逐视各自标定**，用共享尺寸互检；**不要**合并 |
| 只有一张图，或三张不是同一正交方向 | ⚠ 退回逐视标定（自有标尺 + 同深度同方向换算） |
| 明显是透视照片 / 广角 | 先 `img_rectify` 校正（见 §6，注意它**只校正取点那个平面**），再按上面的口径处理 |

## 2. 怎么取点（每视 ≥3 个）

1. 选**已知真实长度**的特征：轴距 / 轮径 / 总长 / 炮管长 / 窗宽……（来源写进 `spec.py` 或项目资料）。
2. 在图上量出该特征**沿本视图像横轴方向**的像素长度。要点：
   - 特征方向必须**平行于图像横轴**（斜着的先做投影折算，否则量的是斜边）；
   - 同一视的 2–3 个特征跨度尽量拉开（跨度越大，比例越稳）；
   - ★ **每视至少 3 点** —— 两点解出来的直线必然过这两点，**残差恒为 0**，判不出透视。
3. **再给至少一条"共享尺寸"**：一条在**两个及以上视图里都能量到**的真实尺寸（全高 / 总长都行）。
   跨视互检全靠它。**没有共享尺寸，三视之间就无法互相证明** —— 结论只能是 `unresolved`。
4. 填成 JSON（示例见下），跑脚本。

```json
{
  "views": {
    "front": {"image_width": 1920, "features": [
      {"name": "总长", "u_mm": 9830, "px": 1474.5},
      {"name": "轴距", "u_mm": 4600, "px": 690.0},
      {"name": "轮径", "u_mm":  660, "px":  99.0}
    ]},
    "side":  {"image_width": 1920, "features": [ /* 同上，用侧视上的已知尺寸 */ ]},
    "top":   {"image_width": 1920, "features": [ /* 同上 */ ]}
  },
  "shared": [
    {"name": "全高", "measure": {"front": {"px": 812.0}, "side": {"px": 406.0}}}
  ],
  "same_scale_asserted": false
}
```

`u_mm` = 该特征在**本视图像横轴方向**的已知物理长度（mm）；`px` = 同一特征在图上的像素长度。
`same_scale_asserted: true` 只在你**确实知道**三视同尺度时才写（如同一张工程图扫描件）。

## 3. 脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""三视图标定（front / side / top）：逐视最小二乘解 px_per_mm + 跨视一致性互检。

用法
    python3 multiview_calib.py calib.json      # 读标定输入，出报告
    python3 multiview_calib.py --selftest      # 合成数据自检（见文末四例）

口径（★ 这一版修掉的核心误解）
  u_mm = 该特征在**本视图图像横轴方向**上的已知物理长度（端点之差，mm）
  px   = 同一特征在参考图上的像素长度
  模型 px = cx + s * u（s = px_per_mm，cx = u=0 落在的像素列）
  ★ 两点必然被直线穿过（残差恒 0）；要判"这张图是不是正交的"，每视至少给 3 点。
  ★ 正交只保证**各视内部** px 与 mm 呈线性（视内残差≈0）。它**不**保证三视共享
    同一个 px_per_mm —— 三张图可以各自有出图比例 / 分辨率 / 裁剪，却都是严格正交。
    因此跨视互检必须换算成**毫米**再比（见 shared_dimensions），不是直接比 s。
"""
import json
import sys

RESID_TOL_PCT = 2.0   # 视内残差上限：>2% -> 该视内部非线性（有透视，或特征点量错）
SHARED_TOL_PCT = 3.0  # 同一物理尺寸由两视各自换算出的毫米值互差上限：>3% -> 两视互不相容
SAME_SCALE_TOL_PCT = 1.0  # 三视 px_per_mm 互差 ≤1% 才可视为"同一出图尺度"


def fit_view(features, image_width=None):
    """最小二乘解 px = cx + s*u。features=[{name,u_mm,px}, ...]（>=2 点）。"""
    pts = [(float(f["u_mm"]), float(f["px"])) for f in features]
    n = len(pts)
    if n < 2:
        raise ValueError("至少 2 个特征点才能定比例（给了 %d 个）" % n)
    mx = sum(p[0] for p in pts) / n
    my = sum(p[1] for p in pts) / n
    sxx = sum((u - mx) ** 2 for u, _ in pts)
    if sxx <= 0:
        raise ValueError("所有特征点在 u 上重合（sxx=0），定不出比例")
    sxy = sum((u - mx) * (p - my) for u, p in pts)
    s = sxy / sxx
    if s <= 0:
        raise ValueError("解出的比例 s<=0（像素与毫米反号？检查 u_mm/px 是否对得上）")
    cx = my - s * mx
    resid = [abs(p - (cx + s * u)) for u, p in pts]
    span = max(u for u, _ in pts) - min(u for u, _ in pts)
    resid_px = max(resid)
    resid_pct = (resid_px / (s * span) * 100.0) if span > 0 else 0.0
    return {"n": n, "s": s, "cx": cx, "sxx": sxx, "span_mm": span,
            "resid_px": resid_px, "resid_pct": resid_pct,
            "pp_offset_px": (None if image_width is None else cx - float(image_width) / 2.0),
            "per_point_px": [round(r, 3) for r in resid]}


def same_scale(fits):
    """三视 px_per_mm 是否可视为同一出图尺度（这是**额外假设**，不是正交的推论）。"""
    ss = [f["s"] for f in fits.values()]
    if not ss or min(ss) <= 0:
        return None, None
    spread = (max(ss) - min(ss)) / min(ss) * 100.0
    return (spread <= SAME_SCALE_TOL_PCT), spread


def shared_check(fits, shared):
    """跨视互检：同一物理尺寸由各视**各自**的 s 换算成毫米，再看是否互相吻合。

    shared = [{"name": "全高", "measure": {"front": {"px": 812.0}, "side": {"px": 150.0}}}, ...]
    """
    rows, bad = [], []
    for item in (shared or []):
        name = item.get("name") or "?"
        vals = {}
        for vname, m in (item.get("measure") or {}).items():
            if vname not in fits:
                continue
            vals[vname] = float(m["px"]) / fits[vname]["s"]
        if len(vals) < 2:
            rows.append({"name": name, "mm": vals, "disagree_pct": None,
                         "note": "只在 1 个视里给了量测 -> 无法互检"})
            continue
        lo, hi = min(vals.values()), max(vals.values())
        d = (hi - lo) / lo * 100.0 if lo > 0 else float("inf")
        rows.append({"name": name, "mm": {k: round(v, 2) for k, v in vals.items()},
                     "disagree_pct": round(d, 3)})
        if d > SHARED_TOL_PCT:
            bad.append("共享尺寸「%s」两视换算互差 %.2f%% > %.1f%%（%s）"
                       % (name, d, SHARED_TOL_PCT,
                          "; ".join("%s=%.1fmm" % (k, v) for k, v in vals.items())))
    return rows, bad


def combine(fits):
    """仅当三视确认同尺度时才合并：按 sxx 反方差加权。"""
    wsum = sum(f["sxx"] for f in fits.values())
    if wsum <= 0:
        raise ValueError("合并权重为 0")
    return sum(f["s"] * f["sxx"] for f in fits.values()) / wsum


def report(data):
    views = data.get("views") or {}
    if not views:
        raise ValueError("输入里没有 views")
    fits, bad = {}, []
    for name, v in views.items():
        f = fit_view(v.get("features") or [], v.get("image_width"))
        fits[name] = f
        if f["resid_pct"] > RESID_TOL_PCT:
            bad.append("%s 视内残差 %.2f%% > %.1f%%（该视图内部非线性：不是正交投影，"
                       "或特征点量错了）" % (name, f["resid_pct"], RESID_TOL_PCT))

    rows, shared_bad = shared_check(fits, data.get("shared"))
    bad += shared_bad

    is_same, spread = same_scale(fits)
    asserted = data.get("same_scale_asserted")

    L = ["=== 逐视最小二乘（px = cx + s*u）===",
         "view     n  px_per_mm   mm_per_px  主点偏移px   残差px   残差%    跨度mm"]
    for name in sorted(fits):
        f = fits[name]
        pp = "-" if f["pp_offset_px"] is None else ("%+.1f" % f["pp_offset_px"])
        L.append("%-8s %d  %9.6f  %9.4f  %10s  %7.3f  %6.3f%%  %8.1f"
                 % (name, f["n"], f["s"], 1.0 / f["s"], pp, f["resid_px"], f["resid_pct"], f["span_mm"]))
    L.append("")
    L.append("=== 三视 px_per_mm 一致性（注意：这是**额外假设**，不是正交的推论）===")
    L.append("  各视 px_per_mm: " + ", ".join("%s=%.6f" % (k, v["s"]) for k, v in sorted(fits.items())))
    L.append("  互差 = %s" % ("-" if spread is None else "%.2f%%" % spread))
    if is_same:
        L.append("  -> 三视 px_per_mm 互差 ≤ %.1f%%：**可以**视为同一出图尺度（如同一张工程图 / "
                 "同一次正交渲染）" % SAME_SCALE_TOL_PCT)
    else:
        L.append("  -> 三视 px_per_mm 互差 > %.1f%%：**不共享同一比例**。这**未必**是透视 ——"
                 % SAME_SCALE_TOL_PCT)
        L.append("     三张图可以各自有出图比例 / 分辨率 / 裁剪，却都严格正交（实测：三视视差全为 0px，"
                 "px_per_mm 却是 0.48 / 0.24 / 0.96，差 300%）。")
        L.append("     处置：**逐视各自标定、各自换算**，再用下面的共享尺寸互检 —— 不要用一个联合比例硬套三视。")
    L.append("")
    if rows:
        L.append("=== 跨视互检（换算成毫米再比，与出图比例无关）===")
        for r in rows:
            mm = r["mm"]
            L.append("  %-10s %s   互差 %s"
                     % (r["name"], ", ".join("%s=%.1fmm" % (k, v) for k, v in sorted(mm.items())),
                        "-" if r["disagree_pct"] is None else "%.2f%%" % r["disagree_pct"]))
    else:
        L.append("=== 跨视互检：输入里没给 `shared` ===")
        L.append("  没有共享尺寸就只能做视内自检 —— 三视之间**无法**互相证明。")
        L.append("  至少给一条在两个视图里都能量到的真实尺寸（如全高 / 总长），否则结论只能是 unresolved。")
    L.append("")

    if bad:
        L.append("=== 结论：不通过（不要硬套联立）===")
        for b in bad:
            L.append("  x " + b)
        L.append("  退回逐视标定：每视只用**自己的**标尺定 s，只换算与本视标尺同深度、同方向的尺寸。")
        L.append("  跨深度尺寸：误差**不是**固定 ±5%，而是随深度线性放大 —— 用 §0.6.1 的 "
                 "err ≈ Δz/(D+Δz) 先估，估不出来就记 unresolved。")
    elif not asserted and not is_same:
        L.append("=== 结论：三视各自正交且共享尺寸一致，但没有同尺度依据 ===")
        L.append("  可以逐视换算；**不要**合并成一个联合 px_per_mm。")
        L.append("  若你确认三视出自同一张图 / 同一次渲染，请在输入里加 \"same_scale_asserted\": true。")
    else:
        s = combine(fits)
        L.append("=== 结论：正交性通过（三视可联立）===")
        L.append("  联合 px_per_mm = %.6f（1 px = %.4f mm），按各视 sxx 反方差加权" % (s, 1.0 / s))
        L.append("  依据：三视 px_per_mm 互差 ≤%.1f%%（同尺度）**且** 共享尺寸互差 ≤%.1f%%。"
                 % (SAME_SCALE_TOL_PCT, SHARED_TOL_PCT))
        L.append("  用途：把各视像素量测统一换算成毫米写入 spec.py，并注明来源（图名 + 像素 + 标尺）。")
    return "\n".join(L)


# ------------------------------------------------------------------ selftest
def _synth(base_s=0.15, cx=960.0, width=1920, us=(0.0, 4600.0, 9830.0),
           view_k=None, in_view_drift=0.0, drift_view="front",
           shared_mm=2000.0, shared_px_override=None):
    """合成标定输入。

    view_k      = {视名: 该视的**整体出图比例系数**}
    shared      = 一条在两个视图里都能量到的真实尺寸（默认 2000mm）
    shared_px_override = {视名: 覆盖该视里这条尺寸的像素值}（用来造"量错 / 不一致"）
    """
    view_k = view_k or {}
    out = {"views": {}, "shared": []}
    for name in ("front", "side", "top"):
        k = float(view_k.get(name, 1.0))
        feats = []
        for i, u in enumerate(us):
            drift = 1.0 + (in_view_drift * (u / float(us[-1])) if name == drift_view else 0.0)
            feats.append({"name": "f%d" % i, "u_mm": u, "px": round(cx + base_s * k * drift * u)})
        out["views"][name] = {"image_width": width, "features": feats}
    # 共享尺寸在 front / side 两视各量一次（各视按自己的出图比例画出来）
    meas = {}
    for name in ("front", "side"):
        k = float(view_k.get(name, 1.0))
        px = base_s * k * shared_mm
        if shared_px_override and name in shared_px_override:
            px = shared_px_override[name]
        meas[name] = {"px": round(px, 3)}
    out["shared"] = [{"name": "共享尺寸", "measure": meas}]
    return out


def _selftest():
    cases = [
        ("A 三视同尺度（应通过并联立）",
         _synth(view_k={"side": 1.004, "top": 0.997}), "联立"),

        ("B 视内透视（front 比例随 u 漂移 15% -> 视内残差红）",
         _synth(in_view_drift=0.15), "视内残差"),

        ("C 三视出图比例不同但都是正交（side 0.5× 且它的标尺同步缩小）"
         " -> 应判「不共享比例，但逐视标定可用」，**不再**误判透视",
         _synth(view_k={"side": 0.5, "top": 0.75}), "逐视"),

        ("D 出图比例不同 + 共享尺寸互差 6%（真的对不上）-> 跨视互检红",
         _synth(view_k={"side": 0.5, "top": 0.75},
                shared_px_override={"side": 0.5 * 0.15 * 2000.0 * 1.06}), "互差"),
    ]
    got = {}
    for title, data, key in cases:
        r = report(data)
        print("######## " + title + " ########")
        print(r)
        print()
        got[title[0]] = (r, key)
    ok_a = "联合 px_per_mm" in got["A"][0]
    ok_b = ("视内残差" in got["B"][0]) and ("不通过" in got["B"][0])
    # C：三种可接受措辞之一，且必须**不**判"不通过"
    ok_c = (("逐视各自标定" in got["C"][0]) or ("逐视换算" in got["C"][0])) \
        and ("不通过" not in got["C"][0])
    ok_d = ("互差" in got["D"][0]) and ("不通过" in got["D"][0])
    good = ok_a and ok_b and ok_c and ok_d
    print("SELFTEST %s  (A 联立 / B 视内残差红 / C 分尺度逐视可用 / D 跨视互检红)  "
          "A=%s B=%s C=%s D=%s" % ("PASS" if good else "FAIL", ok_a, ok_b, ok_c, ok_d))
    return 0 if good else 1


def main(argv):
    if "--selftest" in argv:
        return _selftest()
    if len(argv) < 2:
        print(__doc__)
        return 2
    with open(argv[1], "r", encoding="utf-8") as fh:
        data = json.load(fh)
    print(report(data))
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

跑法：

```bash
python3 multiview_calib.py calib.json     # 出报告
python3 multiview_calib.py --selftest     # 合成数据自检：A 联立 / B 视内残差红 / C 分尺度逐视可用 / D 跨视互检红
```

## 4. 判据（硬）

| 判据 | 阈值 | 含义 | 处置 |
|---|---|---|---|
| 视内残差 | **> 2%** | 该视**内部**比例不一致 ⇒ 真的有透视 / 非线性，或特征点量错了 | 该视不可按比例量 |
| 共享尺寸互差（换算成 mm 后） | **> 3%** | 两视对**同一条真实尺寸**给出的毫米值对不上 ⇒ 两视互不相容（其中一视非正交 / 标尺取错 / 尺寸量错） | 不要联立；逐视换算并标 `unresolved` |
| 三视 px_per_mm 互差 | **> 1%** | 三视**不共享同一出图尺度** | **不等于透视** —— 逐视各自标定即可；只有当你另有"同尺度"依据时才合并 |
| 主点偏移 `cx − 图宽/2` | 参考 | 仅作诊断：偏得离谱 ⇒ 取点或裁剪有问题（不等于非正交） | 复查取点 |

⚠ **`s` 互差超限 ≠ 判透视**。这是本文件最重要的一条口径修正：
老写法把"三视 px_per_mm 不一致"直接读成"非正交/透视"，于是去怀疑取点、去重标定，
**而真正的原因是三张图缩放不同**。正确的分工是：
**视内残差判"这张图正不正交"，共享尺寸判"两视能不能互相印证"，`s` 互差只判"能不能合并成一个比例"。**

全绿**且** `same_scale_asserted`（或 `s` 互差 ≤1%）⇒ 取按 `sxx` 反方差加权的**联合 `px_per_mm`**，
三视统一换算成毫米写进 `spec.py`，附来源（图名 + 像素坐标 + 标尺）。

## 5. ★ 跨深度的误差**不是**固定 ±5%

老写法把"跨深度尺寸"的公差一律放宽到 **±5%**。这是没保证的 —— 透视下比例随深度按 `1/z` 变化：

```text
err ≈ Δz / (D + Δz)            # D = 相机到近处标尺的距离，Δz = 两点之间的深度差
```

**实测（本机 Blender 5.2.2，`world_to_camera_view` 直接量，D = 8 m）**：

| 深度差 Δz | 实测 px_per_mm | 相对近处比例误差 | 解析式 Δz/(D+Δz) |
|---|---|---|---|
| 0 m | 0.295139 | 0.00% | 0.00% |
| 0.25 m | 0.286195 | **−3.03%** | −3.03% |
| 0.5 m | 0.277778 | **−5.88%** | −5.88% |
| 1.0 m | 0.262346 | **−11.11%** | −11.11% |
| 2.0 m | 0.236111 | **−20.00%** | −20.00% |
| 3.0 m | 0.214646 | **−27.27%** | −27.27% |

⇒ 8 m 机位下**只跨 0.5 m 深度就已经超出 ±5%**；跨 3 m 时误差 **−27.3%**。
解析式与实测逐行吻合到小数点后两位，可以直接当公式用。

**纪律**：

1. 跨深度量测**先算** `Δz/(D+Δz)`，把它当这次量测的误差上界（不是 ±5%）；
2. 估不出 `D` 或 `Δz` ⇒ 该尺寸记 **`unresolved`**，不要顺手写个"±5% 应该够"；
3. 想要真按比例量 ⇒ 让被测特征与标尺**同深度**，或对参考图做**真正的相机标定**（内参 + 外参）；
4. 单应校正（`img_rectify`）只解决**一个平面**，见 §6。

## 6. ★ 单应（`img_rectify`）只校正"取点那个平面"

四点单应对**恰好落在取点那个物理平面**上的点严格成立；离开这个平面的点**误差随深度迅速放大**。

**实测（本机 Blender 5.2.2；在 z=0 平面上取 4 角点拟合单应，再用它去映射同一物体的其他点）**：

| 点离开 z=0 平面的距离 | 最大误差 | 折算到该机位（1 m ≈ 190 px） |
|---|---|---|
| 0 m | **0.000 px** | 0 mm |
| 0.05 m | 9.906 px | **52 mm** |
| 0.20 m | 40.194 px | **211 mm** |
| 0.50 m | 103.464 px | **543 mm** |

**纪律**：

1. **平面参考**（图纸翻拍、标定板、贴平的贴纸、平铺在地面的标定框）⇒ 单应完全正确，放心用；
2. **立体物**（车 / 人 / 建筑）⇒ 单应只保证**那个平面**（比如车侧面整体外框所在的平面）是对的；
   车身离这个平面越远的特征（轮眉外扩、翼子板弧度、A 柱前倾）误差越大；
3. 所以四点必须是**同一平面上的真实矩形**，并且**量测只取靠近那个平面的特征**；
4. 要整个立体物都准 ⇒ 走真正的相机标定（内参 + 外参）或多视几何，**不是**单应；
5. 强斜视会把取点误差放大（§6.3 的 543 mm 就是这么来的）—— 先用 `img_scan` 看清轮廓再取点。

## 7. 自证：怎么知道标定是对的

- **有真值时**：自己用正交相机渲的图，`px_per_mm = 渲染宽度px / ortho_scale_mm`
  （`bpy.context.scene.camera.data.ortho_scale`）—— 拿它对拍脚本给出的值，目标误差 **≤1%**。
  ★ 注意这条只在**同一个正交设置**下成立；换 `ortho_scale` 或换分辨率，`px_per_mm` 就变（§0 实测表）。
- **没真值时**：留一个已知尺寸**不参与**拟合，用它做盲测；盲测偏差 >2% 就说明这张图不能按比例量。
- **跨视自证**：给一条共享尺寸（§2 第 3 条），看两视换算出的毫米值互差是否 ≤3%。

## 8. 怎么验（一句话）

`--selftest` 四个合成用例分别给出 **A 联立 / B 视内残差红 / C 分尺度逐视可用 / D 跨视互检红**
（末行 `SELFTEST PASS`）；真图上：正交三视若同尺度则联合比例与渲染真值误差 ≤1%，
若三视缩放不同则脚本报"不共享比例、逐视标定可用"而**不**误判透视。
