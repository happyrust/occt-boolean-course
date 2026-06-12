# Module 3: 交点部分 —— PaveFiller 的安检流水线与珍珠项链

### Teaching Arc
- **Metaphor:** 珍珠项链。一条边（edge）是线绳；求交时发现的交点是穿在绳上的珍珠（**Pave**，记录"哪颗珍珠 + 在绳上的位置参数 t"）；相邻两颗珍珠之间的那段绳叫 **Pave Block**（切分后的边段）。整个求交过程，就是不断往各条"线绳"上穿珍珠，最后所有绳都被珍珠分成小段——建造部分要用的就是这些小段。
- **Opening hook:** 上一章说摄影组（PaveFiller）"跑遍现场找交点"。但两个形状之间有 6 个面、24 条边、16 个顶点……到底按什么顺序检查谁碰谁？乱查会重复劳动，OCCT 的答案是一条严格的"从小到大"流水线。
- **Key insight:** 求交按维度从低到高排队：顶点/顶点 → 顶点/边 → 边/边 → 顶点/面 → 边/面 → 面/面。先处理小家伙，是因为低维交点常常能"顺便解决"高维的相交（两条边在某点相交，这个点也是两面交线上的点），避免重复计算。所有结果（新顶点、交线、切分段）统统写入 DS。
- **"Why should I care?":** 布尔运算 90% 的"翻车"都发生在这一段——容差（tolerance）问题、丢交线、自相交。理解 Pave/PaveBlock 和检查顺序后，你能看懂 OCCT 的报错与 Check 结果，并向 AI 精确转述："Edge/Face 求交丢了一个公共部分，可能要调 fuzzy value。"

### Code Snippets (pre-extracted)

File: `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_PaveFiller.cxx` (lines 253-288，节选，保持原样)
```cpp
  // 00
  PerformVV(aPS.Next(aSteps.GetStep(PIOperation_PerformVV)));
  if (HasErrors())
  {
    return;
  }
  // 01
  PerformVE(aPS.Next(aSteps.GetStep(PIOperation_PerformVE)));
  if (HasErrors())
  {
    return;
  }
  //
  UpdatePaveBlocksWithSDVertices();
  // 11
  PerformEE(aPS.Next(aSteps.GetStep(PIOperation_PerformEE)));
  if (HasErrors())
  {
    return;
  }
  UpdatePaveBlocksWithSDVertices();
  // 02
  PerformVF(aPS.Next(aSteps.GetStep(PIOperation_PerformVF)));
  if (HasErrors())
  {
    return;
  }
  UpdatePaveBlocksWithSDVertices();
  // 12
  PerformEF(aPS.Next(aSteps.GetStep(PIOperation_PerformEF)));
```
翻译要点：注释里的数字是"维度暗号"：0=顶点、1=边、2=面。00=顶点/顶点，01=顶点/边，11=边/边，02=顶点/面，12=边/面。每步之后都检查 HasErrors（出错立刻收工），并把"撞在一起的同位顶点"（SD = Same Domain）同步进珍珠记录。

File: `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_PaveFiller.cxx` (lines 311-330，节选)
```cpp
  // 22
  PerformFF(aPS.Next(aSteps.GetStep(PIOperation_PerformFF)));
  if (HasErrors())
  {
    return;
  }
  //
  UpdateBlocksWithSharedVertices();
  //
  myDS->RefineFaceInfoIn();
  //
  MakeSplitEdges(aPS.Next(aSteps.GetStep(PIOperation_MakeSplitEdges)));
```
翻译要点：压轴大戏 22 = 面/面求交（算出交线，最贵的一步），然后 MakeSplitEdges 把所有被珍珠标记过的边真正切成小段。

