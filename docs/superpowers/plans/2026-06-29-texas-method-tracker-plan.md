# Texas Method 力量训练追踪器 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个纯前端 Web 应用，支持德州训练法的计划自动生成、训练记录、配片可视化，全部数据本地存储，PWA 可离线使用。

**Architecture:** 单 HTML 文件（index.html）包含 Alpine.js UI、Dexie.js 数据层、业务逻辑模块；外加 sw.js（Service Worker）、manifest.json（PWA）和图标文件。所有依赖通过 CDN 引入，零构建工具。

**Tech Stack:** Alpine.js 3.x (CDN) + Dexie.js 4.x (CDN) + 自写 CSS + Service Worker API

## 文件结构

```
D:\code\ai\
├── index.html              # 主应用（HTML + CSS + JS，所有功能）
├── sw.js                   # Service Worker（离线缓存）
├── manifest.json           # PWA 清单
└── icons/
    ├── icon-192.png        # PWA 图标 192x192
    └── icon-512.png        # PWA 图标 512x512
```

**说明**：index.html 包含所有 HTML 模板、CSS 样式和 JavaScript 逻辑。按 `// ===== Section Name =====` 注释分隔各模块。所有 View 仅通过 `x-show` 切换，无 URL 路由。

## 全局约束

- 零构建工具，所有 CDN 依赖通过 `<script>` 标签引入
- 支持 `file://` 协议直接打开（不使用 ES modules）
- 仅支持现代浏览器（Chrome/Firefox/Safari/Edge 近两年版本）
- 不引入 CSS 框架或 npm 依赖
- 数据仅存 IndexedDB，用户手动导出 JSON 备份
- 增量单位：下肢默认 2.5kg，上肢默认 1.5kg（设置页可调）
- 默认训练日：Day1=周六, Day2=周二, Day3=周四（设置页可调）
- 杠铃杆默认 20kg / 红色 #CC0000
- 一次只支持一套活跃模板和一个周计划
- 不做热身组计算、不做图表分析、不做云同步

---

### Task 1: 项目骨架 — HTML Shell + Tab 导航 + Alpine.js 初始化

**Files:**
- Create: `D:\code\ai\index.html`

**Interfaces:**
- Produces: `index.html` 基础结构，含 `<head>`、CDN 引用、空 `<body>` 含 Alpine.js `x-data="app"` 和 4 个 Tab 占位 div

