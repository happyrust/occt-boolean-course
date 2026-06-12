# OCCT 交互式教程编写计划（Roadmap）

> 基于 `src/` 七大模块盘点 + `dox/user_guides` 官方蓝本 + `tests/` 素材库梳理。
> 已完成：**《深入 OCCT 布尔运算》**（模块 1-5 正课 + 番外 I 相交引擎 + 番外 II 实战判例），覆盖 `TKBO` + `TKGeomAlgo` 相交部分。
> 风格基线：中文授课、每模块独立隐喻、真实源码片段、交互元素（流程动画/模拟器/测验）、codebase-to-course 工作流（brief → module HTML → QA → 打包）。

---

## 课程矩阵总览（按优先级）

| # | 课程 | 覆盖工具包 | 官方蓝本 | 优先级 | 预估规模 |
|---|---|---|---|---|---|
| C1 | ✅ **已完成** 形状的解剖学（拓扑与几何数据结构）→ `occt-anatomy-course/` | TKBRep · TKG2d/TKG3d · TKGeomBase | dox/user_guides/modeling_data | ★★★★★ | 6 模块 |
| C2 | ✅ **已完成** 从零造一个零件（建模工作流）→ `occt-modeling-course/` | TKPrim · TKTopAlgo · TKFillet · TKOffset | dox/user_guides/modeling_algos + dox/tutorial | ★★★★★ | 7 模块 |
| C3 | ✅ **已完成** 三角化的艺术（网格化）→ `occt-mesh-course/` | TKMesh | dox/user_guides/mesh | ★★★★ | 4 模块 |
| C4 | ✅ **已完成** 形状急诊室（修复与诊断）→ `occt-healing-course/` | TKShHealing | dox/user_guides/shape_healing | ★★★★ | 5 模块 |
| C5 | ✅ **已完成** 模型的护照（数据交换 STEP/XDE）→ `occt-de-course/` | TKDESTEP · TKDEIGES · TKXCAF · TKDEGLTF | dox/user_guides/step, xde | ★★★★ | 6 模块 |
| C6 | ✅ **已完成** 把模型画到屏幕上（可视化）→ `occt-visu-course/` | TKV3d · TKService · TKOpenGl | dox/user_guides/visualization | ★★★ | 6 模块 |
| C7 | ✅ **已完成** 带 Undo 的文档（OCAF 应用框架）→ `occt-ocaf-course/` | TKCDF · TKLCAF · TKCAF · TKStd | dox/user_guides/ocaf | ★★ | 5 模块 |
| 番外 | 工程图引擎（HLR）/ Draw 深度玩法 / 内核基石（Handle 与集合） | TKHLR · Draw · TKernel/TKMath | dox/user_guides/draw_test_harness, foundation_classes | ★ | 各 2-3 模块 |

**推荐编写顺序：C1 → C2 → C3 → C4 → C5 → C6 → C7 → 番外**（依赖链与读者学习曲线一致）。

---

## C1 形状的解剖学 —— TopoDS / Geom / BRep（最优先）

**为什么第一**：布尔课全程在说「面 = 曲面 + 边界裁剪」「形状带容差」，但从未正面讲过 Shape 的解剖结构。这是读懂其余一切算法的地基；也是新人最常见的认知断层（TopoDS_Shape 为什么没有坐标？）。

- **核心内容**：
  1. 拓扑骨架：Vertex→Edge→Wire→Face→Shell→Solid→Compound 七级阶梯；共享与复用（同一条边被两个面共用）
  2. 几何血肉：Geom 曲线/曲面家族（Line/Circle/BSpline；Plane/Cylinder/BSplineSurface）
  3. 粘合层 BRep：`BRep_TDF`？否——`BRep_Tool` 如何从 Edge 取出 3D 曲线/pcurve/容差；TopLoc_Location 位置复用
  4. 方向（Orientation）：FORWARD/REVERSED 的含义，为什么镜像不用复制几何
  5. 遍历与查询：TopExp_Explorer、TopExp::MapShapesAndAncestors
  6. 动手搭骨架：BRepBuilderAPI_MakeVertex/Edge/Wire/Face 五行造一张脸
