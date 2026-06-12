# Module 4: 建造部分 —— GF 切好整个蛋糕，BOP 按订单装盘

### Teaching Arc
- **Metaphor:** 自助蛋糕台。General Fuse（GF）是后厨：把两个叠在一起的蛋糕沿所有接触面**全部切开**，得到一盘"所有碎块"（S1 独有块 + S2 独有块 + 公共块）。四种布尔操作只是四张不同的"装盘订单"：Fuse=全装走、Common=只装公共夹层、Cut=只装第一个蛋糕没碰到对方的部分、Cut21=反过来。后厨只切一次，订单随便换。
- **Opening hook:** 上一章结束时，所有边被剪成段、所有面被交线划成片——零件已经全切好、整齐躺在 DS 档案室里。现在导演（BOPAlgo_BOP）登场：他从不亲自切割，只负责"挑选与拼装"。
- **Key insight:** R(GF) = Sp1 + Sp2 + Sp12，而每种布尔结果都是这三类零件的子集。这就是为什么 `BOPAlgo_BOP` 直接继承 `BOPAlgo_Builder`（GF 算法类）——布尔操作 = GF + 一个过滤器。BuildShape/BuildRC 干的就是"按订单过滤再拼装"。
- **"Why should I care?":** 当 Fuse 结果"多了一块"或 Cut "切漏了"，你就知道：零件切分（交点部分）可能没错，错的是分类/挑选（IN/OUT/ON 状态判断）。需要复杂组合时（如一次对 10 个体做拆分），你会想起可以直接用 GF/Splitter 而不是循环做两两布尔——一句话就能让 AI 写出快 10 倍的代码。

### Code Snippets (pre-extracted)

File: `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_BOP.cxx` (lines 432-446，节选)
```cpp
  // 1. CheckData
  CheckData();
  if (HasErrors())
  {
    return;
  }
  //
  // 2. Prepare
  Prepare();
  if (HasErrors())
  {
    return;
  }
```
翻译要点：导演开工前两步：验收剧本（参数维度是否兼容等）、布置剪辑台。每步照例"出错即收工"。

File: `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_BOP.cxx` (lines 460-476，节选)
```cpp
  // 3. Fill Images
  // 3.1 Vertices
  FillImagesVertices(aPS.Next(aSteps.GetStep(PIOperation_TreatVertices)));
  if (HasErrors())
  {
    return;
  }
  //
  BuildResult(TopAbs_VERTEX);
  if (HasErrors())
  {
    return;
  }
```
翻译要点："Images（映像）"是建造部分的核心词——记录"原始形状 → 它切分后的替身们"的映射表。先从最低维的顶点开始填表、建结果，然后逐级往上（边、线框、面、壳、体）。

File: `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_BOP.cxx` (lines 583-609，BuildRC 的 Fuse 分支，节选)
```cpp
  // A. Fuse
  if (myOperation == BOPAlgo_FUSE)
  {
    NCollection_Map<TopoDS_Shape, TopTools_ShapeMapHasher> aMFence;
    aType = TypeToExplore(myDims[0]);
    TopExp_Explorer aExp(myShape, aType);
    for (; aExp.More(); aExp.Next())
    {
      const TopoDS_Shape& aS = aExp.Current();
      if (aMFence.Add(aS))
      {
        aBB.Add(aC, aS);
      }
    }
    myRC = aC;
    return;
  }
```
翻译要点：Fuse 订单最简单——"全都要"。遍历 GF 草稿结果里的所有零件，借助一个"防重名册"（aMFence，集合去重）保证每块只装一次，全部装进结果盘 aC。Common/Cut 的分支则要逐块判断"属于谁"再决定去留。

### 概念事实（务必准确表达）
- BOP 建造流程（PerformInternal1）：CheckData → Prepare → FillImagesVertices/BuildResult(顶点) → 逐维度 FillImages*/BuildResult（边→线框→面→壳→体）→ BuildShape（内部先 BuildRC 草稿再精修）→ PrepareHistory（记录"谁变成了谁"的族谱，供撤销/参数化建模用）。
- "状态分类"概念：每个切分零件相对另一组形状有 IN（在内部）/ OUT（在外部）/ ON（在边界上）三种状态。Common 要 IN 的，Cut 要 OUT 的，Fuse 要外壳。
- History（历史）：Modified/Generated/Deleted 三类记录，是 OCAF 参数化命名的基石——这也是 GF "history-based architecture" 的含义。
- Splitter（拆分器）= 用 Tools 切 Objects 但不删除任何 Objects 的零件；Section = 只收集交线交点。它们都是 GF 的不同"装盘订单"。

### Interactive Elements
- [x] **配图** — 模块开头放 `images/builder-cake.png`（自助蛋糕台：整切蛋糕 + 三个不同装盘订单的插画，16:9）。
- [x] **订单对照卡片** — 4 张卡片：Fuse/Common/Cut12/Cut21，每张写"装盘规则"（用 Sp1/Sp2/Sp12 表示）。可用三色圆点图例（S1 独有=蓝、S2 独有=橙、公共=绿）。
- [x] **Group chat animation** — 群名："剪辑机房"。actors: BOP（导演）、DS（档案室）、Builder（GF 后厨）。剧本示例：
  1. BOP: "@DS 把箱子和球的全部切块清单给我"
  2. DS: "送达：箱子独有 ×5 块，球独有 ×3 块，公共 ×1 块"
  3. BOP: "订单是 COMMON——只要公共那块"
  4. BOP: "分类完毕：IN ✅，其余退回"
  5. BOP: "拼装、缝合、登记族谱…成片输出 🎬"
- [x] **Drag-and-drop quiz**（操作→结果配对）— items: "Fuse 的结果"、"Common 的结果"、"Cut12 的结果"、"Cut21 的结果"；targets: "Sp1+Sp2+Sp12"、"Sp12"、"Sp1"、"Sp2"。
- [x] **Code↔English translation** — 第 3 段（BuildRC Fuse 分支）必做，第 2 段（Fill Images）必做。
- [x] **Quiz** — 2-3 题（与拖拽题合计仍算本模块测验）：
  1. debugging："Cut 的结果里残留了一小块本该被切掉的材料，最可能是哪个环节判断错了？"（答：零件的 IN/OUT 状态分类）
  2. 架构："为什么 OCCT 不为 Fuse/Common/Cut 写三套独立算法？"（答：共享 GF——切分一次，过滤不同）
- [x] **Callout** — "aha!"框：History/族谱让上层应用能在你改了草图后"重新找到那个孔"——参数化 CAD 的根基。

### Reference Files to Read
- `references/interactive-elements.md` → "Group Chat Animation"、"Drag-and-Drop"、"Code ↔ English Translation"、"Multiple-Choice Quizzes"、"Callout Boxes"
- `references/content-philosophy.md` → 全文
- `references/gotchas.md` → 全文

### Connections
- **Previous module:** "交点部分"——零件已切好存进 DS。本模块开头承接"零件全切好了，谁来挑选拼装"。
- **Next module:** "调参与排错"——好结果不仅靠流程对，还靠参数调得好。结尾预告："同样的代码，有人跑出完美结果，有人跑出碎渣——差别往往只是几个开关。最后一章教你拨这些开关。"
- **Tone/style notes:** 中文授课；teal accent；沿用导演/档案室/后厨称呼；术语 tooltip：Images/映像、IN/OUT/ON、History、Modified/Generated/Deleted、Splitter、Compound 等。模块 section id 必须是 `module-4`。
