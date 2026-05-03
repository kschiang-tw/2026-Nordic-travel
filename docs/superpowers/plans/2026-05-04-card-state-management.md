# 景點卡片狀態管理 — 實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-card state management (✅ 完成 / 📅 調整 / 🗑️ 放棄) with long-press popover, two-step adjust modal, localStorage persistence, and JSON export/import backup to the single-file static travel app.

**Architecture:** All new code added inline to `travel-app.html` — CSS appended to existing `<style>` block, JS as a single new IIFE appended to existing `<script>` block at file end. Card identity is derived at runtime from `data-card-id = ${dayId}-${hash(cardName)}` and injected by JS on load. State persists to `localStorage["hygge2026-card-states"]` as `{ version: 1, updatedAt, cards: { cardId: { state, moved? } } }`.

**Tech Stack:** Vanilla JS (ES5/ES6 — must work in mobile Safari without bundling), inline HTML/CSS, no dependencies, no build step. Testing is manual browser verification (no Jest/Vitest in project).

**Spec:** [docs/superpowers/specs/2026-05-04-card-state-management-design.md](../specs/2026-05-04-card-state-management-design.md)

---

## File Structure

All changes go in **one file**: `travel-app.html`.

| Section | Lines (current) | Change |
|---------|----------------|--------|
| `<style>` block | ~14–197 | Append new CSS at end (state classes, popover, modal, badge, settings button) |
| Body — DOM templates | end of body, before `<script>` | Insert hidden popover, adjust modal, settings modal markup |
| Body — bottom nav | (search for `id="tabs"` or nav bar) | Add ⚙️ settings button |
| `<script>` block | ~1050–1179 | Append new IIFE `cardStateManager()` after the existing IIFEs |

The new IIFE has these internal sections (commented headers, all in one closure):

```
// === Storage ===            load(), save()
// === Card identity ===      hash(), generateCardId(), indexAllCards()
// === Apply state ===        applyMoved(state), applyVisualState(state)
// === Popover ===            attachLongPress(), showPopover(), hidePopover()
// === Adjust modal ===       openAdjustModal(), renderDayList(), renderInsertPoints(), performMove()
// === Settings + backup ===  openSettings(), exportBackup(), importBackup()
// === Init ===               main entry point
```

---

## Task 1: Storage layer + card identification

**Goal:** Every `.cd` gets a stable `data-card-id` on page load. Storage helpers are usable from console.

**Files:**
- Modify: `travel-app.html` (append IIFE skeleton at end of `<script>`, before `</script>`)

- [ ] **Step 1: Locate insertion point**

Find the closing `})();` of the last existing IIFE in `<script>` (the `loadDayWeather` one at end of file). Insert new code immediately after it, before `</script>`.

- [ ] **Step 2: Add IIFE skeleton with storage + identity**

Append this code:

```js
// ── 卡片狀態管理 ──
(function cardStateManager() {
  var STORAGE_KEY = 'hygge2026-card-states';
  var SCHEMA_VERSION = 1;

  // === Storage ===
  function load() {
    try {
      var raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return { version: SCHEMA_VERSION, updatedAt: null, cards: {} };
      var data = JSON.parse(raw);
      if (!data || data.version !== SCHEMA_VERSION || !data.cards) {
        return { version: SCHEMA_VERSION, updatedAt: null, cards: {} };
      }
      return data;
    } catch (e) {
      console.warn('[cardState] load failed, resetting', e);
      return { version: SCHEMA_VERSION, updatedAt: null, cards: {} };
    }
  }

  function save(state) {
    state.version = SCHEMA_VERSION;
    state.updatedAt = new Date().toISOString();
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
    } catch (e) {
      console.warn('[cardState] save failed', e);
      alert('儲存失敗，可能是瀏覽器空間不足');
    }
  }

  // === Card identity ===
  // cyrb53 hash → base36, deterministic across sessions
  function hash(str) {
    var h1 = 0xdeadbeef, h2 = 0x41c6ce57;
    for (var i = 0; i < str.length; i++) {
      var ch = str.charCodeAt(i);
      h1 = Math.imul(h1 ^ ch, 2654435761);
      h2 = Math.imul(h2 ^ ch, 1597334677);
    }
    h1 = Math.imul(h1 ^ (h1 >>> 16), 2246822507);
    h1 ^= Math.imul(h2 ^ (h2 >>> 13), 3266489909);
    h2 = Math.imul(h2 ^ (h2 >>> 16), 2246822507);
    h2 ^= Math.imul(h1 ^ (h1 >>> 13), 3266489909);
    var n = 4294967296 * (2097151 & h2) + (h1 >>> 0);
    return n.toString(36);
  }

  function generateCardId(dayId, cardName) {
    return dayId + '-' + hash(cardName.trim());
  }

  function indexAllCards() {
    var cards = document.querySelectorAll('.pg .tli > .cd');
    cards.forEach(function(card) {
      var pg = card.closest('.pg');
      if (!pg || !pg.id || pg.id.indexOf('p-') !== 0) return;
      var dayId = pg.id.slice(2); // 'p-d3' -> 'd3'
      var cnEl = card.querySelector('.cn');
      if (!cnEl) return;
      card.setAttribute('data-card-id', generateCardId(dayId, cnEl.textContent));
      card.setAttribute('data-origin-day', dayId);
    });
  }

  // === Init ===
  function init() {
    indexAllCards();
    window.__cardState = { load: load, save: save, hash: hash };
  }

  init();
})();
```

