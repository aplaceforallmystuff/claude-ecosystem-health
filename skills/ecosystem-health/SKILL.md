---
name: ecosystem-health
description: Detect drift between skills, agents, MCP servers, vault paths, and CLI tools. Read-only diagnostics with remediation pointers.
use_when: Monthly full sweep, weekly quick check, after renaming/archiving any component, or when something "used to work" but stopped
user-invocable: true
argument-hint: "[--quick] [--check vault-paths|skill-refs|mcp-servers|cli-tools|config-drift|staleness|orphans]"
---

# Ecosystem Health

Detect drift between your interconnected Claude Code components: skills, agents, MCP servers, vault paths, CLI tools, and configuration policies. Read-only by default -- reports findings and points to existing tools for remediation.

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

## Check Summary

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| 1 | Vault Paths | CRITICAL | Hardcoded vault paths pointing to non-existent locations |
| 2 | Skill References | HIGH | References to renamed, archived, or missing skills |
| 3 | MCP Servers | CRITICAL | MCP tool references not matching configured servers |
| 4 | CLI Tools | MEDIUM | CLI tools referenced in CLAUDE.md not installed or broken |
| 5 | Config Drift | HIGH | Skills/agents violating stated CLAUDE.md policies |
| 6 | Staleness | LOW | Skills/agents not modified in 90+ days (full mode only) |
| 7 | Orphans | LOW | Non-invocable skills with zero references (full mode only) |

> **Detailed check implementations:** See [references/checks.md](references/checks.md) for full scan/validate/classify procedures.

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
# Add your project-level CLAUDE.md paths here:
# echo ~/path/to/your/project/CLAUDE.md
```

Store all file paths for reuse across checks.

### Step 2: Run Checks

Execute checks per the selected mode, following the detailed procedures in [references/checks.md](references/checks.md).

---

## Output

### Report File

Write report to a location in your project or system directory:

```
# Examples:
# ~/.claude/reports/ecosystem-health-report.md
# ~/your-vault/System/Ecosystem Health Report.md
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
```

Health thresholds:
- **HEALTHY:** 0 critical, 0-2 warnings
- **NEEDS ATTENTION:** 0 critical, 3+ warnings OR 1 critical
- **DEGRADED:** 2+ critical findings

Each finding includes: **File**, **Line**, **Issue**, **Expected**, **Fix via**.

### Console Summary

After writing the report, display:

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

## Integration

### Weekly Review

Use as part of your weekly review process:

- **Weekly:** Run `/ecosystem-health --quick` (checks 1-5)
- **Monthly (first review of month):** Run `/ecosystem-health` (all 7 checks)

### Daily Note (Optional)

After running, log a summary to today's daily note:

```markdown
## Ecosystem Health Check

**Mode:** Quick/Full | **Health:** HEALTHY/NEEDS ATTENTION/DEGRADED
**Findings:** N critical, N warnings, N info
```

---

## Pitfalls, Lessons Learned & Operational Notes

> See [references/pitfalls.md](references/pitfalls.md) for detailed pitfalls from production runs and operational notes (config file size, MCP aliasing, CLI-over-MCP drift, vault path false positives, fuzzy matching, first-run expectations).
