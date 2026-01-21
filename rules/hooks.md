# Hooks System

## Hook Types

- **PreToolUse**: Before tool execution (validation, parameter modification)
- **PostToolUse**: After tool execution (auto-format, checks)
- **Stop**: When session ends (final verification)

## Current Hooks (in ~/.claude/hooks/*)

### PreToolUse
- **git push review**: Opens editor for review before push
- **doc blocker**: Blocks creation of unnecessary .md/.txt files

### PostToolUse
- **PR creation**: Logs PR URL and GitHub Actions status
- **ruff format**: Auto-formats Python files after edit
- **Type check**: Runs `uv run poe typecheck` after editing type-related files
- **Proto regen**: Reminds to run `uv run poe proto` after editing actions/events

### Stop
- **Lint audit**: Runs `uv run poe lint` on all modified files before session ends

## Recommended Hooks for ubo_app

### PostToolUse: Auto-format Python

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "uv run ruff format $CLAUDE_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

### PostToolUse: Proto Reminder

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "if echo $CLAUDE_FILE_PATH | grep -q 'store/.*\\.py'; then echo '⚠️  Remember: run uv run poe proto if you added/modified actions or events'; fi"
          }
        ]
      }
    ]
  }
}
```

### Stop: Final Lint Check

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "uv run poe lint"
      }
    ]
  }
}
```

## Auto-Accept Permissions

Use with caution:
- Enable for trusted, well-defined plans
- Disable for exploratory work
- Never use dangerously-skip-permissions flag
- Configure `allowedTools` in `~/.claude.json` instead

## TodoWrite Best Practices

Use TodoWrite tool to:
- Track progress on multi-step tasks
- Verify understanding of instructions
- Enable real-time steering
- Show granular implementation steps

Todo list reveals:
- Out of order steps
- Missing items
- Extra unnecessary items
- Wrong granularity
- Misinterpreted requirements

## ubo_app Specific Reminders

After modifying certain files, remember:

| File Pattern | Action Needed |
|--------------|---------------|
| `ubo_app/store/**/*.py` | May need `uv run poe proto` |
| `ubo_app/services/*/reducer.py` | Check type hints match state |
| `ubo_app/services/090-web-ui/web-app/**` | Run `npm run build` |
| `*.proto` | Run `uv run poe proto:compile` |
| `tests/**/*.py` | Run in Docker for snapshot consistency |
