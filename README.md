# TrackFlow skill for AI agents

An [Agent Skill](https://agentskills.io) that teaches Claude, ChatGPT, Codex, Cursor, Antigravity, Devin,
Hermes, GitHub Copilot and other AI agents how to use **[TrackFlow](https://trackflow.webanalyticssolution.com)**
and its MCP server to set up, test and audit website conversion tracking, with as few errors as possible.

With the skill installed and TrackFlow connected, you can say things like:

> "Track purchases and add-to-cart on my Shopify store for Facebook and TikTok."
> "Audit my tracking and tell me why Meta shows fewer purchases than my shop."
> "Send my CRM's qualified leads to Google Ads as offline conversions."

and the AI does the whole job: it inspects your site, builds the tags, connects the ad platforms, tests
everything, publishes and verifies the live events.

## What the skill covers

| Topic | File |
|---|---|
| How to work with TrackFlow, rules, setup and audit workflows | [`SKILL.md`](skills/trackflow/SKILL.md) |
| Every MCP tool and parameter | [`mcp-tools.md`](skills/trackflow/references/mcp-tools.md) |
| REST API and server-side event ingest | [`rest-api.md`](skills/trackflow/references/rest-api.md) |
| Browser tracker commands | [`tracker-js.md`](skills/trackflow/references/tracker-js.md) |
| Installing on Shopify, WordPress/WooCommerce, Wix, Webflow, Squarespace, Next.js, GTM … | [`install-cms.md`](skills/trackflow/references/install-cms.md) |
| Event names and parameters per ad platform, what to send and what not to send | [`event-parameters.md`](skills/trackflow/references/event-parameters.md) |
| Meta Pixel + CAPI, TikTok, Google Ads, GA4, Snapchat, Pinterest, LinkedIn, Reddit, X, Microsoft … | [`ad-platforms.md`](skills/trackflow/references/ad-platforms.md) |
| Built-in tag manager: tags, triggers, variables, recipes, versions | [`tag-manager.md`](skills/trackflow/references/tag-manager.md) |
| Checkout webhooks, CRMs, offline conversions | [`integrations.md`](skills/trackflow/references/integrations.md) |
| Linking server orders to the ad click (visitor-id passthrough) | [`visitor-id.md`](skills/trackflow/references/visitor-id.md) |
| Refunds, cancellations, renewals and cash on delivery | [`lifecycle-events.md`](skills/trackflow/references/lifecycle-events.md) |
| First-party tracking domain (ad-blocker resistance), blocking rules | [`first-party-domain.md`](skills/trackflow/references/first-party-domain.md) |
| Reading reports and diagnosing problems | [`reports-and-audit-data.md`](skills/trackflow/references/reports-and-audit-data.md) |
| Setup and audit checklists | [`setup-playbook.md`](skills/trackflow/references/setup-playbook.md), [`audit-playbook.md`](skills/trackflow/references/audit-playbook.md) |

## Step 1: connect TrackFlow to your AI

Open the TrackFlow dashboard → **Connect AI**, pick your AI app and follow the steps. The MCP server is:

```
https://trackflow.webanalyticssolution.com/mcp
```

Most apps sign in to TrackFlow by themselves (OAuth). Apps that cannot do that use an API key from
**Connect AI → API keys**.

## Step 2: give your AI the skill

The skill is the folder [`skills/trackflow`](skills/trackflow). Pick your app:

### Claude Code

```
/plugin marketplace add mnsmasum62786/trackflow-skill
/plugin install trackflow@trackflow-skill
```

Or copy `skills/trackflow` to `~/.claude/skills/trackflow`.

### Claude (claude.ai and the desktop app)

1. Turn on **Code execution and file creation** in Settings → Capabilities.
2. Download [the skill zip](https://github.com/mnsmasum62786/trackflow-skill/releases/latest/download/trackflow.zip)
   (or zip the `skills/trackflow` folder yourself).
3. Customize → Skills → **+** → Upload a skill → choose the zip.

### OpenAI Codex

Copy `skills/trackflow` to `~/.agents/skills/trackflow` (all projects) or `.agents/skills/trackflow`
in your project. Or ask Codex: *"Use $skill-installer to install skills/trackflow from
github.com/mnsmasum62786/trackflow-skill"*.

### ChatGPT

Where Skills are enabled for your workspace, upload the zip from the Claude step above. Otherwise add
`SKILL.md` and the `references/*.md` files to a Project (Project files) or a custom GPT (Knowledge), or
paste this into the chat:

```
Read and follow https://raw.githubusercontent.com/mnsmasum62786/trackflow-skill/main/skills/trackflow/SKILL.md
and the reference files it links to before you use the TrackFlow tools.
```

### Cursor

Copy `skills/trackflow` to `~/.cursor/skills/trackflow` (or `.cursor/skills/trackflow` in a project).

### Google Antigravity

Copy `skills/trackflow` to `.agents/skills/trackflow` in your workspace, or to
`~/.gemini/config/skills/trackflow` for every workspace.

### Devin

Add `skills/trackflow` to a connected repository at `.agents/skills/trackflow`. Devin finds it automatically.

### Hermes Agent

```
hermes skills install mnsmasum62786/trackflow-skill/skills/trackflow
```

### GitHub Copilot (VS Code)

Copy `skills/trackflow` to `.github/skills/trackflow` in your project or `~/.copilot/skills/trackflow`.

### Gemini CLI

```
gemini skills install https://github.com/mnsmasum62786/trackflow-skill.git --path skills/trackflow
```

### Any other AI

Give it this repository URL and say: *"Read AGENTS.md in this repo and follow it when you use TrackFlow."*
Most agents also read skills from `~/.agents/skills/`.

## Step 3: try it

- "List my TrackFlow sites and tell me what each one tracks."
- "Audit the tracking on example.com and give me a fix list."
- "Set up Meta Pixel and Conversions API for purchases on my WooCommerce shop."

## Keeping it up to date

The skill follows the live TrackFlow platform. Pull the latest version (or re-run the install command)
when TrackFlow adds features. Issues and suggestions are welcome.

## License

MIT
