# Study Mode + Theme Toggle Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a Study Mode toggle that hides all game mechanics and auto-presents flashcards, plus a light/dark theme toggle for visual comfort.

**Architecture:** Single-file modification to `index.html`. CSS classes on `<body>` control visibility (`study-mode`, `light-theme`). JS conditionals gate game-only behavior. State fields `studyMode` and `lightTheme` persist via existing localStorage save/load.

**Tech Stack:** Vanilla HTML/CSS/JS (single-file app)

---

### Task 1: Add state fields and URL parameter handling

**Files:**
- Modify: `index.html:3288-3321` (state object)
- Modify: `index.html:4730-4744` (initialization)

**Step 1: Add state fields**

In the state object (line 3288), add two new fields at the end (before the closing `}`):

```javascript
studyMode: false,
lightTheme: false,
```

**Step 2: Add URL parameter check and body class application at init**

After `tryLoad()` (line 4730), add URL parameter handling and a function to apply mode classes:

```javascript
// Study mode: URL param overrides saved state
const urlParams = new URLSearchParams(window.location.search);
if (urlParams.get('mode') === 'study') state.studyMode = true;

function applyMode() {
  document.body.classList.toggle('study-mode', state.studyMode);
  document.body.classList.toggle('light-theme', state.lightTheme);
  document.querySelector('header h1').textContent = state.studyMode ? 'PEL Flashcards' : 'PEL Clicker';
  // Update toggle states
  const modeToggle = document.getElementById('modeToggle');
  const themeToggle = document.getElementById('themeToggle');
  if (modeToggle) modeToggle.checked = state.studyMode;
  if (themeToggle) themeToggle.checked = state.lightTheme;
}
```

**Step 3: Call applyMode() during init, gate game-only init code**

Replace the init block (lines 4730-4744) with:

```javascript
tryLoad();
if (!state.firstPlayDate) state.firstPlayDate = new Date().toISOString().slice(0, 10);
if (!state.currentPrestigeStart) state.currentPrestigeStart = Date.now();

// URL param override
const urlParams = new URLSearchParams(window.location.search);
if (urlParams.get('mode') === 'study') state.studyMode = true;

geminiToggle.checked = state.useGeminiDeck;
applyMode();
updateStudyStats();

if (!state.studyMode) {
  renderAchievements();
  updateTugBar();
  updateUI();
  checkDailyBonus();
  setInterval(gameTick, 50);
  setInterval(spawnParticle, 2500);
  for (let i = 0; i < 4; i++) setTimeout(spawnParticle, i * 500);
} else {
  // Auto-present first card in study mode
  setTimeout(() => showQuiz(), 300);
}

setInterval(trySave, 10000);
```

**Step 4: Verify manually**

- Open `index.html` — should load as normal game mode
- Open `index.html?mode=study` — title should say "PEL Flashcards", game elements still visible (CSS not added yet)
- Check console for errors

**Step 5: Commit**

```
feat: add studyMode/lightTheme state fields and URL param handling
```

---

### Task 2: Add toggle controls to the HTML

**Files:**
- Modify: `index.html:2018-2028` (study-tracker area)

**Step 1: Add mode and theme toggle HTML**

Insert a new toggle row between the header (line 2016) and the study-tracker (line 2018):

```html
<div class="mode-toggles">
  <label class="mode-toggle-label">
    <span class="toggle-switch"><input type="checkbox" id="modeToggle"><span class="toggle-slider"></span></span>
    Study Mode
  </label>
  <label class="mode-toggle-label">
    <span class="toggle-switch"><input type="checkbox" id="themeToggle"><span class="toggle-slider"></span></span>
    <span id="themeIcon">☀️</span> Light
  </label>
</div>
```

**Step 2: Wire up toggle event listeners**

Add after the `applyMode()` function definition (from Task 1):

```javascript
document.getElementById('modeToggle').addEventListener('change', (e) => {
  state.studyMode = e.target.checked;
  applyMode();
  trySave();
  if (state.studyMode) {
    // Entering study mode: show a card
    setTimeout(() => showQuiz(), 300);
  } else {
    // Entering game mode: init game systems
    renderAchievements();
    updateTugBar();
    updateUI();
  }
});

document.getElementById('themeToggle').addEventListener('change', (e) => {
  state.lightTheme = e.target.checked;
  applyMode();
  trySave();
});
```

**Step 3: Verify manually**

- Toggles visible in both modes
- Clicking study mode toggle changes `state.studyMode` (check via console)
- Clicking theme toggle changes `state.lightTheme`
- Reloading preserves toggle states

