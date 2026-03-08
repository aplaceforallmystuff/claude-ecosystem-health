# Ecosystem Health: Pitfalls & Lessons Learned

Discovered during production runs and subsequent fixes. These represent real issues you'll encounter.

---

## 1. Large Config Files Cause False Positives

`.claude.json` can exceed 40k tokens. If you use the Read tool instead of `jq`, you'll get truncated output and miss project-level MCP server configurations nested inside `projects["/path/to/project"].mcpServers`. This was the single biggest source of false positives on the first run -- 4 of 6 "phantom" MCP servers were actually configured at project level.

**Fix:** Always use `jq` to extract server names (as documented in Check 3). Never use Read on `.claude.json`.

## 2. MCP Server Aliasing (Docker/Proxy Patterns)

Some MCP servers act as proxies for multiple services. For example, a Docker-based MCP server might provide tools from many different APIs through a single configured server. Tools prefixed with the proxy server name are valid even though individual service names aren't configured as separate servers.

**Fix:** Maintain a known-aliases list in your configuration. When checking server names, resolve aliases before flagging as phantom.

## 3. Disabled vs. Missing Servers

A server can be configured in `.claude.json` AND listed in `disabledMcpServers`. The configuration exists but the server won't actually work. This is a different failure mode than a phantom server -- the intent was there, but the server was turned off.

## 4. CLI-over-MCP Drift is the Most Common Finding

In a mature ecosystem, the most frequent warnings come from Check 5 (Configuration Drift), not Check 3 (MCP Health). When CLI tools replace MCP servers, existing skills/agents retain old MCP references because there's no automated migration.

**Fix:** Batch operations by category in the report. Consider adding a `--fix-drift` mode in a future version.

## 5. Vault Path Patterns vs. References

Vault path scanning (Check 1) will match paths inside bash code blocks, grep patterns, and search examples. For instance, `grep -rn "~/my-vault/" ...` contains the path but isn't a broken reference -- it's a search command.

**Fix:** Skip paths that appear inside fenced code blocks or are clearly part of grep/find/search commands.

## 6. Skill Name Fuzzy Matching is Hard

Check 2 tries to match references like `/my-old-skill-name` against active skill names. Skills get renamed and references don't update automatically. Simple string matching misses these -- check for partial matches and common rename patterns (adding/removing suffixes like `-note`, `-skill`, `-tool`).

## 7. First Run Will Find Real Issues

Expect the first run to surface genuine problems, not just configuration drift. In production, the first run found: broken vault paths, wrong skill name references, and MCP server issues (a mix of genuine phantoms and false positives from the large config file issue). Plan for remediation time after the first run.

---

## Operational Notes

- All checks are read-only -- no files are modified by this skill
- Phantom MCP detection accounts for server name aliasing (proxy servers wrap multiple services)
- Skills in `_archive/` are excluded from all checks
- User-invocable skills are exempt from orphan detection
- When checking vault paths, skip glob patterns and search-pattern strings
- The first run is both diagnostic AND educational -- document what you find for future runs
- Report output is intentionally verbose on first run; subsequent runs should be shorter as issues get fixed