- [ ] **Step 3: Browser verification**

1. Open `travel-app.html` in browser (or `index.html` which redirects)
2. Open DevTools Console
3. Run: `document.querySelectorAll('.cd[data-card-id]').length`
   - Expected: ≥ 60 (number of timeline cards across all days)
4. Run: `document.querySelector('#p-d3 .cd').dataset.cardId`
   - Expected: a string like `d3-xxxxx`
5. Run: `window.__cardState.load()`
   - Expected: `{ version: 1, updatedAt: null, cards: {} }`
6. Run: `window.__cardState.save({ cards: { test: { state: 'done' } } })` then `window.__cardState.load()`
   - Expected: returned object has `cards.test.state === 'done'` and `updatedAt` is now an ISO string
7. Cleanup: `localStorage.removeItem('hygge2026-card-states')`

- [ ] **Step 4: Commit**

```bash
git add travel-app.html
git commit -m "Add card state storage layer and identity indexing"
```

---

## Task 2: Visual state styling (done / skip)

**Goal:** Loading the page applies saved `state: 'done' | 'skip'` to cards visually.

**Files:**
- Modify: `travel-app.html` (`<style>` block + new function in IIFE)

- [ ] **Step 1: Add CSS**

Append inside `<style>` (after the `.day-wx-na` rule near line 197, just before `</style>`):

```css
/* ── CARD STATE ── */
.cd.is-done { opacity:.55; border-left:2.5px solid #5cb85c !important; background:rgba(80,180,80,.06); }
.cd.is-skip { opacity:.45; border-left:2.5px solid #d9534f !important; background:rgba(180,80,80,.06); }
.cd.is-skip .cn { text-decoration:line-through; }
.state-badge { font-size:10px; padding:2px 7px; border-radius:8px; font-weight:600; margin-left:auto; flex-shrink:0; }
.state-badge.s-done { background:rgba(80,180,80,.2); color:#5cb85c; border:1px solid rgba(80,180,80,.35); }
.state-badge.s-skip { background:rgba(180,80,80,.2); color:#d9534f; border:1px solid rgba(180,80,80,.35); }
```

- [ ] **Step 2: Add `applyVisualState` function**

Inside the IIFE, add this function before `init()`:

```js
  // === Apply state ===
  function applyVisualState(state) {
    // Clear all existing visual state first (idempotent)
    document.querySelectorAll('.cd.is-done, .cd.is-skip').forEach(function(c) {
      c.classList.remove('is-done', 'is-skip');
    });
    document.querySelectorAll('.state-badge').forEach(function(b) { b.remove(); });

    Object.keys(state.cards).forEach(function(cardId) {
      var entry = state.cards[cardId];
      if (!entry || !entry.state) return;
      var card = document.querySelector('.cd[data-card-id="' + cardId + '"]');
      if (!card) return;
      card.classList.add(entry.state === 'done' ? 'is-done' : 'is-skip');
      var ch = card.querySelector('.ch');
      if (!ch) return;
      var badge = document.createElement('span');
      badge.className = 'state-badge s-' + entry.state;
      badge.textContent = entry.state === 'done' ? '✓ 完成' : '✕ 放棄';
      ch.appendChild(badge);
    });
  }
```

- [ ] **Step 3: Wire into init**

Modify `init()`:

```js
  function init() {
    indexAllCards();
    var state = load();
    applyVisualState(state);
    window.__cardState = { load: load, save: save, hash: hash, applyVisualState: applyVisualState };
  }
```

- [ ] **Step 4: Browser verification**

1. Reload page, console:
   ```js
   var s = window.__cardState.load();
   var firstId = document.querySelector('#p-d3 .cd').dataset.cardId;
   s.cards[firstId] = { state: 'done' };
   window.__cardState.save(s);
   window.__cardState.applyVisualState(s);
   ```
   - Expected: first card on D3 fades and shows green ✓ 完成 badge
2. Repeat with `state: 'skip'` on another card → faded with red ✕ 放棄 + strikethrough on name
3. Reload page → state persists (badge & opacity reapply)
4. Cleanup: `localStorage.removeItem('hygge2026-card-states'); location.reload()`

- [ ] **Step 5: Commit**

```bash
git add travel-app.html
git commit -m "Render done/skip visual state for cards on load"
```

---

## Task 3: Long-press popover UI

**Goal:** Long-pressing a card (500ms) opens a 3-button popover. Clicking 完成 / 放棄 / 還原 updates state and visuals immediately.

**Files:**
- Modify: `travel-app.html` (CSS + new DOM + new functions in IIFE)

- [ ] **Step 1: Add popover CSS**

Append to `<style>`:

```css
/* ── POPOVER ── */
.card-popover {
  position:absolute; z-index:100;
  background:#252840; border:1px solid #3a3d55; border-radius:12px;
  padding:5px; display:flex; gap:4px;
  box-shadow:0 6px 20px rgba(0,0,0,.5);
  font-family:inherit;
}
.card-popover[hidden] { display:none; }
.cpb {
  display:flex; align-items:center; gap:4px;
  font-size:12px; padding:7px 10px; border-radius:8px;
  cursor:pointer; user-select:none; color:#e0e0e0;
  background:transparent; border:none;
}
.cpb:hover, .cpb:active { background:#3a3d55; }
.cpb.cpb-done { color:#7ed882; }
.cpb.cpb-adjust { color:#f0a030; }
.cpb.cpb-skip { color:#e57373; }
.cpb.cpb-restore { color:#aaa; }
.cd.is-pressing { transform:scale(.97); transition:transform .1s; }
```

