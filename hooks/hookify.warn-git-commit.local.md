---
name: warn-git-commit
enabled: true
event: bash
pattern: ^git\s+commit
action: warn
---

⚠️ **Git commit detected**

You're about to create a git commit. Please verify:

- [ ] All changes have been reviewed
- [ ] Commit message is descriptive
- [ ] No sensitive data is being committed
- [ ] Tests pass (if applicable)

If this is intentional, proceed. Otherwise, cancel and review changes first.
