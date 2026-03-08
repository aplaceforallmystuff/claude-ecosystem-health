# Ecosystem Health: Check Implementations

Detailed scan/validate/classify procedures for each of the 7 checks.

---

## Check 1: Vault Path Validation (CRITICAL)

**Goal:** Find hardcoded vault paths in skills/agents that point to non-existent locations.

### Scan

Search all skill, agent, and command files for vault path patterns:

```bash
# Search for hardcoded vault paths -- customize these patterns for your vault location
grep -rn "~/your/vault/path/" ~/.claude/skills/ ~/.claude/agents/ --include="*.md" | grep -v "_archive"
grep -rn "/Users/yourusername/your/vault/path/" ~/.claude/skills/ ~/.claude/agents/ --include="*.md" | grep -v "_archive"
```

### Validate

For each path found:
1. Extract the full path (resolve `~/` to `$HOME`)
2. Check if the path exists on filesystem using `ls` or `test -e`
3. If it contains a glob pattern (e.g., `*/SKILL.md`), skip validation -- those are search patterns, not references
4. If path ends with `/` (directory reference), check directory exists

### Classify

- **Critical:** Path to a specific file that doesn't exist
- **Warning:** Path to a directory that doesn't exist
- **OK:** Path exists

### Customization

Update the grep patterns in the Scan step to match your vault location. Common patterns:
- `~/Documents/Vault/` for local vaults
- `~/Obsidian/` for Obsidian users
- `~/Sync/` for synced vaults

---

## Check 2: Skill Cross-Reference Validation (HIGH)

**Goal:** Find references to skills that don't exist (renamed, archived, or typos).

### Scan

Search for skill name references in all skills, agents, commands:

```bash
# Pattern 1: Skill invocations like /skill-name or `/skill-name`
grep -rn "\/[a-z][a-z0-9-]*" ~/.claude/skills/ ~/.claude/agents/ ~/.claude/commands/ --include="*.md" | grep -v "_archive" | grep -v "http"

# Pattern 2: Skill tool references like "log-to-daily skill" or "Invoke log-to-daily"
grep -rn "skill\|Skill tool\|invoke.*skill" ~/.claude/skills/ ~/.claude/agents/ --include="*.md" | grep -v "_archive"
```

### Validate

1. Build list of all active skill names from `~/.claude/skills/*/SKILL.md` frontmatter `name:` field (or directory name as fallback)
2. For each reference found that looks like a skill name:
   - Check if it exists in the active skills list
   - Check if it exists in `_archive/` (was archived)
   - Check for close matches (typos, old names)

**Filtering:** Ignore references that are clearly:
- URLs (contain `://` or start with `http`)
- File paths (contain `/Users/` or `~/`)
- Command-line flags (start with `--`)
- Code comments or examples in fenced code blocks
- Built-in CLI commands (/help, /clear, /compact, /context, etc.)

### Classify

- **Critical:** Skill referenced in an active agent/command but doesn't exist anywhere
- **Warning:** Skill referenced but found in `_archive/` (was archived, reference is stale)
- **OK:** Skill reference resolves to active skill

---

## Check 3: MCP Server Health (CRITICAL)

**Goal:** Find MCP server references in skills/agents that aren't actually configured.

### Scan

```bash
# Find all mcp__*__ tool patterns in skills, agents, commands
grep -rohn "mcp__[A-Za-z0-9_-]*__[A-Za-z0-9_]*" ~/.claude/skills/ ~/.claude/agents/ ~/.claude/commands/ --include="*.md" | grep -v "_archive" | sort -u
```

### Extract Server Names

From each `mcp__SERVERNAME__toolname` pattern, extract `SERVERNAME`.

### Validate Against Configuration

**IMPORTANT:** `.claude.json` can be 40k+ tokens. Do NOT use the Read tool -- use `grep` or `jq` to search it:

```bash
# Extract ALL configured MCP server names (global + all project-level)
cat ~/.claude.json | jq -r '
  (.mcpServers // {} | keys[]),
  (.projects // {} | to_entries[] | .value.mcpServers // {} | keys[])
' 2>/dev/null | sort -u
```

```bash
# Also check settings.json for additional servers
cat ~/.claude/settings.json | jq -r '
  (.mcpServers // {} | keys[])
' 2>/dev/null | sort -u
```

1. Build a combined list of ALL configured servers from both files (global AND project-level)
2. For each server name found in code:
   - Check if it appears in the combined configured list
   - For local servers (path contains `~/Dev/`), verify directory exists

### npm Outdated (Full mode only)

For each configured local MCP server with a `package.json`:

```bash
cd ~/Dev/mcp-[name] && npm outdated --json 2>/dev/null
```

Skip this step in `--quick` mode.

### Check disabledMcpServers

Also check if servers appear in the `disabledMcpServers` list:

```bash
cat ~/.claude.json | jq -r '
  (.disabledMcpServers // [])[]
' 2>/dev/null
```