- [ ] **Step 2: Add popover DOM**

Just before the closing `</body>` tag (search for `</body>`), insert:

```html
<div id="card-popover" class="card-popover" hidden></div>
```

- [ ] **Step 3: Add popover + long-press logic**

Inside the IIFE, add before `init()`:

```js
  // === Popover ===
  var popover = null;
  var activeCard = null;
  var pressTimer = null;
  var pressMoved = false;

  function getPopover() {
    if (!popover) popover = document.getElementById('card-popover');
    return popover;
  }

  function showPopover(card) {
    var pop = getPopover();
    var cardId = card.dataset.cardId;
    var state = load();
    var entry = state.cards[cardId] || {};
    var current = entry.state;

    var buttons = '';
    if (current === 'done' || current === 'skip') {
      buttons += '<button class="cpb cpb-restore" data-action="restore">↩️ 還原</button>';
      buttons += '<button class="cpb cpb-adjust" data-action="adjust">📅 調整</button>';
    } else {
      buttons += '<button class="cpb cpb-done" data-action="done">✅ 完成</button>';
      buttons += '<button class="cpb cpb-adjust" data-action="adjust">📅 調整</button>';
      buttons += '<button class="cpb cpb-skip" data-action="skip">🗑️ 放棄</button>';
    }
    pop.innerHTML = buttons;

    // Position above the card, right-aligned
    var rect = card.getBoundingClientRect();
    pop.hidden = false;
    var pw = pop.offsetWidth;
    pop.style.top = (window.scrollY + rect.top - pop.offsetHeight - 6) + 'px';
    pop.style.left = (window.scrollX + rect.right - pw) + 'px';
    activeCard = card;
  }

  function hidePopover() {
    if (popover) popover.hidden = true;
    activeCard = null;
  }

  function setCardState(card, newState) {
    var state = load();
    var cardId = card.dataset.cardId;
    if (!state.cards[cardId]) state.cards[cardId] = {};
    if (newState === null) {
      delete state.cards[cardId].state;
      if (Object.keys(state.cards[cardId]).length === 0) delete state.cards[cardId];
    } else {
      state.cards[cardId].state = newState;
    }
    save(state);
    applyVisualState(state);
  }

  function attachLongPress(card) {
    function start(e) {
      // Skip long-press on links and chips inside the card
      if (e.target.closest('a, .chip')) return;
      pressMoved = false;
      card.classList.add('is-pressing');
      pressTimer = setTimeout(function() {
        card.classList.remove('is-pressing');
        if (!pressMoved) showPopover(card);
      }, 500);
    }
    function cancel() {
      clearTimeout(pressTimer);
      card.classList.remove('is-pressing');
    }
    function move() {
      pressMoved = true;
      cancel();
    }
    card.addEventListener('touchstart', start, { passive: true });
    card.addEventListener('touchend', cancel);
    card.addEventListener('touchmove', move, { passive: true });
    card.addEventListener('mousedown', start);
    card.addEventListener('mouseup', cancel);
    card.addEventListener('mouseleave', cancel);
  }

  // Global click handler for popover buttons + outside-click dismiss
  function handlePopoverClick(e) {
    var pop = getPopover();
    if (pop.hidden) return;
    var btn = e.target.closest('.cpb');
    if (btn && activeCard) {
      var action = btn.dataset.action;
      var card = activeCard;
      hidePopover();
      if (action === 'done') setCardState(card, 'done');
      else if (action === 'skip') setCardState(card, 'skip');
      else if (action === 'restore') setCardState(card, null);
      else if (action === 'adjust') openAdjustModal(card.dataset.cardId);
      e.stopPropagation();
      return;
    }
    // Click outside popover and outside any card → close
    if (!e.target.closest('.card-popover') && !e.target.closest('.cd[data-card-id]')) {
      hidePopover();
    }
  }

  // Stub for now — implemented in Task 5/6
  function openAdjustModal(cardId) {
    console.log('[adjust] not yet implemented for', cardId);
  }
```

- [ ] **Step 4: Wire into init**

Update `init()`:

```js
  function init() {
    indexAllCards();
    var state = load();
    applyVisualState(state);

    document.querySelectorAll('.cd[data-card-id]').forEach(attachLongPress);
    document.addEventListener('click', handlePopoverClick, true);

    window.__cardState = { load: load, save: save, hash: hash, applyVisualState: applyVisualState };
  }
```

- [ ] **Step 5: Browser verification**

1. Reload page
2. Long-press (mouse hold 0.5s, or touch hold) on a card on D3 → popover appears with 3 buttons
3. Click "✅ 完成" → popover closes, card fades + green badge appears
4. Long-press the same card → popover now shows "↩️ 還原" + "📅 調整"
5. Click "↩️ 還原" → card returns to normal
6. Long-press again, click "🗑️ 放棄" → red badge + strikethrough
7. Click outside popover (e.g. background) → popover closes
8. Click a chip or link inside a card and hold → popover should NOT appear (chips/links are excluded)
9. Reload page → states persist
10. Test on a real mobile device or DevTools device emulator with touch
11. Cleanup: `localStorage.removeItem('hygge2026-card-states'); location.reload()`

