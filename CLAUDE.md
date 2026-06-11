# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# CareerAdvice — 高考理工科专业推荐测试

## 项目简介

面向高考生的 MBTI 风格单页测试网站。从 51 道题库随机抽 17 题 + 固定 Q18 命运转盘，基于四维度画像+欧氏距离匹配推荐 3 个理工科专业大类。

纯前端单文件（HTML+CSS+JS全部内嵌），部署到 GitHub Pages / Vercel。

## 文件结构

```
D:\考研\CareerAdvice\
├── index.html              # 主页面（单文件，HTML+CSS+JS 全部内嵌）
├── hero-bg.jpg             # Hero 全屏背景图（攀登者雪山，1200px，216KB）
├── 实施计划.md               # 原始需求文档
├── cxadvice.txt            # 宣传文案/广告语
├── test-slide1~3.png       # Playwright 自动截图（三个推荐专业各一张）
├── poster-promo.html       # 宣传海报（待创建）
└── .claude/
    └── settings.local.json # 本地权限配置
```

## 技术栈

- 纯 HTML+CSS+JS，零依赖构建工具
- CDN 引入：html2canvas 1.4.1 (海报生成)、qrcodejs 1.0.0 (二维码)、Google Fonts (Zhi Mang Xing 毛笔字体 + Noto Serif SC 衬线体)
- 单页滚动布局，`max-width: 640px` 容器
- 雪山曙光色系：`#f0f3f7` 背景、`#3a6b8c` 山湖蓝主色、`#d4794a` 曙光橙强调色、`#7a9a6e` 辅助绿

## 常用操作

- **本地预览**：直接在浏览器打开 `index.html`（零构建，纯静态文件）
- **清空重测**：刷新页面即可，所有状态仅存于 JS 内存
- **海报保存**：做完测试 → 滑到对应推荐卡片 → 点击「📸 分享此推荐」→ 自动下载 PNG（依赖 html2canvas + qrcodejs CDN）
- **截图测试**：`python -c "..."` inline Playwright 脚本（视口 414×896，模拟 iPhone 11 Pro）

## 核心架构

### 四维度计分系统

每道题的每个选项通过 `effects` 数组影响 4 个维度的分数：

```
effects: [[dimIndex, value], ...]
// dimIndex: 0=抽象(-)/具象(+), 1=物性(-)/生命(+), 2=系统(-)/细节(+), 3=创新(-)/规范(+)
// value: 负值=偏向左极, 正值=偏向右极（范围约 ±3）
```

- **维度0**：抽象理论 (负) ↔ 具象实操 (正)
- **维度1**：物理机械 (负) ↔ 生命人体 (正)
- **维度2**：系统全局 (负) ↔ 细节钻研 (正)
- **维度3**：创新探索 (负) ↔ 规范成熟 (正)

每题可同时影响多个维度（如 `[[0, -2], [1, 1]]` 同时降低维度0、升高维度1）。
未答题不影响分数；修改答案会**完全重算**分数（`recalculateScores()` 从零累加所有已答题）。

### 专业大类画像

12 个 `MAJORS` 条目，每个有 4 维 `profile` 数组（与用户分数同坐标系），值域约 `[-4, 4]`：

```js
{ id: 'cs', name: '计算机类', profile: [-3.5, -3, -2, -1.5], ... }
// 含义：偏抽象、偏物理机械、偏系统、偏创新
```

### 推荐引擎

1. 计算用户分数向量与每个专业 profile 向量的**欧氏距离**
2. 按距离升序取前 3
3. Q18 结果干预：
   - **A（顺其自然）**：保持原序
   - **B（逆风翻盘）**：70% 概率交换 #2 和 #3
   - **C（出其不意）**：从 #4/#5 随机选一个替换 #3

### 题库结构 (QUESTION_BANK)

51 道题，每题格式：

```js
{ id: 1, text: '题目文字', options: [
  { text: '选项A', effects: [[dimIndex, value], ...] },
  // effects 为空数组 [] 表示该选项不影响任何维度
]}
```

抽题逻辑：Fisher-Yates 洗牌后取前 17 道，Q18（命运转盘）永远固定在最后。

### 应用状态 (state)

```js
state = {
  questions: [],       // 随机选出的 17 题
  answers: {},         // { questionId: optionIndex }
  q18Result: null,     // 'A' | 'B' | 'C' | null
  scores: [0,0,0,0],   // 当前四维分数
  recommendations: [], // computeRecommendations() 填充
  totalQuestions: 18
}
```

### 海报生成

通过隐藏的 `#poster-capture` div 构建海报 DOM，用 html2canvas 截图后触发下载。`generatePoster(index)` 接受专业索引（0/1/2），生成单专业海报。QR 码使用 `window.location.href` 动态读取当前 URL，部署到任何域名均自动适配。

### 页面流程

Hero（全屏雪山背景 + 毛笔苍裂标语「不因少了一座雪峰而影响你的志愿填报」）→ (点击「开始探索」) → 17题+Q18答题区 + 进度条 + 提交按钮 → 结果页（四维条 + Emoji滑窗 ×3 + 独立分享按钮 + 选科提醒）

### Emoji 角色系统

12 个专业各有 `MAJOR_EMOJIS` 映射：`[主角色, 道具1, 道具2, 道具3, 道具4]`。结果页以横向矩形横幅展示：左侧大 emoji（52px）+ 右侧 4 个道具小 emoji 一字排开（26px）。

### 自动化测试

用 Playwright + Chromium 进行手机端截图测试（414×896 视口），对三个推荐结果滑窗各截一张图验证 UI。

```bash
# 首次安装
pip install playwright
python -m playwright install chromium

# 运行截图测试（脚本为临时 Python inline，无独立 .py 文件）
# 测试端口：直接用 file:// 协议打开 index.html，无本地 server 依赖
```

截图输出：`test-slide1.png`、`test-slide2.png`、`test-slide3.png`（对应三个推荐专业滑窗）。

## 上次进度（2026-06-10）

- ✅ Hero页全屏雪山背景 + 毛笔苍裂标语（Zhi Mang Xing 字体 + SVG 滤镜）
- ✅ **雪山曙光色系重构**（山湖蓝 #3a6b8c / 曙光橙 #d4794a / 石板绿 #7a9a6e）
- ✅ Hero 隐藏实现细节（去除「51题库随机抽」「约2分钟」等 meta 标签）
- ✅ 答题交互、进度条
- ✅ 51题题库 + 随机抽17题
- ✅ 四维度计分引擎 + 欧氏距离匹配
- ✅ Q18 命运转盘（分阶段动画 + 扇区标签 + 脉冲聚焦）
- ✅ **结果页 Emoji 横向矩形横幅**（触屏滑动 + 箭头导航 + 圆点指示器）
- ✅ **单专业独立分享海报**（generatePoster 支持 index 参数）
- ✅ CSS 响应式（max-width: 480px 断点）
- ✅ Playwright 自动化截图测试（完整流程通过）
- ⬜ 宣传海报独立页面（poster-promo.html）
- ⬜ 部署上线
- ⬜ 真机测试

## 下一步

1. 真机测试 → 修触摸/布局问题
2. 多组答案验证推荐合理性
3. 创建 poster-promo.html 宣传海报页
4. 部署到 Vercel 或 GitHub Pages