- **隐喻方向**：解剖课（骨架/血肉/韧带）；乐高的「共享积木」仓库
- **真实素材**：`src/ModelingData/TKBRep/BRep/BRep_Tool.cxx`、`TopExp_Explorer`；tests/mkface
- **交互元素提案**：七级阶梯拖拽排序（DnD 引擎已有）；「拆一个立方体」点击钻取器（Compound→…→Vertex）；orientation 翻转模拟器

## C2 从零造一个零件 —— 建模工作流（与 C1 并列最优先）

**为什么**：这是用户问得最多的「怎么用 OCCT 干活」；把分散的 API 串成一条真实工序单，承接布尔课（布尔是其中一道工序）。

- **核心内容**（以一个法兰/支架零件贯穿全课）：
  1. 草图：gp 坐标系 + BRepBuilderAPI_MakeWire 画轮廓
  2. 长料：BRepPrimAPI（Box/Cylinder/Prism 拉伸/Revol 旋转）
  3. 复杂成形：BRepOffsetAPI_MakePipe（扫掠）/ ThruSections（放样）
  4. 精加工：BRepFilletAPI_MakeFillet/MakeChamfer（圆角倒角；TKFillet）
  5. 抽壳与偏置：BRepOffsetAPI_MakeThickSolid（TKOffset）
  6. 装配工序：布尔运算回顾（呼应已完课程）
  7. 出厂检验：BRepCheck_Analyzer、GProp 体积/质心
- **隐喻方向**：车间工序单/流水线；一道菜的完整菜谱
- **真实素材**：dox/tutorial（官方瓶子教程可对照）；tests/prim、tests/fillet、tests/offset
- **交互元素提案**：工序流水线动画（已有 flow 引擎）；「选错工序会怎样」分支模拟器；每模块结尾零件进度图

## C3 三角化的艺术 —— BRepMesh

**为什么**：可视化、3D 打印、求交加速（IntPolyh）都踩在网格上；deflection 参数是用户第二常调的旋钮（第一是 fuzzy，已讲）。

- **核心内容**：BRepMesh_IncrementalMesh 入口；linear/angular deflection 的几何意义；Poly_Triangulation 存哪了（挂在 Face 上）；网格质量与性能权衡；与 STL 导出的关系
- **隐喻方向**：马赛克镶嵌画；地图等高线的取舍
- **真实素材**：`src/ModelingAlgorithms/TKMesh/BRepMesh/`；tests/mesh
- **交互元素提案**：deflection 滑杆 → 圆的折线逼近实时变化（纯 SVG/CSS 可做）；三角形数量 vs 误差的天平动画

## C4 形状急诊室 —— ShapeHealing

**为什么**：真实项目 80% 的痛苦来自导入的坏模型；与番外 II 的容差/判例文化无缝衔接。

- **核心内容**：ShapeAnalysis（确诊：自由边/小边/方向错）；ShapeFix_Shape 全家桶（修复 pipeline）；ShapeUpgrade（升级：合并小面/统一连续性）；什么时候该修、什么时候该放弃
- **隐喻方向**：急诊室分诊台（挂号→检查→手术→复查）
- **真实素材**：`src/ModelingAlgorithms/TKShHealing/ShapeFix/`；tests/heal
- **交互元素提案**：「病例卡」检索台（复用模块 7 卷宗台模式）；修复前后对比滑块

## C5 模型的护照 —— DataExchange（STEP/IGES/XDE/glTF）

**为什么**：几乎所有工程对接都从 STEP 进出；XDE（颜色/装配/属性）是最被低估的宝藏。

