# Texas Method 力量训练个人计划与记录软件 — 需求规格说明书

> 版本 1.0 | 2026-06-29

---

## 1. 项目概述

### 1.1 产品定义

面向个人力量训练者的德州训练法（Texas Method）计划安排与训练记录 Web 应用。自动生成三周期训练计划，用户按日执行并记录完成情况，全程本地存储，零服务器依赖。

### 1.2 目标用户

- 采用或准备采用德州训练法的中级力量训练者
- 对杠铃片配算有需求，希望训练时有可视化辅助
- 偏好简单、不需要云账号体系的工具

### 1.3 核心价值

- **计划自动化**：填一次模板 → 每周自动生成三大项+推举的目标重量
- **训练零录入负担**：计划数据自动填充，用户只记录完成差异
- **配片可视化**：输入目标重量 → 显示杠铃两侧装配方案 + 图形预览

---

## 2. 技术方案

### 2.1 技术栈

| 层 | 选型 | 理由 |
|---|------|------|
| UI 框架 | Alpine.js（CDN 引入） | 极轻量，无需构建，声明式数据绑定 |
| 数据库 | Dexie.js + IndexedDB | Promise API 封装，查询简洁 |
| 样式 | 自写 CSS（配合一些 CSS 自定义属性） | 无构建工具，不引入 Tailwind |
| PWA | Service Worker（手动编写） | 离线缓存 + 手机"添加到主屏幕" |
| 部署 | 纯静态文件（HTML + JS + CSS） | 可本地打开或托管任意静态服务器 |

### 2.2 架构概览

```
浏览器
├── Alpine.js UI（4 个 Tab 视图）
│   ├── 计划视图（PlanView）
│   ├── 训练视图（TrainingView）← 含杠铃可视化
│   ├── 历史视图（HistoryView）
│   └── 设置视图（SettingsView）
├── 业务模块（纯 JS 函数）
│   ├── PlanEngine      — 计划生成/递增计算
│   ├── PlateMath       — 配片最优解计算
│   └── BarbellRenderer — CSS 渲染杠铃图形
├── Dexie.js（IndexedDB 封装）
│   ├── TemplateStore
│   ├── WeekPlanStore
│   └── TrainingLogStore
└── Service Worker（离线缓存 + PWA 安装）
```

---

## 3. 功能需求

### 3.1 计划管理（Tab: 计划）

#### 3.1.1 模板配置

用户定义训练模板，作为自动生成计划的基准：

- **训练日槽位**：三个槽位 Day1（Volume）→ Day2（Recovery）→ Day3（Intensity），不绑定具体星期几
- **核心动作（固定）**：深蹲、卧推、推举、硬拉
  - 深蹲：三天均出现（Volume 5×5 / Recovery 2×5 / Intensity 1×5）
  - 卧推与推举：交替做主项（一周卧推冲强度，推举做容量；下一周互换）
  - 硬拉：仅 Intensity Day（1×5）
- **辅助动作（可选）**：划船、高翻、引体向上等，用户可勾选添加
- **百分比可调**：每个动作槽位的目标重量百分比可手动调整（例如 Volume Day 深蹲从 90% 改为 80%）
- **组次可调**：每个动作的组数 × 次数可修改，支持进阶阶段改为 3RM/2RM/1RM

#### 3.1.2 周计划生成

- **生成逻辑**：
  1. 读取模板配置
  2. 基于上一周 Intensity Day 的实际完成重量 × 百分比 → 生成本周 Volume/Recovery 重量
  3. 基于上一周 Intensity Day 重量 + 递增幅度 → 生成本周 Intensity Day 建议重量
- **日期分配**：
  - 默认使用用户在设置中配置的偏好日期（如：Day1=周六, Day2=周二, Day3=周四）
  - 生成后可手动调整任意训练日日期
- **递增确认**：
  - 系统展示本周建议增量（下肢 ±2.5kg、上肢 ±1.5kg 默认，可配置）
  - 用户必须手动确认或调整后才生效（不会自动递增）
  - 若上周某动作未完成/标记失败，该动作本周重量 **建议不增加**（维持上周重量），由用户最终决定

#### 3.1.3 计划卡片展示