- [ ] **Step 1: 创建 index.html 基础结构**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>德州训练法 · 训练追踪</title>
  <link rel="manifest" href="manifest.json">
  <meta name="theme-color" content="#1a1a2e">
  <style>
    /* ===== CSS Reset & Variables ===== */
    :root {
      --bg: #f5f5f5;
      --surface: #ffffff;
      --text: #1a1a2e;
      --text-secondary: #666;
      --primary: #cc0000;
      --primary-light: #ff4444;
      --success: #2ecc71;
      --danger: #e74c3c;
      --border: #e0e0e0;
      --radius: 12px;
      --shadow: 0 2px 8px rgba(0,0,0,0.1);
      --nav-height: 64px;
    }

    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: var(--bg);
      color: var(--text);
      padding-bottom: calc(var(--nav-height) + 16px);
      max-width: 800px;
      margin: 0 auto;
      min-height: 100dvh;
    }

    /* ===== Bottom Tab Navigation ===== */
    .tab-nav {
      position: fixed;
      bottom: 0;
      left: 50%;
      transform: translateX(-50%);
      width: 100%;
      max-width: 800px;
      height: var(--nav-height);
      background: var(--surface);
      border-top: 1px solid var(--border);
      display: flex;
      justify-content: space-around;
      align-items: center;
      z-index: 100;
    }
    .tab-btn {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 2px;
      border: none;
      background: none;
      color: var(--text-secondary);
      font-size: 11px;
      cursor: pointer;
      padding: 8px 16px;
      border-radius: 8px;
      transition: color 0.2s;
    }
    .tab-btn.active {
      color: var(--primary);
      font-weight: 600;
    }
    .tab-btn .tab-icon { font-size: 22px; }

    /* ===== View Container ===== */
    .view {
      padding: 16px;
      display: none;
    }
    .view.active {
      display: block;
    }

    /* ===== Card ===== */
    .card {
      background: var(--surface);
      border-radius: var(--radius);
      padding: 16px;
      margin-bottom: 12px;
      box-shadow: var(--shadow);
    }
    .card-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 8px;
    }
    .card-title { font-size: 16px; font-weight: 600; }

    /* ===== Badge ===== */
    .badge {
      display: inline-block;
      padding: 2px 10px;
      border-radius: 12px;
      font-size: 11px;
      font-weight: 600;
    }
    .badge-volume { background: #ffe0e0; color: #cc0000; }
    .badge-recovery { background: #e0f0ff; color: #0066cc; }
    .badge-intensity { background: #ffe0ff; color: #9900cc; }

    /* ===== Buttons ===== */
    .btn {
      padding: 10px 20px;
      border: none;
      border-radius: 8px;
      font-size: 14px;
      font-weight: 600;
      cursor: pointer;
      transition: opacity 0.2s;
    }
    .btn:active { opacity: 0.7; }
    .btn-primary { background: var(--primary); color: #fff; }
    .btn-outline { background: none; border: 2px solid var(--border); color: var(--text); }
    .btn-sm { padding: 6px 12px; font-size: 12px; }

    /* ===== Form ===== */
    .form-group { margin-bottom: 12px; }
    .form-label { display: block; font-size: 13px; font-weight: 600; margin-bottom: 4px; color: var(--text-secondary); }
    .form-input {
      width: 100%;
      padding: 10px 12px;
      border: 1px solid var(--border);
      border-radius: 8px;
      font-size: 15px;
    }
    select.form-input { appearance: none; background: var(--surface); }

    /* ===== Empty state ===== */
    .empty-state {
      text-align: center;
      padding: 40px 20px;
      color: var(--text-secondary);
    }
    .empty-state .empty-icon { font-size: 48px; margin-bottom: 12px; }
  </style>
  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.14.1/dist/cdn.min.js"></script>
  <script src="https://unpkg.com/dexie@4.0.8/dist/dexie.js"></script>
</head>
<body x-data="app">

  <!-- ===== Tab Navigation ===== -->
  <nav class="tab-nav">
    <button class="tab-btn" :class="{ active: activeTab === 'plan' }" @click="activeTab = 'plan'">
      <span class="tab-icon">📋</span> 计划
    </button>
    <button class="tab-btn" :class="{ active: activeTab === 'train' }" @click="activeTab = 'train'">
      <span class="tab-icon">🏋️</span> 训练
    </button>
    <button class="tab-btn" :class="{ active: activeTab === 'history' }" @click="activeTab = 'history'">
      <span class="tab-icon">📜</span> 历史
    </button>
    <button class="tab-btn" :class="{ active: activeTab === 'settings' }" @click="activeTab = 'settings'">
      <span class="tab-icon">⚙️</span> 设置
    </button>
  </nav>

  <!-- ===== Plan View Placeholder ===== -->
  <div class="view" :class="{ active: activeTab === 'plan' }" x-show="activeTab === 'plan'">
    <h2 style="padding:16px">📋 计划 — 待实现</h2>
  </div>

  <!-- ===== Training View Placeholder ===== -->
  <div class="view" :class="{ active: activeTab === 'train' }" x-show="activeTab === 'train'">
    <h2 style="padding:16px">🏋️ 训练 — 待实现</h2>
  </div>

  <!-- ===== History View Placeholder ===== -->
  <div class="view" :class="{ active: activeTab === 'history' }" x-show="activeTab === 'history'">
    <h2 style="padding:16px">📜 历史 — 待实现</h2>
  </div>

  <!-- ===== Settings View Placeholder ===== -->
  <div class="view" :class="{ active: activeTab === 'settings' }" x-show="activeTab === 'settings'">
    <h2 style="padding:16px">⚙️ 设置 — 待实现</h2>
  </div>

  <script>
    // ===== Alpine App Shell =====
    document.addEventListener('alpine:init', () => {
      Alpine.data('app', () => ({
        activeTab: 'plan',

        async init() {
          console.log('App initialized');
        }
      }));
    });
  </script>
</body>
</html>
```

- [ ] **Step 2: 在浏览器中打开 index.html，验证 4 个 Tab 切换正常**

用浏览器打开 `D:\code\ai\index.html`，依次点击底部 4 个 Tab 按钮，确认视图切换正常且按钮高亮跟随。

- [ ] **Step 3: Commit**

```
git add index.html
git commit -m "feat: add project scaffold with Alpine.js shell and tab navigation"
```

---

### Task 2: 数据库层 — Dexie.js Schema + 默认数据初始化

**Files:**
- Modify: `D:\code\ai\index.html` — 在 `</style>` 前、`<script>` 块中替换 JS 内容

**Interfaces:**
- Produces: `db` 全局对象（Dexie 实例），含 `templates`、`weekPlans`、`trainingLogs`、`settings` 四个表
- Produces: `getSettings()` 异步函数 → 返回 Settings 对象，无则创建默认值
- Produces: `getDefaultTemplate()` 同步函数 → 返回经典德州模板对象

- [ ] **Step 1: 在 index.html 中的 Alpine init 之前，添加 Dexie.js 数据库初始化和默认数据定义**

替换 `<script>` 标签中 `// ===== Alpine App Shell =====` 之后的所有 JS 代码为以下内容：

```javascript
// ===== Database (Dexie.js) =====
const db = new Dexie('TexasMethodDB');
db.version(1).stores({
  templates: '++id',
  weekPlans: '++id, startDate',
  trainingLogs: '++id, dayPlanId, date',
  settings: '++id'
});

// ===== Default Settings =====
const DEFAULT_SETTINGS = {
  dayPreferences: { day1: 6, day2: 2, day3: 4 },  // 周六/周二/周四
  increments: { lowerBody: 2.5, upperBody: 1.5 },
  barbell: {
    barWeight: 20,
    barColor: '#CC0000',
    availablePlates: [
      { weight: 25, color: '#CC0000', diameterRatio: 1.0, thicknessRatio: 1.0 },
      { weight: 20, color: '#0066CC', diameterRatio: 0.92, thicknessRatio: 0.85 },
      { weight: 15, color: '#FFCC00', diameterRatio: 0.82, thicknessRatio: 0.7 },
      { weight: 10, color: '#00AA44', diameterRatio: 0.72, thicknessRatio: 0.55 },
      { weight: 5,  color: '#FFFFFF', diameterRatio: 0.55, thicknessRatio: 0.4 },
      { weight: 2.5, color: '#222222', diameterRatio: 0.4, thicknessRatio: 0.25 },
      { weight: 1.25, color: '#999999', diameterRatio: 0.32, thicknessRatio: 0.18 },
      { weight: 1, color: '#C0C0C0', diameterRatio: 0.28, thicknessRatio: 0.14 },
      { weight: 0.75, color: '#D0D0D0', diameterRatio: 0.24, thicknessRatio: 0.11 },
      { weight: 0.5, color: '#CD7F32', diameterRatio: 0.2, thicknessRatio: 0.08 },
      { weight: 0.25, color: '#B87333', diameterRatio: 0.16, thicknessRatio: 0.06 }
    ]
  },
  ui: { showBarbellPreview: true }
};

async function getSettings() {
  let s = await db.settings.toCollection().first();
  if (!s) {
    const id = await db.settings.add(JSON.parse(JSON.stringify(DEFAULT_SETTINGS)));
    s = await db.settings.get(id);
  }
  return s;
}

// ===== Default Template =====
function getDefaultTemplate() {
  return {
    name: '经典德州',
    exercises: [
      // 深蹲 — 三天都练
      { name: '深蹲', category: 'core', slots: [
        { day: 'volume', sets: 5, reps: 5, percentageOf: 'intensityDay', percentage: 90 },
        { day: 'recovery', sets: 2, reps: 5, percentageOf: 'volumeDay', percentage: 80 },
        { day: 'intensity', sets: 1, reps: 5, isPR: true }
      ]},
      // 卧推 — 主项在 Intensity 冲重量，副项在 Volume 做容量
      { name: '卧推', category: 'core', slots: [
        { day: 'volume', sets: 5, reps: 5, percentageOf: 'intensityDay', percentage: 90 },
        { day: 'intensity', sets: 1, reps: 5, isPR: true }
      ]},
      // 推举 — 与卧推交替
      { name: '推举', category: 'core', slots: [
        { day: 'volume', sets: 5, reps: 5, percentageOf: 'intensityDay', percentage: 90 },
        { day: 'intensity', sets: 1, reps: 5, isPR: true }
      ]},
      // 硬拉 — 仅 Intensity Day
      { name: '硬拉', category: 'core', slots: [
        { day: 'intensity', sets: 1, reps: 5, isPR: true }
      ]}
    ]
  };
}

// ===== Alpine App Shell =====
document.addEventListener('alpine:init', () => {
  Alpine.data('app', () => ({
    activeTab: 'plan',
    settings: null,
    template: null,
    currentWeekPlan: null,
    trainingLogs: [],

    async init() {
      this.settings = await getSettings();
      // 加载或创建模板
      const tmpl = await db.templates.toCollection().first();
      if (!tmpl) {
        const id = await db.templates.add(getDefaultTemplate());
        this.template = await db.templates.get(id);
      } else {
        this.template = tmpl;
      }
      console.log('App initialized', this.settings, this.template);
    }
  }));
});
```

- [ ] **Step 2: 用浏览器打开 index.html，打开 DevTools Console，验证输出**

打开 `D:\code\ai\index.html` → F12 → Console，应看到 `App initialized` 及 settings 和 template 对象。

- [ ] **Step 3: 打开 Application → IndexedDB → TexasMethodDB，验证 tables 和 schema 存在**

在 DevTools 的 Application 面板中展开 IndexedDB → TexasMethodDB，确认 templates、weekPlans、trainingLogs、settings 四个表均已创建，且 settings 和 templates 中有初始数据。

- [ ] **Step 4: Commit**

```
git add index.html
git commit -m "feat: add Dexie.js database layer with default settings and template"
```

---

### Task 3: 设置页 — Settings View 完整 UI + 读写

**Files:**
- Modify: `D:\code\ai\index.html` — 替换设置视图占位 HTML + 扩展 Alpine.js 数据和方法

**Interfaces:**
- Consumes: `db.settings` 表结构, `DEFAULT_SETTINGS` 常量, `getSettings()` 函数
- Produces: Alpine methods — `saveSettings()`, `resetSettings()`, `updatePlateColor(weight, color)`

- [ ] **Step 1: 替换设置视图的占位 HTML**

找到 `<!-- ===== Settings View Placeholder ===== -->` 及其所在 div，替换为：

```html
<!-- ===== Settings View ===== -->
<div class="view" :class="{ active: activeTab === 'settings' }" x-show="activeTab === 'settings'">
  <h2 style="padding:8px 0 16px;">⚙️ 设置</h2>

  <!-- 训练偏好 -->
  <div class="card">
    <div class="card-header"><span class="card-title">📅 训练日偏好</span></div>
    <div class="form-group">
      <label class="form-label">Day1（Volume）</label>
      <select class="form-input" x-model="settings.dayPreferences.day1">
        <option value="0">周日</option><option value="1">周一</option>
        <option value="2">周二</option><option value="3">周三</option>
        <option value="4">周四</option><option value="5">周五</option>
        <option value="6">周六</option>
      </select>
    </div>
    <div class="form-group">
      <label class="form-label">Day2（Recovery）</label>
      <select class="form-input" x-model="settings.dayPreferences.day2">
        <option value="0">周日</option><option value="1">周一</option>
        <option value="2">周二</option><option value="3">周三</option>
        <option value="4">周四</option><option value="5">周五</option>
        <option value="6">周六</option>
      </select>
    </div>
    <div class="form-group">
      <label class="form-label">Day3（Intensity）</label>
      <select class="form-input" x-model="settings.dayPreferences.day3">
        <option value="0">周日</option><option value="1">周一</option>
        <option value="2">周二</option><option value="3">周三</option>
        <option value="4">周四</option><option value="5">周五</option>
        <option value="6">周六</option>
      </select>
    </div>
    <div class="form-group">
      <label class="form-label">下肢递增幅度 (kg)</label>
      <input type="number" class="form-input" x-model.number="settings.increments.lowerBody" step="0.25" min="0">
    </div>
    <div class="form-group">
      <label class="form-label">上肢递增幅度 (kg)</label>
      <input type="number" class="form-input" x-model.number="settings.increments.upperBody" step="0.25" min="0">
    </div>
  </div>

  <!-- 配片与外观 -->
  <div class="card">
    <div class="card-header"><span class="card-title">🏋️ 配片与外观</span></div>
    <div class="form-group">
      <label class="form-label">杠铃杆重量 (kg)</label>
      <input type="number" class="form-input" x-model.number="settings.barbell.barWeight" step="0.5" min="0">
    </div>
    <div class="form-group">
      <label class="form-label">杠铃杆颜色</label>
      <input type="color" class="form-input" x-model="settings.barbell.barColor" style="height:40px;padding:4px;">
    </div>
    <div class="form-group">
      <label class="form-label">可用配片规格</label>
      <div style="display:flex;flex-wrap:wrap;gap:8px;">
        <template x-for="plate in settings.barbell.availablePlates" :key="plate.weight">
          <label style="display:flex;align-items:center;gap:4px;font-size:13px;cursor:pointer;">
            <input type="checkbox" x-model="plate.enabled" :checked="plate.enabled !== false">
            <span :style="{color: plate.color}">●</span>
            <span x-text="plate.weight + 'kg'"></span>
          </label>
        </template>
      </div>
    </div>
    <div class="form-group">
      <label class="form-label">训练时显示杠铃预览</label>
      <label style="display:flex;align-items:center;gap:8px;cursor:pointer;">
        <input type="checkbox" x-model="settings.ui.showBarbellPreview">
        <span style="font-size:13px;">显示</span>
      </label>
    </div>

    <!-- 配片颜色自定义 -->
    <details style="margin-top:8px;">
      <summary style="cursor:pointer;font-weight:600;font-size:14px;">🎨 自定义配片颜色</summary>
      <template x-for="plate in settings.barbell.availablePlates" :key="'color-' + plate.weight">
        <div style="display:flex;align-items:center;gap:8px;margin-top:8px;">
          <span style="font-size:13px;min-width:60px;" x-text="plate.weight + 'kg'"></span>
          <input type="color" x-model="plate.color" style="width:40px;height:30px;">
        </div>
      </template>
    </details>
  </div>

  <!-- 保存 / 系统 -->
  <div class="card">
    <div class="card-header"><span class="card-title">💾 操作</span></div>
    <div style="display:flex;gap:8px;flex-wrap:wrap;">
      <button class="btn btn-primary" @click="saveSettings()">保存设置</button>
      <button class="btn btn-outline" @click="resetSettings()" style="color:var(--danger);">恢复默认</button>
    </div>
    <div style="margin-top:12px;padding-top:12px;border-top:1px solid var(--border);">
      <button class="btn btn-outline" @click="resetAllData()" style="color:var(--danger);">
        ⚠️ 清除全部数据
      </button>
      <p style="font-size:11px;color:var(--text-secondary);margin-top:4px;">此操作不可撤销，建议先导出备份</p>
    </div>
    <p style="font-size:11px;color:var(--text-secondary);margin-top:8px;">Texas Method Tracker v1.0 · 数据仅存储于本浏览器</p>
  </div>
</div>
```

- [ ] **Step 2: 在 Alpine `app` data 对象中添加设置相关方法**

在 `Alpine.data('app', () => ({` 返回对象内的 `init()` 方法之后，添加：

```javascript
async saveSettings() {
  const s = await db.settings.toCollection().first();
  if (s) {
    await db.settings.update(s.id, JSON.parse(JSON.stringify(this.settings)));
  } else {
    await db.settings.add(JSON.parse(JSON.stringify(this.settings)));
  }
  alert('设置已保存 ✓');
},

async resetSettings() {
  if (!confirm('恢复默认设置？当前设置将被覆盖。')) return;
  const s = await db.settings.toCollection().first();
  if (s) {
    await db.settings.update(s.id, JSON.parse(JSON.stringify(DEFAULT_SETTINGS)));
  }
  this.settings = await getSettings();
  alert('已恢复默认设置');
},

async resetAllData() {
  if (!confirm('⚠️ 确定清除全部数据？此操作不可撤销！\n\n建议先去「历史」页导出备份。')) return;
  if (!confirm('再次确认：清除所有训练计划、日志和设置？')) return;
  await db.templates.clear();
  await db.weekPlans.clear();
  await db.trainingLogs.clear();
  await db.settings.clear();
  location.reload();
}
```

- [ ] **Step 3: 浏览器验证设置页**

打开 index.html → 切换至「设置」Tab → 修改训练日偏好和递增幅度 → 点击「保存设置」→ 刷新页面 → 确认设置已持久化。

- [ ] **Step 4: Commit**

```
git add index.html
git commit -m "feat: add settings view with training preferences and plate config"
```

---

### Task 4: 配片计算引擎 — PlateMath

**Files:**
- Modify: `D:\code\ai\index.html` — 在 `// ===== Database (Dexie.js) =====` 之后添加 PlateMath 函数

**Interfaces:**
- Consumes: `settings.barbell.availablePlates[]`（PlateConfig 数组）、`settings.barbell.barWeight`
- Produces: `calculatePlates(targetWeight, barWeight, availablePlates)` → `{ perSide: PlateConfig[], totalWeight: number, exact: boolean }`
- Produces: `formatPlateList(perSide)` → 字符串如 "20+20+5+1.25"

- [ ] **Step 1: 在数据库初始化代码之后、默认设置之前，添加 PlateMath 函数**

在 `// ===== Default Settings =====` 之前插入：

```javascript
// ===== PlateMath =====
function calculatePlates(targetWeight, barWeight, availablePlates) {
  const sideWeight = (targetWeight - barWeight) / 2;
  if (sideWeight <= 0) {
    return { perSide: [], totalWeight: barWeight, exact: sideWeight === 0 };
  }
  // 过滤启用的配片，按重量降序排序
  const plates = availablePlates
    .filter(p => p.enabled !== false)
    .sort((a, b) => b.weight - a.weight)
    .map(p => p.weight);

  const result = findBestCombination(sideWeight, plates);
  // 将重量映射回 PlateConfig
  const perSide = (result.combination || []).map(w =>
    availablePlates.find(p => p.weight === w)
  ).filter(Boolean);
  const totalWeight = barWeight + perSide.reduce((sum, p) => sum + p.weight * 2, 0);
  return {
    perSide: perSide,
    totalWeight: Math.round(totalWeight * 100) / 100,
    exact: Math.abs(totalWeight - targetWeight) < 0.01
  };
}

function findBestCombination(target, plates) {
  // plates 已降序排列，贪心优先
  let best = null;
  // 贪心：先用最大片填
  let remaining = target;
  const greedy = [];
  for (const w of plates) {
    while (remaining >= w - 0.001) {
      greedy.push(w);
      remaining = Math.round((remaining - w) * 1000) / 1000;
    }
  }
  if (remaining < 0.005) {
    return { combination: greedy, remainder: 0 };
  }
  // 贪心不够精确时回溯
  best = { combination: greedy, remainder: remaining };
  backtrack([], target, plates, 0);
  return best;

  function backtrack(current, rem, plates, idx) {
    if (rem < 0) return;
    if (rem < 0.005) {
      if (!best || current.length < best.combination.length) {
        best = { combination: [...current], remainder: 0 };
      }
      return;
    }
    if (idx >= plates.length) return;
    // 剪枝：如果当前组合已比最佳差，跳过
    if (best && current.length >= best.combination.length) return;
    // 尝试用当前片
    current.push(plates[idx]);
    backtrack(current, Math.round((rem - plates[idx]) * 1000) / 1000, plates, idx);
    current.pop();
    // 跳过当前片
    backtrack(current, rem, plates, idx + 1);
  }
}

function formatPlateList(perSide) {
  if (!perSide || perSide.length === 0) return '空杆';
  return perSide.map(p => p.weight + 'kg').join(' + ');
}
```

- [ ] **Step 2: 在浏览器 Console 中手动测试 PlateMath**

打开 index.html → F12 Console，执行：

```javascript
// 等待 app 初始化后
const settings = await getSettings();
const result = calculatePlates(100, 20, settings.barbell.availablePlates);
console.log('100kg:', formatPlateList(result.perSide), 'exact:', result.exact);
// 预期: 每侧 "20kg + 20kg" exact: true

const result2 = calculatePlates(92.5, 20, settings.barbell.availablePlates);
console.log('92.5kg:', formatPlateList(result2.perSide), 'exact:', result2.exact);
// 预期: 每侧类似 "20kg + 15kg + 1.25kg" exact: true

const result3 = calculatePlates(47.5, 20, settings.barbell.availablePlates);
console.log('47.5kg:', formatPlateList(result3.perSide), 'exact:', result3.exact);
// 预期: 每侧类似 "10kg + 2.5kg + 1.25kg" exact: true
```

验证配片组合正确且为最少片数。

- [ ] **Step 3: Commit**

```
git add index.html
git commit -m "feat: add PlateMath engine with greedy + backtracking algorithm"
```

---

### Task 5: 杠铃可视化渲染器 — BarbellRenderer

**Files:**
- Modify: `D:\code\ai\index.html` — 在 PlateMath 之后添加 BarbellRenderer 函数，在 `<style>` 中添加杠铃 CSS

**Interfaces:**
- Consumes: `calculatePlates()` 的输出 `{ perSide }`、`settings.barbell.barColor`、`settings.ui.showBarbellPreview`
- Produces: `renderBarbellHTML(targetWeight, barWeight, barColor, plateResult)` → 返回 HTML 字符串
- Produces: CSS 类 `.barbell-container`、`.barbell-bar`、`.barbell-plate`

- [ ] **Step 1: 在 `<style>` 块末尾（`</style>` 之前）添加杠铃可视化 CSS**

```css
/* ===== Barbell Visualization ===== */
.barbell-container {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 12px 0;
  gap: 0;
  overflow-x: auto;
  max-width: 100%;
}
.barbell-plate-group {
  display: flex;
  flex-direction: row-reverse;
  align-items: center;
}
.barbell-plate {
  border-radius: 4px;
  border: 1.5px solid rgba(0,0,0,0.2);
  flex-shrink: 0;
}
.barbell-bar {
  height: 12px;
  border-radius: 6px;
  flex-shrink: 0;
  min-width: 80px;
}
.barbell-collar {
  width: 6px;
  height: 32px;
  background: #C0C0C0;
  border-radius: 2px;
  border: 1px solid rgba(0,0,0,0.2);
  flex-shrink: 0;
}
.barbell-label {
  text-align: center;
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 4px;
}
```

- [ ] **Step 2: 在 PlateMath 代码块之后添加 renderBarbellHTML 函数**

```javascript
// ===== BarbellRenderer =====
function renderBarbellHTML(targetWeight, barWeight, barColor, plateResult) {
  const { perSide } = plateResult;
  const BAR_LENGTH = 140; // px
  const PLATE_BASE_HEIGHT = 60; // px — 最大片的像素高度
  const PLATE_BASE_WIDTH = 10;  // px — 最大片的像素厚度

  // 构建左侧配片 HTML（从内到外：小片在里，大片在外）
  function buildPlateSide(plates) {
    if (!plates || plates.length === 0) return '';
    // 按重量升序（小片在内侧紧贴杆，大片在外侧）
    const sorted = [...plates].sort((a, b) => a.weight - b.weight);
    let html = '';
    for (const plate of sorted) {
      const h = Math.round(plate.diameterRatio * PLATE_BASE_HEIGHT);
      const w = Math.round((plate.thicknessRatio || 0.5) * PLATE_BASE_WIDTH);
      const color = plate.color;
      html += `<div class="barbell-plate" style="
        height:${h}px;width:${w}px;background:${color};
      " title="${plate.weight}kg"></div>`;
    }
    return html;
  }

  const leftPlates = buildPlateSide(perSide);
  const rightPlates = buildPlateSide([...perSide].reverse()); // 右侧镜像

  return `
    <div class="barbell-container">
      <div class="barbell-plate-group">${leftPlates}</div>
      <div class="barbell-collar"></div>
      <div class="barbell-bar" style="background:${barColor};width:${BAR_LENGTH}px;"></div>
      <div class="barbell-collar"></div>
      <div class="barbell-plate-group">${rightPlates}</div>
    </div>
    <div class="barbell-label">${targetWeight}kg · 每侧 ${formatPlateList(perSide)}</div>
  `;
}
```

- [ ] **Step 3: 浏览器 Console 验证杠铃渲染**

```javascript
const settings = await getSettings();
const result = calculatePlates(100, 20, settings.barbell.availablePlates);
const html = renderBarbellHTML(100, 20, settings.barbell.barColor, result);
document.body.innerHTML += html;
```

应看到水平杠铃杆 + 两端叠放的配片 + 下方标注文字。100kg 应显示 20+20 在每侧。

- [ ] **Step 4: Commit**

```
git add index.html
git commit -m "feat: add BarbellRenderer with CSS visualization"
```

---

### Task 6: 计划引擎 — PlanEngine（生成 + 递增 + 交替）

**Files:**
- Modify: `D:\code\ai\index.html` — 在 BarbellRenderer 之后添加 PlanEngine 函数

**Interfaces:**
- Consumes: `db.weekPlans`、`db.trainingLogs`、`getSettings()`、Template 数据结构
- Produces: `generateWeekPlan(template, settings, lastWeekPlan, lastWeekLogs)` → `WeekPlan` 对象（未保存）
- Produces: `calculateIncrementAdvice(template, settings, lastWeekPlan, lastWeekLogs)` → `{ [exerciseName]: { suggested: number, reason: string } }`
- Produces: `getNextDate(dayIndex, referenceDate)` → 计算下一个训练日日期

- [ ] **Step 1: 在 BarbellRenderer 之后添加 PlanEngine**

```javascript
// ===== PlanEngine =====
function getNextDate(dayOfWeek, afterDate) {
  // dayOfWeek: 0=周日 ... 6=周六
  // 返回 >= afterDate 的下一个 dayOfWeek 的日期字符串 YYYY-MM-DD
  const d = new Date(afterDate || Date.now());
  d.setDate(d.getDate() + 1); // 从明天开始找
  while (d.getDay() !== dayOfWeek) {
    d.setDate(d.getDate() + 1);
  }
  return d.toISOString().split('T')[0];
}

function generateWeekPlan(template, settings, lastWeekPlan, lastWeekTrainingLogs) {
  const now = new Date();
  const weekStart = now.toISOString().split('T')[0];

  // 计算三个训练日的日期
  const day1Date = getNextDate(settings.dayPreferences.day1, new Date(Date.now() - 86400000));
  const day2Date = getNextDate(settings.dayPreferences.day2, new Date(day1Date));
  const day3Date = getNextDate(settings.dayPreferences.day3, new Date(day2Date));

  // 确定本周的卧推/推举交替
  let benchIsIntensity = true; // 默认第一周卧推冲强度
  if (lastWeekPlan) {
    const lastIntensity = lastWeekPlan.days.find(d => d.dayType === 'intensity');
    if (lastIntensity) {
      const hasBenchPR = lastIntensity.exercises.some(e => e.name === '卧推' && e.isPR);
      benchIsIntensity = !hasBenchPR; // 交替
    }
  }

  // 获取上周 Intensity Day 的实际完成重量
  const lastIntensityWeights = {};
  if (lastWeekPlan && lastWeekTrainingLogs) {
    const lastIntensityDay = lastWeekPlan.days.find(d => d.dayType === 'intensity');
    if (lastIntensityDay) {
      const log = lastWeekTrainingLogs.find(l => l.dayPlanId === lastIntensityDay.id);
      for (const ex of lastIntensityDay.exercises) {
        if (log) {
          const logged = log.exercises.find(le => le.name === ex.name);
          if (logged) {
            const allCompleted = logged.sets.every(s => s.completed);
            lastIntensityWeights[ex.name] = {
              weight: ex.targetWeight,
              increment: allCompleted
            };
          }
        } else {
          lastIntensityWeights[ex.name] = { weight: ex.targetWeight, increment: false };
        }
      }
    }
  }

  // 如果没有上周数据，使用模板默认（需要在生成时由用户提供起始 PR）
  // 这里返回的 WeekPlan 中 weights 为 null 表示需要用户输入起始重量
  const days = [
    buildDayPlan(template, 'volume', day1Date, benchIsIntensity, lastIntensityWeights, settings),
    buildDayPlan(template, 'recovery', day2Date, benchIsIntensity, lastIntensityWeights, settings),
    buildDayPlan(template, 'intensity', day3Date, benchIsIntensity, lastIntensityWeights, settings)
  ];

  const lastWeekNumber = lastWeekPlan ? lastWeekPlan.weekNumber : 0;
  return {
    weekNumber: lastWeekNumber + 1,
    startDate: weekStart,
    confirmed: false,
    completed: false,
    days: days
  };
}

function buildDayPlan(template, dayType, date, benchIsIntensity, lastIntensityWeights, settings) {
  const exercises = [];

  for (const ex of template.exercises) {
    const slot = ex.slots.find(s => s.day === dayType);
    if (!slot) continue;

    // 卧推/推举交替逻辑
    if (ex.name === '卧推' && !slot.isPR) {
      // 卧推做容量 = 推举冲强度（本周推举是主项）
      // slot 出现在 volume 说明卧推这周是副项
    }
    if (ex.name === '推举' && !slot.isPR) {
      // 推举做容量 = 卧推冲强度
    }

    // 计算目标重量
    let targetWeight = null;
    if (slot.isPR) {
      // Intensity Day PR 冲顶
      const last = lastIntensityWeights[ex.name];
      if (last && last.weight) {
        const cat = getExerciseCategory(ex.name);
        const inc = cat === 'lower' ? settings.increments.lowerBody : settings.increments.upperBody;
        targetWeight = last.increment ? last.weight + inc : last.weight;
      }
    } else if (slot.percentageOf && slot.percentage) {
      // 基于某基准的百分比
      const baseKey = slot.percentageOf; // 'intensityDay' | 'volumeDay'
      // 这里简化：percentageOf 指向 intensityDay 的对应动作
      const baseWeight = lastIntensityWeights[ex.name]
        ? lastIntensityWeights[ex.name].weight
        : null;
      if (baseWeight) {
        targetWeight = Math.round(baseWeight * slot.percentage / 100 / 2.5) * 2.5; // 取整到 2.5
      }
    }

    exercises.push({
      name: ex.name,
      targetSets: slot.sets,
      targetReps: slot.reps,
      targetWeight: targetWeight,
      isPR: slot.isPR || false
    });
  }

  return {
    date: date,
    dayType: dayType,
    exercises: exercises
  };
}

function getExerciseCategory(name) {
  const lower = ['深蹲', '硬拉'];
  const upper = ['卧推', '推举'];
  if (lower.includes(name)) return 'lower';
  if (upper.includes(name)) return 'upper';
  return 'accessory';
}

function getExerciseDefaultWeight(name) {
  const defaults = { '深蹲': 100, '卧推': 70, '推举': 45, '硬拉': 120 };
  return defaults[name] || 40;
}
```

- [ ] **Step 2: 浏览器 Console 手动测试 PlanEngine**

```javascript
const settings = await getSettings();
const template = await db.templates.toCollection().first();
// 测试第一周生成（无历史数据）
const plan = generateWeekPlan(template, settings, null, null);
console.log('Week', plan.weekNumber);
console.log('Day1 (Volume):', plan.days[0].date, plan.days[0].exercises);
console.log('Day2 (Recovery):', plan.days[1].date, plan.days[1].exercises);
console.log('Day3 (Intensity):', plan.days[2].date, plan.days[2].exercises);
```

应输出三天的计划，重量为 null（首次需用户设置起始 PR）。

- [ ] **Step 3: Commit**

```
git add index.html
git commit -m "feat: add PlanEngine with week generation and bench/press alternation"
```

---

### Task 7: 计划页 — Plan View 完整 UI

**Files:**
- Modify: `D:\code\ai\index.html` — 替换计划视图占位 HTML + 添加计划相关 Alpine 方法

**Interfaces:**
- Consumes: `generateWeekPlan()`、`db.weekPlans`、`db.trainingLogs`、Alpine app state (`currentWeekPlan`, `template`, `settings`)
- Produces: Alpine methods — `createNewWeekPlan()`, `confirmWeekPlan()`, `adjustDayDate(dayIndex, newDate)`, `adjustExerciseWeight(dayIndex, exIndex, newWeight)`

- [ ] **Step 1: 替换计划视图占位 HTML**

```html
<!-- ===== Plan View ===== -->
<div class="view" :class="{ active: activeTab === 'plan' }" x-show="activeTab === 'plan'">
  <h2 style="padding:8px 0 16px;">📋 训练计划</h2>

  <!-- 无计划状态 -->
  <div x-show="!currentWeekPlan" class="empty-state">
    <div class="empty-icon">📋</div>
    <p style="margin-bottom:16px;">还没有训练计划</p>
    <button class="btn btn-primary" @click="createNewWeekPlan()">生成新一周计划</button>
  </div>

  <!-- 有计划时展示 -->
  <template x-if="currentWeekPlan">
    <div>
      <!-- 周计划概览 -->
      <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:16px;">
        <div>
          <span style="font-weight:600;" x-text="'第 ' + currentWeekPlan.weekNumber + ' 周'"></span>
          <span style="font-size:12px;color:var(--text-secondary);margin-left:8px;"
                x-text="currentWeekPlan.confirmed ? '✅ 已确认' : '⚠️ 待确认'"></span>
        </div>
        <div style="display:flex;gap:8px;">
          <button class="btn btn-sm btn-outline" @click="createNewWeekPlan()"
                  x-show="currentWeekPlan.confirmed && currentWeekPlan.completed">生成下周</button>
          <button class="btn btn-sm btn-primary" @click="confirmWeekPlan()"
                  x-show="!currentWeekPlan.confirmed">确认计划</button>
        </div>
      </div>

      <!-- 三个训练日卡片 -->
      <template x-for="(day, di) in currentWeekPlan.days" :key="di">
        <div class="card">
          <div class="card-header">
            <span class="card-title">
              <span class="badge"
                    :class="{ 'badge-volume': day.dayType === 'volume',
                              'badge-recovery': day.dayType === 'recovery',
                              'badge-intensity': day.dayType === 'intensity' }"
                    x-text="day.dayType === 'volume' ? 'Volume 高量' :
                            day.dayType === 'recovery' ? 'Recovery 恢复' : 'Intensity 冲强度'"></span>
            </span>
            <input type="date" class="form-input" style="width:auto;"
                   :value="day.date"
                   @change="day.date = $event.target.value"
                   :disabled="currentWeekPlan.confirmed">
          </div>

          <!-- 动作列表 -->
          <table style="width:100%;font-size:14px;border-collapse:collapse;">
            <thead>
              <tr style="color:var(--text-secondary);font-size:12px;">
                <th style="text-align:left;padding:4px;">动作</th>
                <th style="text-align:center;padding:4px;">组 × 次</th>
                <th style="text-align:right;padding:4px;">目标重量</th>
              </tr>
            </thead>
            <tbody>
              <template x-for="(ex, ei) in day.exercises" :key="ei">
                <tr style="border-top:1px solid var(--border);">
                  <td style="padding:8px 4px;">
                    <span x-text="ex.name"></span>
                    <span x-show="ex.isPR" style="font-size:10px;color:var(--primary);">🏆PR</span>
                  </td>
                  <td style="text-align:center;padding:8px 4px;" x-text="ex.targetSets + '×' + ex.targetReps"></td>
                  <td style="text-align:right;padding:8px 4px;">
                    <span x-show="ex.targetWeight !== null" style="font-weight:600;"
                          x-text="ex.targetWeight + ' kg'"></span>
                    <input x-show="ex.targetWeight === null && !currentWeekPlan.confirmed"
                           type="number" step="2.5" min="0"
                           class="form-input" style="width:80px;text-align:right;"
                           :placeholder="getExerciseDefaultWeight(ex.name)"
                           @change="ex.targetWeight = parseFloat($event.target.value)">
                  </td>
                </tr>
              </template>
            </tbody>
          </table>
        </div>
      </template>
    </div>
  </template>