- [ ] **Step 6: Commit**

```bash
git add travel-app.html
git commit -m "Add long-press popover for completing and skipping cards"
```

---

## Task 4: Apply moved state on load

**Goal:** Cards with `moved: { toDay, index }` in storage are physically relocated to the target day at the target index when the page loads.

**Files:**
- Modify: `travel-app.html` (add function + wire into init)

- [ ] **Step 1: Add `applyMoved` function**

Inside the IIFE, add before `applyVisualState`:

```js
  function applyMoved(state) {
    // Build a sorted list of moves (stable ordering by toDay then index)
    var moves = [];
    Object.keys(state.cards).forEach(function(cardId) {
      var entry = state.cards[cardId];
      if (entry && entry.moved) {
        moves.push({ cardId: cardId, toDay: entry.moved.toDay, index: entry.moved.index });
      }
    });
    // Apply in (toDay, index) order to preserve relative positions
    moves.sort(function(a, b) {
      if (a.toDay !== b.toDay) return a.toDay < b.toDay ? -1 : 1;
      return a.index - b.index;
    });

    moves.forEach(function(m) {
      var card = document.querySelector('.cd[data-card-id="' + m.cardId + '"]');
      if (!card) return;
      var tli = card.closest('.tli');
      if (!tli) return;
      var targetPg = document.getElementById('p-' + m.toDay);
      if (!targetPg) return;
      // Find the LAST .tl in the target day (avoids splitting between section labels)
      var tls = targetPg.querySelectorAll('.tl');
      var targetTl = tls[tls.length - 1];
      if (!targetTl) return;
      var existing = targetTl.querySelectorAll(':scope > .tli');
      var insertBefore = existing[m.index] || null;
      if (insertBefore) targetTl.insertBefore(tli, insertBefore);
      else targetTl.appendChild(tli);
    });
  }
```

- [ ] **Step 2: Wire into init (before applyVisualState)**

```js
  function init() {
    indexAllCards();
    var state = load();
    applyMoved(state);
    applyVisualState(state);

    document.querySelectorAll('.cd[data-card-id]').forEach(attachLongPress);
    document.addEventListener('click', handlePopoverClick, true);

    window.__cardState = { load: load, save: save, hash: hash, applyVisualState: applyVisualState, applyMoved: applyMoved };
  }
```

- [ ] **Step 3: Browser verification**

1. Reload page. In console:
   ```js
   var s = window.__cardState.load();
   var firstId = document.querySelector('#p-d3 .cd').dataset.cardId;
   s.cards[firstId] = { moved: { toDay: 'd5', index: 0 } };
   window.__cardState.save(s);
   location.reload();
   ```
2. Navigate to D5 (馬爾摩) — the moved D3 card should now be at the top of D5's last `.tl`
3. Navigate back to D3 — that card no longer appears
4. Cleanup: `localStorage.removeItem('hygge2026-card-states'); location.reload()` → card back on D3

- [ ] **Step 4: Commit**

```bash
git add travel-app.html
git commit -m "Apply card moves on page load"
```

---

## Task 5: Adjust modal — Step 1 (day picker)

**Goal:** Clicking "📅 調整" opens a full-screen modal showing all 16 days.

**Files:**
- Modify: `travel-app.html` (CSS + DOM + function)

- [ ] **Step 1: Add modal CSS**

Append to `<style>`:

```css
/* ── ADJUST MODAL ── */
.adjust-modal { position:fixed; inset:0; z-index:200; }
.adjust-modal[hidden] { display:none; }
.am-backdrop { position:absolute; inset:0; background:rgba(0,0,0,.55); }
.am-panel {
  position:absolute; left:0; right:0; bottom:0;
  background:var(--bg); border-top-left-radius:18px; border-top-right-radius:18px;
  max-height:85vh; display:flex; flex-direction:column;
  padding:14px 14px 24px;
  border-top:1px solid var(--bd);
}
.am-header { display:flex; align-items:center; gap:8px; margin-bottom:10px; }
.am-back { background:none; border:none; color:var(--a); font-size:14px; cursor:pointer; padding:4px 8px; }
.am-back[hidden] { display:none; }
.am-title { font-size:14px; font-weight:600; flex:1; }
.am-close { background:none; border:none; color:var(--tm); font-size:18px; cursor:pointer; padding:4px 8px; }
.am-body { overflow-y:auto; flex:1; }
.am-day {
  display:flex; align-items:center; gap:10px;
  padding:11px 12px; margin-bottom:6px;
  background:var(--sf); border:1px solid var(--bd); border-radius:11px;
  cursor:pointer;
}
.am-day:active { background:var(--sf2); }
.am-day-no { font-size:11px; font-weight:700; color:var(--tm); letter-spacing:1px; }
.am-day-info { flex:1; }
.am-day-city { font-size:13px; font-weight:500; }
.am-day-date { font-size:11px; color:var(--tm); margin-top:1px; }
.insert-point {
  height:30px; display:flex; align-items:center; justify-content:center;
  font-size:11px; color:var(--a); cursor:pointer;
  border:1px dashed rgba(95,176,207,.3); border-radius:8px;
  margin:4px 0; background:rgba(95,176,207,.04);
}
.insert-point:hover, .insert-point:active { background:rgba(95,176,207,.15); border-style:solid; }
.am-existing {
  padding:8px 11px; background:var(--sf); border:1px solid var(--bd); border-radius:10px;
  font-size:12px; color:var(--tm);
  display:flex; align-items:center; gap:6px;
}
.am-existing.is-target { outline:2px solid var(--a2); }
```

