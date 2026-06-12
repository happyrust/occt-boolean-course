# 深入 OCCT 布尔运算 · 交互式教程

系统拆解 OpenCASCADE (OCCT) 布尔运算 (`TKBO`) 的中文交互式课程：从魔法表象、参与角色、相交引擎、结果构建，到选项调优、几何细节与实战判例，共 7 个模块。

纯静态站点，打开 `index.html` 即可浏览。

## 本地构建

课程正文按模块拆分在 `modules/*.html`，由 `build.sh` 拼装：

```bash
bash build.sh   # 生成 index.html
```

## 目录结构

- `index.html` — 完整课程（已构建，可直接部署）
- `modules/` — 各模块正文片段
- `briefs/` — 各模块编写大纲
- `images/` — 课程配图
- `styles.css` / `main.js` — 样式与交互逻辑
- `COURSE-ROADMAP.md` — 系列课程编写计划
