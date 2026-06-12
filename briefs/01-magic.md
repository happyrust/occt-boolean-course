# Module 1: 一行代码的魔法 —— 当你调用 BRepAlgoAPI_Fuse 时发生了什么

### Teaching Arc
- **Metaphor:** 橡皮泥游戏。两团橡皮泥（红色和蓝色）放在一起：把它们捏成一团 = Fuse（并）；只留下两团重叠的部分 = Common（交）；用蓝色那团当"模具"从红色上挖走一块 = Cut（差）；只描出两团相碰的轮廓线 = Section（截面）。四种玩法，同一对橡皮泥。
- **Opening hook:** 你在 CAD 软件里点过"合并""开孔""切除"按钮吗？比如给一个零件打螺丝孔。点下去的那一瞬间，底层几何内核（很多 CAD 软件用的就是 OCCT）就在执行"布尔运算"。这门课带你看清那一瞬间内部发生的一切。
- **Key insight:** OCCT 把 Fuse / Common / Cut / Section 都实现为"同一台机器的不同出口"——它们共享同一套求交与切分流程（General Fuse），只是最后挑选的零件不同。
- **"Why should I care?":** 知道四种操作的语义和正确叫法（Fuse/Common/Cut/Cut21/Section），你就能向 AI 精确描述需求（"对这两个 solid 做 Cut，tool 是圆柱"），并且当结果不对时，第一时间判断是"选错了操作"还是"算法出了问题"。

### Code Snippets (pre-extracted)

File: `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_Operation.hxx` (lines 18-26)
```cpp
enum BOPAlgo_Operation
{
  BOPAlgo_COMMON,
  BOPAlgo_FUSE,
  BOPAlgo_CUT,
  BOPAlgo_CUT21,
  BOPAlgo_SECTION,
  BOPAlgo_UNKNOWN
};
```
英文翻译要点：这是一个"枚举"（enum，给一组固定选项起名字的清单）。COMMON=交集，FUSE=并集，CUT=用工具切对象，CUT21=反过来用对象切工具，SECTION=只要相交的线，UNKNOWN=还没选。

File: `src/ModelingAlgorithms/TKBO/BRepAlgoAPI/BRepAlgoAPI_Fuse.cxx` (lines 43-50)
```cpp
BRepAlgoAPI_Fuse::BRepAlgoAPI_Fuse(const TopoDS_Shape&          S1,
                                   const TopoDS_Shape&          S2,
                                   const Message_ProgressRange& theRange)
    : BRepAlgoAPI_BooleanOperation(S1, S2, BOPAlgo_FUSE)
{
  Build(theRange);
}
```
翻译要点：这是 Fuse 的"构造函数"——你把两个形状 S1、S2 递给它，它在出生时就登记好"我要做 FUSE"，然后立刻调用 Build() 开始干活。一行 `BRepAlgoAPI_Fuse aFuse(box, sphere);` 背后就是这段。

File: `src/ModelingAlgorithms/TKBO/BRepAlgoAPI/BRepAlgoAPI_BooleanOperation.cxx` (lines 122-140，Build 的开头)
```cpp
void BRepAlgoAPI_BooleanOperation::Build(const Message_ProgressRange& theRange)
{
  // Set Not Done status by default
  NotDone();
  // Clear from previous runs
  Clear();
  // Check for availability of arguments and tools
  // Both should be present
  if (myArguments.IsEmpty() || myTools.IsEmpty())
  {
    AddError(new BOPAlgo_AlertTooFewArguments);
    return;
  }
  // Check if the operation is set
  if (myOperation == BOPAlgo_UNKNOWN)
  {
    AddError(new BOPAlgo_AlertBOPNotSet);
    return;
  }
```
翻译要点：Build 一上来先"自我清零"（标记未完成、清掉上次的结果），再做两个体检：有没有给够形状？有没有选操作类型？任何一项不满足就记录一条错误并放弃。这是防御式编程的典型样子。

### 概念事实（务必准确表达）
- 参与运算的两组形状分别叫 **Objects（对象组）** 和 **Tools（工具组）**，每组可以有任意多个形状（不只是两个）。
- 数学表示：GF(S1,S2) = Sp1 + Sp2 + Sp12（S1 独有部分 + S2 独有部分 + 公共部分）。Fuse=三者全要，Common=只要 Sp12，Cut12=只要 Sp1，Cut21=只要 Sp2。
- 形状的统一类型叫 `TopoDS_Shape`（顶点/边/面/体的通用包装）。

### Interactive Elements
- [x] **Hero 图片** — 模块开头放 `images/hero-boolean.png`（橡皮泥/几何体四种布尔结果的插画，16:9）。用 `<img>` 全宽展示，配一句图注。
- [x] **四操作卡片** — 4 张 pattern card：FUSE / COMMON / CUT / SECTION，每张配 emoji 或 SVG 小图标 + 一句话语义 + 对应橡皮泥玩法。
- [x] **Code↔English translation** — 上面 3 段代码各做一个翻译块（或挑前 2 段，第 3 段视篇幅）。
- [x] **Data flow animation** — actors: 你的代码 → BRepAlgoAPI_Fuse → Build() → 结果 Shape()。steps: ①你 new 一个 Fuse 并传入两个形状 ②构造函数登记 FUSE ③Build 体检参数 ④返回融合后的形状。
- [x] **Quiz** — 3 题，scenario 风格：
  1. "你想在零件上打一个圆孔，应该用哪个操作？Objects 和 Tools 分别放什么？"（答：CUT；Objects=零件，Tools=圆柱）
  2. "Cut 和 Cut21 有什么区别？"（场景式：A 切 B vs B 切 A）
  3. "你只想要两个曲面的交线（不要体），用哪个？"（答：SECTION）

### Reference Files to Read
- `references/interactive-elements.md` → "Pattern Cards"、"Message Flow / Data Flow Animation"、"Code ↔ English Translation"、"Multiple-Choice Quizzes"、"Callout Boxes"
- `references/content-philosophy.md` → 全文
- `references/gotchas.md` → 全文

### Connections
- **Previous module:** 无（本模块是开篇，需要用 2-3 屏交代：什么是 OCCT、什么是布尔运算、本课程学什么）
- **Next module:** "认识演员表" —— 将介绍完成这次魔法的四个幕后角色（API 门面 / PaveFiller / BOP Builder / 数据结构 DS）。结尾埋一句："Build() 里那句 IntersectShapes 和 BuildResult 是谁在干活？下一章见。"
- **Tone/style notes:** 中文授课，口吻像聪明朋友聊天；accent 色 teal (#2A7B9B)；术语首次出现都加 glossary tooltip（OCCT、CAD、几何内核、布尔运算、枚举、构造函数、TopoDS_Shape、Objects、Tools 等）。模块 section id 必须是 `module-1`。
