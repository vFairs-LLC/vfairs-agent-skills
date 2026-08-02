# vFairs Agent Skills

Official Claude skills for use alongside the vFairs MCP server. These skills give Claude structured, tested workflows for common event-management questions — registration health, exhibitor engagement, session analytics, and more (additional skills coming soon).

## Requirements

- The vFairs MCP connector must be connected in your Claude account (Claude Code, Cowork, or claude.ai).


## Install (claude.ai or Cowork — no coding required)

If you're using Claude in your browser or the Cowork app, you don't need any commands:

1. Open Claude and click **Customize** (usually in the sidebar or settings menu).
2. Go to the **Plugins** tab.
3. Under **Personal plugins**, click the **+** button.
4. Select **Add marketplace**.
5. Paste this into the box: `vFairs-LLC/vfairs-agent-skills`
6. Press Enter / click **Add**. You'll now see **vfairs-skills** listed as an available marketplace.
7. Click into it and find `registration-health-check`.
8. Click **Install**, then review and approve the permissions it asks for (this lets it talk to the vFairs MCP connector).

That's it — no terminal, no code. The skill is now available any time you chat with Claude.

Before you start: make sure the vFairs MCP connector is already connected to your Claude account (ask your admin if you're not sure). The skill needs that connection to actually pull event data.

## Install (Claude Code)

Add this marketplace once:

```
/plugin marketplace add vFairs-LLC/vfairs-agent-skills
```

Then install the skill(s) you want:

```
/plugin install registration-health-check@vfairs-skills
```

Browse everything available:

```
/plugin marketplace browse vfairs-skills
```


## Usage

Once installed, just ask naturally — Claude will invoke the skill automatically when relevant, e.g.:

> "How's registration going for our Spring Conference? We wanted 500 by end of month."

## Updating

Third-party marketplaces don't auto-update by default. Pull the latest version with:

```
/plugin marketplace update vfairs-skills
/reload-plugins
```

## Available skills

| Skill | What it does |
|---|---|
| `registration-health-check` | Attendee registration pace, status breakdown, profile completeness, and target tracking |
| `pre-event-readiness-audit` | Pre-launch checklist for sessions, speakers, and booths, with an overall readiness score |

## Support

Questions or issues — open a GitHub issue in this repo, or contact your vFairs account team.