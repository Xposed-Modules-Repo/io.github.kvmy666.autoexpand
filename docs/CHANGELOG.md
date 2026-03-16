# Changelog

## v1.2.0 (upcoming — versionCode 5)
### Added
- **Keyboard Enhancer:** Persistent toolbar injected below Gboard keyboard
  - Clipboard manager with history, pin, favorite, delete, search
  - Select-all (long press)
  - Cursor navigation (jump to start / end of field)
  - Text shortcut buttons (short-tap and long-press variants)
- Settings: Keyboard Enhancer section with enable toggle, height multiplier, shortcut text fields, max clipboard entries

## v1.1.0 (versionCode 3)
### Added
- Swipe-down on heads-up to toggle expand/collapse instead of launching app
- Flexible height fix: `setActualHeight` + `requestLayout` on row and parent for OPlus floating window

## v1.0.4 (versionCode 4)
### Fixed
- Minor stability improvements

## v1.0.0 (versionCode 1)
### Added
- Auto-expand notifications in shade (`setSystemExpanded`, `setSystemChildExpanded`, `setExpandable`)
- Auto-expand heads-up banners (`calculateVisibleType`, `getIntrinsicHeight`, `getPinnedHeadsUpHeight`, `setHeadsUp`, `setExpandedWhenPinned`)
- Auto-expand on lock screen (`setOnKeyguard`)
- Disable back gesture haptic (`VibrationHelper.doVibrateCustomized`)
- Per-app exclusion list
