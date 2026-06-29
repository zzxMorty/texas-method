# Task 2: 数据库层 — Dexie.js Schema + 默认数据初始化

**Files:**
- Modify: `D:\code\ai\index.html` — replace the JavaScript content in the `<script>` block

**Interfaces:**
- Produces: `db` 全局对象（Dexie 实例），含 `templates`、`weekPlans`、`trainingLogs`、`settings` 四个表
- Produces: `getSettings()` 异步函数 → 返回 Settings 对象，无则创建默认值
- Produces: `getDefaultTemplate()` 同步函数 → 返回经典德州模板对象

## Step 1: Replace the Alpine app shell JS with DB layer + app shell

Find the existing `<script>` tag in index.html. Replace ONLY the JavaScript code inside it (keep the `<script>` tags themselves). The current JS is:

```
document.addEventListener('alpine:init', () => {
  Alpine.data('app', () => ({
    activeTab: 'plan',

    async init() {
      console.log('App initialized');
    }
  }));
});
```

Replace it with the FULL content below, which adds the Dexie.js database layer, default settings, default template, and an expanded Alpine app shell:

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
  dayPreferences: { day1: 6, day2: 2, day3: 4 },
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
      { name: '深蹲', category: 'core', slots: [
        { day: 'volume', sets: 5, reps: 5, percentageOf: 'intensityDay', percentage: 90 },
        { day: 'recovery', sets: 2, reps: 5, percentageOf: 'volumeDay', percentage: 80 },
        { day: 'intensity', sets: 1, reps: 5, isPR: true }
      ]},
      { name: '卧推', category: 'core', slots: [
        { day: 'volume', sets: 5, reps: 5, percentageOf: 'intensityDay', percentage: 90 },
        { day: 'intensity', sets: 1, reps: 5, isPR: true }
      ]},
      { name: '推举', category: 'core', slots: [
        { day: 'volume', sets: 5, reps: 5, percentageOf: 'intensityDay', percentage: 90 },
        { day: 'intensity', sets: 1, reps: 5, isPR: true }
      ]},
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

Make sure:
- The Dexie.js CDN script is loaded BEFORE this code (it should be, since Dexie is in `<head>` without defer, and this script is at the end of `<body>`)
- The `<script>` tag containing this code is NOT `type="module"` (must work with file://)
- No syntax errors in the JS

## Step 2: Verify in browser

Open index.html → F12 Console → should see "App initialized" with settings and template objects.
Open Application → IndexedDB → TexasMethodDB → verify 4 tables exist with initial data in settings and templates.

## Step 3: Commit

Commit with message: "feat: add Dexie.js database layer with default settings and template"
