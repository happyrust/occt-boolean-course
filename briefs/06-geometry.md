# Module 6（番外篇）: 相交计算的几何引擎 —— 三条求交路线与夜路描缝

### Teaching Arc
- **Metaphor:** 测绘队。模块 3 说 FF 站「算交线、最贵」，本章打开那台机器的外壳：里面是一支测绘队，接到「这两张面交在哪」的订单后走三条路线之一——**查公式**（两张面都是平面/圆柱/球/锥这类「正规几何」，交线有现成数学公式，像查代数表，结果直接是直线/圆/椭圆）；**半查半量**（一边正规一边自由）；**夜路描缝**（两张全是自由曲面：没有公式，只能实地测量——找到一个起点，打着手电沿两面交缝一步一步走，每步钉一颗图钉，最后把图钉连成光滑曲线画到图纸上）。
- **Opening hook:** 模块 3 的流水线告诉你「FF 站会算出交线」，但它没告诉你：交线到底是怎么「算」出来的？一张圆柱面和一张自由曲面相交，世界上根本没有现成公式。打开机器，看看里面。
- **Key insight:** OCCT 把曲面分成两类：有公式的「正规几何」（ts=1：平面/圆柱/球/锥）和只能采样的「自由曲面」（ts=0：B 样条等）。两正规→解析解（快且准）；有自由→行走法（walking）：从起点出发沿交缝逐步推进得到折线（WLine），再**拟合**成 B 样条曲线，同时如实记录拟合误差（这就是结果容差比输入大的来源）。所有昂贵的辅助仪器（投影器、分类器）统一放在 IntTools_Context「器材间」里按需缓存。
- **"Why should I care?":** ① 性能：同一零件用正规几何建模 vs 全转 NURBS，布尔速度可差一个量级——因为前者查公式、后者走夜路；② 排错：结果交线边的容差变大不是 bug，是测量+拟合的诚实误差报告；模块 5 的 Fuzzy 调的就是这台机器的判定松紧。

### Code Snippets (pre-extracted，保持原样)

File: `src/ModelingAlgorithms/TKGeomAlgo/IntPatch/IntPatch_Intersection.cxx` (lines 1263-1278)
```cpp
  // Surface type definition
  int ts1 = 0;
  switch (typs1)
  {
    case GeomAbs_Plane:
    case GeomAbs_Cylinder:
    case GeomAbs_Sphere:
    case GeomAbs_Cone:
      ts1 = 1;
      break;
    case GeomAbs_Torus:
      ts1 = bGeomGeom;
      break;
    default:
      break;
  }
```
翻译要点：给曲面「验明正身」。平面/圆柱/球/锥 = 正规几何（ts=1，有公式）；圆环面看情况（轴线对齐才有公式）；其余（B 样条等自由曲面）= 0。两张面各验一次，组合出三条路线：1+1=Geom-Geom 查公式，1+0=Geom-Param 半查半量，0+0=Param-Param 夜路描缝。源码 1296-1300 行的注释原文就是这三种 "Possible intersection types"。

File: `src/ModelingAlgorithms/TKGeomAlgo/IntWalk/IntWalk_PWalking.hxx` (lines 31-44)
```cpp
//! This class implements an algorithm to determine the
//! intersection between 2 parametrized surfaces, marching from
//! a starting point. The intersection line
//! starts and ends on the natural surface's boundaries.
class IntWalk_PWalking
{
public:
  DEFINE_STANDARD_ALLOC

  //! Constructor used to set the data to compute intersection
  //! lines between Caro1 and Caro2.
  //! Deflection is the maximum deflection admitted between two
  //! consecutive points on the resulting polyline.
  //! TolTangency is the tolerance to find a tangent point.
```
翻译要点：官方注释直说了「marching from a starting point」（从起点行进）——就是夜路描缝。结果是 polyline（折线/图钉串）；Deflection = 相邻两颗图钉间允许的最大偏差（步子迈多大）；TolTangency = 判定「切着碰」的容差。

File: `src/ModelingAlgorithms/TKBO/IntTools/IntTools_FaceFace.cxx` (lines 1394-1407)
```cpp
            occ::handle<Geom_Curve> aBSp = GeomInt_IntSS::MakeBSpline(WL, ifprm, ilprm);
            //
            if (myApprox1)
            {
              H1 = GeomInt_IntSS::MakeBSpline2d(WL, ifprm, ilprm, true);
            }
            //
            if (myApprox2)
            {
              H2 = GeomInt_IntSS::MakeBSpline2d(WL, ifprm, ilprm, false);
            }
            //
            IntTools_Curve aIC(aBSp, H1, H2);
            mySeqOfCurve.Append(aIC);
```
翻译要点：WL 就是那串图钉（Walking Line 折线）。MakeBSpline 把图钉连成光滑 3D 曲线；MakeBSpline2d 在两张面各自的「参数地图」上再画一遍（pcurve，2D 影子）；最后 3D 曲线+两张 2D 影子打包成一条 IntTools_Curve 交线入库。