**Step 4: Commit**

```
feat: add Study Mode and Light Theme toggle controls
```

---

### Task 3: Add CSS to hide game elements in study mode

**Files:**
- Modify: `index.html` (CSS section, after existing styles ~line 1998)

**Step 1: Add study-mode CSS rules**

Add before the closing `</style>` tag:

```css
/* === STUDY MODE === */
body.study-mode .click-section,
body.study-mode .shop-section,
body.study-mode .achievements-section,
body.study-mode #prestigeSection,
body.study-mode #tugWrapper,
body.study-mode .tug-status-bar,
body.study-mode #particles,
body.study-mode .year-badge,
body.study-mode #statsBtn,
body.study-mode .per-second,
body.study-mode .prestige-mult,
body.study-mode .score-label {
  display: none !important;
}

body.study-mode .game-container {
  display: none !important;
}

body.study-mode {
  padding-bottom: 20px;
}

/* Mode toggles row */
.mode-toggles {
  display: flex;
  justify-content: center;
  gap: 24px;
  margin: 8px 0 4px;
}

.mode-toggle-label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.82em;
  color: #8da9c4;
  cursor: pointer;
  user-select: none;
}
```

**Step 2: Verify manually**

- Toggle study mode on: game elements disappear, only header + stats bar + study tracker + review btn + reset btn remain
- Toggle study mode off: everything reappears
- URL `?mode=study` hides game elements on load

**Step 3: Commit**

```
feat: CSS rules to hide game elements in study mode
```

---

### Task 4: Modify showQuiz() and handleAnswer() for study mode

**Files:**
- Modify: `index.html:3500-3536` (showQuiz)
- Modify: `index.html:3594-3757` (handleAnswer)
- Modify: `index.html:3781-3791` (quizClose handler)

**Step 1: Make showQuiz() work without upgrade context**

Modify `showQuiz` to accept optional parameters and handle study mode:

```javascript
function showQuiz(upgradeName, cost) {
  const qIndex = pickQuestion();
  const fc = getActiveFlashcards()[qIndex];

  const isGemini = qIndex >= flashcards.length;
  const setNum = isGemini ? qIndex - flashcards.length + 1 : qIndex + 1;
  const setTotal = isGemini ? geminiFlashcards.length : flashcards.length;
  quizSource.innerHTML = isGemini
    ? `<span class="src-ai">Generated ISBE Flashcard</span> ${setNum}/${setTotal}`
    : `PEL Official Flashcard ${setNum}/${setTotal}`;

  if (state.studyMode) {
    quizLabel.textContent = 'PEL Question';
    quizCost.textContent = '';
  } else {
    quizLabel.textContent = `Answer to unlock: ${upgradeName}`;
    quizCost.textContent = fmt(cost) + ' pts';
  }

  quizQuestion.textContent = fc.q;
  quizOptions.innerHTML = '';
  quizAnswered = false;
  quizResult.className = 'quiz-result';
  quizResult.style.display = 'none';
  quizClose.className = 'quiz-close';

  const indices = fc.options.map((_, i) => i);
  for (let i = indices.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [indices[i], indices[j]] = [indices[j], indices[i]];
  }

  indices.forEach((origIdx, displayIdx) => {
    const btn = document.createElement('div');
    btn.className = 'quiz-option';
    btn.textContent = `${displayIdx + 1}. ${fc.options[origIdx]}`;
    btn.addEventListener('click', () => handleAnswer(qIndex, origIdx, btn));
    quizOptions.appendChild(btn);
  });

  quizOverlay.classList.add('active');
}
```

**Step 2: Simplify handleAnswer() result messages in study mode**

In `handleAnswer`, after the stats update and mastery check (line ~3634), add study mode branches for the result display. Wrap the game-specific logic (PTO, tug-of-war, penalties, upgrade purchase) in `if (!state.studyMode)` blocks:

For the **correct** path (after mastery check at line 3634):
```javascript
if (state.studyMode) {
  quizResult.className = 'quiz-result correct-result';
  quizResult.innerHTML = '✅ Correct!';
  quizResult.style.display = 'block';
  quizClose.textContent = 'Next Card';
  screenFlash('green');
} else {
  // ... existing PTO earning, streak messages, completePurchase, etc.
}
```

For the **wrong** path (after consecutiveRight reset at line 3683):
```javascript
if (state.studyMode) {
  quizResult.className = 'quiz-result wrong-result';
  quizResult.innerHTML = '❌ Not quite! The correct answer is highlighted above.';
  quizResult.style.display = 'block';
  quizClose.textContent = 'Next Card';
  screenFlash('red');
} else {
  // ... existing PTO protection, tug-of-war, penalty logic, etc.
}
```