</div>
```

- [ ] **Step 2: 在 Alpine app 中添加计划相关方法**

在 `Alpine.data('app', () => ({` 内添加：

```javascript
// Plan methods
async createNewWeekPlan() {
  let lastWeekPlan = null;
  let lastWeekLogs = [];
  const allPlans = await db.weekPlans.orderBy('weekNumber').reverse().toArray();
  if (allPlans.length > 0) {
    lastWeekPlan = allPlans[0];
    lastWeekLogs = await db.trainingLogs
      .where('dayPlanId')
      .anyOf(lastWeekPlan.days.map(d => d.id))
      .toArray();
  }
  const plan = generateWeekPlan(this.template, this.settings, lastWeekPlan, lastWeekLogs);
  const id = await db.weekPlans.add(plan);
  plan.id = id;
  this.currentWeekPlan = plan;
  // 切换到计划 tab
  this.activeTab = 'plan';
},

async confirmWeekPlan() {
  if (!this.currentWeekPlan) return;
  // 验证：所有动作都有目标重量
  for (const day of this.currentWeekPlan.days) {
    for (const ex of day.exercises) {
      if (ex.targetWeight === null || ex.targetWeight <= 0) {
        alert(`请为 ${ex.name} (${day.dayType}) 设置目标重量`);
        return;
      }
    }
  }
  this.currentWeekPlan.confirmed = true;
  await db.weekPlans.update(this.currentWeekPlan.id, { confirmed: true, days: this.currentWeekPlan.days });
},

async loadCurrentPlan() {
  const plans = await db.weekPlans.orderBy('weekNumber').reverse().toArray();
  this.currentWeekPlan = plans.find(p => !p.completed) || (plans.length > 0 ? plans[0] : null);
}
```

- [ ] **Step 3: 更新 `init()` 方法以在启动时加载当前计划**

在 `init()` 方法末尾添加 `await this.loadCurrentPlan();`

- [ ] **Step 4: 浏览器验证完整流程**

打开 index.html → 计划 Tab → 点击「生成新一周计划」→ 验证三天卡片显示正确的训练日类型标签和日期 → 输入每个动作的起始重量 → 点击「确认计划」→ 刷新页面确认计划持久化。

- [ ] **Step 5: Commit**

```
git add index.html
git commit -m "feat: add plan view with week generation, date adjustment, and weight confirmation"
```

---

### Task 8: 训练记录页 — Training View 完整 UI + 配片展示

**Files:**
- Modify: `D:\code\ai\index.html` — 替换训练视图占位 HTML + 添加训练相关 Alpine 方法

**Interfaces:**
- Consumes: `currentWeekPlan`、`db.trainingLogs`、`calculatePlates()`、`renderBarbellHTML()`、`settings`
- Produces: Alpine methods — `loadTodayTraining()`、`saveTrainingLog()`、`toggleSetComplete(exIndex, setIndex)`

- [ ] **Step 1: 替换训练视图占位 HTML**

```html
<!-- ===== Training View ===== -->
<div class="view" :class="{ active: activeTab === 'train' }" x-show="activeTab === 'train'">
  <h2 style="padding:8px 0 16px;">🏋️ 今日训练</h2>

  <!-- 无待训练计划 -->
  <div x-show="!todayDayPlan" class="empty-state">
    <div class="empty-icon">🏋️</div>
    <p x-show="!currentWeekPlan" style="margin-bottom:12px;">还没有训练计划</p>
    <p x-show="currentWeekPlan && currentWeekPlan.completed" style="margin-bottom:12px;">本周训练已全部完成 🎉</p>
    <p x-show="currentWeekPlan && !currentWeekPlan.completed && !todayDayPlan" style="margin-bottom:12px;">
      今天没有安排训练
    </p>
    <button class="btn btn-primary" @click="goToPlan()"
            x-show="!currentWeekPlan || currentWeekPlan.completed">去计划页</button>
  </div>

  <!-- 今日训练内容 -->
  <template x-if="todayDayPlan">
    <div>
      <div class="card" style="margin-bottom:16px;">
        <span class="badge"
              :class="{ 'badge-volume': todayDayPlan.dayType === 'volume',
                        'badge-recovery': todayDayPlan.dayType === 'recovery',
                        'badge-intensity': todayDayPlan.dayType === 'intensity' }"
              x-text="todayDayPlan.dayType === 'volume' ? 'Volume 高量日' :
                      todayDayPlan.dayType === 'recovery' ? 'Recovery 恢复日' : 'Intensity 冲强度日'"></span>
        <span style="margin-left:8px;font-size:14px;" x-text="todayDayPlan.date"></span>
      </div>

      <!-- 动作卡片 -->
      <template x-for="(ex, ei) in todayDayPlan.exercises" :key="ei">
        <div class="card">
          <div style="display:flex;gap:16px;flex-wrap:wrap;">
            <!-- 左侧：动作信息 -->
            <div style="flex:1;min-width:200px;">
              <div class="card-header">
                <span class="card-title" x-text="ex.name"></span>
                <span x-show="ex.isPR" style="color:var(--primary);">🏆 PR</span>
              </div>
              <div style="font-size:13px;color:var(--text-secondary);margin-bottom:8px;">
                <span x-text="ex.targetSets + '×' + ex.targetReps + ' @ ' + ex.targetWeight + 'kg'"></span>
              </div>

              <!-- 配片方案 -->
              <div x-show="ex.plateResult" style="margin-bottom:8px;">
                <span style="font-size:12px;color:var(--text-secondary);">
                  配片：<span x-text="formatPlateList(ex.plateResult.perSide)"></span>
                </span>
              </div>

              <!-- 每组记录 -->
              <template x-for="(set, si) in getExerciseSets(ei)" :key="si">
                <div style="display:flex;align-items:center;gap:8px;padding:6px 0;border-top:1px solid var(--border);">
                  <span style="font-size:12px;min-width:36px;" x-text="'组' + (si + 1)"></span>
                  <button class="btn btn-sm"
                          :style="{ background: set.completed ? 'var(--success)' : 'var(--danger)', color:'#fff' }"
                          @click="toggleSet(ei, si)">
                    <span x-text="set.completed ? '✓' : '✗'"></span>
                  </button>
                  <span style="font-size:13px;min-width:60px;"
                        x-text="ex.targetWeight + 'kg × ' + ex.targetReps"></span>
                  <input type="number" class="form-input" style="width:50px;padding:4px;font-size:13px;"
                         x-model.number="set.actualReps" min="0"
                         :placeholder="ex.targetReps">
                  <span style="font-size:11px;color:var(--text-secondary);">次</span>

                  <!-- RPE -->
                  <select class="form-input" style="width:54px;padding:4px;font-size:11px;"
                          x-model.number="set.rpe">
                    <option value="">RPE</option>
                    <option value="6">6</option><option value="6.5">6.5</option>
                    <option value="7">7</option><option value="7.5">7.5</option>
                    <option value="8">8</option><option value="8.5">8.5</option>
                    <option value="9">9</option><option value="9.5">9.5</option>
                    <option value="10">10</option>
                  </select>
                </div>
              </template>
            </div>

            <!-- 右侧：杠铃预览 -->
            <div x-show="settings.ui.showBarbellPreview && ex.plateResult"
                 x-html="ex.barbellHTML"
                 style="flex:0 0 auto;min-width:280px;">
            </div>
          </div>
        </div>
      </template>

      <!-- 备注 + 完成按钮 -->
      <div class="card">
        <div class="form-group">
          <label class="form-label">训练备注（感受、状态等）</label>
          <textarea class="form-input" rows="2" x-model="trainingNotes" placeholder="可选..."></textarea>
        </div>
        <button class="btn btn-primary" style="width:100%;" @click="saveTrainingLog()">
          ✅ 完成训练
        </button>
      </div>
    </div>
  </template>
</div>
```

- [ ] **Step 2: 在 Alpine app 中添加训练相关方法和数据字段**

在 app data 对象中添加字段：

```javascript
todayDayPlan: null,
trainingNotes: '',
exerciseSetsData: [],  // { exerciseIndex, sets: [{completed, actualReps, rpe, notes}] }
```

添加方法：

```javascript
async loadTodayTraining() {
  this.todayDayPlan = null;
  this.exerciseSetsData = [];
  this.trainingNotes = '';

  if (!this.currentWeekPlan || !this.currentWeekPlan.confirmed) return;

  const today = new Date().toISOString().split('T')[0];
  // 查找今天或最近未完成的计划日
  let targetDay = null;
  for (const day of this.currentWeekPlan.days) {
    // 检查是否已有日志
    const existingLog = await db.trainingLogs.where('dayPlanId').equals(day.id).first();
    if (existingLog) continue; // 已记录过

    if (day.date === today) {
      targetDay = day;
      break;
    }
  }
  // 如果今天没有匹配，找最近一个未记录的
  if (!targetDay) {
    for (const day of this.currentWeekPlan.days) {
      const existingLog = await db.trainingLogs.where('dayPlanId').equals(day.id).first();
      if (!existingLog) {
        targetDay = day;
        break;
      }
    }
  }

  if (!targetDay) return;

  this.todayDayPlan = targetDay;

  // 为每个动作初始化组数据、配片计算、杠铃渲染
  for (let i = 0; i < targetDay.exercises.length; i++) {
    const ex = targetDay.exercises[i];
    const sets = [];
    for (let s = 0; s < ex.targetSets; s++) {
      sets.push({ completed: true, actualReps: ex.targetReps, rpe: null, notes: '' });
    }
    this.exerciseSetsData.push({ exerciseIndex: i, sets: sets });

    // 配片计算
    if (ex.targetWeight) {
      const result = calculatePlates(ex.targetWeight, this.settings.barbell.barWeight,
        this.settings.barbell.availablePlates);
      ex.plateResult = result;
      ex.barbellHTML = renderBarbellHTML(ex.targetWeight, this.settings.barbell.barWeight,
        this.settings.barbell.barColor, result);
    }
  }
},

getExerciseSets(ei) {
  const data = this.exerciseSetsData.find(d => d.exerciseIndex === ei);
  return data ? data.sets : [];
},

toggleSet(ei, si) {
  const data = this.exerciseSetsData.find(d => d.exerciseIndex === ei);
  if (data) {
    data.sets[si].completed = !data.sets[si].completed;
  }
},

async saveTrainingLog() {
  if (!this.todayDayPlan) return;

  const loggedExercises = this.todayDayPlan.exercises.map((ex, i) => {
    const data = this.exerciseSetsData.find(d => d.exerciseIndex === i);
    return {
      name: ex.name,
      sets: (data ? data.sets : []).map((s, si) => ({
        setNumber: si + 1,
        targetWeight: ex.targetWeight,
        targetReps: ex.targetReps,
        actualReps: s.completed ? (s.actualReps || ex.targetReps) : (s.actualReps || 0),
        completed: s.completed,
        rpe: s.rpe || null,
        notes: s.notes || ''
      }))
    };
  });

  const log = {
    dayPlanId: this.todayDayPlan.id,
    date: new Date().toISOString().split('T')[0],
    exercises: loggedExercises,
    notes: this.trainingNotes
  };

  await db.trainingLogs.add(log);

  // 检查本周是否全部完成
  const weekDays = this.currentWeekPlan.days;
  let allDone = true;
  for (const day of weekDays) {
    const existingLog = await db.trainingLogs.where('dayPlanId').equals(day.id).first();
    if (day.id === this.todayDayPlan.id) continue; // 刚保存的
    if (!existingLog) { allDone = false; break; }
  }
  if (allDone) {
    await db.weekPlans.update(this.currentWeekPlan.id, { completed: true });
    this.currentWeekPlan.completed = true;
  }

  alert('训练记录已保存 ✓');
  this.todayDayPlan = null;
  this.activeTab = 'plan';
},

goToPlan() {
  this.activeTab = 'plan';
}
```

- [ ] **Step 3: 更新 `init()` 以在进入训练 Tab 时自动加载当日训练**

在 app 中添加 watcher。由于 Alpine.js `x-data` 不支持 computed watchers，改用 Tab 切换时的主动调用：在每个 tab-btn 的 `@click` 中追加逻辑。

修改训练 Tab 按钮中的 `@click`：

```html
<button class="tab-btn" :class="{ active: activeTab === 'train' }"
        @click="activeTab = 'train'; loadTodayTraining()">
```

- [ ] **Step 4: 浏览器完整训练流程测试**

计划页确认一个计划 → 切换到训练 Tab → 验证自动显示今日计划 → 修改某组为失败 → 调整 RPE → 填写备注 → 点击「完成训练」→ 验证跳转回计划页。

- [ ] **Step 5: Commit**

```
git add index.html
git commit -m "feat: add training view with auto-fill, set recording, plate visualization"
```

---

### Task 9: 历史记录页 — History View + 导出/导入

**Files:**
- Modify: `D:\code\ai\index.html` — 替换历史视图 HTML + 添加方法

**Interfaces:**
- Consumes: `db.trainingLogs`、`db.weekPlans`、`db.templates`、`db.settings`
- Produces: Alpine methods — `loadHistory()`、`exportData()`、`importData(file)`

- [ ] **Step 1: 替换历史视图 HTML**

```html
<!-- ===== History View ===== -->
<div class="view" :class="{ active: activeTab === 'history' }" x-show="activeTab === 'history'">
  <h2 style="padding:8px 0 16px;">📜 训练历史</h2>

  <!-- 导出/导入 -->
  <div style="display:flex;gap:8px;margin-bottom:16px;">
    <button class="btn btn-sm btn-outline" @click="exportData()">📤 导出备份</button>
    <label class="btn btn-sm btn-outline" style="cursor:pointer;">
      📥 导入恢复
      <input type="file" accept=".json" style="display:none;" @change="importData($event)">
    </label>
  </div>

  <!-- 空状态 -->
  <div x-show="historyLogs.length === 0" class="empty-state">
    <div class="empty-icon">📜</div>
    <p>暂无训练记录</p>
  </div>

  <!-- 日志列表 -->
  <template x-for="log in historyLogs" :key="log.id">
    <div class="card">
      <div class="card-header" style="cursor:pointer;" @click="log._expanded = !log._expanded">
        <div>
          <span style="font-weight:600;" x-text="log.date"></span>
          <span class="badge" style="margin-left:8px;"
                :class="log._dayType === 'volume' ? 'badge-volume' :
                        log._dayType === 'recovery' ? 'badge-recovery' : 'badge-intensity'"
                x-text="log._dayType === 'volume' ? 'Volume' :
                        log._dayType === 'recovery' ? 'Recovery' : 'Intensity'"></span>
        </div>
        <div style="font-size:12px;color:var(--text-secondary);">
          <span x-text="log._completionRate"></span>
          <span x-text="log._expanded ? ' ▲' : ' ▼'"></span>
        </div>
      </div>

      <!-- 展开详情 -->
      <div x-show="log._expanded" style="border-top:1px solid var(--border);padding-top:8px;margin-top:8px;">
        <template x-for="ex in log.exercises" :key="ex.name">
          <div style="margin-bottom:8px;">
            <div style="font-weight:600;font-size:13px;margin-bottom:4px;" x-text="ex.name"></div>
            <template x-for="s in ex.sets" :key="s.setNumber">
              <div style="font-size:12px;padding:2px 0;display:flex;gap:8px;align-items:center;">
                <span x-text="'组' + s.setNumber + ':'"></span>
                <span :style="{color: s.completed ? 'var(--success)' : 'var(--danger)'}"
                      x-text="s.completed ? '✓' : '✗'"></span>
                <span x-text="s.targetWeight + 'kg × ' + (s.completed ? (s.actualReps || s.targetReps) : (s.actualReps || 0))"></span>
                <span x-show="s.rpe" style="color:var(--text-secondary);" x-text="'RPE ' + s.rpe"></span>
              </div>
            </template>
          </div>
        </template>
        <div x-show="log.notes" style="font-size:12px;color:var(--text-secondary);margin-top:8px;padding-top:8px;border-top:1px solid var(--border);">
          📝 <span x-text="log.notes"></span>
        </div>
      </div>
    </div>
  </template>
</div>
```

- [ ] **Step 2: 添加历史相关 Alpine 方法和数据字段**

在 app data 中添加：

```javascript
historyLogs: [],
```

添加方法：

```javascript
async loadHistory() {
  const logs = await db.trainingLogs.orderBy('date').reverse().toArray();
  // 为每个 log 附加 dayType（从关联的 DayPlan 获取）
  for (const log of logs) {
    if (log._dayType) continue; // 已处理过
    const dayPlan = await db.weekPlans
      .where('id').equals(log.dayPlanId).first();
    // dayPlan 可能在嵌套的 days 数组中
    let dayType = 'intensity';
    if (dayPlan) {
      // log.dayPlanId 是 DayPlan 的 id，需要从 WeekPlan.days 中查找
      const allPlans = await db.weekPlans.toArray();
      for (const wp of allPlans) {
        const day = wp.days.find(d => d.id === log.dayPlanId);
        if (day) { dayType = day.dayType; break; }
      }
    }
    log._dayType = dayType;
    // 计算完成率
    let totalSets = 0, completedSets = 0;
    for (const ex of log.exercises) {
      for (const s of ex.sets) {
        totalSets++;
        if (s.completed) completedSets++;
      }
    }
    log._completionRate = completedSets + '/' + totalSets;
    log._expanded = false;
  }
  this.historyLogs = logs;
},

async exportData() {
  const data = {
    version: 1,
    exportedAt: new Date().toISOString(),
    templates: await db.templates.toArray(),
    weekPlans: await db.weekPlans.toArray(),
    trainingLogs: await db.trainingLogs.toArray(),
    settings: await db.settings.toArray()
  };
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `texas-method-backup-${new Date().toISOString().split('T')[0]}.json`;
  a.click();
  URL.revokeObjectURL(url);
},

async importData(event) {
  const file = event.target.files[0];
  if (!file) return;
  if (!confirm('导入将覆盖当前全部数据（模板、计划、日志、设置）。确定继续？')) {
    event.target.value = '';
    return;
  }
  try {
    const text = await file.text();
    const data = JSON.parse(text);
    if (!data.version || !data.templates || !data.weekPlans || !data.trainingLogs || !data.settings) {
      throw new Error('无效的备份文件格式');
    }
    // 清除现有数据
    await db.templates.clear();
    await db.weekPlans.clear();
    await db.trainingLogs.clear();
    await db.settings.clear();
    // 导入
    await db.templates.bulkAdd(data.templates);
    await db.weekPlans.bulkAdd(data.weekPlans);
    await db.trainingLogs.bulkAdd(data.trainingLogs);
    await db.settings.bulkAdd(data.settings);
    alert('数据导入成功 ✓ 页面将刷新');
    location.reload();
  } catch (e) {
    alert('导入失败：' + e.message);
  }
  event.target.value = '';
}
```

- [ ] **Step 3: 更新历史 Tab 按钮 click 事件**

```html
<button class="tab-btn" :class="{ active: activeTab === 'history' }"
        @click="activeTab = 'history'; loadHistory()">
```

- [ ] **Step 4: 浏览器测试历史页和导出/导入**

完成至少一次训练 → 切换到历史 Tab → 验证日志列表显示 → 点击展开详情 → 点击「导出备份」→ 验证 JSON 文件名和内容 → 点击「导入恢复」选择刚才的 JSON → 验证数据恢复。

- [ ] **Step 5: Commit**

```
git add index.html
git commit -m "feat: add history view with log list, detail expansion, and JSON export/import"
```

---

### Task 10: 首次使用引导 — Onboarding Flow

**Files:**
- Modify: `D:\code\ai\index.html` — 在 `<body>` 顶部添加引导弹窗 HTML + Alpine 方法

**Interfaces:**
- Consumes: `db.settings`、`db.templates`
- Produces: `showOnboarding` Alpine 状态、`completeOnboarding()` 方法

- [ ] **Step 1: 在 `<body>` 顶部（Tab 导航之前）添加引导弹窗**

```html
<!-- ===== Onboarding Modal ===== -->
<div x-show="showOnboarding" x-cloak style="position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.5);z-index:200;display:flex;align-items:center;justify-content:center;">
  <div class="card" style="max-width:400px;width:90%;max-height:80vh;overflow-y:auto;">
    <h3 style="margin-bottom:12px;">🏋️ 欢迎使用德州训练法</h3>
    <p style="font-size:13px;color:var(--text-secondary);margin-bottom:16px;">
      数据仅存储在本浏览器中，建议定期导出备份。
    </p>

    <div class="form-group">
      <label class="form-label">选择辅助动作（可多选）</label>
      <div style="display:flex;flex-wrap:wrap;gap:8px;">
        <template x-for="acc in accessoryOptions" :key="acc.name">
          <label style="display:flex;align-items:center;gap:4px;font-size:13px;cursor:pointer;">
            <input type="checkbox" x-model="acc.selected">
            <span x-text="acc.name"></span>
          </label>
        </template>
      </div>
    </div>

    <div class="form-group">
      <label class="form-label">Volume Day 百分比 (%)</label>
      <input type="number" class="form-input" x-model.number="onboardingVolumePct" step="1" min="60" max="100">
      <span style="font-size:11px;color:var(--text-secondary);">Intensity Day 重量的百分比</span>
    </div>

    <div class="form-group">
      <label class="form-label">训练日偏好</label>
      <div style="display:flex;gap:8px;">
        <select class="form-input" x-model.number="settings.dayPreferences.day1" style="flex:1;">
          <option value="0">周日</option><option value="1">周一</option><option value="2">周二</option>
          <option value="3">周三</option><option value="4">周四</option><option value="5">周五</option>
          <option value="6">周六</option>
        </select>
        <select class="form-input" x-model.number="settings.dayPreferences.day2" style="flex:1;">
          <option value="0">周日</option><option value="1">周一</option><option value="2">周二</option>
          <option value="3">周三</option><option value="4">周四</option><option value="5">周五</option>
          <option value="6">周六</option>
        </select>
        <select class="form-input" x-model.number="settings.dayPreferences.day3" style="flex:1;">
          <option value="0">周日</option><option value="1">周一</option><option value="2">周二</option>
          <option value="3">周三</option><option value="4">周四</option><option value="5">周五</option>
          <option value="6">周六</option>
        </select>
      </div>
      <span style="font-size:11px;color:var(--text-secondary);">Day1/Volume · Day2/Recovery · Day3/Intensity</span>
    </div>

    <button class="btn btn-primary" style="width:100%;margin-top:8px;" @click="completeOnboarding()">
      开始训练
    </button>
  </div>
</div>
```

- [ ] **Step 2: 添加引导相关 CSS（x-cloak）**

在 `<style>` 中添加：`[x-cloak] { display: none !important; }`

- [ ] **Step 3: 在 Alpine app 中添加引导相关数据和方法**

```javascript
showOnboarding: false,
accessoryOptions: [
  { name: '划船', selected: false },
  { name: '高翻', selected: false },
  { name: '引体向上', selected: false }
],
onboardingVolumePct: 90,

async checkOnboardingNeeded() {
  const tmpl = await db.templates.toCollection().first();
  const sets = await db.settings.toCollection().first();
  // 如果模板是默认的且未修改过，或设置是默认的，显示引导
  if (!tmpl || (tmpl.name === '经典德州' && tmpl.exercises.length === 4)) {
    this.showOnboarding = true;
  }
},

async completeOnboarding() {
  // 将选中辅助动作加入模板
  const selectedAccessories = this.accessoryOptions.filter(a => a.selected);
  for (const acc of selectedAccessories) {
    this.template.exercises.push({
      name: acc.name,
      category: 'accessory',
      slots: [
        { day: 'volume', sets: 3, reps: 8 },
        { day: 'recovery', sets: 3, reps: 10 }
      ]
    });
  }
  // 应用百分比到模板（更新 Volume Day slot 的 percentage）
  for (const ex of this.template.exercises) {
    for (const slot of ex.slots) {
      if (slot.day === 'volume' && slot.percentageOf === 'intensityDay') {
        slot.percentage = this.onboardingVolumePct;
      }
    }
  }
  await db.templates.update(this.template.id, { exercises: this.template.exercises });
  // 保存设置
  await this.saveSettings();
  this.showOnboarding = false;
  this.activeTab = 'plan';
}
```

- [ ] **Step 4: 更新 `init()` 以在启动时检测是否需要引导**

在 `init()` 方法最后添加 `await this.checkOnboardingNeeded();`

- [ ] **Step 5: 浏览器验证引导流程**

清除 IndexedDB 数据 → 刷新页面 → 应看到引导弹窗 → 勾选划船 → 调整百分比 → 点击「开始训练」→ 确认模板已更新（检查计划生成时是否包含划船）。

- [ ] **Step 6: Commit**

```
git add index.html
git commit -m "feat: add onboarding flow for first-time setup"
```

---

### Task 11: PWA 支持 — Service Worker + Manifest + 图标

**Files:**
- Create: `D:\code\ai\sw.js`
- Create: `D:\code\ai\manifest.json`
- Create: `D:\code\ai\icons\icon-192.png`、`D:\code\ai\icons\icon-512.png`

**Interfaces:**
- Consumes: index.html 中的缓存资源列表
- Produces: 离线可用性、PWA 安装能力

- [ ] **Step 1: 创建 manifest.json**

```json
{
  "name": "德州训练法 · 训练追踪",
  "short_name": "德州训练",
  "description": "Texas Method 力量训练个人计划与记录",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#f5f5f5",
  "theme_color": "#cc0000",
  "icons": [
    {
      "src": "icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

- [ ] **Step 2: 创建图标（使用 Canvas API 生成简单 SVG 转 PNG 的占位图标）**

执行以下 PowerShell 创建 icons 目录：

```powershell
New-Item -ItemType Directory -Force -Path "D:\code\ai\icons"
```

用浏览器打开以下 data URI 生成图标（在 index.html Console 中执行）：

```javascript
// 在浏览器 Console 中运行，生成并下载 PNG 图标
function generateIcon(size, filename) {
  const canvas = document.createElement('canvas');
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext('2d');
  // 背景
  ctx.fillStyle = '#cc0000';
  ctx.beginPath();
  ctx.roundRect(8, 8, size-16, size-16, size*0.15);
  ctx.fill();
  // 文字
  ctx.fillStyle = '#ffffff';
  ctx.font = `bold ${size*0.35}px sans-serif`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText('🏋️', size/2, size/2);

  canvas.toBlob(blob => {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
  });
}
generateIcon(192, 'icon-192.png');
generateIcon(512, 'icon-512.png');
```

将下载的文件移至 `D:\code\ai\icons\` 目录。

- [ ] **Step 3: 创建 sw.js**

```javascript
// Texas Method Tracker — Service Worker v1.0
const CACHE_NAME = 'texas-method-v1';
const ASSETS = [
  './',
  './index.html',
  './manifest.json',
  'https://cdn.jsdelivr.net/npm/alpinejs@3.14.1/dist/cdn.min.js',
  'https://unpkg.com/dexie@4.0.8/dist/dexie.js'
];

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(ASSETS))
  );
  self.skipWaiting();
});

self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', event => {
  // 只处理 GET 请求
  if (event.request.method !== 'GET') return;
  event.respondWith(
    caches.match(event.request).then(cached => {
      const fetchPromise = fetch(event.request).then(response => {
        if (response && response.status === 200) {
          const clone = response.clone();
          caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        }
        return response;
      }).catch(() => cached);
      return cached || fetchPromise;
    })
  );
});
```

- [ ] **Step 4: 在 index.html 中注册 Service Worker**

在 `init()` 方法末尾添加：

```javascript
// 注册 Service Worker
if ('serviceWorker' in navigator) {
  try {
    await navigator.serviceWorker.register('sw.js');
    console.log('SW registered');
  } catch (e) {
    console.log('SW registration failed (expected for file:// protocol):', e.message);
  }
}
```

> **注**：Service Worker 在 `file://` 协议下无法注册（浏览器安全策略），这是预期行为。通过 HTTP 服务器（如 `npx serve .` 或 Python `http.server`）访问时即可注册。

