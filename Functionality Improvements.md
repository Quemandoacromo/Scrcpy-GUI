# Functionality Improvements

Based on a full code review of the Flutter codebase (v1.6.0-rc.1) covering architecture, Flutter best practices, widget efficiency, platform code, and Dart quality.

---

## Priority 1 — Critical / High Impact

### 1. Replace recursive settings polling with `Timer.periodic`
**File:** `lib/main.dart` lines 115–132
`_startSettingsPolling()` recursively reschedules itself via `Future.delayed` with no stored reference. It cannot be cancelled in `dispose()`, and overlapping async chains can accumulate. Replace with a stored `Timer.periodic` cancelled in `dispose()`.

```dart
Timer? _settingsTimer;

void _startSettingsPolling() {
  _settingsTimer = Timer.periodic(const Duration(seconds: 2), (_) async {
    final newSettings = await _settingsService.loadSettings();
    if (!mounted) return;
    if (newSettings.showBatFilesTab != _currentSettings.showBatFilesTab) {
      setState(() { _currentSettings = newSettings; });
    }
  });
}

@override
void dispose() {
  _settingsTimer?.cancel();
  super.dispose();
}
```

**Ideal long-term fix:** Make `SettingsService` a `ChangeNotifier` and notify listeners when settings save — eliminates all polling entirely.

---

### 2. Remove `static` from `DeviceManagerService.devicesInfo`
**File:** `lib/services/device_manager_service.dart` line 35
`devicesInfo` is a public static mutable `Map`. Widgets read it directly, bypassing Provider entirely. Mutations do not trigger `notifyListeners()`. Makes the service untestable.
Remove `static`, expose as an instance getter, and access it exclusively through `context.read<DeviceManagerService>()`.

---

### 3. Remove `static` from `SettingsService.currentSettings`
**File:** `lib/services/settings_service.dart` lines 7–9
`CommandBuilderService.fullCommand` reads `SettingsService.currentSettings` as a hidden static dependency. The command does not rebuild when settings change because there is no listener relationship. Inject `SettingsService` (or `AppSettings`) into `CommandBuilderService` via its constructor.

---

### 4. Make `TerminalService` an injectable instance class
**File:** `lib/services/terminal_service.dart`
`TerminalService` is 942 lines of entirely static methods — no interface, no injection seam. Every downstream service and widget is hard-wired to real process execution, making the entire service layer untestable without a real device. Convert to an instance class behind an abstract interface and inject it into `DeviceManagerService` and other consumers.

---

### 5. Make option model classes immutable
**File:** `lib/models/scrcpy_options.dart` — all 10 option classes
All option classes expose public mutable `var` fields. Any widget can mutate them directly, bypassing `notifyListeners()`. The `copyWith()` methods imply immutable updates but nothing enforces them. Add `@immutable` and make all fields `final`.

---

### 6. Fix `runCommand` distinguishing errors from empty output
**File:** `lib/services/terminal_service.dart` lines 65–69
`runCommand` returns `""` for both a successful command with no output and a failed command. Callers (e.g., `adbDevices`) cannot distinguish "ADB not installed" from "no devices connected." Introduce a `CommandResult` type carrying `stdout`, `stderr`, and `exitCode`.

---

### 7. Narrow `CommandActionsPanel` rebuild scope
**File:** `lib/pages/home_panels/command_actions_panel.dart` line 41
The entire `CommandActionsPanel` is wrapped in `Consumer<CommandBuilderService>`, so every option change in every panel triggers a full rebuild of the device dropdown, buttons, and text field. Only `fullCommand` is actually needed. Use `context.select`:

```dart
final command = context.select<CommandBuilderService, String>(
  (svc) => svc.fullCommand,
);
```

---

### 8. Guard all `debugPrint` calls with `kDebugMode`
**File:** `lib/services/command_builder_service.dart` lines 155–213, `lib/models/scrcpy_options.dart`
`fullCommand` triggers 11 unconditional `debugPrint` calls per keypress (once per option class + service). While no-op in release mode, these calls add noise, slow debug sessions, and are inappropriate inside model classes. Wrap all logging:

```dart
if (kDebugMode) debugPrint('[CommandBuilderService] $message');
```

