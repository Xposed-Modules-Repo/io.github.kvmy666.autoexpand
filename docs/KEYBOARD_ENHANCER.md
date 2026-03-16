# Keyboard Enhancer — Feature Spec (v1.2.0)

## Overview
Inject a persistent toolbar row below Gboard's keyboard view, providing: clipboard manager, select-all, cursor navigation, and text shortcuts.

**Target package:** `com.google.android.inputmethod.latin`
**Hook class:** `com.autoexpand.xposed.KeyboardHook`
**Pref key:** `keyboard_enhancer_enabled` (bool, default true)

---

## Phase 1: Reconnaissance (COMPLETED FIRST — zero behavior change)

Hook these `InputMethodService` lifecycle methods with **logging only**:

| Method | Hook type | Log target |
|---|---|---|
| `onCreateInputView()` | after | Return view class name, dimensions |
| `setInputView(View)` | before | View class, parent class |
| `onStartInputView(EditorInfo, Boolean)` | before | EditorInfo fields |
| `getWindow()` | after | Window layout attributes |

Walk the view hierarchy from `onCreateInputView` result → log all child class names + dimensions.

**Logcat filter:** `tag:AutoExpand`

**Expected discoveries:** Real Gboard view class names, actual hierarchy depth, window layout params.

---

## Phase 2: PoC Toolbar Injection

Wrap the view returned by `onCreateInputView()` in a vertical `LinearLayout`:

```
LinearLayout (vertical, MATCH_PARENT × WRAP_CONTENT)
  ├── [original keyboard view]   ← kept intact, weight 0
  └── [toolbar LinearLayout]     ← injected at bottom, solid color #FF1A1A2E
```

Height calculation (in order of preference):
1. Find Enter/Return key by content description → measure its height → `toolbarHeight = (enterKeyHeight * multiplier).toInt()`
2. Fallback: `keyboardView.height * 0.09` (9% of keyboard height)
3. Hard fallback: 48dp

Post height calculation as a `Runnable` after wrapping (view not measured yet at hook time).

---

## Phase 3: Full Feature Implementation

### 3a. ClipboardDatabase (ClipboardDatabase.kt)
Raw `SQLiteOpenHelper`, no Room dependency.

```sql
CREATE TABLE clipboard_entries (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  text        TEXT NOT NULL,
  timestamp   INTEGER NOT NULL,
  is_pinned   INTEGER NOT NULL DEFAULT 0,
  is_favorite INTEGER NOT NULL DEFAULT 0
)
```

- DB path: `/data/data/com.autoexpand.xposed/databases/clipboard.db`
- Auto-prune: delete oldest non-pinned, non-favorite when `count > clipboard_max_entries`
- All DB ops run on `Executors.newSingleThreadExecutor()`
- Methods: `insert(text)`, `getAll(sortBy)`, `pin(id)`, `favorite(id)`, `delete(id)`, `search(query)`

### 3b. Clipboard Capture
Register `ClipboardManager.addPrimaryClipChangedListener()` inside `onCreateInputView()` hook (runs in Gboard's process).

On clip change:
1. Read `clipboardManager.primaryClip?.getItemAt(0)?.text`
2. Skip if empty
3. Skip if identical to most recent DB entry (dedup)
4. Insert on background thread

### 3c. Toolbar Buttons

#### Button 1 — Clipboard (📋)
- Tap → `PopupWindow` anchored above toolbar, full-width
- Content: `ScrollView` + dynamic `LinearLayout` (avoid RecyclerView dependency)
- Tab strip: All / Pinned / Favorites / Regular
- Sort toggle: newest-first / oldest-first
- Each item: tap → `inputConnection.commitText(text, 1)`, long-press → `AlertDialog` (Pin / Favorite / Delete)

#### Button 2 — Select All (⬛)
- Long press (≥ `ViewConfiguration.getLongPressTimeout()` ms): `inputConnection.setSelection(0, extractedText.text.length)`
- Short tap: no-op (avoids accidental selection)

#### Button 3 — Cursor Navigation (← →)
- Nested horizontal `LinearLayout`, 50/50 split
- Left arrow: `inputConnection.setSelection(0, 0)` — go to start
- Right arrow: `inputConnection.setSelection(len, len)` — go to end

#### Button 4 — Text Shortcut (★)
- Short tap: `inputConnection.commitText(shortcut_text_1, 1)`
- Long press: `inputConnection.commitText(shortcut_text_2, 1)` (skip if empty)

### 3d. InputConnection Access
Call `(param.thisObject as InputMethodService).currentInputConnection` at button-press time.
Null-check: disable/grey buttons when `currentInputConnection == null`.

### 3e. Toolbar Styling
- Background: `Color.parseColor("#CC1A1A2E")` (semi-transparent dark)
- Attempt to sample Gboard root view background color and match
- Icon characters: Unicode symbols (📋, ⬛, ←, →, ★) or drawables
- Dividers between buttons (1dp vertical lines, #33FFFFFF)

---

## Phase 4: Settings UI

New Compose `Card` section in `MainActivity.kt` below the existing feature cards:

```
── Keyboard Enhancer ──────────────────────
  [Toggle] Enable keyboard toolbar
  [Slider] Toolbar height: 1.0× (range 0.5×–2.0×)
  [TextField] Shortcut 1 text
  [TextField] Shortcut 2 text (long-press)
  [NumberField] Max clipboard entries: 500
───────────────────────────────────────────
```

Reuse existing `ToggleRow` composable pattern. Slider uses `androidx.compose.material3.Slider`.
TextField changes save immediately via `prefs.edit().putString(key, value).apply()`.

---

## Preference Keys

| Key | Type | Default |
|---|---|---|
| `keyboard_enhancer_enabled` | bool | true |
| `toolbar_height_multiplier` | float | 1.0 |
| `shortcut_text_1` | string | `"ههههههههههههههههههههههههههههه"` |
| `shortcut_text_2` | string | `""` |
| `clipboard_max_entries` | int | 500 |

---

## Risk Mitigations

| Risk | Mitigation |
|---|---|
| Gboard view hierarchy differs from AOSP | Phase 1 reconnaissance maps real class names first |
| `onCreateInputView` not the right injection point | Log all lifecycle methods; fallback to `setInputView` hook |
| `InputConnection` null at button press | Null-check + grey out buttons |
| Clipboard PopupWindow layout issues | Start with simple `ScrollView` + dynamic views before tabs |
| SQLite on main thread | `Executors.newSingleThreadExecutor()` for all DB ops |
| Toolbar breaks keyboard height/insets | `WRAP_CONTENT` on container; test gesture + 3-button nav |