- 当前周三个训练日以卡片形式展示
- 每张卡片显示：训练日类型标签、日期、动作列表、目标重量
- 折叠区域：上周完成情况快速回顾（哪些天完成、哪些动作失败/未完成）

---

### 3.2 训练记录（Tab: 训练）

#### 3.2.1 今日训练入口

- 自动定位：匹配今天日期对应的日计划
- 若今天不在计划内：显示最近一次未完成的计划日
- 若无任何待训练计划：引导用户先去"计划"页生成

#### 3.2.2 动作记录卡片

每个动作展示为一张卡片，预填内容：

| 字段 | 来源 | 用户操作 |
|------|------|---------|
| 动作名 | 计划 | 只读 |
| 目标组数×次数 | 计划 | 只读 |
| 目标重量 (kg) | 计划 | 只读 |
| 配片方案 | PlateMath 计算 | 只读，自动展示 |
| 杠铃预览图 | BarbellRenderer | 只读，自动渲染 |
| 每组完成情况 | — | ✓完成 / ✗失败 + 次数微调 |
| RPE (1-10) | — | 可选滑杆或数字输入 |
| 本组备注 | — | 可选文本 |

每组默认预填为"完成"状态（目标次数），用户只需标记失败组和调整实际次数。

#### 3.2.3 汇总备注

- 整个训练日的统一备注区（感受、伤病、状态等自由文本）

#### 3.2.4 完成后操作

- 点击「完成训练」→ 记录写入 TrainingLog，标记 DayPlan 为已完成
- 自动跳转回计划页查看本周进度

---

### 3.3 配片计算与可视化

#### 3.3.1 配片计算（PlateMath）

- 输入：目标重量、杠铃杆重、可用配片规格列表
- 输出：每侧最优装配方案（用最少片数达到目标重量）
- 算法：贪心 + 回溯，优先使用大片，确保重量精确匹配
- 无法精确匹配时：给出最接近的方案并标注偏差

#### 3.3.2 杠铃可视化（BarbellRenderer）

- 纯 CSS 渲染，无 Canvas/SVG
- 渲染元素：
  - **杠铃杆**：水平色条，颜色可配置（默认红色/银色），长度固定比例
  - **配片**：按真实比例叠在杆两端——宽度=片厚比例，高度=片径比例
  - **颜色编码**：默认按健身惯例
    | 重量 | 颜色 |
    |------|------|
    | 25kg | 红色 |
    | 20kg | 蓝色 |
    | 15kg | 黄色 |
    | 10kg | 绿色 |
    | 5kg | 白色 |
    | 2.5kg | 黑色 |
    | 1.25kg | 灰色 |
    | 1kg | 银色 |
    | 0.75kg | 浅灰 |
    | 0.5kg | 铜色 |
    | 0.25kg | 古铜 |
    | 卡扣 | 银色 |
- **展示位置**：训练页每个动作卡片右侧（桌面端并排；移动端卡片下方）
- **个性化设置**（见 3.5.2）：杆颜色、片颜色、大小比例均可自定义

---

### 3.4 历史查看（Tab: 历史）

#### 3.4.1 日志列表

- 按训练日倒序排列
- 每条显示：日期、训练日类型标签、动作数、完成率（完成组数/计划组数）
- 点击展开详情：完整组级记录 + 备注

#### 3.4.2 数据导出/导入

- **导出**：将全部数据（模板 + 计划 + 日志 + 设置）导出为一个 JSON 文件，文件名含导出日期
- **导入**：选择 JSON 文件，覆盖或合并当前数据，导入前弹窗确认
- 两个按钮放在历史页顶部

---

### 3.5 设置（Tab: 设置）

#### 3.5.1 训练偏好

| 设置项 | 默认值 | 说明 |
|--------|--------|------|
| Day1（Volume）默认日期 | 周六 | 可选择周一到周日 |
| Day2（Recovery）默认日期 | 周二 | 可选择周一到周日 |
| Day3（Intensity）默认日期 | 周四 | 可选择周一到周日 |
| 下肢递增幅度 (kg) | 2.5 | 深蹲、硬拉每次增量 |
| 上肢递增幅度 (kg) | 1.5 | 卧推、推举每次增量 |

