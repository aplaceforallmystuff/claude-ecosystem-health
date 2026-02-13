---
name: ecosystem-health
description: Detect drift between skills, agents, MCP servers, vault paths, and CLI tools. Read-only diagnostics with remediation pointers.
use_when: Monthly full sweep, weekly quick check, after renaming/archiving any component, or when something "used to work" but stopped
user-invocable: true
argument-hint: "[--quick] [--check vault-paths|skill-refs|mcp-servers|cli-tools|config-drift|staleness|orphans]"
---

# Ecosystem Health

Detect drift between interconnected Claude Code components: skills, agents, MCP servers, vault paths, CLI tools, and configuration policies. Read-only by default — reports findings and points to existing tools for remediation.

## When to Use

- Monthly full sweep (first weekly review of each month)
- Weekly quick check during weekly review
- After renaming, archiving, or reconfiguring any skill/agent/MCP server
- When something "used to work" but stopped

## Modes

| Invocation | Checks Run | Use Case |
|-----------|-----------|----------|
| `/ecosystem-health` | All 7 | Monthly full sweep |
| `/ecosystem-health --quick` | Checks 1-5 only (skip npm outdated in Check 3) | Weekly quick check |
| `/ecosystem-health --check [name]` | Single named check | Targeted diagnosis |

**Check names for --check:** `vault-paths`, `skill-refs`, `mcp-servers`, `cli-tools`, `config-drift`, `staleness`, `orphans`

---

## Process

### Step 0: Determine Mode

Parse the user's invocation:

- No arguments: **Full mode** (all 7 checks, npm outdated included)
- `--quick`: **Quick mode** (checks 1-5, skip npm outdated)
- `--check [name]`: **Single check mode** (run only the named check)

Initialize counters for OK, Warning, Critical findings per check.

### Step 1: Scan Source Files

Build the file lists that all checks will use:

```bash
# All active skill files (exclude _archive)
find ~/.claude/skills -name "SKILL.md" -not -path "*/_archive/*" -type f

# All agent files (top-level only)
ls ~/.claude/agents/*.md 2>/dev/null | grep -v _archive

# All command files
find ~/.claude/commands -name "*.md" -type f

# Hook scripts
cat ~/.claude/settings.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
hooks = d.get('hooks', {})
for event, matchers in hooks.items():
    for mg in matchers:
        for h in mg.get('hooks', []):
            print(h.get('command', ''))
"

# CLAUDE.md files
echo ~/.claude/CLAUDE.md
# Add your project-level CLAUDE.md path here:
# echo ~/path/to/your/project/CLAUDE.md
```

Store all file paths for reuse across checks.

---

## Check 1: Vault Path Validation (CRITICAL)

**Goal:** Find hardcoded vault paths in skills/agents that point to non-existent locations.

### Scan

Search all skill, agent, and command files for vault path patterns:

```bash
# Search for hardcoded vault paths — customize these patterns for your vault location
grep -rn "~/your/vault/path/" ~/.claude/skills/ ~/.claude/agents/ --include="*.md" | grep -v "_archive"
grep -rn "/Users/yourusername/your/vault/path/" ~/.claude/skills/ ~/.claude/agents/ --include="*.md" | grep -v "_archive"
```

### Validate

For each path found:
1. Extract the full path (resolve `~/` to `$HOME`)
2. Check if the path exists on filesystem using `ls` or `test -e`
3. If it contains a glob pattern (e.g., `*/SKILL.md`), skip validation — those are search patterns, not references
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

# Pattern 2: Skill tool references like "my-skill skill" or "Invoke my-skill"
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

**IMPORTANT:** `.claude.json` can be very large (40k+ tokens in complex setups). Do NOT use the Read tool — use `jq` to search it:

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

### Check disabledMcpServers

Also check if servers appear in the `disabledMcpServers` list:

```bash
cat ~/.claude.json | jq -r '
  (.disabledMcpServers // [])[]
' 2>/dev/null
```

Servers that are configured BUT also listed in `disabledMcpServers` should be flagged as a warning — the references in code will fail at runtime even though the config exists.

### npm Outdated (Full mode only)