- [ ] **Step 5: 验证 PWA**

用本地服务器打开 index.html（`npx serve D:\code\ai` 或 Python `python -m http.server`）→ 打开 Chrome DevTools → Application → Service Workers → 确认 SW 已注册 → 地址栏应出现安装图标 → 点击安装 → 独立窗口打开。

- [ ] **Step 6: Commit**

```
git add sw.js manifest.json icons/
git commit -m "feat: add PWA support with service worker, manifest, and icons"
```

---

### Task 12: 润色 — 响应式 CSS + 边缘情况 + 最终验证

**Files:**
- Modify: `D:\code\ai\index.html` — CSS 补充 + Bug 修复

**Interfaces:**
- Consumes: 所有已完成的功能
- Produces: 完整的、可在桌面和移动端正常使用的应用

- [ ] **Step 1: 添加响应式 CSS**

在 `<style>` 块末尾添加：

```css
/* ===== Responsive ===== */
@media (max-width: 767px) {
  .view {
    padding: 12px;
  }
  .card {
    padding: 12px;
  }
  .card-header {
    flex-wrap: wrap;
    gap: 4px;
  }
  table {
    font-size: 13px;
  }
  .btn {
    padding: 8px 16px;
    font-size: 13px;
  }
}

/* ===== Utility ===== */
.text-danger { color: var(--danger); }
.text-success { color: var(--success); }
.text-secondary { color: var(--text-secondary); font-size: 12px; }

/* ===== PR highlight ===== */
.pr-badge {
  display: inline-block;
  background: var(--danger);
  color: #fff;
  font-size: 9px;
  padding: 1px 6px;
  border-radius: 4px;
  margin-left: 4px;
  vertical-align: middle;
}

/* ===== Smooth transitions ===== */
.view {
  animation: fadeIn 0.2s ease;
}
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(4px); }
  to { opacity: 1; transform: translateY(0); }
}
```