- [ ] **Step 2: Add modal DOM**

Just before `</body>`, after the existing `<div id="card-popover">`:

```html
<div id="adjust-modal" class="adjust-modal" hidden>
  <div class="am-backdrop" data-am-close></div>
  <div class="am-panel">
    <div class="am-header">
      <button class="am-back" data-am-back hidden>← 返回</button>
      <div class="am-title">移到哪一天？</div>
      <button class="am-close" data-am-close>✕</button>
    </div>
    <div class="am-body"></div>
  </div>
</div>
```

- [ ] **Step 3: Add adjust modal logic**

Replace the stub `openAdjustModal` from Task 3 with this full implementation, plus add helpers. Place inside the IIFE:

```js
  // === Adjust modal ===
  // Day metadata: must match #p-dN ids in HTML
  var DAY_INFO = [
    { id: 'd1',  city: '曼谷',       date: '5/23 週六' },
    { id: 'd2',  city: '哥本哈根',    date: '5/24 週日' },
    { id: 'd3',  city: '哥本哈根',    date: '5/25 週一' },
    { id: 'd4',  city: '哥本哈根',    date: '5/26 週二' },
    { id: 'd5',  city: '馬爾摩',      date: '5/27 週三' },
    { id: 'd6',  city: '哥本哈根',    date: '5/28 週四' },
    { id: 'd7',  city: '奧登塞',      date: '5/29 週五' },
    { id: 'd8',  city: '奧登塞',      date: '5/30 週六' },
    { id: 'd9',  city: '奧胡斯',      date: '5/31 週日' },
    { id: 'd10', city: '比隆',       date: '6/1 週一' },
    { id: 'd11', city: '斯德哥爾摩',  date: '6/2 週二' },
    { id: 'd12', city: '斯德哥爾摩',  date: '6/3 週三' },
    { id: 'd13', city: '斯德哥爾摩',  date: '6/4 週四' },
    { id: 'd14', city: '斯德哥爾摩',  date: '6/5 週五' },
    { id: 'd15', city: '斯德哥爾摩',  date: '6/6 週六' },
    { id: 'd16', city: '曼谷',       date: '6/7 週日' }
  ];

  var modal = null;
  var modalCardId = null;
  var modalStep = 1;
  var modalTargetDay = null;

  function getModal() {
    if (!modal) modal = document.getElementById('adjust-modal');
    return modal;
  }

  function openAdjustModal(cardId) {
    modalCardId = cardId;
    modalStep = 1;
    modalTargetDay = null;
    renderDayList();
    getModal().hidden = false;
  }

  function closeAdjustModal() {
    getModal().hidden = true;
    modalCardId = null;
  }

  function renderDayList() {
    var m = getModal();
    m.querySelector('.am-title').textContent = '移到哪一天？';
    m.querySelector('[data-am-back]').hidden = true;
    var body = m.querySelector('.am-body');
    body.innerHTML = '';
    DAY_INFO.forEach(function(d, i) {
      var row = document.createElement('div');
      row.className = 'am-day';
      row.innerHTML =
        '<div class="am-day-no">DAY ' + (i+1 < 10 ? '0'+(i+1) : (i+1)) + '</div>' +
        '<div class="am-day-info">' +
          '<div class="am-day-city">' + d.city + '</div>' +
          '<div class="am-day-date">' + d.date + '</div>' +
        '</div>';
      row.addEventListener('click', function() {
        modalTargetDay = d.id;
        modalStep = 2;
        renderInsertPoints(d.id);
      });
      body.appendChild(row);
    });
  }

  // Stub for next task
  function renderInsertPoints(targetDayId) {
    var body = getModal().querySelector('.am-body');
    body.innerHTML = '<div style="padding:20px; color:var(--tm); text-align:center">第 2 步施工中（Task 6）</div>';
    getModal().querySelector('.am-title').textContent = '選擇插入位置';
    getModal().querySelector('[data-am-back]').hidden = false;
  }

  // Modal close handlers (delegated)
  function handleModalClick(e) {
    if (!e.target.closest('#adjust-modal')) return;
    if (e.target.matches('[data-am-close]')) {
      closeAdjustModal();
    } else if (e.target.matches('[data-am-back]')) {
      modalStep = 1;
      modalTargetDay = null;
      renderDayList();
    }
  }
```

- [ ] **Step 4: Wire into init**

Update `init()` to add modal click handler:

```js
  function init() {
    indexAllCards();
    var state = load();
    applyMoved(state);
    applyVisualState(state);

    document.querySelectorAll('.cd[data-card-id]').forEach(attachLongPress);
    document.addEventListener('click', handlePopoverClick, true);
    document.addEventListener('click', handleModalClick);

    window.__cardState = { load: load, save: save, hash: hash, applyVisualState: applyVisualState, applyMoved: applyMoved, openAdjustModal: openAdjustModal };
  }
```

- [ ] **Step 5: Browser verification**

