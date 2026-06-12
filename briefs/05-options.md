# Module 5: 调参与排错 —— 布尔运算的调音台

### Teaching Arc
- **Metaphor:** 录音棚调音台。同一首歌（同一对形状），推子位置不同，出来的是金曲还是噪音。布尔运算暴露了一排"推子"：Fuzzy Value（容差推子）、Run Parallel(并行开关）、Non-Destructive（安全模式）、Glue（贴合模式）、Check Inverted / OBB（体检和加速开关）。会调推子的人，翻车率低一个数量级。
- **Opening hook:** 前四章你已经看懂了整条流水线。但现实是：两个看起来明明相交的零件，Fuse 出来却是空的；或者跑一次要 30 秒。先别怀疑人生——大概率只是推子没调对。
- **Key insight:** 布尔运算的鲁棒性≠魔法，它是一组可控参数：容差吃掉建模误差（fuzzy），并行换速度（parallel），安全模式保护输入（non-destructive），glue 跳过昂贵求交（当你确定形状只是贴着没穿透）。出错时按"症状→推子"查表，而不是盲目重建模型。
- **"Why should I care?":** 这章是直接可复制的生产力：你能把"调 fuzzy 到 1e-5 重试""开 glue shift""先 BRepAlgoAPI_Check 体检"这样的指令精确地说给 AI 或写进自己的代码，从"碰运气建模"升级为"系统性排错"。

### Code Snippets (pre-extracted)

File: `dox/specification/boolean_operations/boolean_operations.md`（GF 官方用法示例，C++，节选）
```cpp
BOPAlgo_Builder aBuilder;
// Setting arguments
NCollection_List<TopoDS_Shape> aLSObjects = …; // Objects
aBuilder.SetArguments(aLSObjects);

// Set parallel processing mode (default is false)
bool bRunParallel = true;
aBuilder.SetRunParallel(bRunParallel);

// Set Fuzzy value (default is Precision::Confusion())
double aFuzzyValue = 1.e-5;
aBuilder.SetFuzzyValue(aFuzzyValue);

// Set safe processing mode (default is false)
bool bSafeMode = true;
aBuilder.SetNonDestructive(bSafeMode);

// Set Gluing mode for coinciding arguments (default is off)
BOPAlgo_GlueEnum aGlue = BOPAlgo_GlueShift;
aBuilder.SetGlue(aGlue);

// Perform the operation
aBuilder.Perform();

// Check for the errors
if (aBuilder.HasErrors())
{
  return;
}

// result of the operation
const TopoDS_Shape& aResult = aBuilder.Shape();
~~~~
```
（注意：写模块时请去掉结尾的 `~~~~`，那是文档标记，不是代码。）
翻译要点：一行一个推子。SetFuzzyValue=容差，SetRunParallel=多核，SetNonDestructive=不修改输入形状（安全但稍慢），SetGlue=形状只贴不穿时跳过昂贵求交。Perform 之后**永远**先 HasErrors 再取结果。

File: `src/ModelingAlgorithms/TKBO/BOPAlgo/BOPAlgo_Options.cxx` (lines 105-108)
```cpp
void BOPAlgo_Options::SetFuzzyValue(const double theFuzz)
{
  myFuzzyValue = std::max(theFuzz, Precision::Confusion());
}
```
翻译要点：经典的"参数夹紧"：不管你传多小，容差永远不低于系统精度 Precision::Confusion()（1e-7）。读源码时看到 std::max/std::min 包住输入，多半是防御性的下限/上限保护。

File: `src/ModelingAlgorithms/TKBO/BRepAlgoAPI/BRepAlgoAPI_BooleanOperation.cxx` (lines 44-47，CSF_DEBUG_BOP 调试彩蛋)
```cpp
    OSD_Environment         env("CSF_DEBUG_BOP");
    TCollection_AsciiString pathdump = env.Value();
    myIsDump                         = (!pathdump.IsEmpty());
    myPath                           = pathdump.ToCString();
```
翻译要点：隐藏调试开关——设置环境变量 CSF_DEBUG_BOP=某个目录，每当布尔运算的参数或结果"体检不合格"，OCCT 会自动把两个输入形状 + 复现脚本 dump 到该目录，便于离线复现 bug。这是 OCCT 开发者自己排错的方式。

### 概念事实（务必准确表达）
- Fuzzy value：额外加在所有子形状容差上的"宽容度"，专治"差一点点就相交/重合"的模型（默认 Precision::Confusion()≈1e-7）。调大可以吞掉建模缝隙，但过大会吞掉真实细节。
- Glue 选项三档：Off（默认）/ GlueShift（形状可能部分重合但不穿透）/ GlueFull（完全贴合，连 Shift 都不用算）。用对了能跳过大量求交，加速明显；用错了（实际有穿透）会得到错误结果。
- Non-Destructive / Safe mode：保证输入形状完全不被修改（默认会就地更新容差等）。要保留原始模型时必开。
- 体检工具 `BRepAlgoAPI_Check`：可检查单个形状的有效性，或检查两个形状对某操作的适配性（自相交、维度不符等）。"垃圾进，垃圾出"——许多布尔翻车其实是输入本身坏了。
- 错误/警告机制：HasErrors()（致命，无结果）与 HasWarnings()（结果可用但有隐患），均可 DumpErrors/DumpWarnings 到流。
- Draw 命令对照（动手实验用）：bfuzzyvalue / brunparallel / bnondestructive / bglue / bfillds / bbuild / bbop。

### Interactive Elements
- [x] **推子面板（pattern cards）** — 5-6 张选项卡片：Fuzzy / Parallel / Non-Destructive / Glue / CheckInverted / UseOBB。每张：图标 + 一句"什么时候拨它" + 默认值。
- [x] **症状→药方查表** — 两列对照表或卡片组："Fuse 结果为空/缺块 → 调大 fuzzy 试试 + 先 Check 输入"；"跑得慢 → RunParallel + （贴合场景）Glue"；"原始模型被改了 → 开 Non-Destructive"；"结果有自相交 → 检查输入 + 看 Warnings"。
- [x] **Code↔English translation** — 第 1 段（推子总览）必做，第 2、3 段任选其一或都做。
- [x] **Quiz**（终章综合，4 题，全部场景式）—
  1. "两个零件理论上面贴面，Fuse 后出现肉眼难见的缝隙导致结果是两个独立体。先拨哪个推子？"（答：Fuzzy value 调大，如 1e-5）
  2. "你要对 200 对零件批量做 Cut，最先打开什么？"（答：RunParallel；若都是贴合关系再加 Glue）
  3. "运算后用户的原始模型容差被改了，客户投诉。怎么避免？"（答：SetNonDestructive(true)）
  4. "AI 写的布尔代码偶尔崩溃，你想给 OCCT 官方提 issue，怎么抓现场？"（答：设 CSF_DEBUG_BOP 让它自动 dump 输入和脚本）
- [x] **Callout** — "aha!"框：HasErrors/HasWarnings 双轨报告是大型几何库的通用模式——错误≠警告，忽略警告等于埋雷。
- [x] **课程收尾屏** — 一屏总结全课程旅程（四个角色 + 两段流水线 + 一排推子），并给出下一步行动建议（读 specification 文档原文、用 Draw 命令做实验、看 GTests 用例）。

### Reference Files to Read
- `references/interactive-elements.md` → "Pattern Cards"、"Code ↔ English Translation"、"Multiple-Choice Quizzes"、"Callout Boxes"
- `references/content-philosophy.md` → 全文
- `references/gotchas.md` → 全文

### Connections
- **Previous module:** "建造部分"——流程已全部讲完，本章是"驾驶舱参数"。开头承接其结尾的"推子"预告。
- **Next module:** 无（终章，含课程总结收尾屏）。
- **Tone/style notes:** 中文授课；teal accent；术语 tooltip：fuzzy value、tolerance、并行、OBB（包围盒）、自相交、Draw（OCCT 的命令行测试台）、环境变量等。模块 section id 必须是 `module-5`。