Skip `updateTugBar()` and `checkAchievements()` calls in study mode at the end of handleAnswer.

**Step 3: Modify quizClose handler to auto-present next card**

Replace the quizClose click handler (line 3781):

```javascript
quizClose.addEventListener('click', () => {
  quizOverlay.classList.remove('active');
  if (state.studyMode) {
    setTimeout(() => showQuiz(), 500);
  }
});
```

Also update the overlay-click-to-close handler similarly (line 3786):

```javascript
quizOverlay.addEventListener('click', (e) => {
  if (e.target === quizOverlay && quizAnswered) {
    quizOverlay.classList.remove('active');
    if (state.studyMode) {
      setTimeout(() => showQuiz(), 500);
    } else {
      pendingUpgrade = null;
    }
  }
});
```

**Step 4: Verify manually**

- Study mode: cards appear automatically, no cost/upgrade text, simple correct/wrong messages
- After Continue, next card appears after 500ms
- Game mode: everything works exactly as before (PTO, tug, upgrades, etc.)
- Keyboard shortcuts (1-4, Space/Enter) work in both modes

**Step 5: Commit**

```
feat: study mode auto-present flow with simplified quiz UI
```

---

### Task 5: Add light theme CSS

**Files:**
- Modify: `index.html` (CSS section, after study-mode styles)

**Step 1: Add light theme CSS overrides**

```css
/* === LIGHT THEME === */
body.light-theme {
  background: linear-gradient(135deg, #f5f7fa 0%, #e4e9f0 50%, #dce2e9 100%);
  color: #2d3748;
}

body.light-theme header h1 {
  background: linear-gradient(90deg, #2b8a7e, #38a89d, #2b8a7e);
  background-size: 200% auto;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

body.light-theme .stats-bar span {
  background: rgba(0, 0, 0, 0.06);
  color: #4a5568;
}

body.light-theme .stats-bar .stat-val {
  color: #2b8a7e;
}

body.light-theme .study-tracker {
  background: rgba(0, 0, 0, 0.04);
}

body.light-theme .study-bar {
  background: rgba(0, 0, 0, 0.08);
}

body.light-theme .mode-toggle-label {
  color: #4a5568;
}

/* Quiz modal - light */
body.light-theme .quiz-modal {
  background: #fff;
  color: #2d3748;
  box-shadow: 0 8px 32px rgba(0,0,0,0.15);
}

body.light-theme .modal-overlay {
  background: rgba(0, 0, 0, 0.4);
}

body.light-theme .quiz-option {
  background: #f7fafc;
  color: #2d3748;
  border-color: #e2e8f0;
}

body.light-theme .quiz-option:hover {
  background: #edf2f7;
  border-color: #2b8a7e;
}

body.light-theme .quiz-question {
  color: #1a202c;
}

body.light-theme .quiz-header {
  border-color: #e2e8f0;
  color: #4a5568;
}

/* Review panel - light */
body.light-theme .review-panel {
  background: #fff;
  color: #2d3748;
}

body.light-theme .review-card {
  background: #f7fafc;
  border-color: #e2e8f0;
}

body.light-theme .review-card .rc-question {
  color: #1a202c;
}

/* Toast - light */
body.light-theme .toast {
  background: #fff;
  color: #2d3748;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

/* Stats panel - light */
body.light-theme .stats-panel {
  background: #fff;
  color: #2d3748;
}

/* Reset modal - light */
body.light-theme .prestige-confirm-modal {
  background: #fff;
  color: #2d3748;
}

/* Version label - light */
body.light-theme div[style*="fixed"][style*="bottom"] {
  color: rgba(0, 0, 0, 0.15) !important;
}

/* Game elements in light theme (when not in study mode) */
body.light-theme .click-section {
  color: #2d3748;
}

body.light-theme .shop-section .section-title,
body.light-theme .achievements-section .section-title {
  color: #2d3748;
}

body.light-theme .shop-item {
  background: #f7fafc;
  border-color: #e2e8f0;
  color: #2d3748;
}

body.light-theme .ach-card {
  background: #f7fafc;
  border-color: #e2e8f0;
  color: #2d3748;
}

body.light-theme .filter-btn {
  background: #edf2f7;
  color: #4a5568;
}

body.light-theme .filter-btn.active {
  background: #2b8a7e;
  color: #fff;
}

body.light-theme .gemini-toggle-note {
  color: #718096;
}
```