For each configured local MCP server with a `package.json`:

```bash
cd ~/Dev/mcp-[name] && npm outdated --json 2>/dev/null
```

Skip this step in `--quick` mode.

### Classify

- **Critical:** MCP server referenced in skills/agents but NOT configured in either config file (phantom tool)
- **Warning:** MCP server configured but listed in `disabledMcpServers` (will fail at runtime)
- **Warning:** MCP server configured but its local directory is missing
- **Warning:** (Full mode) MCP server has major version updates available
- **OK:** Server configured and (if local) directory exists

### Server Aliasing

Some MCP servers act as proxies for multiple services. For example, a Docker-based MCP server might provide tools from many different services through a single configured server name. When you see `mcp__PROXY_NAME__service_tool`, check if `PROXY_NAME` is a configured proxy server before flagging as phantom.

Add your known aliases to the Customization section below.

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
2. For each skill, check if `user-invocable: true` in frontmatter — exempt these
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

---

## Output

### Report File

Write report to a system location in your vault or project:

```
# Examples:
# ~/your-vault/00 System/Ecosystem Health Report.md
# ~/.claude/reports/ecosystem-health-report.md
```

### Report Format

```markdown
---
type: system-health
generated: [ISO 8601 timestamp]
mode: Full|Quick|Single ([check-name])
---

# Ecosystem Health Report

**Generated:** YYYY-MM-DD HH:MM | **Mode:** Full/Quick/Single

## Summary

| Category | OK | Warning | Critical |
|----------|-----|---------|----------|
| Vault Paths | X | X | X |
| Skill References | X | X | X |
| MCP Servers | X | X | X |
| CLI Tools | X | X | X |
| Config Drift | X | X | X |
| Staleness | X | X | X |
| Orphans | X | X | X |
| **Total** | **X** | **X** | **X** |

**Health:** HEALTHY / NEEDS ATTENTION / DEGRADED

Health thresholds:
- **HEALTHY:** 0 critical, 0-2 warnings
- **NEEDS ATTENTION:** 0 critical, 3+ warnings OR 1 critical
- **DEGRADED:** 2+ critical findings

---

## Critical Issues

List each critical finding:

### [Check Name]: [Brief Description]

**File:** `path/to/affected/file`
**Line:** N
**Issue:** What's wrong
**Expected:** What should be there
**Fix via:** Manual edit / existing tool name

---

## Warnings

Same format as Critical, grouped by check.

---

## Informational

Stale/orphan findings, lower priority items.

---

## Remediation Summary

| Finding | Severity | Fix Via | Files Affected |
|---------|----------|---------|----------------|
| Broken vault path | Critical | Manual edit | file.md |
| Wrong skill name | Critical | Manual edit | file.md |
| Phantom MCP tool | Warning | Manual edit | file.md |

---

*Generated by /ecosystem-health. Run /ecosystem-health --quick for weekly checks.*
```

### Console Summary

After writing the report, display a brief summary:

```
Ecosystem Health: [HEALTHY|NEEDS ATTENTION|DEGRADED]

  Critical: N findings
  Warnings: N findings
  Info: N findings

Report saved to: [report path]
```

---

## Remediation Guide

This skill is **read-only**. For fixes, use these tools:

| Issue Type | Fix Via |
|-----------|---------|
| Broken vault paths | Manual edit of affected skill/agent |
| Wrong skill names | Manual edit of affected file |
| Phantom MCP tools | Manual edit to remove or replace with CLI |
| CLI-over-MCP policy violations | Manual edit to use CLI tool instead |
| Stale skills | Archive or manual review |
| Outdated MCP deps | Update per server |
| Orphaned skills | Review for archival |

---

## Pitfalls & Lessons Learned

These were discovered during production use. They represent real issues you'll encounter.

### 1. Large Config Files Cause False Positives

`.claude.json` can exceed 40k tokens in complex setups. If you use the Read tool instead of `jq`, you'll get truncated output and miss project-level MCP server configurations nested inside `projects["/path/to/project"].mcpServers`. This was the single biggest source of false positives on the first run — 4 of 6 "phantom" MCP servers were actually configured at project level.