---

### 9. Remove dual notification mechanism in `DeviceManagerService`
**File:** `lib/services/device_manager_service.dart` lines 51–66
`selectedDevice` updates both a `ValueNotifier<String?>` and calls `notifyListeners()`. Two parallel observation channels for the same value create confusion and risk inconsistent rebuild ordering. Remove `selectedDeviceNotifier`; use `context.select` or `Selector` from Provider for granular listening.

---

### 10. Use constructor injection for service dependencies
**File:** `lib/main.dart` lines 43–65
`CommandBuilderService` is constructed and then wired via a public setter — it exists in a partially initialised state between those two lines. Require dependencies at construction time:

```dart
// Before
commandBuilder.deviceManagerService = deviceManager;

// After
final commandBuilder = CommandBuilderService(deviceManagerService: deviceManager);
```

---

## Priority 2 — Medium Impact

### 11. Fix `_isLockedExpanded` wrong `orElse` fallback in `SurroundingPanel`
**File:** `lib/widgets/surrounding_panel.dart` lines 112–129
When a `panelId` is not found in `panelOrder`, the `orElse` fallback returns `settings.panelOrder.first` — an entirely unrelated panel's settings. This is a correctness bug. Replace with a safe default:

```dart
orElse: () => PanelSettings(id: '', displayName: '', visible: true, lockedExpanded: false),
```

---

### 12. Remove collapsed panel children from the layout tree
**File:** `lib/widgets/surrounding_panel.dart` lines 292–313
Collapsed panels use a `ConstrainedBox(maxHeight: 0)` which keeps the full child subtree built and participating in layout passes. For panels with many widgets (e.g., `InputControlPanel`), this wastes work. Use `Visibility(visible: isExpanded, maintainState: true, child: ...)` to remove the child from layout while preserving state.

---

### 13. Fix macOS AppleScript quote injection
**File:** `lib/services/terminal_service.dart` lines 133–138
The scrcpy command is interpolated directly into an AppleScript string literal. If the command contains a double quote (e.g., `--window-title="My App"`), the AppleScript becomes malformed. Escape before interpolation:

```dart
final escaped = wrappedCommand.replaceAll('"', '\\"');
```

---

### 14. Inject `CommandsService` instead of instantiating per button press
**File:** `lib/pages/home_panels/command_actions_panel.dart` lines 235, 307
`CommandsService()` is `new`-ed on every `_runCommand` and `_favoriteCommand` call. Add it to the Provider tree and access via `context.read<CommandsService>()`.

---

### 15. Convert `pages` getter to a `late final` field in `main.dart`
**File:** `lib/main.dart` lines 148–155
`pages` is a getter, so a new `HomePage` instance is constructed on every `build()` call. Assign it once in `initState` and rebuild only when `_currentSettings` actually changes.

---

### 16. Add a bool lock to prevent overlapping ADB poll ticks
**File:** `lib/services/device_manager_service.dart` line 87
The 2-second `Timer.periodic` for device polling does not wait for the previous tick to complete. If `_loadDeviceData` takes longer than 2 seconds (e.g., slow ADB daemon on first connection), ticks overlap. Add a simple `_isChecking` guard:

```dart
bool _isChecking = false;

_pollingTimer = Timer.periodic(const Duration(seconds: 2), (_) async {
  if (_isChecking) return;
  _isChecking = true;
  await _checkDeviceChanges();
  _isChecking = false;
});
```

---

### 17. Fix `disconnectWireless` always returning `success: true`
**File:** `lib/services/terminal_service.dart` lines 779–788
The `else` branch returns `success: true` even when `disconnectResult` contains an ADB error string (e.g., `error: no such device`). The UI shows a green snackbar on error. Check the content and return `success: false` appropriately.

---

### 18. Remove redundant `_isLoading` flag in `HomePage`
**File:** `lib/pages/home_page.dart` lines 60–81
`_loadSettings()` is synchronous and called from `initState`. The `_isLoading` flag flips `true → false` within the same synchronous call, showing a `CircularProgressIndicator` for a single frame. Remove `_isLoading` entirely and initialise `panelOrder` directly in `initState`.

---

