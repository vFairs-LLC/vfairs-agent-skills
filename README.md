# vFairs Agent Skills

Official Claude skills for use alongside the vFairs MCP server. These skills give Claude structured, tested workflows for common event-management questions — registration health, exhibitor engagement, session analytics, and more (additional skills coming soon).

## Requirements

- The vFairs MCP connector must be connected in your Claude account (Claude Code, Cowork, or claude.ai).
- Claude Code or Cowork, to install via the plugin marketplace below.

## Install

Add this marketplace once:

```
/plugin marketplace add vFairs-LLC/vfairs-agent-skills
```

Then install the skill(s) you want:

```
/plugin install registration-health-check@vfairs
```

Browse everything available:

```
/plugin marketplace browse vfairs
```

## Usage

Once installed, just ask naturally — Claude will invoke the skill automatically when relevant, e.g.:

> "How's registration going for our Spring Conference? We wanted 500 by end of month."

## Updating

Third-party marketplaces don't auto-update by default. Pull the latest version with:

```
/plugin marketplace update vfairs
/reload-plugins
```

## Available skills

| Skill | What it does |
|---|---|
| `registration-health-check` | Attendee registration pace, status breakdown, profile completeness, and target tracking |

## Support

Questions or issues — open a GitHub issue in this repo, or contact your vFairs account team.