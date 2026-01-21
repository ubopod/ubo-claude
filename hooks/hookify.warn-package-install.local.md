---
name: warn-package-install
enabled: true
event: bash
pattern: (npm|yarn|pnpm)\s+install|pip\s+install|brew\s+install|apt(-get)?\s+install
action: warn
---

⚠️ **Package installation detected**

You're about to install packages. Please confirm:

- [ ] The package name is correct
- [ ] The package is from a trusted source
- [ ] This dependency is necessary for the project
- [ ] Version constraints are appropriate (if specified)

**Consider:**
- Check package popularity and maintenance status
- Review package.json/requirements.txt changes after install
- Audit dependencies periodically with `npm audit` or similar

Proceed if this installation is intentional and approved.