- **核心内容**：STEPControl_Reader/Writer 最小流程；转换精度与单位；XCAF 文档结构（装配树/颜色/图层/名字）；glTF/OBJ/STL 网格类导出（TKRWMesh）；常见失败模式（丢颜色/丢装配层级）
- **隐喻方向**：海关与翻译官（护照=中性格式，签证=单位/精度协商）
- **真实素材**：`src/DataExchange/TKDESTEP/`；tests/de、tests/de_wrapper
- **交互元素提案**：格式选择器（需求→推荐格式）；装配树点击钻取器

## C6 把模型画到屏幕上 —— Visualization

- **核心内容**：V3d_Viewer/V3d_View/AIS_InteractiveContext 三件套；AIS_Shape 展示一个形状的最小代码；选择与高亮机制（SelectMgr）；显示模式（线框/着色）；与网格课的关系（显示=三角化结果）
- **隐喻方向**：剧场（舞台 Viewer/演员 AIS/导演 Context/灯光）
- **真实素材**：`src/Visualization/TKV3d/AIS/`；samples
- **交互元素提案**：剧场角色 flow 动画；「为什么我的形状不显示」排错决策树

## C7 带 Undo 的文档 —— OCAF

- **核心内容**：TDocStd_Document；Label 树与 Attribute（数据挂在标签上）；事务与 Undo/Redo；Bin/Xml 持久化；什么样的应用值得上 OCAF
- **隐喻方向**：公证处档案馆（每页纸有编号，盖章才生效，可翻旧档）
- **真实素材**：`src/ApplicationFramework/`；tests/caf
- **交互元素提案**：Label 树点击展开器；事务提交/回滚模拟器

## 番外候选（按需）

1. ✅ **已完成** 工程图引擎 HLR（TKHLR）→ `occt-hlr-course/`：3D→2D 投影消隐，3 模块
2. ✅ **已完成** Draw 深度玩法 → `occt-draw-course/`：自定义命令、批量回归，2 模块（判例篇已铺垫）
3. ✅ **已完成** 内核基石 → `occt-kernel-course/`：Handle 引用计数、NCollection 容器、异常体系（TKernel/TKMath），3 模块——适合想读源码的进阶读者

> 🏁 **全系列收官（2026-06-12）**：七门正课 + 布尔课 + 三个番外共 **11 门课、48 个模块**全部完成，门户页 `occt-courses-portal/` 一页索引。本 roadmap 使命达成，归档留念。

---

## 工程化约定（沿用已验证流程）

1. 每门课独立目录 `occt-<topic>-course/`，复用 codebase-to-course 骨架（_base/styles/main.js/build.sh）
2. 流程：盘点源码 → 逐模块 brief（briefs/NN-xx.md）→ 模块 HTML → 结构化 QA（标签/id/JSON/锚点脚本）→ agent-browser 实测交互 → 打包 zip
3. 红线：代码片段保持原样不删改；每模块 ≥1 个交互元素；quiz 考应用不考记忆；id 全文档唯一（`flow-actor-<课>-<模块>-*`）
4. 素材三板斧：`dox/user_guides`（概念蓝本）+ 源码注释（权威细节）+ `tests/`（真实案例，呼应判例篇方法论）

## 建议的下一步

🎓 **C1–C7 七门正课全部完课**（2026-06-12），**番外 I（HLR）亦完课**，并新增统一门户页 `occt-courses-portal/index.html`（与各课程目录同级部署即可，相对链接串联九门课）。剩余可选加餐：番外 II《Draw 深度玩法》（蓝本 dox/user_guides/draw_test_harness，2 模块）与番外 III《内核基石》（Handle/NCollection/异常，蓝本 dox/user_guides/foundation_classes，3 模块）。

> C2 实施备注：贯穿零件按官方瓶子教程执行（dox/tutorial 原码直引），模块顺序对齐瓶子真实工序（草图→长料→圆角→焊颈→抽壳→螺纹→检验），未采用 roadmap 草案中的法兰/支架例（理由：官方代码权威、七道工序天然覆盖全部目标 API）。