#### 3.5.2 配片与外观

| 设置项 | 说明 |
|--------|------|
| 杠铃杆重量 (kg) | 默认 20，可修改 |
| 可用配片规格 | 多选：25/20/15/10/5/2.5/1.25/1/0.75/0.5/0.25 kg |
| 杠铃杆颜色 | 默认红色（#CC0000），色块选择器 |
| 配片颜色 | 每个规格独立选色，默认按健身惯例 |
| 训练时显示杠铃预览 | 开关，默认开启 |

#### 3.5.3 系统

| 设置项 | 说明 |
|--------|------|
| 重置全部数据 | 清除 IndexedDB，需二次确认 |
| 关于 | 版本号、技术说明 |

---

## 4. 非功能需求

### 4.1 离线与 PWA

- 首次访问后，所有静态资源被 Service Worker 缓存
- 在无网络环境下可正常使用全部功能（数据本地存储）
- 手机端可"添加到主屏幕"，以独立应用形态运行
- manifest.json 配置应用名、图标、启动方式

### 4.2 响应式设计

- 桌面端：卡片并排布局，杠铃预览在动作卡片右侧
- 移动端：单列布局，杠铃预览在动作卡片下方
- 断点：≥768px 为桌面布局

### 4.3 数据安全

- 所有数据仅存储在浏览器 IndexedDB 中
- 用户需定期手动导出 JSON 备份
- 清除浏览器数据将导致数据丢失（首次使用时提示用户）

### 4.4 性能

- 首屏加载 < 2s（依赖 CDN 的 Alpine.js + Dexie.js 约 30KB gzipped）
- 杠铃渲染使用 CSS，不触发重排
- IndexedDB 查询使用索引，保证大数据量下流畅

---

## 5. 数据模型

### 5.1 Template（训练模板）

```typescript
interface Template {
  id: number;
  name: string;                          // "经典德州" | "自定义"
  exercises: TemplateExercise[];
  // 递增幅度统一读取 Settings.increments，模板不单独存储
}

interface TemplateExercise {
  name: string;                          // "深蹲" | "卧推" | "推举" | "硬拉" | ...
  category: 'core' | 'accessory';        // 核心 vs 辅助
  slots: {
    day: 'volume' | 'recovery' | 'intensity';
    sets: number;
    reps: number;
    percentageOf?: 'intensityDay' | 'volumeDay'; // 以哪个为基准算百分比
    percentage?: number;                  // 百分比值 (90 表示 90%)
    isPR?: boolean;                       // 是否为 PR 冲顶组
  }[];
}
```

### 5.2 WeekPlan（周计划）

```typescript
interface WeekPlan {
  id: number;
  weekNumber: number;
  startDate: string;                     // YYYY-MM-DD（该周第一天）
  confirmed: boolean;
  completed: boolean;
  days: DayPlan[];
}

interface DayPlan {
  id: number;
  weekPlanId: number;
  date: string;                          // YYYY-MM-DD
  dayType: 'volume' | 'recovery' | 'intensity';
  exercises: DayExercise[];
}

interface DayExercise {
  name: string;
  targetSets: number;
  targetReps: number;
  targetWeight: number;                  // kg
  isPR: boolean;
}
```

### 5.3 TrainingLog（训练记录）

```typescript
interface TrainingLog {
  id: number;
  dayPlanId: number;
  date: string;                          // 实际训练日期
  exercises: LoggedExercise[];
  notes: string;                         // 整体备注
}

interface LoggedExercise {
  name: string;
  sets: {
    setNumber: number;
    targetWeight: number;
    targetReps: number;
    actualReps: number | null;           // null = 跳过/未做
    completed: boolean;                  // 完成 vs 失败
    rpe: number | null;                  // 1-10, 选填
    notes: string;                       // 本组备注
  }[];
}
```

### 5.4 Settings（用户设置）