**Step 2: Update theme icon in applyMode()**

In the `applyMode()` function, add:
```javascript
const themeIcon = document.getElementById('themeIcon');
if (themeIcon) themeIcon.textContent = state.lightTheme ? '🌙' : '☀️';
```

**Step 3: Verify manually**

- Toggle light theme: background becomes light, text becomes dark, all panels/modals readable
- Light theme + Study Mode: clean, minimal flashcard experience
- Light theme + Game Mode: game elements styled appropriately
- Dark theme (default): no visual changes from current behavior

**Step 4: Commit**

```
feat: add light theme CSS with full coverage of all UI elements
```

---

### Task 6: Handle edge cases and polish

**Files:**
- Modify: `index.html` (various locations)

**Step 1: Handle "all cards mastered" in study mode**

In `showQuiz()`, before picking a question, check if all active cards are mastered:

```javascript
function showQuiz(upgradeName, cost) {
  // Check if all cards mastered (study mode completion)
  if (state.studyMode) {
    const active = getActiveFlashcards();
    const allMastered = active.every((_, i) => state.mastered[i]);
    if (allMastered) {
      quizLabel.textContent = '🎉 All Cards Learned!';
      quizCost.textContent = '';
      quizSource.innerHTML = '';
      quizQuestion.textContent = 'You\'ve mastered all ' + active.length + ' cards! Toggle the AI deck for more, or review your cards.';
      quizOptions.innerHTML = '';
      quizResult.style.display = 'none';
      quizClose.className = 'quiz-close show';
      quizClose.textContent = 'Review Cards';
      quizOverlay.classList.add('active');
      // Override close to open review
      quizClose.onclick = () => {
        quizOverlay.classList.remove('active');
        quizClose.onclick = null;
        renderReview();
        reviewOverlay.classList.add('active');
      };
      return;
    }
  }

  const qIndex = pickQuestion();
  // ... rest of function
```

**Step 2: Adjust reset modal text for study mode**

In the reset button handler area (line 2184), the text says "PEL bux, upgrades, and achievements" which is game-specific. Add a study-mode variant:

In the `applyMode()` function, add:
```javascript
const resetBody = document.querySelector('#resetOverlay .prestige-confirm-body');
if (resetBody) {
  resetBody.innerHTML = state.studyMode
    ? 'This will erase your <strong>study progress</strong> and <strong>learned cards</strong>.'
    : 'This will erase your <strong>PEL bux</strong>, <strong>upgrades</strong>, and <strong>achievements</strong>.';
}
```

**Step 3: Stop game intervals when switching to study mode**

Store interval IDs so they can be cleared when switching modes. Update the mode toggle handler and init:

```javascript
let gameTickInterval = null;
let particleInterval = null;

function startGameIntervals() {
  if (!gameTickInterval) gameTickInterval = setInterval(gameTick, 50);
  if (!particleInterval) particleInterval = setInterval(spawnParticle, 2500);
}

function stopGameIntervals() {
  if (gameTickInterval) { clearInterval(gameTickInterval); gameTickInterval = null; }
  if (particleInterval) { clearInterval(particleInterval); particleInterval = null; }
}
```

Update the mode toggle change handler to call `stopGameIntervals()` / `startGameIntervals()`.

**Step 4: Verify full flow manually**

- Study mode from URL: loads directly into flashcard mode, card auto-appears
- Study mode toggle: instant switch, card appears, game elements hidden
- Game mode toggle: game resumes, intervals restart
- All cards mastered: completion message appears
- Reset in study mode: appropriate messaging
- Light/dark theme: works in both modes
- Refresh preserves both toggle states
- Keyboard shortcuts work in study mode

**Step 5: Commit**

```
feat: study mode edge cases — completion state, interval management, reset text
```

---

### Task 7: Version bump and final commit

**Files:**
- Modify: `index.html:4748-4749` (version label)

**Step 1: Update version**

Change `v1.3.0` to `v1.4.0`.

**Step 2: Final manual test**

- Fresh load (clear localStorage): defaults to game mode, dark theme
- `?mode=study`: clean flashcard experience
- Toggle study mode on/off multiple times: no errors, smooth transitions
- Toggle light theme: all elements properly styled
- Answer cards in study mode, switch to game mode: progress preserved
- Answer cards in game mode, switch to study mode: progress preserved

**Step 3: Commit**

```
feat: bump version to v1.4.0 for Study Mode + Theme Toggle release
```
