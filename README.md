# agent-skills

A collection of [Agent Skills](https://agentskills.io/) for use with GitHub Copilot, Claude, and other AI coding assistants.

## Skills

| Skill | Description |
|---|---|
| [`generate-mcp`](skills/generate-mcp/SKILL.md) | End-to-end guide for building a production-quality Go MCP server — from API research through CI and release automation |
| [`audit-network-security`](skills/audit-network-security/SKILL.md) | Structured security audit of a UniFi network using unifi-mcp: device inventory, firmware currency, WiFi config, firewall rules, segmentation, DNS anomalies, VPN, and voucher hygiene — produces a prioritised findings report |

## Usage

### Project skill (single repository)

Copy the skill directory into your repository:

```bash
cp -r skills/generate-mcp /path/to/your-repo/.github/skills/
```

### Personal skill (all projects)

Copy the skill directory to your home directory:

```bash
cp -r skills/generate-mcp ~/.copilot/skills/
# or for Claude:
cp -r skills/generate-mcp ~/.claude/skills/
```

Copilot will automatically load the skill when you ask it to create a new MCP server.

## License

MIT