- [ ] **Step 2: 修复已知边缘情况**

在 Alpine app 中处理以下边缘情况：

**a) 当 currentWeekPlan 为 null 时，init 不应报错**

确保 `init()` 中 `loadCurrentPlan()` 和 `loadTodayTraining()` 在 `currentWeekPlan` 为 null 时不崩溃。

**b) 计划页无上周数据时生成计划的处理**

在 `createNewWeekPlan()` 中，如果 `lastWeekPlan` 为 null（第一周），所有 Intensity Day 动作为 null 重量，提示用户手动输入起始 PR。

**c) 配片计算中 targetWeight 为 null 的保护**

在 `renderBarbellHTML` 中添加：

```javascript
if (!targetWeight || targetWeight <= 0) return '<div class="barbell-label">请先设置目标重量</div>';
```

**d) 导入数据后刷新 Settings**

在 `importData` 成功后 reload 页面（已实现）。

- [ ] **Step 3: 完整验收测试**

按照以下验收标准逐项测试：

| AC | 场景 | 测试方法 |
|----|------|---------|
| AC-1 | 首次打开应用 | 清除 IndexedDB → 刷新 → 应看到引导弹窗 |
| AC-2 | 生成周计划 | 完成引导 → 计划页 → 生成计划 → 三天卡片展示 |
| AC-3 | 训练日自动填充 | 计划确认 → 训练 Tab → 自动显示今日动作和重量 |
| AC-4 | 记录失败组 | 训练页 → 点击某组 ✗ 按钮 → 标记失败 |
| AC-5 | 完成训练 | 点击完成 → 数据存入 IndexedDB → 计划页更新 |
| AC-6 | 递增确认 | 生成第二周 → 检查建议增量 → 手动确认 |
| AC-7 | 配片可视化 | 训练页 → 动作卡片右侧显示杠铃图形 |
| AC-8 | 自定义颜色 | 设置 → 修改杆颜色 → 训练页即时生效 |
| AC-9 | 导出导入 | 历史页导出 JSON → 清除数据 → 导入恢复 |
| AC-10 | 离线使用 | 本地服务器 → 断网 → 刷新仍可用 |
| AC-11 | PWA 安装 | Chrome → 安装 → 独立窗口运行 |
| AC-12 | 卧推/推举交替 | 连续两周 → 主项互换 |

- [ ] **Step 4: 修复测试中发现的问题**

记录所有发现的问题并修复，然后重新验证。

- [ ] **Step 5: Commit**

```
git add index.html
git commit -m "feat: add responsive CSS, edge case fixes, and final polish"
```

---