1. Reload page
2. Long-press any card on D2 → popover opens
3. Click "📅 調整" → bottom-sheet modal slides up with 16 day rows
4. Click row "DAY 05 馬爾摩" → modal switches to step 2 with placeholder text and "← 返回" button
5. Click "← 返回" → back to day list
6. Click ✕ → modal closes
7. Click backdrop (dark area) → modal closes
8. Reopen → all 16 days listed in order
9. Cleanup: nothing to reset (no state was saved)

- [ ] **Step 6: Commit**

```bash
git add travel-app.html
git commit -m "Add adjust modal with day picker (step 1)"
```

---

## Task 6: Adjust modal — Step 2 (insert points + perform move)

**Goal:** Selecting a day shows the day's existing items with "↓ 插入此處" lines between them. Clicking an insert point moves the card and updates state.

**Files:**
- Modify: `travel-app.html` (replace `renderInsertPoints` stub + add `performMove`)

- [ ] **Step 1: Replace `renderInsertPoints` and add `performMove`**

Replace the stub `renderInsertPoints` from Task 5 with:

```js
  function renderInsertPoints(targetDayId) {
    var m = getModal();
    m.querySelector('.am-title').textContent = '選擇插入位置';
    m.querySelector('[data-am-back]').hidden = false;
    var body = m.querySelector('.am-body');
    body.innerHTML = '';

    var pg = document.getElementById('p-' + targetDayId);
    if (!pg) {
      body.innerHTML = '<div style="padding:20px; color:var(--tm); text-align:center">找不到該天</div>';
      return;
    }
    // Use the LAST .tl in the day (matches applyMoved logic)
    var tls = pg.querySelectorAll('.tl');
    var tl = tls[tls.length - 1];
    var items = tl ? tl.querySelectorAll(':scope > .tli') : [];

    function makeInsertPoint(index) {
      var ip = document.createElement('div');
      ip.className = 'insert-point';
      ip.textContent = '↓ 插入此處';
      ip.addEventListener('click', function() {
        performMove(modalCardId, targetDayId, index);
      });
      return ip;
    }

    function makeExistingItem(tli) {
      var card = tli.querySelector(':scope > .cd');
      var name = card && card.querySelector('.cn') ? card.querySelector('.cn').textContent : '(行程)';
      var icon = card && card.querySelector('.ci') ? card.querySelector('.ci').textContent : '•';
      var time = card && card.querySelector('.ctm') ? card.querySelector('.ctm').textContent : '';
      var div = document.createElement('div');
      div.className = 'am-existing';
      var isTarget = card && card.dataset.cardId === modalCardId;
      if (isTarget) div.classList.add('is-target');
      div.innerHTML = '<span>' + icon + '</span><span style="flex:1">' + name + (isTarget ? ' <em style="color:var(--a2);font-style:normal;font-size:10px">（這張）</em>' : '') + '</span>' + (time ? '<span style="color:var(--a2);font-weight:600">' + time + '</span>' : '');
      return div;
    }

    body.appendChild(makeInsertPoint(0));
    items.forEach(function(tli, i) {
      body.appendChild(makeExistingItem(tli));
      body.appendChild(makeInsertPoint(i + 1));
    });
    if (items.length === 0) {
      // Empty day — only one insert point already added is enough
    }
  }

  function performMove(cardId, targetDayId, index) {
    var state = load();
    if (!state.cards[cardId]) state.cards[cardId] = {};
    state.cards[cardId].moved = { toDay: targetDayId, index: index };
    save(state);

    // Re-apply on the live DOM without full reload
    // Simplest correct approach: reload to re-trigger applyMoved + applyVisualState
    closeAdjustModal();
    location.reload();
  }
```

> **Note on reload:** We use `location.reload()` after move because re-applying the move live requires undoing the previous position too, which is fiddly. Reload is correct and the page is fast.

- [ ] **Step 2: Browser verification**

1. Reload page
2. Long-press first card on D3 → 調整 → DAY 05
3. Step 2 shows D5's existing items with "↓ 插入此處" between each
4. Click the insert point at the very top → page reloads, navigate to D5 → moved card is at top
5. Navigate to D3 → that card is gone
6. Long-press the moved card on D5 → 調整 → DAY 03 → click insert point at bottom → reload → card is at bottom of D3 (returns to original day, last position)
7. Test moving same-day reorder: long-press card on D3 → 調整 → DAY 03 → pick a different position → reload → card moved within D3
8. The card being moved is highlighted with blue outline + 「（這張）」 in the step 2 list
9. Cleanup: `localStorage.removeItem('hygge2026-card-states'); location.reload()`

- [ ] **Step 3: Commit**

```bash
git add travel-app.html
git commit -m "Implement insert-point picker and perform card move"
```

---

## Task 7: Settings button + Export/Import backup

**Goal:** A ⚙️ button in the header area opens a settings modal with 「📤 匯出備份」 and 「📥 匯入備份」 buttons.

**Files:**
- Modify: `travel-app.html` (CSS + DOM + functions)

- [ ] **Step 1: Find the header location**

Search for the existing `.hdr` block (around line 194). The settings button will go inside `.hdr-top`'s right side, next to/replacing the flags area, or as a small absolute-positioned button. Inspect the existing markup:

```bash
grep -n 'class="hdr-top"\|class="flags"' travel-app.html
```

