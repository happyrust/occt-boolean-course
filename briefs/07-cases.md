# Module 7（番外篇 II）: 实战判例篇 —— 卷宗室里的相交事故

### Teaching Arc
- **Metaphor:** 卷宗室（判例库）。OCCT 二十多年攒下的每一桩相交事故都没有被扔掉，而是写成「判例」永久归档在 `tests/` 目录：出过的 bug → 最小复现脚本 → 每晚 CI 复审 → 同样的事故永不二犯。读判例是理解相交算法最快的路——每个判例都是一份真实案发现场的重建记录。
- **Opening hook:** 模块 6 拆完了机器。但机器在真实世界里翻过哪些车？圆锥碰球丢过交点、圆柱碰平面多算过交线……这些事故现在都躺在卷宗室里，编号在案，每晚被重审一遍。
- **Key insight:** 判例的核心是「案发现场重建」（测试数据构造），共三种手法：①**现场搭建**——用 Draw 命令程序化造形（`pcone`/`psphere`/`box`），零外部依赖；②**物证封存**——复杂形状存成 `.brep` 文件，用 `restore [locate_data_file xxx.brep]` 调取（数据放在 `CSF_TestDataPath` 指向的外部数据库）；③**搭建+摆位**——调取物证后再 `trotate`/`explode` 摆出案发姿态。验收靠仪器：`bopcurves` 报告交线数与实测容差，`xdistcs` 沿曲线抽样测它到两张曲面的距离。
- **"Why should I care?":** 你修过的 bug 也应该入档：会写判例 = 会构造最小复现 = 向 AI/同事精确描述问题的能力。读懂 `bopcurves` 的输出（NbCurv、Tolerance Reached、points found），就能独立验收任何一次相交计算。

### Code Snippets (pre-extracted，保持原样)

File: `tests/lowalgos/intss/bug21494_1` (节选)
```tcl
pcone bc 15 0 10
psphere bs 10
explode bc f
explode bs f

set log [bopcurves bc_1 bs_1 -2d]

regexp {Tolerance Reached=+([-0-9.+eE]+)\n+([-0-9.+eE]+)} $log full Toler NbCurv

if { ![regexp {1 point\(s\) found} $log full] } {
  puts "Error: Cone apex and Pole of sphere are excluded from the intersection result"
}

if {$NbCurv != 1} {
  puts "Error: Please check NbCurves for intersector"
}

if { $Toler > 2.0e-7} {
  puts "Error: Big tolerance value"
}
```
翻译要点：案由 OCC21494「锥球相交失败」。现场搭建：造一个底半径 15、高 10 的圆锥 + 半径 10 的球，explode 拆出面。按快门（bopcurves），用正则从测量报告里抠出实测容差和交线数。三条验收标准：①锥顶/球极那个「单独的交点」不能丢（交点也是结果！）；②交线必须恰好 1 条；③实测容差不得超过 2e-7。

File: `src/ModelingAlgorithms/TKBO/GTests/IntTools_FaceFace_Test.cxx` (lines 187-214，节选)
```cpp
  // Half cylinder with axis along Y, positioned far from plane center
  // The cylinder is at X=20, which is outside the plane's circular boundary (radius 10)
  const gp_Pnt aCylCenter(20.0, 0.0, 0.0);
  const gp_Dir aCylAxis(0.0, 1.0, 0.0);
  const double aCylRadius = 2.0;
```
以及断言部分：
```cpp
  EXPECT_EQ(aNbCurves, 0) << "Expected no intersection curves, but found " << aNbCurves
                          << ". The cylinder at X=20 is outside the circular plane (radius 10).";
```
翻译要点：C++ 判例（GTest）。无限平面和圆柱面在数学上必然相交，但这张「面」只是平面上半径 10 的圆形区域，而圆柱站在 X=20——圆外。正确答案是 0 条交线。教学点：**曲面相交 ≠ 面相交**，面 = 曲面 + 边界裁剪；曾经有 bug 在这里翻车（裁剪验证不严，多报了交线）。

File: `tests/lowalgos/intss/bug29807_i1003` (节选)
```tcl
restore [locate_data_file bug29807-obj.brep] b1
restore [locate_data_file bug29807-tool.brep] b2

trotate b2 +23.85857157145715500000 +12.00000000000000000000 +5.50000000000000000000 7 -7.14142842854285 0 -5
```
翻译要点：物证封存 + 摆位。形状太复杂没法现场搭，就从外部证物库调取 .brep 文件（locate_data_file 沿环境变量 CSF_TestDataPath 搜索），再用 trotate 旋转到精确的案发角度——注意那串 20 位小数，案发姿态差一点都复现不出来。

