---
allowed-tools: Bash(gh:*), Bash(git:*)
argument-hint: [help|config|PR number]
description: PR review comment handler
---

# PR Resolver

## Language Detection

Read language setting: !`git config --global pr-resolver.lang 2>/dev/null || echo "en"`

## Command Routing

Check `$1` argument:

- If `$1` = "config" → Go to **Config Section** below
- If `$1` = "help" or other → Route to language-specific file based on detected language

## Language Routing

For non-config commands, based on the language setting above:

- If language is `ko` → Follow instructions in **pr-resolver-ko.md**
- Otherwise (default `en`) → Follow instructions in **pr-resolver-en.md**

---

# Config Section

Configuration is stored using git config (global). This section handles all config commands regardless of language.

## Load Current Config

Read settings: !`git config --global --get-regexp '^pr-resolver\.' 2>/dev/null || echo ""`

## Show Config (no additional args after "config")

Display current configuration:

```
╔═══════════════════════════════════════════════════════════╗
║                 PR Resolver Configuration                  ║
╚═══════════════════════════════════════════════════════════╝

Language: {lang or "en (default)"}

Actions:
  ┌─────────────────┬─────────┬──────────────┐
  │ Action          │ Enabled │ Reaction     │
  ├─────────────────┼─────────┼──────────────┤
  │ fixed           │ ✓       │ 👍 (+1)      │
  │ will_fix_later  │ ✓       │ 👀 (eyes)    │
  │ explain         │ ✓       │ -            │
  │ disagree        │ ✓       │ -            │
  │ skip            │ ✓       │ 👍 (+1)      │
  │ praise          │ ✓       │ ❤️ (heart)   │
  └─────────────────┴─────────┴──────────────┘
```

Show actual values from git config, fall back to defaults if not set.

## Update Config

### Language: `/pr-resolver config lang <en|ko>`
```bash
git config --global pr-resolver.lang {value}
```
Display: "✅ Language set to {value}"

### Action enable/disable: `/pr-resolver config action <name> <enable|disable>`
```bash
git config --global pr-resolver.action.{name}.enabled {true|false}
```
Display: "✅ Action '{name}' {enabled|disabled}"

### Action reaction: `/pr-resolver config action <name> reaction <+1|eyes|heart|rocket|null>`
```bash
git config --global pr-resolver.action.{name}.reaction {value}
```
Display: "✅ Action '{name}' reaction set to {value}"

### Reset: `/pr-resolver config reset`
```bash
git config --global --remove-section pr-resolver 2>/dev/null || true
```
Display: "✅ Configuration reset to defaults"

After config operation, exit.