- [ ] **Step 2: Add settings button CSS**

Append to `<style>`:

```css
/* ── SETTINGS ── */
.settings-btn {
  background:var(--sf); border:1px solid var(--bd); border-radius:50%;
  width:32px; height:32px; cursor:pointer;
  display:grid; place-items:center;
  font-size:15px; color:var(--tm);
  margin-left:6px;
}
.settings-btn:active { background:var(--sf2); }

.settings-modal { position:fixed; inset:0; z-index:200; }
.settings-modal[hidden] { display:none; }
.sm-backdrop { position:absolute; inset:0; background:rgba(0,0,0,.55); }
.sm-panel {
  position:absolute; left:14px; right:14px; top:50%; transform:translateY(-50%);
  background:var(--bg); border:1px solid var(--bd); border-radius:14px;
  padding:18px;
}
.sm-title { font-size:15px; font-weight:600; margin-bottom:12px; }
.sm-row { margin-bottom:10px; }
.sm-btn {
  width:100%; padding:11px; border-radius:10px;
  background:var(--sf); border:1px solid var(--bd); color:var(--tx);
  font-size:13px; font-weight:500; cursor:pointer;
  display:flex; align-items:center; gap:8px; justify-content:center;
}
.sm-btn:active { background:var(--sf2); }
.sm-note { font-size:11px; color:var(--tm); margin-top:6px; line-height:1.5; }
.sm-close {
  background:none; border:none; color:var(--tm); font-size:14px; cursor:pointer;
  display:block; margin:8px auto 0;
}
```

- [ ] **Step 3: Add settings button to header**

Inside the existing `.hdr-top` block, add the button. Find the existing structure and add the button just before the closing `</div>` of `.hdr-top`. Use Edit:

Locate this line (or similar — search for `class="flags"`):

```html
      <div class="flags">
```

If `.flags` is wrapped within `.hdr-top`, add the settings button immediately after `.flags` closes, still inside `.hdr-top`:

```html
        <button class="settings-btn" id="btn-settings" aria-label="設定">⚙️</button>
```

> If the structure is different, place the `<button>` inside `.hdr-top` after the `flags` div but before `.hdr-top` closes.

- [ ] **Step 4: Add settings modal DOM**

Just before `</body>` (after `#adjust-modal`):

```html
<div id="settings-modal" class="settings-modal" hidden>
  <div class="sm-backdrop" data-sm-close></div>
  <div class="sm-panel">
    <div class="sm-title">設定 / 備份</div>
    <div class="sm-row">
      <button class="sm-btn" id="btn-export">📤 匯出備份</button>
      <div class="sm-note">下載目前所有卡片狀態為 JSON 檔。可儲存到 iCloud / Google Drive 作為備份。</div>
    </div>
    <div class="sm-row">
      <button class="sm-btn" id="btn-import">📥 匯入備份</button>
      <input type="file" id="import-file" accept=".json,application/json" hidden>
      <div class="sm-note">從 JSON 備份還原狀態（會覆蓋目前資料）。</div>
    </div>
    <button class="sm-close" data-sm-close>關閉</button>
  </div>
</div>
```

- [ ] **Step 5: Add export/import logic**

Inside the IIFE, add before `init()`:

```js
  // === Settings + backup ===
  function exportBackup() {
    var state = load();
    var json = JSON.stringify(state, null, 2);
    var blob = new Blob([json], { type: 'application/json' });
    var url = URL.createObjectURL(blob);
    var a = document.createElement('a');
    var date = new Date().toISOString().slice(0, 10);
    a.href = url;
    a.download = 'hygge2026-backup-' + date + '.json';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    setTimeout(function() { URL.revokeObjectURL(url); }, 1000);
  }

  function importBackup(file) {
    var reader = new FileReader();
    reader.onload = function() {
      try {
        var data = JSON.parse(reader.result);
        if (!data || data.version !== SCHEMA_VERSION || typeof data.cards !== 'object') {
          alert('檔案格式不正確，無法匯入');
          return;
        }
        if (!confirm('將覆蓋目前所有狀態，確定要匯入？')) return;
        save(data);
        location.reload();
      } catch (e) {
        alert('讀取失敗：' + e.message);
      }
    };
    reader.onerror = function() { alert('讀取檔案失敗'); };
    reader.readAsText(file);
  }

  function openSettings() {
    document.getElementById('settings-modal').hidden = false;
  }
  function closeSettings() {
    document.getElementById('settings-modal').hidden = true;
  }

  function handleSettingsClick(e) {
    if (e.target.id === 'btn-settings') {
      openSettings();
    } else if (e.target.matches('[data-sm-close]')) {
      closeSettings();
    } else if (e.target.id === 'btn-export') {
      exportBackup();
      closeSettings();
    } else if (e.target.id === 'btn-import') {
      document.getElementById('import-file').click();
    }
  }
```

- [ ] **Step 6: Wire into init**

Update `init()`:

```js
  function init() {
    indexAllCards();
    var state = load();
    applyMoved(state);
    applyVisualState(state);

    document.querySelectorAll('.cd[data-card-id]').forEach(attachLongPress);
    document.addEventListener('click', handlePopoverClick, true);
    document.addEventListener('click', handleModalClick);
    document.addEventListener('click', handleSettingsClick);

    var importFile = document.getElementById('import-file');
    if (importFile) {
      importFile.addEventListener('change', function(e) {
        var f = e.target.files && e.target.files[0];
        if (f) importBackup(f);
        e.target.value = ''; // allow re-importing same file
      });
    }

    window.__cardState = {
      load: load, save: save, hash: hash,
      applyVisualState: applyVisualState, applyMoved: applyMoved,
      openAdjustModal: openAdjustModal, exportBackup: exportBackup
    };
  }
```

