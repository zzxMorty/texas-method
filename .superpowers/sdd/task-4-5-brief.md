# Tasks 4+5: PlateMath + BarbellRenderer

**Files:**
- Modify: `D:\code\ai\index.html` — add PlateMath functions + BarbellRenderer function + barbell CSS

## Part A: PlateMath (Task 4)

Find the `// ===== Database (Dexie.js) =====` section in the script block. After the DB code and BEFORE `// ===== Default Settings =====`, insert:

```javascript
// ===== PlateMath =====
function calculatePlates(targetWeight, barWeight, availablePlates) {
  const sideWeight = (targetWeight - barWeight) / 2;
  if (sideWeight <= 0) {
    return { perSide: [], totalWeight: barWeight, exact: sideWeight === 0 };
  }
  const plates = availablePlates
    .filter(p => p.enabled !== false)
    .sort((a, b) => b.weight - a.weight)
    .map(p => p.weight);

  const result = findBestCombination(sideWeight, plates);
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
  let best = null;
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
    if (best && current.length >= best.combination.length) return;
    current.push(plates[idx]);
    backtrack(current, Math.round((rem - plates[idx]) * 1000) / 1000, plates, idx);
    current.pop();
    backtrack(current, rem, plates, idx + 1);
  }
}

function formatPlateList(perSide) {
  if (!perSide || perSide.length === 0) return '空杆';
  return perSide.map(p => p.weight + 'kg').join(' + ');
}
```

## Part B: BarbellRenderer CSS (Task 5)

Add to the `<style>` block, before `</style>`:

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

## Part C: BarbellRenderer JS (Task 5)

After the PlateMath code, add:

```javascript
// ===== BarbellRenderer =====
function renderBarbellHTML(targetWeight, barWeight, barColor, plateResult) {
  const { perSide } = plateResult;
  const BAR_LENGTH = 140;
  const PLATE_BASE_HEIGHT = 60;
  const PLATE_BASE_WIDTH = 10;

  function buildPlateSide(plates) {
    if (!plates || plates.length === 0) return '';
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
  const rightPlates = buildPlateSide([...perSide].reverse());

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

## Verify

Check the file has no JS syntax errors. Verify the CSS and JS blocks are properly placed.

## Commit

Commit with message: "feat: add PlateMath engine and BarbellRenderer with CSS visualization"
