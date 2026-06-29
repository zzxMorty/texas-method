# Tasks 6+7: PlanEngine + Plan View

**Files:**
- Modify: `D:\code\ai\index.html` — add PlanEngine functions + replace plan view placeholder + add Alpine plan methods

## Part A: PlanEngine (Task 6)

In the `<script>` block, after the BarbellRenderer code, add these PlanEngine functions:

```javascript
// ===== PlanEngine =====
function getNextDate(dayOfWeek, afterDate) {
  const d = new Date(afterDate || Date.now());
  d.setDate(d.getDate() + 1);
  while (d.getDay() !== dayOfWeek) {
    d.setDate(d.getDate() + 1);
  }
  return d.toISOString().split('T')[0];
}

function generateWeekPlan(template, settings, lastWeekPlan, lastWeekTrainingLogs) {
  const now = new Date();
  const weekStart = now.toISOString().split('T')[0];

  const day1Date = getNextDate(settings.dayPreferences.day1, new Date(Date.now() - 86400000));
  const day2Date = getNextDate(settings.dayPreferences.day2, new Date(day1Date));
  const day3Date = getNextDate(settings.dayPreferences.day3, new Date(day2Date));

  let benchIsIntensity = true;
  if (lastWeekPlan) {
    const lastIntensity = lastWeekPlan.days.find(d => d.dayType === 'intensity');
    if (lastIntensity) {
      const hasBenchPR = lastIntensity.exercises.some(e => e.name === '卧推' && e.isPR);
      benchIsIntensity = !hasBenchPR;
    }
  }

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
            lastIntensityWeights[ex.name] = { weight: ex.targetWeight, increment: allCompleted };
          }
        } else {
          lastIntensityWeights[ex.name] = { weight: ex.targetWeight, increment: false };
        }
      }
    }
  }

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

    let targetWeight = null;
    if (slot.isPR) {
      const last = lastIntensityWeights[ex.name];
      if (last && last.weight) {
        const cat = getExerciseCategory(ex.name);
        const inc = cat === 'lower' ? settings.increments.lowerBody : settings.increments.upperBody;
        targetWeight = last.increment ? last.weight + inc : last.weight;
      }
    } else if (slot.percentageOf && slot.percentage) {
      const baseWeight = lastIntensityWeights[ex.name]
        ? lastIntensityWeights[ex.name].weight
        : null;
      if (baseWeight) {
        targetWeight = Math.round(baseWeight * slot.percentage / 100 / 2.5) * 2.5;
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

  return { date: date, dayType: dayType, exercises: exercises };
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

## Part B: Plan View HTML (Task 7)

Find the plan view placeholder div (it has `activeTab === 'plan'` with an empty-state inside). Replace the inner content with:

```html
<h2 style="padding:8px 0 16px;">📋 训练计划</h2>

<div x-show="!currentWeekPlan" class="empty-state">
  <div class="empty-icon">📋</div>
  <p style="margin-bottom:16px;">还没有训练计划</p>
  <button type="button" class="btn btn-primary" @click="createNewWeekPlan()">生成新一周计划</button>
</div>

<template x-if="currentWeekPlan">
  <div>
    <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:16px;">
      <div>
        <span style="font-weight:600;" x-text="'第 ' + currentWeekPlan.weekNumber + ' 周'"></span>
        <span style="font-size:12px;color:var(--text-secondary);margin-left:8px;"
              x-text="currentWeekPlan.confirmed ? '✅ 已确认' : '⚠️ 待确认'"></span>
      </div>
      <div style="display:flex;gap:8px;">
        <button type="button" class="btn btn-sm btn-outline" @click="createNewWeekPlan()"
                x-show="currentWeekPlan.confirmed && currentWeekPlan.completed">生成下周</button>
        <button type="button" class="btn btn-sm btn-primary" @click="confirmWeekPlan()"
                x-show="!currentWeekPlan.confirmed">确认计划</button>
      </div>
    </div>

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
```

## Part C: Alpine Plan Methods (Task 7)

In the Alpine.data('app', ...) object, after the existing methods (saveSettings, resetSettings, resetAllData), add:

```javascript
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
  this.activeTab = 'plan';
},

async confirmWeekPlan() {
  if (!this.currentWeekPlan) return;
  for (const day of this.currentWeekPlan.days) {
    for (const ex of day.exercises) {
      if (ex.targetWeight === null || ex.targetWeight <= 0) {
        alert('请为 ' + ex.name + ' (' + day.dayType + ') 设置目标重量');
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

Also update the `init()` method to call `await this.loadCurrentPlan();` at the end, right before `console.log('App initialized', ...)`.

## Verify

Browser: open index.html → Plan tab → click "生成新一周计划" → verify 3 day cards render with correct type badges → input weights → "确认计划" → refresh → plan persists.

## Commit

Commit with message: "feat: add PlanEngine and plan view with week generation and confirmation"
