---
description: Sync documentation from source-of-truth files. Extract poe tasks, environment variables, and verify documentation currency.
---

# Update Documentation

This command syncs documentation from source-of-truth files in the ubo_app codebase.

## What This Command Does

1. **Extract poe tasks** from pyproject.toml
2. **Document environment variables** from tests/.env and code
3. **Verify CLAUDE.md accuracy** against current codebase
4. **Identify stale documentation**
5. **Generate changelog updates** if needed

## Source of Truth Files

| File | Contains |
|------|----------|
| `pyproject.toml` | poe tasks, dependencies, tool configs |
| `tests/.env` | Test environment variables |
| `CLAUDE.md` | AI assistant context and conventions |
| `README.md` | User-facing documentation |

## Workflow

### Step 1: Extract Poe Tasks Reference

Read `pyproject.toml` and generate a tasks reference:

```bash
# List all available poe tasks
uv run poe --help
```

**Current poe tasks categories:**

| Category | Tasks | Description |
|----------|-------|-------------|
| **Lint** | `lint`, `lint:fix` | Run ruff linter |
| **Type Check** | `typecheck` | Run pyright type checker |
| **Test** | `test` | Run pytest |
| **Sanity** | `sanity` | Run typecheck + lint + test |
| **Proto** | `proto`, `proto:generate`, `proto:compile` | Generate and compile protobuf files |
| **Docker** | `build-docker-images` | Build dev and test Docker images |
| **Web App** | `build-web-app` | Build the web UI application |
| **Device Deploy** | `device:deploy`, `device:deploy:complete` | Deploy to Raspberry Pi |
| **Device Test** | `device:test`, `device:test:complete` | Run tests on device |

### Step 2: Document Environment Variables

Extract from `tests/.env` and codebase:

| Variable | Purpose | Default |
|----------|---------|---------|
| `UBO_LOG_LEVEL` | Logging verbosity | `DEBUG` |
| `UBO_CACHE_PATH` | Cache directory path | `/tmp/ubo-test-cache` |
| `UBO_DATA_PATH` | Data directory path | `/tmp/ubo-test-data` |
| `UBO_DEBUG_VISUAL` | Enable visual debugging | `False` |
| `UBO_DEBUG_TASKS` | Debug async tasks | `False` |
| `UBO_DEBUG_DOCKER` | Debug Docker operations | `False` |
| `UBO_DEBUG_MENU` | Debug menu rendering | `False` |
| `UBO_TEST_INVESTIGATION_MODE` | Enable test investigation | `False` |
| `DOCKER_HOST` | Docker socket path | `/var/run/docker.sock` |

**Convention:** All ubo_app environment variables use the `UBO_` prefix.

### Step 3: Verify CLAUDE.md Accuracy

Check that CLAUDE.md reflects current:

- [ ] Service directory structure (000-xxx, 030-xxx, etc.)
- [ ] Available poe commands
- [ ] Testing procedures
- [ ] Protobuf workflow
- [ ] Deployment commands

```bash
# Verify service directories match documentation
ls -d ubo_app/services/0*/

# Verify poe tasks match documentation
grep -A 50 '\[tool.poe.tasks\]' pyproject.toml
```

### Step 4: Identify Stale Documentation

Find markdown files not modified recently:

```bash
# Find docs not modified in 90+ days
find . -name "*.md" -mtime +90 -type f | grep -v node_modules | grep -v .venv

# Check last modification dates
ls -la *.md
```

### Step 5: Verify README Sections

Ensure README.md contains current information for:

- [ ] Installation instructions
- [ ] Development setup (`uv sync --dev`)
- [ ] Docker testing commands
- [ ] Device deployment commands
- [ ] Service architecture description

## Quick Reference: Common Updates

### When adding a new poe task:

1. Add to `pyproject.toml` under `[tool.poe.tasks]`
2. Update this reference table
3. Update CLAUDE.md if it's a commonly-used command

### When adding a new environment variable:

1. Add to `tests/.env` if test-relevant
2. Use `UBO_` prefix
3. Document purpose in this file
4. Update CLAUDE.md conventions section

### When adding a new service:

1. Create directory with priority prefix (e.g., `030-newservice/`)
2. Update CLAUDE.md service list
3. Add to README architecture section if significant

## Validation Commands

```bash
# Verify pyproject.toml syntax
uv run poe --help

# Verify all services have required files
for dir in ubo_app/services/0*/; do
  echo "=== $dir ==="
  ls "$dir"/*.py 2>/dev/null | head -5
done

# Check for undocumented environment variables
grep -r "os.environ\|os.getenv\|UBO_" ubo_app/ --include="*.py" | grep -v __pycache__
```

## Output

After running this command, you should have:

1. **Updated task reference** - Current poe tasks with descriptions
2. **Environment variable list** - All `UBO_` variables documented
3. **Stale docs list** - Files needing review
4. **Diff summary** - What changed since last update

## Integration with Other Commands

- Run after `/plan` to document new features
- Run before releases to ensure docs are current
- Use with `/code-review` to verify documentation coverage

## Related Files

| File | Purpose |
|------|---------|
| `pyproject.toml` | Task definitions, dependencies |
| `tests/.env` | Test environment configuration |
| `CLAUDE.md` | AI assistant instructions |
| `README.md` | User documentation |
| `CHANGELOG.md` | Version history |
