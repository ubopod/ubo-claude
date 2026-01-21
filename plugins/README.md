# Plugins and Marketplaces

Plugins extend Claude Code with new tools and capabilities. This guide covers installation only - see the [full article](https://x.com/affaanmustafa/status/2012378465664745795) for when and why to use them.

---

## Marketplaces

Marketplaces are repositories of installable plugins.

### Adding a Marketplace

```bash
# Add official Anthropic marketplace
claude plugin marketplace add https://github.com/anthropics/claude-plugins-official

# Add community marketplaces
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep
```

### Recommended Marketplaces

| Marketplace | Source |
|-------------|--------|
| claude-plugins-official | `anthropics/claude-plugins-official` |
| claude-code-plugins | `anthropics/claude-code` |
| Mixedbread-Grep | `mixedbread-ai/mgrep` |

---

## Installing Plugins

```bash
# Open plugins browser
/plugins

# Or install directly
claude plugin install pyright-lsp@claude-plugins-official
```

### Recommended Plugins

**Development:**
- `pyright-lsp` - Python type checking
- `hookify` - Create hooks conversationally
- `code-simplifier` - Refactor code
- `feature-dev` - Feature development

**Code Quality:**
- `code-review` - Code review
- `security-guidance` - Security checks

**Integrations:**
- `github` - GitHub integration
- `figma` - Figma design integration

**Claude Code Management:**
- `claude-md-management` - CLAUDE.md file management
- `claude-code-setup` - Claude Code setup and configuration

---

## Quick Setup

```bash
# Add marketplaces
claude plugin marketplace add https://github.com/anthropics/claude-plugins-official
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Open /plugins and install what you need
```

---

## Plugin Files Location

```
~/.claude/plugins/
|-- cache/                    # Downloaded plugins
|-- installed_plugins.json    # Installed list
|-- known_marketplaces.json   # Added marketplaces
|-- marketplaces/             # Marketplace data
```