File: `src/ModelingAlgorithms/TKBO/BOPDS/BOPDS_Pave.hxx` (lines 25-44，节选)
```cpp
//! The class BOPDS_Pave is to store
//! information about vertex on an edge
class BOPDS_Pave
{
public:
  DEFINE_STANDARD_ALLOC

  //! Empty constructor
  BOPDS_Pave();

  //! Constructor with index and parameter
  BOPDS_Pave(const int theIndex, const double theParameter);

  //! Modifier
  //! Sets the index of vertex <theIndex>
  void SetIndex(const int theIndex);
```
翻译要点：一颗"珍珠"只存两样东西：顶点的编号（DS 档案室里的索引）+ 它在曲线上的位置参数 t。极简，但足以把任何边切分成段。

### 概念事实（务必准确表达）
- 干涉（interference）有 BRep 型 6 种（V/V、V/E、V/F、E/E、E/F、F/F）+ 非 BRep 型 4 种（V/Solid、E/Solid、F/Solid、Solid/Solid，即"完全在体内"）。
- 完整检查顺序：VV→VE→EE→VF→EF→FF→V/Solid→E/Solid→F/Solid→Solid/Solid（从低维到高维）。
- 形状都带容差（tolerance）：两个东西的距离 ≤ 容差之和就算"相交"。V/V 干涉会产生一个新顶点（包住两个旧顶点容差球的小球心）。
- Common Block：E/E 或 E/F 重叠时，几何上重合的多个 PaveBlock 绑成一个"共块"。
- 每条有限边天生至少有一个 PaveBlock（两端点就是首尾两颗珍珠）。

### Interactive Elements
- [x] **配图** — 模块中部放 `images/pave-necklace.png`（珍珠项链与几何边切分的对照插画，16:9），配图注："Pave=珍珠，PaveBlock=两珠之间的绳段"。
- [x] **Data flow animation**（本课程必备元素，务必做好）— 流水线动画。actors: VV → VE → EE → VF → EF → FF → MakeSplitEdges。steps 描述每站干什么（"合并撞在一起的顶点"→"把顶点投影到边上穿珍珠"→…→"面面求交算出交线"→"沿珍珠剪断所有边"）。
- [x] **珍珠项链示意** — 用纯 HTML/CSS 画一条水平"边"：线段上 4 个圆点（珍珠），珍珠之间分成 3 段不同颜色的 PaveBlock，hover 高亮每段。可以用 flex + border 实现，不需要 canvas。
- [x] **Code↔English translation** — 第 1 段（流水线）必做；第 3 段（Pave 类）必做；第 2 段视篇幅。
- [x] **Quiz** — 3 题：
  1. tracing："两个立方体棱对棱搭在一起，最先在流水线的哪一站被发现？"（答：EE，边/边）
  2. debugging："结果里两个明明重合的角点变成了两个独立顶点，最该检查什么？"（答：容差/fuzzy value——V/V 干涉没触发）
  3. 概念应用："为什么先做 VV 再做 FF，而不是反过来？"（答：低维结果可复用，避免高维重复计算）
- [x] **Callout** — "aha!"框：容差思想——几何内核里没有"精确相等"，只有"在容差内相等"。浮点世界的生存法则。

### Reference Files to Read
- `references/interactive-elements.md` → "Message Flow / Data Flow Animation"、"Code ↔ English Translation"、"Multiple-Choice Quizzes"、"Callout Boxes"
- `references/content-philosophy.md` → 全文
- `references/gotchas.md` → 全文

### Connections
- **Previous module:** "认识演员表"——已建立摄影组(PaveFiller)/档案室(DS)的角色设定，本模块直接沿用这些称呼。
- **Next module:** "建造部分"——交点找完了，导演（BOP）如何把切好的零件挑选拼装成最终结果。结尾预告："现在所有边都被剪成了段、所有面都被交线划成了片——谁来决定哪些零件进成片？"
- **Tone/style notes:** 中文授课；teal accent；沿用摄影组/档案室称呼；术语 tooltip：tolerance/容差、interference/干涉、Pave、PaveBlock、Common Block、SD 顶点、参数 t 等。模块 section id 必须是 `module-3`。