```typescript
interface Settings {
  dayPreferences: {
    day1: number;                        // 0=周日 ... 6=周六, 默认 6
    day2: number;                        // 默认 2（周二）
    day3: number;                        // 默认 4（周四）
  };
  increments: {
    lowerBody: number;
    upperBody: number;
  };
  barbell: {
    barWeight: number;                   // 默认 20
    barColor: string;                    // 默认 "#CC0000"
    availablePlates: PlateConfig[];
  };
  ui: {
    showBarbellPreview: boolean;         // 默认 true
  };
}

interface PlateConfig {
  weight: number;                        // kg
  color: string;                         // CSS 颜色
  diameterRatio: number;                 // 相对最大片直径比例 (0-1)
  thicknessRatio: number;                // 相对厚度比例
}
```

---

## 6. 页面路由与 Tab 切换

使用 Alpine.js 的 `x-show` 实现 SPA 式 Tab 切换，无 URL 路由（纯本地应用）。

| Tab | 图标建议 | 视图 |
|-----|---------|------|
| 计划 | 📋 | PlanView |
| 训练 | 🏋️ | TrainingView |
| 历史 | 📜 | HistoryView |
| 设置 | ⚙️ | SettingsView |

---

## 7. 关键交互规则

### 7.1 递增逻辑

```
IF 上一周 Intensity Day 某动作全部组完成（completed = true）
  → 建议新重量 = 上周 Intensity 重量 + 对应递增幅度
ELSE
  → 建议新重量 = 上周 Intensity 重量（不增加）
  
→ 建议在计划页展示，用户手动确认/修改
```

### 7.2 卧推/推举交替

```
IF 上一周 Intensity Day 主项 = 卧推
  → 本周 Intensity Day 主项 = 推举
  → 本周卧推在 Volume Day 做容量（5×5 @ 副项百分比）
ELSE
  → 本周 Intensity Day 主项 = 卧推
  → 本周推举在 Volume Day 做容量（5×5 @ 副项百分比）
```

### 7.3 计划确认流程

```
Template → 生成 WeekPlan（Draft）→ 用户查看 → 
  → 调整日期/重量 → 确认（confirmed = true）
  → DayPlan 出现在 Training Tab
```

---

## 8. 边界与约束

### 8.1 明确不做的功能

- **热身组计算**：用户明确不需要
- **多维度图表/分析**：用户只要基本日志列表
- **云端同步/账号体系**：纯本地存储
- **多训练计划并行**：同一时间只有一套活跃模板和一个周计划
- **社交分享/排行榜**

### 8.2 技术约束

- 无构建工具，依赖全部 CDN 引入
- 仅支持现代浏览器（Chrome/Firefox/Safari/Edge 近两年版本）
- 不引入任何 CSS 框架或 JS 运行时框架之外的 npm 依赖

---

## 9. 后续迭代方向（非本次范围）

以下方向在需求讨论中未纳入当前版本，但可作为未来参考：

- 训练计时器（组间休息倒计时）
- 减载周自动调度
- 训练日自动提醒（Notification API）
- 多套计划切换（如同时跑两套德州变体）
- 语音播报配片方案（训练时不看屏幕）

---

## 10. 验收标准

| 编号 | 场景 | 预期结果 |
|------|------|---------|
| AC-1 | 首次打开应用 | 引导设置模板（选择辅助动作、确认百分比、设置训练日偏好） |
| AC-2 | 生成第一周计划 | 展示三天的卡片，用户调整日期后确认 |
| AC-3 | 训练日自动填充 | 训练页显示当天计划的全部动作以及目标重量 |
| AC-4 | 记录一组失败 | 标记某组失败，记录实际次数，RPE 可选填 |
| AC-5 | 完成训练 | 数据存入日志，计划页进度更新 |
| AC-6 | 第二周生成 | 展示建议增量，需手动确认；若第一周有失败动作，对应重量不增加 |
| AC-7 | 配片计算 | 输入目标重量，显示两侧最优装配方案 + 杠铃图形 |
| AC-8 | 自定义颜色 | 设置页修改杆颜色和片颜色，训练页即时生效 |
| AC-9 | 导出导入 | 导出 JSON 文件 → 换浏览器导入 → 数据完整恢复 |
| AC-10 | 离线使用 | 断网后刷新页面，应用正常加载并使用 |
| AC-11 | 手机安装 | Chrome/Safari "添加到主屏幕"，独立窗口打开 |
| AC-12 | 卧推/推举交替 | 连续两周 Intensity Day 主项自动互换 |
