# Notification Hooks — Technical Reference (MainHook.kt)

## Hook Target: `com.android.systemui`

All hooks in `MainHook.kt` run inside the SystemUI process.

---

## Hooks Summary

### 1. Context Capture — `android.app.Application.onCreate` [after]
- **Purpose:** Capture `appContext` for ContentProvider queries; write `autoexpand_active` marker to Global Settings so the UI can detect module activation.
- **Behavior:** Sets `appContext`, writes current timestamp to `Settings.Global["autoexpand_active"]`, triggers initial cache refresh.

---

### 2. Shade Hooks

#### `ExpandableNotificationRow.setSystemExpanded(Boolean)` [before]
- **Force `true`** when feature enabled and package not excluded.
- **Force `false`** when disabled or excluded.

#### `ExpandableNotificationRow.setSystemChildExpanded(Boolean)` [before]
- Same logic as `setSystemExpanded` — covers grouped/child notifications.

#### `ExpandableNotificationRow.setExpandable(Boolean)` [before]
- Forces `true` so the row always registers as expandable, preventing the system from clamping it to collapsed height.

**Feature key:** `expand_shade_enabled`

---

### 3. Lock Screen Hook

#### `ExpandableNotificationRow.setOnKeyguard(Boolean)` [before]
- Forces argument to `false` — tells the row it is NOT on the keyguard, so the expanded layout is shown.

**Feature key:** `expand_lockscreen_enabled`

---

### 4. Heads-Up Hooks

The heads-up expand system uses a custom instance field `aeCollapsed` (stored via `setAdditionalInstanceField`) to track expand/collapse state. This avoids the race condition where `mExpandedWhenPinned` is read by layout methods before `setHeadsUp` finishes.

#### `NotificationContentView.calculateVisibleType()` [after]
- If `isHeadsUp == true` and result == 2 (contracted), overrides to 1 (expanded) — provided `aeCollapsed != true` and an expanded child view exists.
- **Class:** `com.android.systemui.statusbar.notification.row.NotificationContentView`

#### `ExpandableNotificationRow.getIntrinsicHeight()` [after]
- If heads-up and not collapsed: returns `mExpandedChild.measuredHeight` instead of default contracted height.
- Guards: `isHeadsUp == true`, `aeCollapsed != true`, `expandedHeight > 0`.

#### `ExpandableNotificationRow.getPinnedHeadsUpHeight(Boolean)` [after]
- Overrides with `getMaxExpandHeight()` if that is larger — ensures the floating OPlus window is sized to fit the expanded content.

#### `ExpandableNotificationRow.setHeadsUp(Boolean)` [before + after]
- **before:** Sets `aeCollapsed = false` and `mExpandedWhenPinned = true`. Sets `inSetHeadsUp = true` guard.
- **after:** Clears `inSetHeadsUp`, re-sets `mExpandedWhenPinned = true`, calls `notifyHeightChanged(false)`.

#### `ExpandableNotificationRow.setExpandedWhenPinned(Boolean)` [before] (hookAllMethods)
- Syncs `aeCollapsed = !value` from external callers.
- Skips sync if `inSetHeadsUp == true` (avoids fighting with the `setHeadsUp` hook).

**Feature key:** `expand_headsup_enabled`

---

### 5. Swipe-to-Toggle Heads-Up

#### `OplusHeadsUpTouchHelper.onInterceptTouchEvent(MotionEvent)` [after]
- **Class:** `com.oplus.systemui.notification.headsup.windowframe.OplusHeadsUpTouchHelper`
- Tracks swipe direction passively (no touch consumption).
- On `ACTION_DOWN`: resets `f1DownStartY`, `f1IsDownwardSwipe`, `f1HasToggled`, `f1SwipeTime`, `f1CurrentRow`.
- On `ACTION_MOVE` (when intercept returns true): if `dy > 10`, marks `f1IsDownwardSwipe = true`, captures the touching heads-up row via `getMTouchingHeadsUpView()`.

#### `StatusBarNotificationActivityStarter.onNotificationClicked` [before] (hookAllMethods)
- **Class:** `com.android.systemui.statusbar.phone.StatusBarNotificationActivityStarter`
- If recent downward swipe (< 2 s): cancels click, calls `toggleHeadsUpExpandState(row)` using `param.args[1]`.

#### `StatusBarNotificationActivityStarter.startNotificationIntent` [before] (hookAllMethods)
- Same guard logic; finds row by scanning `param.args` for `ExpandableNotificationRow` or `NotificationEntry`, falls back to `f1CurrentRow`.

#### `toggleHeadsUpExpandState(row)` — internal helper
- Reads `aeCollapsed`, flips it, sets `mExpandedWhenPinned`, queries new `getIntrinsicHeight()`, calls `setActualHeight()`, then `requestLayout()` on row and parent.

**Feature key:** `disable_headsup_popup_enabled`

---

### 6. Back Gesture Haptic

#### `VibrationHelper.doVibrateCustomized(Context, Int, Boolean)` [before]
- **Class:** `com.oplus.systemui.navigationbar.gesture.VibrationHelper`
- Sets `param.result = null` to suppress the vibration entirely.

**Feature key:** `disable_back_haptic_enabled`

---

## Height Reference Values (OnePlus 15 / OxygenOS 16)
| State | Height (px) |
|---|---|
| Collapsed heads-up | 216 |
| Expanded heads-up | 763 |
| Max heads-up (getPinnedHeadsUpHeight) | 644 |
| Max expanded (getMaxExpandHeight) | 1253 |

---

## Preference Cache
- Cache TTL: 2000 ms (`CACHE_INTERVAL_MS`)
- Reads via ContentProvider at `content://com.autoexpand.xposed.prefs`
- Cursor columns: `key`, `type` (`bool` / `string_set` / `string`), `value`
- Bool default (missing key): `true` — all features default to enabled
