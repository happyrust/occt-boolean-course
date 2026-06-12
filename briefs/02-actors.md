# Module 2: 认识演员表 —— 布尔运算的四个幕后角色

### Teaching Arc
- **Metaphor:** 电影摄制组。`BRepAlgoAPI` 是制片人（对外签约、接收需求、交付成片）；`BOPAlgo_PaveFiller` 是勘景摄影组（跑遍现场，找出所有"交叉点"）；`BOPAlgo_BOP/Builder` 是导演兼剪辑（按剧本把素材剪成成片）；`BOPDS_DS` 是场记板+档案室（所有镜头、标记、中间结果都记录在册，谁要都来查）。
- **Opening hook:** 上一章你看到 `Build()` 一行代码返回了结果。但 Build 自己其实"什么几何活都不干"——它是个项目经理，真正的活儿被分包给了四个部门。
- **Key insight:** OCCT 布尔运算 = 两段流水线 + 一个公共档案室：交点部分（Intersection Part，由 PaveFiller 负责）→ 建造部分（Building Part，由 BOP/Builder 负责），两者通过数据结构 DS（BOPDS）交换一切信息。这个"分层 + 共享数据结构"的架构让 GF/BOP/Section/Splitter 全家共用一套底层。
- **"Why should I care?":** 出问题时你能立刻定位部门：交不出交点（结果缺边、缺交线）→ 查 PaveFiller/IntTools；交点对但结果挑错了零件（多了/少了块）→ 查 BOP 的 Building 逻辑。向 AI 提问时直接点名"PaveFiller 阶段"或"BuildShape 阶段"，回答质量天差地别。

### Code Snippets (pre-extracted)

File: `src/ModelingAlgorithms/TKBO/BRepAlgoAPI/BRepAlgoAPI_BooleanOperation.cxx` (lines 174-211，节选)
```cpp
  // If necessary perform intersection of the argument shapes
  if (myIsIntersectionNeeded)
  {
    // Combine Objects and Tools into a single list for intersection
    NCollection_List<TopoDS_Shape> aLArgs = myArguments;
    for (NCollection_List<TopoDS_Shape>::Iterator it(myTools); it.More(); it.Next())
    {
      aLArgs.Append(it.Value());
    }

    // Perform intersection
    IntersectShapes(aLArgs, aPS.Next(70));
```
翻译要点：制片人把 Objects 和 Tools 合并成一张总名单，先送去做"求交"（IntersectShapes 内部就是雇佣 PaveFiller）。注意进度条权重：求交占 70%——它是整个布尔运算中最贵的环节。

File: `src/ModelingAlgorithms/TKBO/BRepAlgoAPI/BRepAlgoAPI_BooleanOperation.cxx` (lines 196-208)
```cpp
  // Builder Initialization
  if (myOperation == BOPAlgo_SECTION)
  {
    myBuilder = new BOPAlgo_Section(myAllocator);
    myBuilder->SetArguments(myDSFiller->Arguments());
  }
  else
  {
    myBuilder = new BOPAlgo_BOP(myAllocator);
    myBuilder->SetArguments(myArguments);
    ((BOPAlgo_BOP*)myBuilder)->SetTools(myTools);
    ((BOPAlgo_BOP*)myBuilder)->SetOperation(myOperation);
  }
```
翻译要点：根据操作类型挑选"导演"：要截面线就请 `BOPAlgo_Section`，其余（Fuse/Common/Cut）都请 `BOPAlgo_BOP`，并把对象组、工具组、操作类型交给他。

### 概念事实（务必准确表达）
- TKBO 工具包里的核心包：`BRepAlgoAPI`（用户 API 层）、`BOPAlgo`（算法层：PaveFiller、Builder、BOP、Section、Splitter）、`BOPDS`（数据结构层：DS、Pave、PaveBlock、FaceInfo…）、`IntTools`（底层求交工具）、`BOPTools`（辅助工具）。
- 数据结构 DS 存储：参数形状、所有子形状（每个有唯一索引编号）、干涉（interference）记录、Pave/PaveBlock、交点交线等。
- 类继承关系：`BRepAlgoAPI_Fuse/Common/Cut/Section` 都继承 `BRepAlgoAPI_BooleanOperation` → `BRepAlgoAPI_BuilderAlgo`。算法层 `BOPAlgo_BOP` 继承 `BOPAlgo_Builder`（GF 是基类，BOP 是子类——印证"BOA 是 GFA 的特例"）。
- GFA 是基础算法：Boolean（BOA）、Splitter（SPA）、Section（SA）都基于它构建；它们共享交点部分，区别只在建造部分。

### Interactive Elements
- [x] **架构分层图** — 用卡片+箭头做 4 层静态架构图（带 hover 效果）：你的代码 → BRepAlgoAPI（制片人）→ BOPAlgo（摄影组+导演）→ BOPDS/IntTools（档案室+器材库）。每层一句职责描述。
- [x] **Group chat animation**（本课程必备元素，务必做好）— 群名："布尔运算摄制组"。actors: 用户、API（制片人）、PaveFiller（摄影组）、DS（场记）、BOP（导演）。剧本：
  1. 用户: "给这个箱子和球做个 Fuse！"
  2. API: "收到。@PaveFiller 先去把所有交点找出来"
  3. PaveFiller: "跑遍了 6 个面和 12 条边，交点、交线全记到 DS 了"
  4. DS: "已归档：新顶点 ×8，交线 ×1，切分边 ×24 ✅"
  5. API: "@BOP 你来，按 FUSE 剧本剪辑"
  6. BOP: "查 DS 档案…挑出外壳零件…拼接完成 🎬"
  7. API: "用户你好，成片在此 → Shape()"
- [x] **Code↔English translation** — 上面两段代码。
- [x] **Quiz** — 3 题：
  1. debugging 场景："Fuse 结果缺了一条本该有的交线棱边，你应该先怀疑哪个部门？"（答：交点部分/PaveFiller）
  2. 架构题："为什么 Section、Splitter、BOP 可以共享同一个 PaveFiller？"（答：交点部分对所有操作都一样，差别只在建造部分）
  3. 场景题："你想自己写一个新布尔类操作（比如只保留壳），应该继承/复用哪一层？"（答：基于 BOPAlgo_Builder/GF 扩展）
- [x] **Callout** — "aha!"框：分层架构 + 共享数据结构是大型 C++ 系统的常见手法；改任何一层都不影响其他层的合同。

### Reference Files to Read
- `references/interactive-elements.md` → "Group Chat Animation"、"Architecture Diagram"、"Code ↔ English Translation"、"Multiple-Choice Quizzes"、"Callout Boxes"
- `references/content-philosophy.md` → 全文
- `references/gotchas.md` → 全文

### Connections
- **Previous module:** "一行代码的魔法"——已讲四种操作语义和 Build() 的体检逻辑。本模块开头回应它结尾的悬念（IntersectShapes / BuildResult 是谁干的）。
- **Next module:** "交点部分"——深入 PaveFiller 的流水线（VV→VE→EE→VF→EF→FF）和珍珠项链数据结构。结尾预告："摄影组到底怎么『跑遍现场』？它有一条严格的安检流水线。"
- **Tone/style notes:** 中文授课；teal accent；演员命名全课程统一：API=制片人、PaveFiller=摄影组、BOP=导演、DS=场记/档案室。模块 section id 必须是 `module-2`。
