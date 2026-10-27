# Flutter UX audit skills

Claude Code skills that audit a Flutter app's screens and navigation against platform UI/UX guidelines. Each one reports findings with `file:line`, a severity, and a ready-to-paste fix prompt for an AI agent.

| Skill | Checks against |
|---|---|
| [`flutter-android-ux-audit`](skills/flutter-android-ux-audit) | Android Core App Quality, Adaptive App Quality (Tier 3), Android 16 behavior changes, edge-to-edge, navigation principles, Material 3, Google Play policy |
| [`flutter-ios-ux-audit`](skills/flutter-ios-ux-audit) | Apple Human Interface Guidelines (iOS/iPadOS), App Store Review Guidelines (UI-related items) |

Each skill has a `SKILL.md` (the steps) and a `RULES.md` (rule IDs, severity, source, Dart search patterns, Flutter fix).

## Install

```sh
git clone https://github.com/lmiadowicz/flutter-ux-audit-skills.git
cp -R flutter-ux-audit-skills/skills/* ~/.claude/skills/
```

To install for a single project instead, copy them into that repo's `.claude/skills/`.

## Use

In a Flutter repo, run `/flutter-android-ux-audit` or `/flutter-ios-ux-audit`, or ask "audit this app against Apple HIG". The audit is read-only. Fixes are applied only when you ask.

Guidelines were checked in October 2026. Re-verify the rules when Android, iOS or Material release major updates.
