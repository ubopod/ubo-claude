---
name: warn-dangerous-rm
enabled: true
event: bash
pattern: rm\s+(-rf|-fr|--recursive\s+--force|--force\s+--recursive)
action: warn
---

⚠️ **Dangerous rm command detected!**

You're using `rm -rf` which permanently deletes files without confirmation.

**Before proceeding, verify:**
- [ ] The path is correct
- [ ] No important files will be lost
- [ ] You have backups if needed

**Safer alternatives:**
- Use `rm -i` for interactive deletion
- Use `trash` command to move to trash instead
- Double-check the path with `ls` first

This warning exists to prevent accidental data loss.
