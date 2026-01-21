---
name: warn-force-push
enabled: true
event: bash
pattern: git\s+push.*(-f|--force)
action: warn
---

⚠️ **Force push detected!**

You're about to force push, which can:
- Overwrite remote commit history
- Cause issues for other collaborators
- Lose commits that exist only on remote

**Consider:**
- Use `git push --force-with-lease` for safer force pushing
- Verify you're on the correct branch
- Ensure no one else is working on this branch

Proceed only if you understand the implications.
