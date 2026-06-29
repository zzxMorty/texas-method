# Task 3: 设置页 — Settings View 完整 UI + 读写

**Files:**
- Modify: `D:\code\ai\index.html` — replace settings placeholder HTML + add Alpine methods

**Interfaces:**
- Consumes: `db.settings` table, `DEFAULT_SETTINGS` constant, `getSettings()` function (already defined in Task 2)
- Produces: Alpine methods — `saveSettings()`, `resetSettings()`, `resetAllData()`

## Step 1: Replace settings view placeholder

In `D:\code\ai\index.html`, find the settings placeholder div (currently around line 321):

```html
<div class="view" :class="{ 'active': activeTab === 'settings' }">
    <div class="empty-state">
        <div class="empty-icon">&#9881;&#65039;</div>
        <div class="empty-title">设置</div>
        <div class="empty-desc">配置个人信息和训练偏好</div>
    </div>
</div>
```

Replace the INNER content (keep the outer `<div class="view" :class="...">`) with the full settings UI:

```html
<div class="view" :class="{ 'active': activeTab === 'settings' }">
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
        <option value="0">周日</option><option value="1">周一</option><option value="2">周二</option><option value="3">周三</option>
        <option value="4">周四</option><option value="5">周五</option><option value="6">周六</option>
      </select>
    </div>
    <div class="form-group">
      <label class="form-label">Day3（Intensity）</label>
      <select class="form-input" x-model="settings.dayPreferences.day3">
        <option value="0">周日</option><option value="1">周一</option><option value="2">周二</option><option value="3">周三</option>
        <option value="4">周四</option><option value="5">周五</option><option value="6">周六</option>
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
      <button type="button" class="btn btn-primary" @click="saveSettings()">保存设置</button>
      <button type="button" class="btn btn-outline" @click="resetSettings()" style="color:var(--danger);">恢复默认</button>
    </div>
    <div style="margin-top:12px;padding-top:12px;border-top:1px solid var(--border);">
      <button type="button" class="btn btn-outline" @click="resetAllData()" style="color:var(--danger);">
        ⚠️ 清除全部数据
      </button>
      <p style="font-size:11px;color:var(--text-secondary);margin-top:4px;">此操作不可撤销，建议先导出备份</p>
    </div>
    <p style="font-size:11px;color:var(--text-secondary);margin-top:8px;">Texas Method Tracker v1.0 · 数据仅存储于本浏览器</p>
  </div>
</div>
```

## Step 2: Add Alpine methods

In the Alpine.data('app', () => ({...})) call, find the `init()` method. After the closing `}` of `init()`, add these methods BEFORE the final `}));` that closes the Alpine.data call:

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

## Step 3: Verify in browser

Open index.html → switch to Settings tab → verify all form controls render → modify a setting → Save → refresh → verify persistence.

## Step 4: Commit

Commit with message: "feat: add settings view with training preferences and plate config"