**Fix:** Always use `jq` to extract server names (as documented in Check 3). Never use Read on `.claude.json`.

### 2. MCP Server Aliasing (Docker/Proxy Patterns)

Some MCP servers act as proxies for multiple services. For example, a Docker-based MCP server might provide tools from many different APIs through a single configured server. Tools prefixed with the proxy server name are valid even though individual service names aren't configured as separate servers.

**Fix:** Maintain a known-aliases list in your configuration. When checking server names, resolve aliases before flagging as phantom.

### 3. Disabled vs. Missing Servers

A server can be configured in `.claude.json` AND listed in `disabledMcpServers`. The configuration exists but the server won't actually work. This is a different failure mode than a phantom server — the intent was there, but the server was turned off.

**Fix:** Check 3 includes a `disabledMcpServers` scan step for this reason.

### 4. CLI-over-MCP Drift is the Most Common Finding

In a mature ecosystem, the most frequent warnings come from Check 5 (Configuration Drift), not Check 3 (MCP Health). When CLI tools are introduced to replace MCP servers, existing skills/agents/commands retain their old MCP references because there's no automated migration. This results in dozens of warnings that are all the same category.

**Fix:** Batch operations by category in the report.

### 5. Vault Path Patterns vs. References

Vault path scanning (Check 1) will match paths inside bash code blocks, grep patterns, and search examples. For instance, `grep -rn "~/my-vault/" ...` contains the path but isn't a broken reference — it's a search command.

**Fix:** Skip paths that appear inside fenced code blocks (``` delimiters) or are clearly part of grep/find/search commands.

### 6. Skill Name Fuzzy Matching is Hard

Check 2 tries to match references like `/my-old-skill-name` against active skill names. But skills get renamed, and the references don't update automatically. Simple string matching misses these — you need to also check for partial matches and common rename patterns (adding/removing suffixes).

### 7. Code Block False Positives in Model Check

Check 5's `grep "^model:"` for agent model fields will match lines inside fenced code blocks that happen to start with `model:`. Only the `model:` field in the YAML frontmatter block (between the first `---` delimiters at the top of the file) is a real model declaration.

### 8. First Run Will Find Real Issues

Expect the first run to surface genuine problems, not just configuration drift. Plan for remediation time after the first run.

---

## Customization

### Required Changes

Before using this skill, customize these items:

1. **Vault paths** (Check 1): Replace `~/your/vault/path/` with your actual vault location
2. **CLI tools** (Check 4): Add the CLI tools your setup depends on
3. **CLI-over-MCP policies** (Check 5): Define which MCP tools have CLI replacements
4. **Report output path** (Output section): Set where reports should be saved
5. **MCP server aliases** (Check 3): List any proxy/Docker MCP servers that wrap multiple services
6. **Project CLAUDE.md** (Step 1): Add your project-level CLAUDE.md path

### Optional Configuration

- **Staleness threshold**: Adjust the `90` day threshold in Check 6 to match your cadence
- **Orphan exemptions**: Add additional exemption criteria beyond `user-invocable` in Check 7
- **Severity thresholds**: Adjust HEALTHY/NEEDS ATTENTION/DEGRADED thresholds

---

## Integration

### Weekly Review

Use as part of your weekly review process:

- **Weekly:** Run `/ecosystem-health --quick` (checks 1-5)
- **Monthly (first review of month):** Run `/ecosystem-health` (all 7 checks)

### Daily Note

After running, log a summary to today's daily note:

```markdown
## Ecosystem Health Check

**Mode:** Quick/Full | **Health:** HEALTHY/NEEDS ATTENTION/DEGRADED
**Findings:** N critical, N warnings, N info
```

---

## Notes

- All checks are read-only — no files are modified by this skill
- Phantom MCP detection accounts for server name aliasing (proxy servers wrap multiple services)
- Skills in `_archive/` are excluded from all checks
- User-invocable skills are exempt from orphan detection
- When checking vault paths, skip glob patterns and search-pattern strings
- The first run is both diagnostic AND educational — document what you find for future runs
- Report output is intentionally verbose on first run; subsequent runs should be shorter as issues get fixed
