---
allowed-tools: Bash(gh:*), Bash(git:*), Bash(cat:*), Bash(mkdir:*), Bash(cp:*)
argument-hint: [help|config|PR number]
description: PR review comment handler with i18n support
---

# PR Resolver

Handle PR review comments with replies and reactions.

## Initialization

### Load Config
1. Config path: `~/.claude-code-skills/config/pr-resolver.json`
2. If not exists, copy from default: `~/.claude-code-skills/config/pr-resolver.default.json`
3. Read config: !`cat ~/.claude-code-skills/config/pr-resolver.json 2>/dev/null || cat ~/.claude-code-skills/config/pr-resolver.default.json`

### Load i18n
1. Get language from config (default: "en")
2. Load i18n file: `~/.claude-code-skills/i18n/{lang}.json`
3. Read i18n: !`cat ~/.claude-code-skills/i18n/en.json`

Use the loaded i18n strings for all user-facing messages.

## Command Routing

Check `$1` argument:

- If `$1` = "help" → Go to **Help Section**
- If `$1` = "config" → Go to **Config Section**
- Otherwise → Go to **Main Flow** (treat $1 as PR number if numeric)

---

# Help Section

Display help information using i18n strings:

```
╔═══════════════════════════════════════════════════════════╗
║                    PR Resolver Help                        ║
╚═══════════════════════════════════════════════════════════╝

Usage:
  /pr-resolver [PR number]    - Handle PR review comments
  /pr-resolver help           - Show this help
  /pr-resolver config         - Show current configuration
  /pr-resolver config <key> <value> - Update configuration

Config Options:
  lang <en|ko>                - Set language
  action <name> <property> <value> - Configure actions
  reset                       - Reset to defaults

Examples:
  /pr-resolver                - Auto-detect PR and handle comments
  /pr-resolver 2874           - Handle comments for PR #2874
  /pr-resolver config lang ko - Switch to Korean
  /pr-resolver config action disagree disable - Disable disagree action
  /pr-resolver config action fixed reaction rocket - Change reaction

Actions:
  fixed         - Code fixed (reply + 👍)
  will_fix_later - Address later (reply + 👀)
  explain       - Explain reason (reply only)
  disagree      - Disagree (reply only)
  skip          - Already resolved (👍 only)
  praise        - Respond to praise (❤️ only)
```

After displaying help, exit.

---

# Config Section

Handle `$ARGUMENTS` after "config":

### Show Config (no additional args)
Display current configuration in readable format:

```
╔═══════════════════════════════════════════════════════════╗
║                 PR Resolver Configuration                  ║
╚═══════════════════════════════════════════════════════════╝

Language:              en
Auto-detect PR:        true
Auto-detect commit:    true
Confirm before send:   true
Preview length:        50

Actions:
  ✓ fixed           reply + 👍
  ✓ will_fix_later  reply + 👀
  ✓ explain         reply only
  ✓ disagree        reply only
  ✓ skip            👍 only
  ✓ praise          ❤️ only
```

### Update Config

**Language**: `/pr-resolver config lang <en|ko>`
- Update "lang" in config file
- Display: "✅ Language updated to {lang}"

**Action enable/disable**: `/pr-resolver config action <name> <enable|disable>`
- Update actions.{name}.enabled in config
- Display: "✅ Action '{name}' {enabled|disabled}"

**Action reaction**: `/pr-resolver config action <name> reaction <+1|eyes|heart|rocket|null>`
- Update actions.{name}.reaction in config
- Display: "✅ Action '{name}' reaction set to {reaction}"

**Action label**: `/pr-resolver config action <name> label <text>`
- Update actions.{name}.label in config
- Display: "✅ Action '{name}' label set to '{text}'"

**Reset**: `/pr-resolver config reset`
- Copy default config to user config
- Display: "✅ Configuration reset to defaults"

After config operation, exit.

---

# Main Flow

## Environment Check

1. Check git repo: !`git rev-parse --git-dir 2>/dev/null || echo "NOT_GIT_REPO"`
2. Check gh auth: !`gh auth status 2>&1 | head -3`
3. Remote info: !`git remote -v | head -2`

- If not a git repo: Print i18n.env.not_git_repo and exit
- If gh not authenticated: Print i18n.env.gh_not_auth and exit

## PR Detection

1. If `$1` is numeric → Use as PR number
2. If `$1` is empty → Run `gh pr view --json number -q '.number' 2>/dev/null`
3. On failure → Ask user for PR number using i18n.pr.enter_number

## Comment Retrieval

Extract repo info:
```bash
gh repo view --json owner,name -q '"\(.owner.login)/\(.name)"'
```

Retrieve comments:
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --jq '.[] | {id, path, body: .body[0:50], in_reply_to_id}'
```

Filter: Only show comments where `in_reply_to_id` is null (top-level comments)

Display using i18n strings:
```
📋 {i18n.comments.count}

| # | {i18n.comments.table_header_file} | {i18n.comments.table_header_content} |
|---|------|---------|
| 1 | Repository.kt:64 | SQL injection risk... |
```

If no comments: Print i18n.comments.no_comments and exit

## User Selection

Use AskUserQuestion with i18n strings:

### Step 1: Select Comment
Question: i18n.select.which_comment
Options: Comment numbers

### Step 2: Select Type
Question: i18n.select.comment_type
Options (from i18n.types, only show enabled):
- i18n.types.bug - i18n.types.bug_desc
- i18n.types.suggestion - i18n.types.suggestion_desc
- i18n.types.question - i18n.types.question_desc
- i18n.types.praise - i18n.types.praise_desc
- i18n.types.other - i18n.types.other_desc

### Step 3: Select Action
Question: i18n.select.how_to_handle
Options (from config.actions where enabled=true):
- Use config.actions.{name}.label if set, otherwise i18n.actions.{name}
- Show i18n.actions.{name}_desc

## Reply Generation

### If action is "fixed" or "will_fix_later":

1. Analyze original comment content
2. Generate appropriate reply based on action
3. If commit hash needed: !`git rev-parse HEAD --short`
4. Confirm with user: i18n.reply.confirm_commit
5. Detect comment language → Match reply language
6. Show: i18n.reply.suggested + suggested reply
7. Ask: [i18n.reply.send] [i18n.reply.edit] [i18n.reply.cancel]

### If action is "explain" or "disagree":

1. Ask: i18n.reply.enter_reply
2. Get user input
3. Confirm content

### If action is "skip" or "praise":

Skip reply generation, proceed to reaction only

## Send

### Send reply (if action has reply=true):
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies -f body="{reply}"
```

### Add reaction (if action has reaction set):
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/reactions -f content="{config.actions.{action}.reaction}"
```

### Display result:
```
{i18n.result.sent}
   {i18n.result.comment_id}
   {i18n.result.reaction}
```

## Repeat

Ask with AskUserQuestion:
Question: i18n.repeat.another
Options: [i18n.repeat.yes] [i18n.repeat.no]

- Yes → Go back to Comment Retrieval
- No → Print i18n.exit.bye and exit

## Error Handling

| Situation | Message |
|-----------|---------|
| Not a git repo | i18n.env.not_git_repo |
| gh not authenticated | i18n.env.gh_not_auth |
| PR not found | i18n.pr.not_found |
| No comments | i18n.comments.no_comments |
| API failure | i18n.error.api_error |
| User cancelled | i18n.exit.cancelled |