File: `src/ModelingAlgorithms/TKBO/IntTools/IntTools_Context.hxx` (lines 46-64)
```cpp
//! The intersection Context contains geometrical
//! and topological toolkit (classifiers, projectors, etc).
//! The intersection Context is for caching the tools
//! to increase the performance.
class IntTools_Context : public Standard_Transient
{
public:
  Standard_EXPORT IntTools_Context();
  Standard_EXPORT ~IntTools_Context() override;

  Standard_EXPORT IntTools_Context(const occ::handle<NCollection_BaseAllocator>& theAllocator);

  //! Returns a reference to point classifier
  //! for given face
  Standard_EXPORT IntTools_FClass2d& FClass2d(const TopoDS_Face& aF);

  //! Returns a reference to point projector
  //! for given face
  Standard_EXPORT GeomAPI_ProjectPointOnSurf& ProjPS(const TopoDS_Face& aF);
```
翻译要点：器材间。官方注释明说「for caching the tools to increase the performance」。每张面的投影仪（ProjPS）、点分类器（FClass2d）造一次就留在架子上，下次直接拿——整场布尔运算共用一间器材间（PaveFiller 调度全程传递同一个 Context）。

### 概念事实（务必准确表达）
- 分诊在 IntPatch_Intersection：正规几何（Plane/Cylinder/Sphere/Cone，ts=1）走解析路线（Geom-Geom，IntAna 解析解）；Torus 仅在轴线对齐等特殊情形算正规（bGeomGeom）；其余为参数曲面（ts=0）。
- 三路线：Geom-Geom（双正规，查公式，结果常是直线/圆/椭圆等精确曲线）、Geom-Param（一正规一自由）、Param-Param（双自由，行走法）。
- 行走法（IntWalk_PWalking）：从起点 march 沿交线推进，产出 polyline（WLine，离散点串）；Deflection 控制相邻点最大偏差；交线起止于曲面自然边界。
- 折线 → GeomInt_IntSS::MakeBSpline 拟合成 3D B 样条；MakeBSpline2d 生成两面上的 pcurve；打包成 IntTools_Curve。
- 拟合不是精确解：IntTools_FaceFace::ComputeTolReached3d 计算「3D 曲线与两面 2D 曲线间最大偏差」作为交线的实际容差——结果容差比输入大的根源。
- IntTools_Context 按 face/edge 缓存分类器/投影器/包围盒等，整个布尔流程共享一个 Context。
- Draw 测试台可用 `bopcurves f1 f2 -2d` 直接观看两张面的交线（亲手按快门）。

### Interactive Elements
- [x] **配图** — 模块中部放 `images/seam-survey.png`（夜路描缝隐喻：两张交叠的纸面，交缝处一排图钉，光滑青色丝线穿过图钉，奶油纸面+青色点缀风格，16:9），配图注："图钉=行走法的采样点，丝线=拟合出的 B 样条交线"。
- [x] **分诊模拟器**（本模块核心交互）— 纯 HTML+内联 JS：两排按钮各选「曲面 A / 曲面 B」的类型（平面、圆柱、球、自由曲面），下方即时显示走哪条路线（查公式 / 半查半量 / 夜路描缝）+ 成本标尺（快/中/贵）。函数名用 m6 前缀避免冲突。
- [x] **Data flow animation**（必备）— FF 一张订单的全流程。actors: 接单(FaceFace) → 分诊(IntPatch) → 查公式 / 走夜路 → 图钉折线 WLine → 拟合 B 样条 → 误差报告·交付。packet 动画用唯一 id（flow-actor-m6-*）。
- [x] **Code↔English translation** — 4 段全做（分诊 switch、PWalking 注释、MakeBSpline、Context）。
- [x] **Quiz** — 3 题（应用型）：
  1. tracing："两张汽车引擎盖那样的自由曲面求交，测绘队走哪条路线？"（答：夜路描缝 Param-Param 行走法）
  2. 性能："同一个法兰零件，用平面+圆柱建模 vs 整体转成 NURBS 再做布尔，哪个快？为什么？"（答：前者，正规几何查公式，免行走免拟合）
  3. debugging："布尔结果中交线边的容差比两张输入面都大，最可能的来源？"（答：行走+拟合的逼近误差被 ComputeTolReached3d 如实记录，不是 bug）
- [x] **Callout** — "aha!"框：交线不是「算出来的精确曲线」，而是「测量+拟合的产物」——所以容差会长大，这是诚实，不是出错。info 框：器材间 Context；预告/回扣：模块 5 的 Fuzzy 调的就是这台机器的判定松紧。

### Reference Files to Read
- `references/interactive-elements.md` → "Data Flow Animation"、"Code ↔ English Translation"、"Multiple-Choice Quizzes"、"Callout Boxes"
- `references/content-philosophy.md` → 全文
- `references/gotchas.md` → 全文

### Connections
- **Previous module:** 模块 5 已收尾全课程（「课程完结」彩蛋在模块 5 末尾）。本章定位为**番外篇/进阶加餐**：开篇明说「正课已毕，这是给好奇者的拆机报告」，回扣模块 3 的 FF 站与模块 5 的容差/Fuzzy。
- **Next module:** 无（番外即终章）。结尾给「亲手按快门」指引：Draw 的 `bopcurves` 命令。
- **Tone/style notes:** 中文授课；teal accent；沿用摄影组/档案室称呼（PaveFiller=摄影组调度，本章主角是它外包的「测绘队」）。背景色用 `var(--color-bg)`（与模块 5 的 warm 交替）。术语 tooltip：解析解、B 样条/NURBS、自由曲面、折线/WLine、拟合/approximation、Deflection、pcurve、分类器、投影器等。模块 section id 必须是 `module-6`；flow actor id 用 `flow-actor-m6-*`；quiz 容器 id `quiz-module6`；内联 JS 函数 `m6Pick`。
