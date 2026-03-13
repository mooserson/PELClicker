# Study Mode + Theme Toggle Design

## Overview

Add a Study Mode to PEL Clicker that strips away all game mechanics and presents a pure flashcard experience. Add a light/dark theme toggle for visual comfort. Both modes share the same study progress data.

## Mode Toggle (Study / Game)

### State
- `state.studyMode` (boolean, default `false`)

### Activation
- **URL parameter:** `?mode=study` activates Study Mode on load
- **In-app toggle:** Switch in the header, always visible in both modes

### Study Mode Hides (via `body.study-mode` CSS)
- Click button, score display, pts/sec
- Shop section
- Achievements section
- Prestige section + progress bar
- Tug-of-war bar + PTO indicator
- Season banner + particles
- Year badge
- Daily login toast
- Stats dashboard button

### Study Mode Shows
- Header with title (changes to "PEL Flashcards")
- Study stats bar (Correct, Accuracy, Learned, Streak)
- Mastery progress bar
- Review button + AI deck toggle
- Reset button
- Quiz card — auto-presented, centered, the main focus

### JS Behavior in Study Mode
- `gameTick()` interval skipped
- `spawnParticle()` interval skipped
- `checkDailyBonus()` skipped
- After answering + dismissing, next card auto-shows (~500ms delay)
- `showQuiz()` called without upgrade/cost context
- `pickQuestion()` unchanged (same weighted selection)

## Theme Toggle (Dark / Light)

### State
- `state.lightTheme` (boolean, default `false`)

### Activation
- In-app toggle near the mode toggle (sun/moon icon or switch)

### Light Theme (via `body.light-theme` CSS)
- Background: soft white/light gray gradient
- Text: dark gray/black
- Cards/panels: white with subtle shadow
- Accent colors adjusted for contrast on light backgrounds
- Works in both Game Mode and Study Mode

## Shared State

All study data is mode-agnostic and persists across switches:
- `questionStats`, `mastered`, `correct`, `wrong`
- `streak`, `bestStreak`
- `useGeminiDeck`

Switching modes never resets anything.

## Auto-present Flow (Study Mode)

1. Page loads -> card appears immediately
2. User picks an answer
3. Feedback shown (correct/wrong + correct answer highlight)
4. User presses Space or clicks Continue
5. ~500ms pause -> next card appears
6. Repeat until all cards mastered (show completion state)