- [ ] **Step 7: Browser verification**

1. Reload page → ⚙️ button visible in header
2. Long-press a card on D3 → mark "✅ 完成"
3. Long-press another card on D5 → "📅 調整" → move to D7
4. Click ⚙️ → settings modal opens
5. Click "📤 匯出備份" → browser downloads `hygge2026-backup-YYYY-MM-DD.json`
6. Open the file in a text editor — it should be valid JSON with `version: 1`, `updatedAt`, and `cards` containing the two entries
7. Cleanup current state: `localStorage.removeItem('hygge2026-card-states'); location.reload()` → cards back to defaults
8. Click ⚙️ → "📥 匯入備份" → file picker opens → select the JSON downloaded in step 5 → confirm dialog → page reloads → previous states restored (D3 card completed, D5 card moved to D7)
9. Test invalid file: create a `.json` file with `{"foo":"bar"}` and try to import → alert "檔案格式不正確"
10. Test cancel of confirm dialog: import valid file → click cancel → state unchanged
11. Cleanup: `localStorage.removeItem('hygge2026-card-states'); location.reload()`

- [ ] **Step 8: Commit**

```bash
git add travel-app.html
git commit -m "Add settings panel with JSON export/import backup"
```

---

## Task 8: Polish + edge cases

**Goal:** Tighten interactions: prevent long-press on header/non-card cards, suppress text selection during long-press, ensure popover scrolls with page.

**Files:**
- Modify: `travel-app.html`

- [ ] **Step 1: Suppress text selection during long-press**

In CSS, modify `.cd` (existing line 101) — actually, instead add this rule to avoid touching existing styles:

```css
.cd[data-card-id].is-pressing { user-select:none; -webkit-user-select:none; }
```

- [ ] **Step 2: Reposition popover on scroll**

Add to the IIFE — modify `showPopover` to also listen for scroll:

```js
  var popoverScrollHandler = null;
  function showPopover(card) {
    // ... existing code ...
    // After positioning, attach scroll handler:
    popoverScrollHandler = function() { hidePopover(); };
    document.getElementById('scroll').addEventListener('scroll', popoverScrollHandler, { once: true });
  }

  function hidePopover() {
    if (popover) popover.hidden = true;
    activeCard = null;
    if (popoverScrollHandler) {
      var sc = document.getElementById('scroll');
      if (sc) sc.removeEventListener('scroll', popoverScrollHandler);
      popoverScrollHandler = null;
    }
  }
```

> **Why hide instead of reposition:** The popover is positioned with absolute coords relative to the card; if the user scrolls, easier to dismiss than to recompute on every scroll frame.

- [ ] **Step 3: Verify cards in non-timeline blocks aren't affected**

The existing selectors (`.pg .tli > .cd`) already exclude cards in `.gs`, `.nc`, `.rc`, etc. Verify by reloading and checking:

```js
// Should be 0 — no card in guide pages or static sections
document.querySelectorAll('.gi[data-card-id], .gs[data-card-id]').length
// Should be > 0 — only timeline cards
document.querySelectorAll('.tli > .cd[data-card-id]').length
```

- [ ] **Step 4: Browser verification — full flow**

1. Reload page
2. Mark 3 cards on different days as done/skip/done
3. Move 1 card from D3 to D5
4. Move 1 card within D2 (reorder)
5. Reload → all 5 changes persist
6. Export backup
7. Clear localStorage, reload → defaults restored
8. Import backup → all 5 changes restored
9. Test on mobile (real device or DevTools touch emulation):
   - Long-press works
   - Popover doesn't trigger from chip clicks
   - Adjust modal scrollable on small screens
10. Test settings ⚙️ button visible in header on both desktop and mobile

- [ ] **Step 5: Commit**

```bash
git add travel-app.html
git commit -m "Polish: suppress selection on press, dismiss popover on scroll"
```

---

## Final Verification Checklist

After all tasks complete, run through this end-to-end:

- [ ] Long-press any card → popover with 3 buttons (or 2 if already done/skip)
- [ ] Click 完成 → green badge + faded card; persists across reload
- [ ] Click 放棄 → red badge + strikethrough; persists
- [ ] Click 還原 (on completed/skipped card) → returns to default
- [ ] Click 調整 → modal step 1 → pick day → step 2 → pick insert point → card moves; persists
- [ ] Move card from D3 to D5, then back to D3 → ends up where you placed it
- [ ] Same-day reorder: pick the same day in step 1, choose different position in step 2 → works
- [ ] Settings ⚙️ visible in header
- [ ] Export downloads JSON; file is human-readable and parseable
- [ ] Import valid JSON restores all state after reload
- [ ] Import invalid JSON shows alert, no state change
- [ ] Cancel of import confirm dialog leaves state unchanged
- [ ] Tapping a chip or link inside a card does NOT trigger long-press
- [ ] Existing app features still work: tabs, day navigation, weather load, guide pages
- [ ] No JavaScript errors in console