Servers that are configured BUT also listed in `disabledMcpServers` should be flagged as a warning -- the references in code will fail at runtime even though the config exists.

### Classify

- **Critical:** MCP server referenced in skills/agents but NOT configured in either config file (phantom tool)
- **Warning:** MCP server configured but listed in `disabledMcpServers` (will fail at runtime)
- **Warning:** MCP server configured but its local directory is missing
- **Warning:** (Full mode) MCP server has major version updates available
- **OK:** Server configured and (if local) directory exists

### Server Aliasing

Some MCP servers act as proxies for multiple services. For example, a Docker-based MCP server might provide tools from many different APIs through a single configured server name. When you see `mcp__PROXY_NAME__service_tool`, check if `PROXY_NAME` is a configured proxy server before flagging as phantom.

Maintain a known-aliases list for your setup and resolve aliases before flagging servers as phantom.

---

## Check 4: CLI Tool Availability (MEDIUM)

**Goal:** Verify CLI tools referenced in CLAUDE.md are actually installed and working.

### Check Each Tool

Customize this list with the CLI tools your setup depends on:

```bash
# GitHub CLI
which gh && gh --version 2>/dev/null

# 1Password CLI (if using for secrets)
which op && op --version 2>/dev/null

# Add your custom CLI tools here:
# which my-tool && my-tool --version 2>/dev/null
```

### Classify

- **Critical:** Tool is referenced in CLAUDE.md as required but not installed
- **Warning:** Tool installed but fails to respond (may need auth or configuration)
- **OK:** Tool installed and responds

---

## Check 5: Configuration Drift (HIGH)

**Goal:** Find skills/agents that violate stated policies in CLAUDE.md.

### Policy: CLI over MCP

If your CLAUDE.md states certain CLI tools should be used instead of their MCP equivalents, check for violations:

```bash
# Example: Find MCP tools that should be CLI calls instead
# Customize these patterns based on your policies:
# grep -rn "mcp__my-service__\|mcp__other-service__" ~/.claude/skills/ ~/.claude/agents/ ~/.claude/commands/ --include="*.md" | grep -v "_archive"
```

### Policy: Model Fields

Check agent `model:` frontmatter fields reference valid values:

```bash
grep -rn "^model:" ~/.claude/agents/*.md | grep -v "_archive"
```

Valid models: `opus`, `sonnet`, `haiku` (or empty/absent = inherit).

**Important:** Filter out matches inside fenced code blocks (``` delimiters). Lines like `model: sonnet  # or opus/haiku` inside YAML examples are documentation, not actual frontmatter. Only the `model:` field in the YAML frontmatter block (between the first `---` delimiters) counts.

### Classify

- **Warning:** Skill/agent uses MCP tool when CLI is the stated preference
- **Warning:** Agent specifies invalid model value
- **OK:** No policy violations

---

## Check 6: Staleness Detection (LOW)

**Skip in `--check` mode unless specifically requested.**

**Goal:** Find skills/agents that haven't been modified in 90+ days.

### Scan

```bash
# Skills by modification date (oldest first)
find ~/.claude/skills -name "SKILL.md" -not -path "*/_archive/*" -mtime +90 -type f

# Agents by modification date
find ~/.claude/agents -name "*.md" -not -path "*/_archive/*" -mtime +90 -type f
```

### Cross-Reference

For each stale file, check if it's still referenced by other active components:

```bash
# For a skill named "skill-name", search for references
grep -rl "skill-name" ~/.claude/skills/ ~/.claude/agents/ ~/.claude/commands/ --include="*.md" | grep -v "_archive"
```

### Classify

- **Warning:** Stale (90+ days) AND still referenced by active components (may be outdated)
- **Info:** Stale AND not referenced (orphaned, candidate for archive)
- **OK:** Modified within 90 days, or stale but user-invocable (directly used)

---

## Check 7: Orphan Detection (LOW)

**Skip in `--check` mode unless specifically requested.**

**Goal:** Find skills that nothing references and aren't user-invocable.

### Build Reference Map

1. List all active skill names
2. For each skill, check if `user-invocable: true` in frontmatter -- exempt these
3. For non-invocable skills, search for references across:
   - Other skills
   - Agents
   - Commands
   - Hooks
   - CLAUDE.md files

```bash
# Get skill name from frontmatter
head -10 ~/.claude/skills/[skill-dir]/SKILL.md | grep "^name:" | sed 's/name: //'

# Check if user-invocable
head -10 ~/.claude/skills/[skill-dir]/SKILL.md | grep "user-invocable: true"

# Search for references (use directory name as search term too)
grep -rl "[skill-name]" ~/.claude/skills/ ~/.claude/agents/ ~/.claude/commands/ ~/.claude/CLAUDE.md --include="*.md" 2>/dev/null | grep -v "_archive" | grep -v "[skill-name]/SKILL.md"
```

### Classify

- **Info:** Non-invocable skill with zero external references (dead code, candidate for archive)
- **OK:** Skill is user-invocable OR has at least one external reference
