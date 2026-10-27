---
name: flutter-android-ux-audit
description: Audit a Flutter app's screens and navigation against Android, Material 3 and Google Play UI/UX guidelines, and output findings with copy-paste fix prompts. Use when the user wants an Android, Material or Google Play UX/UI review, guideline check or compliance audit of a Flutter app.
---

Audit a Flutter codebase against [RULES.md](RULES.md) and produce a **finding** for every violation. Each finding comes with a self-contained **fix prompt** another AI agent can run without context. This is read-only: don't edit app code unless the user asks after the report.

## Steps

### 1. Ground the project
Find `pubspec.yaml`. Record:
- Flutter SDK constraint and the router package (go_router, auto_route, beamer, or plain `Navigator`)
- state management
- `targetSdk`/`minSdk` from `android/app/build.gradle(.kts)`
- whether `AndroidManifest.xml` and `styles.xml` exist

Read the repo's `AGENTS.md`/`CLAUDE.md` if present.
**Done when** all of these are written in your notes. targetSdk changes severities in RULES.md.

### 2. Map screens and navigation
Build a **navigation map**:
- every screen: each widget that returns a `Scaffold`, or is a route target
- route definitions
- top-level destinations (bar, rail, drawer)
- per-tab stacks
- modals (dialogs, bottom sheets, full-screen dialogs)
- auth/onboarding/splash gating
- deep-link entry points

Useful starting point: `rg -l "Scaffold\(" lib`, `rg -n "GoRoute\(|AutoRoute\(|MaterialPageRoute|onGenerateRoute|routes:" lib`.
**Done when** every file containing `Scaffold(` is assigned to a screen in the map, or explicitly marked as not a screen.

### 3. Apply every rule
Work through RULES.md section by section (A-NAV, A-BACK, A-E2E, A-LS, A-VIS, A-A11Y, A-PLAY). For each rule:
1. Run its **Detect** patterns.
2. Read each hit in context.
3. Check it against the navigation map.

A grep hit is only a candidate. It becomes a finding only after you've read the code and confirmed the violation. App-wide rules (theme, manifest, edge-to-edge) produce one finding, not one per screen.
**Done when** every rule ID in RULES.md has a status: `violated` (with findings), `passes`, or `n/a` (with a reason, e.g. "no sign-up flow").

### 4. Write the report
Output it in chat. Write it to a file only if the user asks, and never commit it. Structure:

1. **Summary**: counts by severity, then a screen-by-rule table (✅ / ❌ / n/a) covering the screens from step 2.
2. **Findings**: ordered by severity (Blocker, Major, Minor), then by screen. Each one:
   ```
   ### [A-XXX-NN] <short title> — <Severity>
   Screen / scope: <ScreenName or "app-wide">
   Where: lib/path/file.dart:LINE
   Evidence: <the offending code, ≤5 lines>
   Guideline: <one sentence + source name from RULES.md>
   Fix prompt:
   > <self-contained prompt, see below>
   ```
3. **Rule coverage**: one line per rule ID with its status from step 3.
4. **Fix-all prompt**: one combined prompt that lists all findings grouped by file, in severity order.

**Fix prompt contract.** Each prompt must stand alone, because it will be pasted into a fresh agent session. It names:
- the file and line, the widget and the rule ID
- the current behaviour and the target behaviour
- the concrete Flutter API to use (taken from the rule's **Fix**)
- any constraint from step 1 (router package, state management, Flutter version)
- an acceptance check, e.g. "`flutter analyze` passes; on a 600dp+ window a NavigationRail is shown with the same 4 destinations"

It never says "fix the issue above".

**Done when** every `violated` rule has at least one finding, every finding has a fix prompt that meets the contract, and coverage lists every rule ID.

### 5. Offer, don't apply
End with one line asking whether to apply fixes. If the user says yes, fix one finding group at a time and run `flutter analyze` (and `flutter test` if tests exist) after each group.

## Judgement calls
- Some apps deliberately keep the same UI on iOS and Android, such as a custom design system. Report deviations, but mark brand-driven ones as Minor and say "if intentional, record it as a design decision."
- Don't report generated code (`*.g.dart`, `*.freezed.dart`) or `test/`.
- If a rule depends on runtime behaviour you can't confirm from code (contrast on images, actual hit size after layout), report it as **Needs device check** with the steps to verify.
