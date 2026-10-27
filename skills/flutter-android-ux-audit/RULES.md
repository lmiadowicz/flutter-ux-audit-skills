# Android / Google Play UI-UX rules for Flutter

Each rule has an ID, a **severity**, its source, what it means in Flutter, how to **detect** it, and how to **fix** it.
Detection patterns are *candidates*. Read the code around every hit before you call it a finding.

**Severity**
- **Blocker**: the platform enforces it (the app breaks on new targetSdk / Android versions) or Play policy can reject the app.
- **Major**: a Core App Quality or Adaptive App Quality criterion, or a "must" in Android/Material guidance.
- **Minor**: a recommendation. It makes the app feel off-platform but won't get it rejected.

**Sources** (checked 2026-10)
- CAQ: Core app quality, https://developer.android.com/docs/quality-guidelines/core-app-quality
- AAQ: Adaptive app quality, Tier 3, https://developer.android.com/docs/quality-guidelines/adaptive-app-quality/tier-3
- A16: Android 16 behavior changes, https://developer.android.com/about/versions/16/behavior-changes-16
- E2E: Edge-to-edge, https://developer.android.com/develop/ui/views/layout/edge-to-edge
- NAVP: Principles of navigation, https://developer.android.com/guide/navigation/principles
- ADNAV: Adaptive navigation, https://developer.android.com/develop/ui/compose/layouts/adaptive/build-adaptive-navigation
- M3: Material 3, https://m3.material.io (components: navigation bar, rail, drawer, dialogs, sheets, FAB)
- PLAY: Google Play Developer Policy Center, https://play.google.com/about/developer-content-policy/

Before you start, check `android/app/build.gradle(.kts)` for `targetSdk`, and `pubspec.yaml` for the Flutter SDK constraint and the router package. Some rules depend on them.

---

## A-NAV: Navigation structure

### A-NAV-01 Bottom navigation bar is for 3–5 top-level destinations, not actions. Major. M3, CAQ Consistent_UX
- A bar with 1 destination, or with more than 5, is wrong. Use a drawer or a different structure instead.
- A tab whose tap opens a dialog, sheet, camera or "create" screen (the "centre + tab" pattern) is an action, not a destination. Use a FAB for it.
- Each item has an icon and a label. Labels are always shown when there are 3 items, and are recommended for 4–5.
- Prefer Material 3 `NavigationBar` over the M2 `BottomNavigationBar`.
- **Detect:** `rg -n "BottomNavigationBar\(|NavigationBar\(|NavigationDestination\(|BottomNavigationBarItem\(" lib`. Count the items. Read `onTap` / `onDestinationSelected` for branches that `push`, `showDialog` or `showModalBottomSheet` instead of switching the index.
- **Fix:** move actions to a `FloatingActionButton`. Use `NavigationBar` with `NavigationDestination(icon:, selectedIcon:, label:)`. Move destinations beyond five into a `NavigationDrawer` or a screen inside a tab.

### A-NAV-02 Navigation adapts to window size. Major. ADNAV, AAQ, CAQ Display_State_Parity
- Compact width (<600dp): navigation bar. Medium and expanded widths (≥600dp): navigation rail. Expanded (≥840dp): a permanent drawer is optional.
- The same destinations exist in every form.
- **Detect:** a `NavigationBar`/`BottomNavigationBar` with no `LayoutBuilder`, `MediaQuery.sizeOf(context).width` or breakpoint switch around it. Search for `NavigationRail` (often missing entirely).
- **Fix:** branch on width: `NavigationRail` (with `labelType: NavigationRailLabelType.all` or selected) in a `Row` above 600dp, `NavigationBar` below. Generate both from one destination list. The `flutter_adaptive_scaffold` package is an option if the project already accepts that dependency.

### A-NAV-03 Each tab keeps its own back stack and scroll state. Major. CAQ State_Preservation
- Switching tabs and back must not reset the tab to its root or lose its scroll position.
- **Detect:** a `body: pages[_index]` / `switch (_index)` that rebuilds pages on each switch, with no `IndexedStack`, `StatefulShellRoute` (go_router) or `AutoTabsRouter` (auto_route).
- **Fix:** use `StatefulShellRoute.indexedStack` (go_router), `AutoTabsRouter` (auto_route), or `IndexedStack` with one nested `Navigator` per tab. Add `PageStorageKey` to scrollables.