File: `tests/lowalgos/intss/begin` (节选)
```tcl
proc CheckIntersectionResult {theSurf1 theSurf2 theListOfCurves theNbPoints theTolerS1 theTolerS2} {
  upvar #0 $theSurf1 s1
  upvar #0 $theSurf2 s2

  foreach a $theListOfCurves {
    puts "** Check of $a **"
    upvar #0 $a aCurve
    bounds aCurve U1 U2

    if {[dval U2-U1] < 1.0e-9} {
      puts "Error: Wrong range of $a"
    }

    xdistcs aCurve s1 U1 U2 $theNbPoints $theTolerS1
    xdistcs aCurve s2 U1 U2 $theNbPoints $theTolerS2
  }
}
```
翻译要点：本组判例共用的「游标卡尺」。对每条交线：检查参数范围没退化成一个点，然后 xdistcs 沿曲线抽 N 个样本点、实测每个点到两张曲面的距离是否都在容差内——交线必须同时贴着两张面，差一张都不算交线。

### 概念事实（务必准确表达）
- 判例三级编目：`tests/<group>/<grid>/<case>`，如 `tests/lowalgos/intss/bug21494_1`（几何求交判例）、`tests/boolean/bopfuse_simple/A1`（布尔判例）。每组有 begin/end 公共脚本、grids.list 卷宗目录、parse.rules 判读规则。
- 现场搭建命令：box/pcone/psphere/pcylinder 等 Draw 命令；explode 按维度拆形状（f=面，e=边）。
- 物证库：`locate_data_file` 沿 `CSF_TestDataPath` 环境变量列出的目录搜索数据文件；大体量 .brep 物证存放在仓库之外。
- bopcurves 是 IntTools_FaceFace 的 Draw 出口：输出含 "Tolerance Reached="（实测容差）、曲线数、"N point(s) found"（孤立交点数）。
- 相交结果 = 交线（curves）+ 交点（points）两种；锥顶碰球极这类「点接触」产出的是交点，不是曲线。
- GTest 判例（C++）：用 gp_Pln/Geom_CylindricalSurface + BRepBuilderAPI_MakeFace 程序化造面，直接调 IntTools_FaceFace::Perform，EXPECT_EQ 断言曲线/点个数。
- 面 = 曲面 + 边界（wire 裁剪）。无限曲面相交但有界面可以不相交——边界验证是真实出过 bug 的环节。
- CI 每晚跑全部判例 = 复审；任何修改若让旧判例翻案，立即被发现。

### Interactive Elements
- [x] **配图** — 无（沿用模块 2/5 的无图先例，重点在真实代码案例）。
- [x] **卷宗检索台**（本模块核心交互）— 纯 HTML+内联 JS：三个「卷宗抽屉」按钮（案例一 锥球点交、案例二 圆外圆柱、案例三 外部物证），点击在下方展示该案的「案情卡」：案由 / 现场重建手法 / 验收标准 / 判决要点。函数名 m7Open。
- [x] **Data flow animation**（必备）— 一桩 bug 的生命周期：用户报案 → 最小复现 → 入档 tests/ → 每晚 CI 复审 → 翻案即报警 → 永不二犯。actor id 用 flow-actor-m7-*。
- [x] **Code↔English translation** — 4 段全做（bug21494_1 脚本、GTest 断言、bug29807 物证+摆位、begin 游标卡尺）。
- [x] **Quiz** — 3 题（应用型）：
  1. 应用："你修好了一个面面求交 bug，怎么保证它五年后不复发？"（答：写成最小复现判例入档 tests/，CI 每晚复审）
  2. tracing："bopcurves 输出 Tolerance Reached=3e-5、2 curve(s) found，判例断言该查哪些？"（答：曲线数和实测容差两项都要断言，只查一项不够）
  3. debugging："无限平面与圆柱数学上必然相交，但 IntTools_FaceFace 返回 0 条曲线，什么情况下这是正确答案？"（答：面的边界裁剪——圆柱落在面的有效区域之外）
- [x] **Callout** — "aha!"框：曲面相交 ≠ 面相交（边界裁剪）；info 框：判例文化——修 bug 的最后一步是写判例；收尾框：自己动手写一个判例的最小模板。
- [x] 开场用 step-cards 展示判例三级编目结构。

### Reference Files to Read
- `references/interactive-elements.md` → "Data Flow Animation"、"Code ↔ English Translation"、"Multiple-Choice Quizzes"、"Callout Boxes"
- `references/content-philosophy.md` → 全文
- `references/gotchas.md` → 全文

### Connections
- **Previous module:** 番外篇 I（模块 6）拆解了几何引擎与 bopcurves 快门；本章展示这台机器的真实事故档案，bopcurves 输出的解读直接派上用场。
- **Next module:** 无。结尾给「写一个自己的判例」行动模板（box+pcone+bopcurves 五行脚本）。
- **Tone/style notes:** 中文授课；teal accent；沿用测绘队/快门称呼。背景色 `var(--color-bg-warm)`（与模块 6 的 bg 交替）。术语 tooltip：回归测试、CI、最小复现、.brep、Draw、explode、正则表达式、GTest、wire/边界、孤立交点等。模块 section id `module-7`；flow actor `flow-actor-m7-*`；quiz 容器 `quiz-module7`；内联 JS 函数 `m7Open`。
