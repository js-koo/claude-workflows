---
allowed-tools: Bash(gh:*), Bash(git:*)
argument-hint: [PR number]
description: Check PR review comments and handle replies
---

# PR Resolver

Check PR review comments and handle replies.

## Environment Check

First, verify the execution environment:
1. Check git repo: !`git rev-parse --git-dir 2>/dev/null || echo "NOT_GIT_REPO"`
2. Check gh auth: !`gh auth status 2>&1 | head -3`
3. Remote info: !`git remote -v | head -2`

- If not a git repo: Print "❌ Please run in a git repository." and exit
- If gh not authenticated: Print "❌ Please run `gh auth login` first." and exit

## PR Detection

User argument: $1

1. If `$1` exists → Use as PR number
2. If `$1` is empty → Run `gh pr view --json number -q '.number' 2>/dev/null`
3. On failure → Ask user for PR number input

## Comment Retrieval

Extract repo info:
```bash
gh repo view --json owner,name -q '"\(.owner.login)/\(.name)"'
```

Retrieve comments:
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --jq '.[] | {id, path, body: .body[0:100]}'
```

Display comment list in table format:
```
📋 Review comments: {n}

| # | File | Content |
|---|------|---------|
| 1 | Repository.kt:64 | SQL injection risk... |
| 2 | Service.kt:176 | check() validation... |
```

If no comments: Print "✅ No review comments to process." and exit

## User Selection

Use AskUserQuestion to ask sequentially:

### Step 1: Select Comment
"Which comment do you want to handle?"
- Options: Comment numbers (1, 2, 3...)

### Step 2: Select Type
"What is the type of this comment?"
- 🔴 Bug/Issue - Problem that needs fixing
- 🟡 Suggestion - Improvement suggestion (optional)
- 🔵 Question - Needs explanation
- 🟢 Praise/Approval - LGTM type
- ⚪ Other

### Step 3: Select Action
"How do you want to handle it?"
- Fixed - Code has been fixed (reply + 👍)
- Will fix later - Not now (reply + 👀)
- Explain - Explain the reason (reply only)
- Disagree - Do not agree (reply only)
- Skip - Already resolved (👍 only)
- Respond to praise - Express thanks (❤️ only)

## Reply Generation

### If action is "Fixed" or "Will fix later":

1. Analyze original comment content
2. Suggest appropriate reply based on action
3. If commit hash needed: Run `git rev-parse HEAD --short`
4. Confirm "Is this the correct commit? {hash}"
5. Detect comment language (Korean/English) → Write reply in same language
6. Suggest reply and ask user for confirmation/edit

Example (Fixed, Korean):
```
💬 Suggested reply:
"Fixed with parameter binding approach. (commit: abc1234)"

[Send] [Edit] [Cancel]
```

Example (Will fix later, English):
```
💬 Suggested reply:
"Thank you for the suggestion. Will address this in a future update."

[Send] [Edit] [Cancel]
```

### If action is "Explain" or "Disagree":

1. Ask user to directly input reply content
2. Confirm the input content

### If action is "Skip" or "Respond to praise":

Add reaction only without generating reply

## Send

### Send reply (except Skip, Respond to praise):
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies -f body="{reply}"
```

### Add reaction:
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/reactions -f content="{reaction}"
```

Reaction mapping:
| Action | reaction |
|--------|----------|
| Fixed | +1 |
| Will fix later | eyes |
| Skip | +1 |
| Respond to praise | heart |
| Explain/Disagree | (none) |

Display result after sending:
```
✅ Reply sent successfully!
   Comment: #{comment_id}
   Reaction: 👍
```

## Repeat

After sending, ask with AskUserQuestion:
"Do you want to handle another comment?"
- Yes → Go back to comment list display
- No → Print "👋 Exiting PR Resolver." and exit

## Error Handling

| Situation | Message |
|-----------|---------|
| Not a git repo | ❌ Please run in a git repository. |
| gh not authenticated | ❌ Please run `gh auth login` first. |
| PR not found | ❌ Cannot find PR. Please enter the PR number. |
| No comments | ✅ No review comments to process. |
| API failure | ❌ GitHub API error: {error message} |
| User cancelled | 👋 Cancelled. |
