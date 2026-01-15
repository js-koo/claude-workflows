---
allowed-tools: Bash(gh:*), Bash(git:*)
argument-hint: [help|PR number]
description: PR review comment handler
---

# PR Resolver

Handle PR review comments with replies and reactions.

## Default Configuration

Use these defaults throughout the flow:

```
Actions:
  fixed:          reply + 👍 (+1)
  will_fix_later: reply + 👀 (eyes)
  explain:        reply only
  disagree:       reply only
  skip:           👍 only (+1)
  praise:         ❤️ only (heart)
```

## Command Routing

Check `$1` argument:

- If `$1` = "help" → Go to **Help Section**
- Otherwise → Go to **Main Flow** (treat $1 as PR number if numeric)

---

# Help Section

Display help information:

```
╔═══════════════════════════════════════════════════════════╗
║                    PR Resolver Help                        ║
╚═══════════════════════════════════════════════════════════╝

Usage:
  /pr-resolver [PR number]    - Handle PR review comments
  /pr-resolver help           - Show this help

Examples:
  /pr-resolver                - Auto-detect PR and handle comments
  /pr-resolver 2874           - Handle comments for PR #2874

Actions:
  fixed          - Code fixed (reply + 👍)
  will_fix_later - Address later (reply + 👀)
  explain        - Explain reason (reply only)
  disagree       - Disagree (reply only)
  skip           - Already resolved (👍 only)
  praise         - Respond to praise (❤️ only)
```

After displaying help, exit.

---

# Main Flow

## Environment Check

1. Check git repo: !`git rev-parse --git-dir 2>/dev/null || echo "NOT_GIT_REPO"`
2. Check gh auth: !`gh auth status 2>&1 | head -3`
3. Remote info: !`git remote -v | head -2`

- If not a git repo: Print "❌ Please run in a git repository." and exit
- If gh not authenticated: Print "❌ Please run `gh auth login` first." and exit

## PR Detection

1. If `$1` is numeric → Use as PR number
2. If `$1` is empty → Run `gh pr view --json number -q '.number' 2>/dev/null`
3. On failure → Ask user: "Enter PR number:"

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

Display:
```
📋 Found {count} comment(s)

| # | File | Content |
|---|------|---------|
| 1 | Repository.kt:64 | SQL injection risk... |
```

If no comments: Print "✅ No comments to process." and exit

## User Selection

Use AskUserQuestion:

### Step 1: Select Comment
Question: "Which comment to handle?"
Options: Comment numbers (1, 2, 3...)

### Step 2: Select Type
Question: "What type of comment is this?"
Options:
- 🔴 Bug/Issue - Points out a bug or problem
- 🟡 Suggestion - Improvement suggestion
- 🔵 Question - Asking for clarification
- 🟢 Praise - Positive feedback
- ⚪ Other - Other type

### Step 3: Select Action
Question: "How do you want to handle this?"
Options:
- Fixed - Reply with fix confirmation + 👍
- Will fix later - Reply with acknowledgment + 👀
- Explain - Reply with explanation
- Disagree - Reply with disagreement
- Skip - Mark as resolved (👍 only)
- Praise response - Thank reviewer (❤️ only)

## Reply Generation

### If action is "fixed" or "will_fix_later":

1. Analyze original comment content
2. Generate appropriate reply based on action
3. Get commit hash: !`git rev-parse --short HEAD`
4. Ask: "Is this the correct commit? {hash}"
5. Detect comment language → Match reply language
6. Show suggested reply
7. Ask: [Send] [Edit] [Cancel]

### If action is "explain" or "disagree":

1. Ask: "Enter your reply:"
2. Get user input
3. Confirm content

### If action is "skip" or "praise":

Skip reply generation, proceed to reaction only

## Send

### Send reply (if action requires reply):
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies -f body="{reply}"
```

### Add reaction (based on action):
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/reactions -f content="{reaction}"
```

Reaction mapping:
- fixed: +1
- will_fix_later: eyes
- skip: +1
- praise: heart

### Display result:
```
✅ Reply sent successfully!
   Comment ID: {id}
   Reaction: {emoji}
```

## Repeat

Ask: "Handle another comment?"
Options: [Yes] [No]

- Yes → Go back to Comment Retrieval
- No → Print "👋 Done!" and exit

## Error Handling

| Situation | Message |
|-----------|---------|
| Not a git repo | ❌ Please run in a git repository. |
| gh not authenticated | ❌ Please run `gh auth login` first. |
| PR not found | ❌ PR not found. Please enter a valid PR number. |
| No comments | ✅ No comments to process. |
| API failure | ❌ GitHub API error: {message} |
| User cancelled | 👋 Cancelled. |