### 19. Replace `Function(String?)` with `ValueChanged<String?>`
**File:** `lib/widgets/custom_dropdown.dart` line 45, `lib/widgets/sidebar.dart`
Using the raw `Function(String?)` type does not enforce a `void` return type. Use the idiomatic Flutter typedef `ValueChanged<String?>` (and `ValueChanged<int>` for the sidebar).

---

### 20. Parallelise sequential ADB calls in `getDeviceIpAddress`
**File:** `lib/services/terminal_service.dart` lines 669–723
Up to 8 sequential ADB shell invocations are made to find the device IP. On a slow ADB daemon this blocks the wireless setup flow for several seconds. Attempt strategies in parallel using `Future.wait` and take the first successful result.

---

## Priority 3 — Low Impact / Polish

### 21. Add `ValueKey(selectedIndex)` to `AnimatedSwitcher` children
**File:** `lib/main.dart` line 181
Without explicit keys, `AnimatedSwitcher` may not animate transitions between pages of differing types consistently. Wrap each page in `KeyedSubtree(key: ValueKey(selectedIndex))`.

---

### 22. Replace `ListView.builder` in `Sidebar` with a `Column`
**File:** `lib/widgets/sidebar.dart` line 64
The sidebar has at most 6 fixed, non-scrollable items. `ListView.builder` is unnecessary overhead. Use a `Column` with direct children, and replace `GestureDetector + MouseRegion` with `InkWell` for proper desktop hover semantics.

---

### 23. Remove or implement `CustomDropdown.showCheckIcon`
**File:** `lib/widgets/custom_dropdown.dart` lines 47–48
The `showCheckIcon` field is declared and included in the constructor but never referenced in `build`. Either implement it or remove the dead code.

---

### 24. Convert `CustomMultiDropdown` to a `StatefulWidget`
**File:** `lib/widgets/custom_multi_dropdown.dart`
`MultiDropdown` is constructed inside `build` of a `StatelessWidget`. Frequent parent rebuilds (triggered by `CommandBuilderService`) will create a new `MultiDropdown` instance each time, potentially resetting its internal open/search state. Wrap in a `StatefulWidget` to preserve the instance across rebuilds.

---

### 25. Add typed fields to option model classes
**File:** `lib/models/scrcpy_options.dart`
Numeric values like `maxSize`, `bitrate`, `framerate`, `tcpipPort`, and `windowX` are stored as `String`. Invalid input (e.g., `"abc"` for bitrate) passes through silently to scrcpy. Use `int?` / `double?` and add a `validate()` method per class to enable inline UI validation.

---

### 26. Eliminate `_getPanelDisplayName` duplication in `HomePage`
**File:** `lib/pages/home_page.dart` lines 83–113
Panel display names are defined twice: once in `_getPanelDisplayName()` in `HomePage` and once in `defaultPanels` in `settings_model.dart`. These will drift apart over time. Read the display name directly from `PanelSettings.displayName` and remove the redundant method.

---

### 27. Use enum-based page identifiers instead of positional indices
**File:** `lib/main.dart` lines 148–155, 124–126
The conditional insertion of `ScriptsPage` shifts all subsequent page indices, requiring manual index arithmetic in multiple places. Replacing positional indices with a `Page` enum eliminates this fragility entirely.

---

## Architecture Scorecard (at time of review)

| Category | Score | Key Issue |
|---|---|---|
| Separation of Concerns | 6/10 | `TerminalService` is a God Class; static cross-service access blurs boundaries |
| State Management | 5/10 | Polling, static mutable state, dual notifier undermine the reactive model |
| Data Flow | 7/10 | Panel → Service → Command pipeline is clean; `generateCommandPart()` pattern is solid |
| Error Handling | 3/10 | Silent swallowing is the default; no user-facing error reporting |
| Testability | 3/10 | Static `TerminalService` blocks all unit testing without a real device |
| Cross-Platform | 6/10 | Functional but platform branching is scattered across methods |
| Extensibility | 7/10 | `copyWith` + `generateCommandPart()` scales well; panel registration is 4-step but manageable |
| Code Quality | 7/10 | Well-documented, consistent naming, good Dart idioms |