### A-NAV-04 Fixed start destination. Login, onboarding and splash are not on the back stack. Major. NAVP
- After login, onboarding or splash, Back from home must exit the app. It must not return to the login screen.
- **Detect:** `rg -n "Navigator\.(of\(context\)\.)?push(Named)?\(|context\.push\(" lib` inside login, onboarding or splash success handlers.
- **Fix:** use `pushAndRemoveUntil(..., (_) => false)`, `pushReplacement`, `context.go(...)`, or a go_router `redirect` based on auth state.

### A-NAV-05 Up (the AppBar back arrow) never exits the app and isn't shown on the start destination. Major. NAVP
- **Detect:** a hand-built `leading: IconButton(icon: Icon(Icons.arrow_back...` on root or tab screens. `SystemNavigator.pop()` wired to an AppBar button.
- **Fix:** rely on `AppBar`'s `automaticallyImplyLeading`. Use `BackButton()` / `Icons.adaptive.arrow_back` only where a previous route exists.

### A-NAV-06 Deep links build a synthetic back stack. Minor. NAVP
- Opening `/orders/42` from a link, then pressing Back or Up, should go to `/orders`, then home. It should not exit straight away or land on a blank screen.
- **Detect:** flat go_router routes (`GoRoute(path: '/orders/:id')` at top level), or `onGenerateRoute` that pushes a single page.
- **Fix:** nest the detail route under its parent (`GoRoute(path: '/orders', routes: [GoRoute(path: ':id')])`) and navigate with `go`.

### A-NAV-07 Drawer usage. Minor. M3
- Use a `NavigationDrawer` (M3) for 5+ destinations or secondary destinations. It shouldn't duplicate the bottom bar.
- **Detect:** `Drawer(` with `ListTile`s that repeat `NavigationBar` destinations.

---

## A-BACK: System back and predictive back

