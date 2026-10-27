# Apple HIG / App Store UI-UX rules for Flutter (iOS & iPadOS)

Each rule has an ID, a **severity**, its source, what it means in Flutter, how to **detect** it, and how to **fix** it.
Detection patterns are *candidates*. Read the code around every hit before you call it a finding.

**Severity**
- **Blocker**: likely App Store rejection, or breaks a core iOS interaction (such as swipe-back).
- **Major**: a clear HIG best practice that iOS users notice as "not an iPhone app".
- **Minor**: a recommendation or polish.

**Sources** (checked 2026-10; HIG is at https://developer.apple.com/design/human-interface-guidelines/<page>)
- HIG pages: tab-bars, toolbars, sheets, modality, alerts, action-sheets, layout, typography, accessibility, buttons, gestures, dark-mode, color, materials, launching, onboarding, loading, text-fields, writing, right-to-left, going-full-screen
- ASRG: App Store Review Guidelines, https://developer.apple.com/app-store/review/guidelines/

**Flutter context.** `MaterialApp` on iOS already uses Cupertino page transitions (which support swipe-back), Cupertino scroll physics and the SF font. The goal is not "rewrite in Cupertino". The goal is that navigation, modals and system interactions behave as iOS users expect. Prefer *adaptive* APIs: `showAdaptiveDialog`, `AlertDialog.adaptive`, `Switch.adaptive`, `CircularProgressIndicator.adaptive`, `Icons.adaptive.*`, `Theme.of(context).platform`.

---

## I-NAV: Tab bar and hierarchy

### I-NAV-01 The tab bar is for navigation, never actions. Major. HIG tab-bars
- A tab that opens a sheet, camera, composer or "+" flow is an action. Put it in a toolbar button or an in-content button.
- **Detect:** `rg -n "BottomNavigationBar\(|NavigationBar\(|CupertinoTabBar\(" lib`. Read `onTap`/`onDestinationSelected` for branches that `push`, `showModalBottomSheet`, `showCupertinoModalPopup` or `showDialog` instead of switching the index.
- **Fix:** keep tabs as sections. Move the action to the AppBar `actions:` / `CupertinoNavigationBar(trailing:)`, or to a prominent button on the relevant tab.

### I-NAV-02 The tab bar stays visible while navigating within a section. Major. HIG tab-bars
- Pushing a detail screen must not hide the tab bar. The exception is modals, which cover it on purpose. In Flutter, the tab bar disappears when detail routes are pushed on the *root* navigator.
- **Detect:** `Navigator.of(context, rootNavigator: true).push`, go_router routes defined outside the `ShellRoute`/`StatefulShellRoute`, or a single `Navigator` with tabs as a `body` switch.
- **Fix:** give each tab its own navigator (`StatefulShellRoute.indexedStack`, `CupertinoTabScaffold` + `CupertinoTabView`, or `AutoTabsRouter`). Push details onto the tab's navigator. Use the root navigator only for full-screen modals.

### I-NAV-03 Tab count and overflow. Minor. HIG tab-bars
- Use as few tabs as necessary. On iPhone, more than 5 causes a "More" overflow, which HIG says to avoid. On iPad with many sections, prefer a sidebar.
- **Detect:** more than 5 items. A hand-rolled "More" tab.
- **Fix:** merge sections, or move secondary sections into a tab (for example a Settings list).

### I-NAV-04 Tabs are never disabled or hidden conditionally. Major. HIG tab-bars
- **Detect:** destinations built with `if (user.isPremium) ...` or `if (featureFlag)`, or with `enabled: false`/greyed styling.
- **Fix:** always show the tab. When the content isn't available, show an explanatory empty state inside it.

### I-NAV-05 Tab items have short labels and familiar icons. Minor. HIG tab-bars
- Labels are single words where possible. Icons are filled (SF Symbols style). Badges are only for critical, actionable information.
- **Detect:** `label: ''`, `showUnselectedLabels: false`, `labelBehavior: NavigationDestinationLabelBehavior.alwaysHide`, multi-word labels, `Badge` on several tabs at once.
- **Fix:** show labels. Use `CupertinoIcons` filled variants, or the `selectedIcon` filled variant on Material.

### I-NAV-06 Tapping the active tab returns to the root / scrolls to the top. Minor. iOS platform convention
- **Detect:** tab `onTap` that ignores `index == currentIndex`.
- **Fix:** with go_router `StatefulShellRoute`, call `navigationShell.goBranch(index, initialLocation: index == navigationShell.currentIndex)`. Otherwise pop the tab navigator to its first route, or `animateTo(0)` its `ScrollController`.

### I-NAV-07 iPad: tab bar at the top or a sidebar, not a stretched phone bar. Major. HIG tab-bars (iPadOS), layout
- **Detect:** iPad supported (`UIDeviceFamily` includes 2 / `TARGETED_DEVICE_FAMILY = "1,2"` in `ios/Runner.xcodeproj/project.pbxproj`) but the layout uses a bottom bar at every width.
- **Fix:** at ≥600pt width (regular size class) use a `NavigationRail`/sidebar (`Row` + list), or a top tab bar. Keep the same destinations.

---

## I-BAR: Navigation bars and toolbars

### I-BAR-01 The standard back control and edge swipe-back work on every pushed screen. Blocker. HIG toolbars, gestures
- Users swipe from the left edge to go back. In Flutter, the following break it:
  - `WillPopScope`, or `PopScope(canPop: false)` without a need
  - a custom `PageRouteBuilder` used for normal pushes
  - `PageTransitionsTheme` forcing a non-Cupertino builder for `TargetPlatform.iOS`
  - full-screen horizontal gesture widgets at the left edge
- Use a chevron back button, not a "Back" text label, and not an Android arrow.
- **Detect:** `rg -n "WillPopScope|PopScope\(|PageRouteBuilder|pageTransitionsTheme|FadeUpwardsPageTransitionsBuilder|ZoomPageTransitionsBuilder|Icons\.arrow_back\b|Text\('Back'\)" lib`.
- **Fix:** use `MaterialPageRoute`/`CupertinoPageRoute` (or go_router's default pages) for pushes. Map `TargetPlatform.iOS: CupertinoPageTransitionsBuilder()`. Use `PopScope(canPop: !dirty)` only on screens with unsaved input. Replace `Icons.arrow_back` with `Icons.adaptive.arrow_back` or rely on the AppBar's automatic `BackButton`.

### I-BAR-02 Every screen has a useful, concise title, and it isn't the app name. Minor. HIG toolbars
- Top-level screens use a large title that collapses on scroll. Titles are centred on iOS.
- **Detect:** `AppBar(title: Text('<AppName>'))` on several screens. `centerTitle: false` forced app-wide. Top-level screens with no title.
- **Fix:** title each screen by its content. For top-level lists use `SliverAppBar.large`/`CupertinoSliverNavigationBar`. Leave `centerTitle` at its platform default.

### I-BAR-03 Toolbar items are few, clear and placed conventionally. Minor. HIG toolbars
- Leading side: back/cancel. Trailing side: primary actions such as Done. Done/Save/Submit uses the prominent style. Extra actions go in a "More" menu.
- **Detect:** AppBar `actions:` with more than 3 icons. Done/Save on the leading side. Text-labelled and icon actions mixed in one group.
- **Fix:** keep 1–2 trailing actions plus a `PopupMenuButton`/`PullDownButton` (`pull_down_button` package if already used) for the rest.

---

## I-MOD: Modality, sheets, alerts, action sheets

### I-MOD-01 Modals are used only when they add value, with an obvious way out. Major. HIG modality
- Every modal has a visible dismiss control (Cancel/Close in the top bar) *and* swipe-down where the presentation allows it. Don't stack modals: dismiss one before presenting another. Don't build an "app within the app" inside a modal.
- **Detect:** `showModalBottomSheet(... isDismissible: false, enableDrag: false)`. `fullscreenDialog: true` routes without a close button. `showDialog`/sheets called from inside a sheet's builder.
- **Fix:** add a Cancel/close button. Keep `enableDrag: true`. Present the second step inside the same sheet (a nested `Navigator`) or after `Navigator.pop`.

### I-MOD-02 Sheet buttons: Cancel on the leading side, Done on the trailing side, never Done alone. Major. HIG sheets
- A sheet with unsaved changes asks for confirmation (via an action sheet) when dismissed by swipe. Resizable sheets show a grabber. On iPhone, consider a medium detent.
- **Detect:** sheet headers with only a Done/Save button, a close X on the trailing side next to Done, or all three of Cancel, Done and Back together. Missing `showDragHandle: true`. Sheets with forms and no `PopScope`.
- **Fix:** header `Row`/`CupertinoNavigationBar(leading: Cancel, middle: title, trailing: Done (bold))`. Use `showDragHandle: true` and `DraggableScrollableSheet(initialChildSize: 0.5)` for the medium detent. Use `PopScope(canPop: !dirty, onPopInvokedWithResult: ... showCupertinoModalPopup(CupertinoActionSheet(... 'Discard Changes' isDestructiveAction)))`.

### I-MOD-03 Alerts are rare, critical and correctly worded. Major. HIG alerts
- Don't show alerts on launch, for purely informational messages, or to confirm common undoable actions. Use at most 3 buttons. Avoid "OK" unless the alert is purely informational. The cancel button is titled "Cancel". Destructive buttons use the destructive style and never the default/primary role.
- **Detect:** `rg -n "showDialog|AlertDialog\(|CupertinoAlertDialog" lib`. Check: called from `initState`/splash; content that is just a success message; `TextButton(child: Text('OK'))` on confirmations; `Yes`/`No` buttons; destructive actions not marked.
- **Fix:** use `showAdaptiveDialog` + `AlertDialog.adaptive` with `adaptiveAction` helpers, or `CupertinoDialogAction(isDestructiveAction: true)`. Retitle buttons with verbs ("Delete", "Cancel"). Replace informational alerts with inline status, a banner or a snackbar. Replace undoable confirmations with an undo action.

### I-MOD-04 Use an action sheet, not an alert or a Material menu, for choices about a user-initiated action. Minor. HIG action-sheets, alerts
- Examples: "Delete photo?", "Take photo / Choose from library". Include Cancel. Make destructive choices prominent. The sheet must not scroll.
- **Detect:** `showDialog` with a list of choices. `showModalBottomSheet` with `ListTile` options on iOS.
- **Fix:** on iOS, `showCupertinoModalPopup(builder: (_) => CupertinoActionSheet(actions: [...], cancelButton: CupertinoActionSheetAction(child: Text('Cancel'), ...)))`, branching on `Theme.of(context).platform`.

---

## I-LAY: Layout and adaptivity

### I-LAY-01 Respect safe areas: Dynamic Island, notch, home indicator. Major. HIG layout
- **Detect:** `Scaffold`s without an `AppBar` where top content isn't in a `SafeArea`. Bottom-pinned buttons without `SafeArea`/`MediaQuery.paddingOf(context).bottom`. Hard-coded `top: 44/47/59`. `SystemChrome.setEnabledSystemUIMode(SystemUiMode.immersive...)` outside media or games.
- **Fix:** use `SafeArea` (or `minimum:` padding). Let scroll content extend under bars, but inset interactive elements.

### I-LAY-02 Adapt to size classes and orientation, not device type. Major. HIG layout
- Keep functionality the same at every size. iPad layouts use the extra space (multi-column, sidebar). They don't stretch a phone layout.
- **Detect:** `Platform.isIOS && MediaQuery.size.width > 700`-style device sniffing. `setPreferredOrientations([portraitUp])` on an iPad-supporting app. `UISupportedInterfaceOrientations~ipad` limited in `Info.plist`. No `LayoutBuilder`/breakpoints. Full-width buttons and text on iPad.
- **Fix:** branch on `MediaQuery.sizeOf(context).width` (≥600 regular). Constrain reading width (`ConstrainedBox(maxWidth: 600–700)`). Support all orientations on iPad (required for iPad multitasking).

### I-LAY-03 Gestures don't conflict with system gestures and always have an alternative. Major. HIG gestures, accessibility
- **Detect:** horizontal `GestureDetector`/`PageView`/`Dismissible` starting at the left edge (conflicts with back swipe). Bottom-edge swipe handlers (conflicts with the home indicator). `Dismissible` or swipe actions with no button or menu alternative.
- **Fix:** inset the gesture area. Add an equivalent button or context menu (`CupertinoContextMenu`, trailing menu) for every swipe action.

### I-LAY-04 RTL support. Minor. HIG right-to-left
- **Detect:** `EdgeInsets.only(left:`/`right:`, `Alignment.centerLeft`, `Positioned(left:` in layout code. `Icons.arrow_back`/`chevron_right` without `matchTextDirection`.
- **Fix:** use `EdgeInsetsDirectional`, `AlignmentDirectional`, `PositionedDirectional`, `Icons.adaptive.*` or `Icon(..., textDirection)`. Only flip icons that represent direction.

---

## I-TYP: Typography and Dynamic Type

### I-TYP-01 Support Dynamic Type through the accessibility sizes. Major. HIG typography, accessibility
- **Detect:** `rg -n "textScaleFactor|textScaler|TextScaler\.(noScaling|linear)" lib`, or a global `MediaQuery` override to 1.0. Fixed-height containers around text. `maxLines: 1` on important text without wrapping. `FittedBox` used to shrink text back.
- **Fix:** don't disable scaling. If you must, clamp only the maximum (≥ 2.0) with `textScaler.clamp(maxScaleFactor: ...)`. Use min-height constraints and allow wrapping. At very large sizes, switch horizontal rows to vertical stacks.

### I-TYP-02 Legible sizes and weights. Minor. HIG typography
- Body text defaults to 17pt. Nothing is smaller than 11pt. Avoid light/thin weights for UI text. Use few typefaces.
- **Detect:** `fontSize:` below 11. `FontWeight.w100–w300` on body text. More than 2 font families.
- **Fix:** use `Theme.of(context).textTheme` / `CupertinoTheme.of(context).textTheme` roles instead of literal sizes.

---

## I-A11Y: Accessibility

### I-A11Y-01 Hit targets are at least 44×44pt (absolute minimum 28×28), with spacing. Major. HIG accessibility, buttons
- Leave about 12pt of padding around bezelled controls and about 24pt around borderless ones.
- **Detect:** `MaterialTapTargetSize.shrinkWrap`, `VisualDensity.compact`, `IconButton(padding: EdgeInsets.zero, constraints: BoxConstraints())`, `CupertinoButton(minSize: <44)` (`minimumSize` in newer Flutter), `GestureDetector` around small `Icon`s, tightly packed icon rows.
- **Fix:** ensure a 44×44 hit area (`SizedBox.square(dimension: 44)` / padding) while keeping the visual size.

### I-A11Y-02 Contrast: 4.5:1 for text up to 17pt, 3:1 for ≥18pt or bold. Aim for 7:1 on custom small text. Major. HIG accessibility, dark-mode
- **Detect:** grey or low-opacity text literals, white on light brand colours, text over images without a scrim. Check both light and dark.
- **Fix:** use semantic colours (`CupertinoColors.label.resolveFrom(context)`, `secondaryLabel`, `colorScheme.onSurface`). Add a scrim behind text on imagery.

### I-A11Y-03 VoiceOver labels and roles. Major. HIG accessibility
- **Detect:** `IconButton` without `tooltip`. `Image` without `semanticLabel`. `GestureDetector`/`InkWell` buttons without `Semantics(button: true, label:)`. Meaningful custom-painted widgets without `Semantics`.
- **Fix:** add labels. Mark decorative images `excludeFromSemantics: true`. Merge related content with `MergeSemantics`.

### I-A11Y-04 Reduce Motion and no colour-only meaning. Minor. HIG accessibility, color
- **Detect:** looping, parallax or large animations that don't check `MediaQuery.disableAnimationsOf(context)`. Status conveyed only by colour.
- **Fix:** gate animations on `disableAnimations`, using a crossfade instead. Add icons or text to colour cues.

### I-A11Y-05 Custom buttons have a pressed state and async feedback. Minor. HIG buttons
- **Detect:** `GestureDetector(onTap:)` styled as a button with no pressed or highlight feedback. Async submit buttons with no in-button progress or disable while running.
- **Fix:** use `CupertinoButton`/`FilledButton` (they have built-in pressed states). Show `CircularProgressIndicator.adaptive()` in the button while waiting. Give only the most likely action the prominent style. Never give it to a destructive action.

---

## I-APP: Appearance

### I-APP-01 Light and Dark Mode are both supported, following the system. Major. HIG dark-mode
- HIG advises against an app-only appearance switch. If one exists, it must default to System.
- **Detect:** no `darkTheme` / `themeMode: ThemeMode.light`. `CupertinoThemeData(brightness: Brightness.light)` forced. Literal `Colors.white` backgrounds and `Colors.black` text.
- **Fix:** add a dark theme with `ThemeMode.system`. Use dynamic colours (`CupertinoDynamicColor`, `colorScheme` roles). Soften pure-white backgrounds.

### I-APP-02 Translucent bars and iOS 26 Liquid Glass. Minor (advisory). HIG materials, color, tab-bars
- Content should scroll under the floating tab bar and toolbars rather than stop at an opaque slab. Don't use glass/blur effects in the content layer. Avoid tab and toolbar label colours that match the content backgrounds behind them.
- Flutter draws its own bars. Check how closely the app's Cupertino widgets follow current iOS visuals, and note it rather than mandate custom glass.
- **Detect:** `extendBody: false` with an opaque, heavily tinted bottom bar. `BackdropFilter` blur on content cards.
- **Fix:** use `extendBody: true` with bottom padding on lists. Reserve blur for bars. Keep bar tints subtle.

---

## I-FLOW: Launch, onboarding, loading, input, copy

### I-FLOW-01 Launch screen: plain and like the first screen, with no text, ads or long splash. Minor. HIG launching
- **Detect:** `ios/Runner/Base.lproj/LaunchScreen.storyboard` containing labels or marketing. Artificial `Future.delayed` on a Flutter splash route. Splash screens shown on every launch.
- **Fix:** keep the launch screen to background plus an optional logo (`flutter_native_splash`). Remove artificial delays. Restore the last state on relaunch.

### I-FLOW-02 Onboarding is short and skippable. Permissions, ratings and paywalls are deferred. Major. HIG onboarding, ASRG 5.1.1
- **Detect:** multi-page onboarding with no Skip. Permission `request()` calls in onboarding or `main()` unrelated to the core function. `requestReview()` / paywall shown before the user has used the app. Licence text in onboarding.
- **Fix:** add Skip. Request each permission at its moment of use, after a short explanation (any pre-prompt must not mislead, and its button should say "Continue", not mimic "Allow"). Prompt for a review only after a success moment.

### I-FLOW-03 Show content fast, and indicate loading. Minor. HIG loading
- **Detect:** full-screen blocking spinners for list loads. A Material `CircularProgressIndicator()` on iOS. Blank screens while fetching.
- **Fix:** use skeletons/placeholders or cached content. `CircularProgressIndicator.adaptive()`. Keep the rest of the app usable.

### I-FLOW-04 Text fields use the right keyboard, autofill, secure entry and a clear button. Minor. HIG text-fields
- **Detect:** `TextField`/`TextFormField` for email, phone, numbers or passwords without `keyboardType`, `autofillHints`, `obscureText` or `textInputAction`. No `AutofillGroup` on login. Long forms without next-field order.
- **Fix:** add `keyboardType: TextInputType.emailAddress`, `autofillHints: [AutofillHints.email]`, `textInputAction: TextInputAction.next`, a clear suffix button (`CupertinoTextField(clearButtonMode: OverlayVisibilityMode.editing)`), and wrap the form in `AutofillGroup`.

### I-FLOW-05 Empty states and errors tell the user what to do next. Settings are linked, not described. Minor. HIG writing
- **Detect:** empty lists rendering nothing or "No data". Raw exception text shown to users. "Go to Settings > Privacy > ..." prose.
- **Fix:** add an empty state with an explanation and an action. Write human error messages with a retry. Use `openAppSettings()` (permission_handler) or `app_settings`.

---

## I-STORE: App Store Review items that show up in the UI

### I-STORE-01 In-app account deletion. Blocker if sign-up exists. ASRG 5.1.1(v)
- **Detect:** account creation exists, but there's no "Delete Account" in Settings/Profile (a link to email support isn't enough).
- **Fix:** add a discoverable "Delete Account" flow that starts deletion in the app.

### I-STORE-02 Sign in with Apple, when using third-party social login. Blocker. ASRG 4.8
- **Detect:** `google_sign_in`, `flutter_facebook_auth` and similar packages, but no `sign_in_with_apple` (unless the 4.8 exceptions apply).
- **Fix:** add Sign in with Apple as an equivalent option. Follow Apple's button style rules (`SignInWithAppleButton`).

### I-STORE-03 It isn't a thin web wrapper and it feels native. Major. ASRG 4.2
- **Detect:** the main screens are mostly a `WebView`. Android-only patterns everywhere on iOS: Material back arrows, FAB-heavy UI, `Toast`s, a drawer as the only navigation.
- **Fix:** replace core flows with native widgets. Adapt the navigation and controls listed above for iOS.

### I-STORE-04 Every permission has a specific purpose string. Blocker. ASRG 5.1.1
- **Detect:** in `ios/Runner/Info.plist`, `NS*UsageDescription` keys missing for permissions the plugins use, or vague text ("This app needs camera access").
- **Fix:** write purpose strings that say what the data is used for and why ("Used to scan receipts you add to expenses").
