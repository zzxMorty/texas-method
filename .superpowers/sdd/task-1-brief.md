# Task 1: 项目骨架 — HTML Shell + Tab 导航 + Alpine.js 初始化

**Files:**
- Create: `D:\code\ai\index.html`

**Interfaces:**
- Produces: `index.html` 基础结构，含 `<head>`、CDN 引用、空 `<body>` 含 Alpine.js `x-data="app"` 和 4 个 Tab 占位 div

## Step 1: 创建 index.html 基础结构

Create the complete index.html as a single-file web app. The file must work when opened directly via `file://` protocol (no server needed for basic operation, though PWA will need HTTP for SW registration later).

CDN versions to use:
- Alpine.js: `https://cdn.jsdelivr.net/npm/alpinejs@3.14.1/dist/cdn.min.js`
- Dexie.js: `https://unpkg.com/dexie@4.0.8/dist/dexie.js`

Requirements:
1. HTML5 doctype, lang="zh-CN", UTF-8
2. Responsive viewport meta tag
3. Title: "德州训练法 · 训练追踪"
4. Link to manifest.json (will be created later)
5. Theme color meta tag: #1a1a2e
6. Inline `<style>` block with:
   - CSS custom properties (variables) for colors: --bg (#f5f5f5), --surface (#ffffff), --text (#1a1a2e), --text-secondary (#666), --primary (#cc0000), --primary-light (#ff4444), --success (#2ecc71), --danger (#e74c3c), --border (#e0e0e0), --radius (12px), --shadow, --nav-height (64px)
   - Reset: *, body styles with max-width 800px centered, min-height 100dvh
   - Bottom tab navigation: fixed at bottom, max-width 800px, flex space-around
   - Tab buttons: column flex, icon + text, active state with primary color
   - View container: padding 16px, display:none default, display:block when .active class
   - Card component: white bg, border-radius, padding 16px, margin-bottom 12px, box-shadow
   - Card header: flex space-between with title
   - Badge styles: .badge-volume (red bg), .badge-recovery (blue bg), .badge-intensity (purple bg)
   - Button styles: .btn, .btn-primary (red bg white text), .btn-outline, .btn-sm
   - Form styles: .form-group, .form-label, .form-input (full width, border, rounded)
   - Empty state: centered text with icon
7. Alpine.js loaded with defer, Dexie.js loaded normally (before Alpine)
8. Body with x-data="app":
   - Bottom tab nav with 4 buttons (📋 计划, 🏋️ 训练, 📜 历史, ⚙️ 设置)
   - Each button uses :class binding for active state, @click to switch activeTab
   - 4 placeholder view divs with x-show for the 4 tabs
9. Inline `<script>` block after body:
   - Alpine:init event listener
   - Alpine.data('app', ...) with activeTab: 'plan' and async init() that logs "App initialized"

The file should be immediately openable in a browser and show working tab switching.

## Step 2: Verify in browser

Open `D:\code\ai\index.html` in browser. Confirm:
- 4 tabs visible at bottom
- Clicking each tab switches the view and highlights the active button
- No console errors

## Step 3: Commit

Commit with message: "feat: add project scaffold with Alpine.js shell and tab navigation"