### A-BACK-01 Use PopScope, not WillPopScope. Blocker when targetSdk ≥ 36, otherwise Major. A16, CAQ Back_Button_Nav, Back_Gesture_Nav
- On Android 16 with targetSdk 36, predictive back is on by default and `onBackPressed`/`KEYCODE_BACK` are no longer delivered. `WillPopScope` (deprecated) blocks predictive back.
- **Detect:** `rg -n "WillPopScope|onWillPop|SystemNavigator\.pop|RawKeyboardListener|LogicalKeyboardKey\.goBack" lib`. Check `android/app/src/main/AndroidManifest.xml` for `android:enableOnBackInvokedCallback="false"`.
- **Fix:** `PopScope(canPop: !hasUnsavedChanges, onPopInvokedWithResult: (didPop, result) { if (!didPop) _confirmDiscard(); })`. Set `android:enableOnBackInvokedCallback="true"` on `<application>`. Remove the `false` opt-out (it's temporary).

### A-BACK-02 No custom on-screen back prompts. Major. CAQ Back_Button_Nav
- "Press back again to exit" toasts, and dialogs that ask "Exit app?" on every Back from home, fight system back.
- **Detect:** `rg -n "back again|press back|exit the app|DateTime.now\(\).difference" lib` near PopScope/WillPopScope.
- **Fix:** let Back from the start destination exit. Only intercept Back to protect unsaved user input.

### A-BACK-03 Back never loses user data. Major. CAQ State_Preservation
- A screen with a form, draft or editor must not silently drop input on Back or swipe.
- **Detect:** screens with `TextEditingController`/`Form` and no `PopScope(canPop: ...)`.
- **Fix:** `PopScope(canPop: !dirty)` plus a discard confirmation dialog, or autosave the draft.

### A-BACK-04 Predictive back page transitions. Minor. A16, M3
- **Detect:** `pageTransitionsTheme` missing in `ThemeData`, or custom `PageRouteBuilder` fades used for normal pushes.
- **Fix:** `ThemeData(pageTransitionsTheme: const PageTransitionsTheme(builders: {TargetPlatform.android: PredictiveBackPageTransitionsBuilder()}))` (Flutter 3.22+). Keep `MaterialPageRoute` for normal pushes.

---

## A-E2E: Edge-to-edge and insets

### A-E2E-01 The app is edge-to-edge and doesn't opt out. Blocker when targetSdk ≥ 36, otherwise Major. E2E, A16
- With targetSdk 35+ on Android 15+, the app draws behind the system bars. With targetSdk 36 on Android 16, the opt-out is ignored.
- **Detect:** `rg -n "windowOptOutEdgeToEdgeEnforcement" android`. `rg -n "setEnabledSystemUIMode|SystemUiOverlayStyle|systemNavigationBarColor|statusBarColor" lib`. An opaque `systemNavigationBarColor` or `statusBarColor` assumes the old non-edge-to-edge model (these are deprecated on Android 15).
- **Fix:** remove the opt-out. Call `SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge)` at startup. Use transparent bar colours with correct `statusBarIconBrightness`/`systemNavigationBarIconBrightness` per theme, set via `AppBarTheme.systemOverlayStyle` or `AnnotatedRegion<SystemUiOverlayStyle>`.

### A-E2E-02 Content and controls respect insets. Major. E2E
- Tappable content must not sit under the status bar, the navigation bar or the display cutout. Scrolling lists should scroll *behind* the bars, with padding at the end.
- **Detect:** `Scaffold`s without an `AppBar` or `SafeArea` whose first child is at the top. Hard-coded `top: 24`, `height: 56 + 24` and similar status-bar maths. Bottom-pinned buttons (`bottomNavigationBar: ElevatedButton`, `Positioned(bottom: 0)`) without `SafeArea`/`MediaQuery.paddingOf(context).bottom`. `removePadding` misuse.
- **Fix:** wrap in `SafeArea` (or use `minimum:`), use `MediaQuery.paddingOf`/`viewPaddingOf`, and give lists `padding: EdgeInsets.only(bottom: MediaQuery.paddingOf(context).bottom)`. Material `NavigationBar`, `BottomAppBar` and `AppBar` handle insets themselves.

### A-E2E-03 Edge swipes don't fight the system back gesture. Minor. E2E (systemGestures insets)
- Horizontal drag handlers (carousels, drawers opened by drag, custom sliders) at the left or right screen edge conflict with gesture back.
- **Detect:** full-width `GestureDetector(onHorizontalDrag...)`, `PageView` or `Dismissible` without edge padding.
- **Fix:** inset the interactive area by `MediaQuery.systemGestureInsetsOf(context)`, or provide a button alternative.

---

## A-LS: Large screens, foldables, configuration changes

### A-LS-01 Don't lock orientation or resizability. Blocker when targetSdk ≥ 36 (ignored on sw600dp+, so the UI breaks), otherwise Major. A16, CAQ Fullscreen_Display, AAQ Config_Changes
- **Detect:** `rg -n "setPreferredOrientations" lib`. `rg -n "screenOrientation|resizeableActivity|maxAspectRatio|minAspectRatio|PROPERTY_COMPAT_ALLOW_RESTRICTED_RESIZABILITY" android`.
- **Fix:** remove the locks (games excepted). Make layouts work in landscape and on tablets (A-LS-02). If a phone-only portrait lock is a product decision, apply it only below 600dp shortest side.

### A-LS-02 Layouts fill the window without stretching. Major. CAQ Fullscreen_Display, Line_Length, AAQ Config_Changes
- Text blocks should be 45–75 characters per line. Forms and buttons must not stretch edge to edge on a 1280dp-wide tablet. Use list-detail or multi-column layouts at expanded width.
- **Detect:** screens with no `LayoutBuilder`, `ConstrainedBox(maxWidth:)` or breakpoint logic. Full-width `ElevatedButton`s / `SizedBox(width: double.infinity)` in forms. Fixed `width: 375`-style magic numbers. `Image` with a fixed height that crops badly in landscape.
- **Fix:** wrap reading content in `Center(child: ConstrainedBox(constraints: BoxConstraints(maxWidth: 600–840)))`. Use two-pane list-detail at ≥840dp. Replace magic sizes with constraints.

### A-LS-03 State survives rotation, fold/unfold and resizing. Major. CAQ State_Preservation, AAQ Config_Changes
- Scroll position, typed text, selected tab and media position must survive.
- **Detect:** state kept in `build()` locals. `TextEditingController` created in `build`. Missing `PageStorageKey`. No `restorationScopeId` on `MaterialApp` (needed for process-death restore).
- **Fix:** keep state in `State`/state management. Add `restorationScopeId: 'app'` and `RestorationMixin` for important forms. Add `PageStorageKey`s.

### A-LS-04 Keyboard, mouse, trackpad and stylus basics. Major. AAQ Keyboard_Input, Mouse_Trackpad_Basic, Stylus_Basic
- Every clickable thing is reachable and clickable. Lists scroll with a mouse wheel and trackpad. Text fields work with a hardware keyboard. Tab order is logical.
- **Detect:** custom `ScrollBehavior` with `dragDevices` that excludes `PointerDeviceKind.mouse`/`trackpad` where drag-scroll is needed. `GestureDetector` used as a button (no focus, no hover, no keyboard activation). `FocusNode(skipTraversal: true)` on interactive items.
- **Fix:** use `InkWell`/`TextButton`/`IconButton` (focusable, hoverable, Enter/Space activatable) instead of bare `GestureDetector`. Wrap forms in a `FocusTraversalGroup`. Add a `Shortcuts`/`CallbackShortcuts` for primary actions where useful.

---

## A-VIS: Visual design and theming

### A-VIS-01 Material 3 components and theming. Minor. CAQ (use Material components), M3
- **Detect:** `useMaterial3: false`. M2-only widgets (`BottomNavigationBar`, `RaisedButton`/`FlatButton` in old code, `ButtonBar`). Hard-coded `Color(0xFF...)` scattered through widgets instead of `Theme.of(context).colorScheme`.
- **Fix:** `ThemeData(useMaterial3: true, colorScheme: ColorScheme.fromSeed(seedColor: ...))`. Read colours from `colorScheme` roles (`primary`, `onSurface`, `surfaceContainer`...).

### A-VIS-02 Light and dark themes are both supported. Major. CAQ Theme_Support
- **Detect:** `MaterialApp` without `darkTheme`, or with `themeMode: ThemeMode.light`. `rg -n "Colors\.(white|black)|Color\(0x" lib/**/screens lib/**/widgets` used as backgrounds or text colours. `WebView`s with a forced light background.
- **Fix:** add `darkTheme` (`ColorScheme.fromSeed(..., brightness: Brightness.dark)`) and `themeMode: ThemeMode.system`. Replace literal colours with scheme roles.

### A-VIS-03 Crisp graphics. Minor. CAQ Graphic_Quality
- **Detect:** raster icons (`Image.asset('...png')`) used as UI icons. Assets without `2.0x/` and `3.0x/` variants. Images scaled up beyond their resolution.
- **Fix:** use `Icon`/`Symbols`, `flutter_svg`, or resolution-aware asset folders.

### A-VIS-04 Component conventions. Minor. M3
- One primary FAB per screen, for the screen's main action. Not stacked FABs.
- Dialogs: short title, `TextButton` actions, confirm on the trailing side, destructive actions explicit ("Delete", not "OK").
- Snackbars are for brief, non-critical feedback with at most one action. Don't use them for errors that block the task.
- Modal bottom sheets: `showDragHandle: true`, dismissible by drag down and scrim tap.
- **Detect:** `rg -n "FloatingActionButton|showDialog|AlertDialog|SnackBar|showModalBottomSheet" lib` and review each one.

---

## A-A11Y: Accessibility

### A-A11Y-01 Touch targets are at least 48×48dp. Major. CAQ Touch_Target_Size
- **Detect:** `materialTapTargetSize: MaterialTapTargetSize.shrinkWrap`. `visualDensity: VisualDensity.compact` or negative density on tappables. `IconButton(... constraints: BoxConstraints(...)` below 48, or `padding: EdgeInsets.zero`. `GestureDetector`/`InkWell` wrapping an `Icon` with `size` < 48 and no padding. `SizedBox(height: 32, child: TextButton(...))`.
- **Fix:** remove the shrinking, add padding, or wrap in `SizedBox.square(dimension: 48)`. The visual can stay small. The hit area must not.

### A-A11Y-02 Contrast: 4.5:1 for normal text, 3:1 for large text (≥18pt, or ≥14pt bold) and graphics. Major. CAQ Visual_Contrast
- **Detect:** grey-on-white literals (`Colors.grey`, `Colors.grey[400]`, `Color(0xFF9E9E9E)` on light), `withOpacity(0.3–0.5)` on text, white text on light brand colours. Compute the ratio for the literal pairs you find.
- **Fix:** use `colorScheme.onSurface`/`onSurfaceVariant` on the matching surface. Adjust the brand tones.

### A-A11Y-03 Every non-text element has a label. Major. CAQ Content_Description
- **Detect:** `IconButton` without `tooltip:`. `Image` without `semanticLabel:` (and not `excludeFromSemantics: true` when decorative). `GestureDetector`/`InkWell` around icons without a `Semantics(label:, button: true)`. `Icon` conveying meaning without `semanticLabel`.
- **Fix:** add `tooltip`/`semanticLabel`/`Semantics(button: true, label: ...)`. Mark decorative images `excludeFromSemantics: true`.

### A-A11Y-04 Text scales with the system font size. Major. CAQ (accessibility), M3
- **Detect:** `rg -n "textScaleFactor|textScaler|TextScaler\.(noScaling|linear)|MediaQuery\.of\(context\)\.copyWith" lib`. Global clamps to 1.0. Fixed-height containers (`SizedBox(height: 40, child: Text(...))`, `maxLines: 1` without `overflow`) that clip at 200%.
- **Fix:** don't disable scaling. If you must clamp, clamp the maximum (≥ 2.0) with `textScaler.clamp(maxScaleFactor: 2.0)`. Use min-height constraints instead of fixed heights.

### A-A11Y-05 Information isn't conveyed by colour alone. Minor. CAQ, M3
- **Detect:** status shown only as a coloured dot or border (error fields with a red border and no text, a chart legend with colour only).
- **Fix:** add an icon, text or pattern. Use `InputDecoration(errorText:)`.

---

## A-PLAY: Play policy items that show up in the UI

### A-PLAY-01 In-app account deletion. Blocker if the app lets users create accounts. PLAY (User Data, account deletion)
- **Detect:** sign-up flows (`signUp`, `createUserWithEmailAndPassword`, register screens) with no "Delete account" in settings or profile.
- **Fix:** add a discoverable "Delete account" in Settings/Profile that starts deletion (confirmation, then a backend call). A web deletion link must also be declared in Play Console.

### A-PLAY-02 Permissions are requested in context, with a disclosure where required. Major. PLAY (permissions, prominent disclosure)
- **Detect:** `rg -n "\.request\(\)|requestPermission|Permission\.(location|camera|contacts|microphone|notification)" lib` called from `main()`, splash or `initState` of the first screen.
- **Fix:** request at the moment of use, after a short in-app explanation. Handle denial with a path to settings (`openAppSettings()`).

### A-PLAY-03 No deceptive or interruptive UI. Major. PLAY (Deceptive behavior, Ads)
- Interstitial ads must not appear on launch, on exit or in the middle of a task. No fake system dialogs, no disguised close buttons, no countdowns on dismiss.
- **Detect:** `InterstitialAd`/`AppOpenAd` shown in `initState`, splash or on back press. Custom dialogs styled like system permission prompts.

### A-PLAY-04 Notifications are relevant and channelled. Minor. CAQ Notification_Quality
- **Detect:** `flutter_local_notifications` with a single `AndroidNotificationChannel` for all categories. Promotional notifications.
- **Fix:** one channel per category with appropriate importance. Group stacked notifications. Use `MessagingStyle` for chat.
