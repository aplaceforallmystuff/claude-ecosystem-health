# Changelog

All notable changes to this project will be documented in this file.

## [1.0.0] - 2026-02-13

### Added
- Initial release with 7 diagnostic checks:
  - Check 1: Vault Path Validation (Critical)
  - Check 2: Skill Cross-Reference Validation (High)
  - Check 3: MCP Server Health (Critical)
  - Check 4: CLI Tool Availability (Medium)
  - Check 5: Configuration Drift (High)
  - Check 6: Staleness Detection (Low)
  - Check 7: Orphan Detection (Low)
- Three operating modes: full, quick, single-check
- Structured health report with severity classification
- Health thresholds: HEALTHY / NEEDS ATTENTION / DEGRADED
- Pitfalls & Lessons Learned section from production use
- Customization guide for adapting to different setups
- `disabledMcpServers` detection in Check 3
- Code block false positive prevention guidance
- MCP server aliasing/proxy pattern documentation
