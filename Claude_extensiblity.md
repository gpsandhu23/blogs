# The Extensible AI: How Claude is trying to become the front door for knowledge work

Claude Code has already become the front door for software development for a lot of developers. Developers don't context-switch between docs, terminals, and editors — they work through Claude. They connect it to GitHub, wire it into their CI pipeline, point it at their codebase, and let it reason across all of it. The result is an AI that doesn't just answer questions about code — it writes, reviews, debugs, and ships it.

This didn't happen because Claude is a better chatbot. It happened because Anthropic focussed on making Claude extensible. MCP servers connect it to tools. Skills teach it how to use those tools well. Plugins package the whole thing up so teams can share it. Extensibility is the difference between a demo and a daily driver.

But here's the thing: software development is just one kind of knowledge work. Across every organization, people — salespeople, lawyers, accountants, designers, scientists — do work that follows patterns, uses specialized tools, and requires domain expertise. The same architecture that made Claude indispensable for developers can do the same for them.

This is already happening. Anthropic recently open-sourced 11 domain-specific plugins — sales, finance, legal, customer support, bio-research, and more — each bundling the tool connections, domain expertise, and workflows for a specific role. With Cowork, Anthropic's agentic desktop app, these plugins bring the Claude Code experience to every knowledge worker. A sales rep says "prep me for my meeting with Acme" and Claude pulls CRM data, researches attendees, and drafts an agenda. A finance team runs `/finance:reconciliation` and Claude orchestrates the entire account reconciliation workflow. A researcher asks about a drug target and Claude queries PubMed, ChEMBL, and Open Targets in one pass.

The organizations that figure out how to encode their workflows, tools, and domain knowledge into this system will have a compounding advantage. Every plugin, every skill, every connector makes Claude more useful — and that usefulness compounds across the organization.

How did we get here? Anthropic built this in three layers — MCP, Skills, and Plugins — each solving a gap the previous left open. Let's walk through the evolution, then deep-dive into plugins, the latest and most complete mechanism.

---

## Layer 1 — MCP: The Connective Tissue

The first problem was isolation. Claude could reason brilliantly about whatever you pasted into the chat window, but it couldn't reach out and touch anything. It couldn't query your database, check your CRM, read your Slack messages, or trigger your CI pipeline. Every time you wanted Claude to work with an external tool, you had to copy-paste data in and results out.

MCP — the Model Context Protocol — solved this. It's an open-source standard that lets Claude connect to external tools and services through a universal interface. Think of it as a standardized plug: any tool that speaks MCP can connect to Claude, and Claude can discover and invoke it without bespoke integration code.

### How it works

MCP servers expose capabilities — tools, resources, prompts — over a standard protocol. Claude discovers what's available and invokes tools as needed. The protocol supports local servers (running on your machine via stdio) and remote servers (hosted services over HTTP or SSE), so it works for everything from a local database to a cloud API.

Configuration is a single JSON file:

```json
{
  "mcpServers": {
    "hubspot": {
      "type": "http",
      "url": "https://mcp.hubspot.com/sse"
    },
    "slack": {
      "type": "http",
      "url": "https://mcp.slack.com/sse"
    },
    "snowflake": {
      "command": "npx",
      "args": ["-y", "@snowflake/mcp-server"]
    }
  }
}
```

That's it. Three lines per tool and Claude can read your CRM, search your Slack, and query your data warehouse.

### The connector model

The real power shows up in how plugins use MCP. Rather than hardcoding specific products, Anthropic's plugins use a category-based connector model. The sales plugin, for example, defines 10 connector categories:

| Category | Placeholder | Included servers | Other options |
|----------|-------------|-----------------|---------------|
| CRM | `~~CRM` | HubSpot, Close | Salesforce, Pipedrive |
| Chat | `~~chat` | Slack | Microsoft Teams |
| Email | `~~email` | Gmail, Microsoft 365 | — |
| Data enrichment | `~~data enrichment` | Clay, ZoomInfo, Apollo | Clearbit, Lusha |
| Meeting transcription | `~~conversation intelligence` | Fireflies | Gong, Chorus, Otter.ai |
| Calendar | `~~calendar` | Google Calendar, Microsoft 365 | — |

Skills and commands reference `~~CRM` or `~~chat` instead of specific products. When you swap from HubSpot to Salesforce, you change one line in `.mcp.json` and everything still works. The workflows are tool-agnostic.

### The gap MCP left

MCP gave Claude access. But access is not competence. Connecting Claude to your CRM doesn't mean it knows how your sales team preps for calls. Connecting it to your accounting system doesn't mean it understands your close process. MCP solved the plumbing — but someone still needed to teach Claude how to do the work.

---

## Layer 2 — Skills: Teaching Claude How to Work

Skills are where domain expertise lives. They're prompt-based extensions — markdown files with YAML frontmatter — that teach Claude how to perform specific tasks. When a skill is loaded, Claude doesn't just have access to tools; it knows what to do with them.

### How they work

A skill is a directory containing a `SKILL.md` file. The YAML frontmatter defines the skill's name and description. The description is critical — it tells Claude when to activate the skill automatically, based on what the user is asking for.

Here's the frontmatter from the sales plugin's call-prep skill:

```yaml
---
name: call-prep
description: Prepare for a sales call with account context, attendee
  research, and suggested agenda. Trigger with "prep me for my call
  with [company]", "I'm meeting with [company] prep me", or "get me
  ready for [meeting]".
---
```

Below the frontmatter, the skill contains detailed instructions: what data to gather, how to structure the output, how to handle different meeting types (discovery, demo, negotiation, QBR), and how to gracefully degrade when certain connectors aren't available. Another interesting implmentation detail is that skills are progressively loaded, so that the context window of the models is not filled with irrelevant details that the user task is not related to. Skills also include 

The call-prep skill, for instance, includes an architecture diagram showing how it works standalone (web research + user input) versus supercharged (CRM history + email threads + Slack discussions + call transcripts + calendar auto-detection). It's over 250 lines of domain expertise encoded as instructions Claude follows.

### The design distinction: model-invoked vs. user-invoked

Skills and commands serve different invocation patterns:

- **Skills** activate automatically when relevant. Claude reads the description, recognizes that the user's request matches, and draws on the skill's instructions. The user never needs to know the skill exists — they just get better results.
- **Commands** are explicit. The user types `/sales:pipeline-review` and Claude executes a defined workflow.

This distinction matters. Skills make Claude proactively competent. A developer doesn't type `/debug` — they describe a bug and Claude automatically applies debugging procedures. A salesperson doesn't need to remember command names — they say "prep me for my call with Acme" and the call-prep skill activates.

### The complexity spectrum

Skills range from trivial to sophisticated. A simple skill might be three lines:

```markdown
---
name: greet
description: Greet the user warmly
---

Greet the user and ask how you can help today.
```

A skill directory can contain much more than just instructions:

```
my-skill/
├── SKILL.md          # Required: instructions + metadata
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
└── assets/           # Optional: templates, resources
```

At the complex end, the bio-research plugin's `single-cell-rna-qc` skill uses all of these — a `SKILL.md` with detailed instructions, a `scripts/` directory with Python analysis code, and a `references/` directory with domain-specific documentation. It's a complete scientific workflow — from data loading to quality control plotting — packaged as a skill.

### The gap skills left

Skills work beautifully within a project. But they're hard to share. If your sales team builds a great set of skills, how do they distribute them to the entire org? How do they version them? How do they bundle skills with the MCP connections they depend on? How do they prevent naming conflicts when multiple skill sets are loaded?

Skills solved competence. What was still missing was packaging and distribution.

---

## Layer 3 — Plugins: The Complete Package

Plugins are the answer to distribution. They bundle skills, commands, agents, hooks, MCP servers, and LSP servers into a single, self-contained package that can be versioned, shared through marketplaces, and installed with two commands.

Everything is file-based — markdown and JSON. No code, no infrastructure, no build steps. This is a deliberate design choice that makes plugins accessible to anyone who can write a markdown file.

### What's in a plugin

Every plugin follows the same structure:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json        # Manifest: name, version, description
├── .mcp.json               # Tool connections
├── commands/                # Slash commands users invoke explicitly
├── skills/                  # Domain knowledge Claude draws on automatically
├── agents/                  # Specialized sub-agents
└── hooks/
    └── hooks.json           # Event handlers
```

The manifest is minimal:

```json
{
  "name": "sales",
  "version": "1.1.0",
  "description": "Prospect, craft outreach, and build deal strategy
    faster. Prep for calls, manage your pipeline, and write personalized
    messaging that moves deals forward.",
  "author": {
    "name": "Anthropic"
  }
}
```

Namespacing prevents conflicts. When you install the sales plugin, its skills become `/sales:call-prep`, `/sales:pipeline-review`, etc. Multiple plugins can define a `research` command without collision.

### The six building blocks

Plugins can contain up to six types of components. Most plugins use two or three; sophisticated ones use all six.

**1. Commands — explicit actions users trigger**

Commands are markdown files in the `commands/` directory. The user types a slash command and Claude follows the instructions. The finance plugin ships five commands:

- `/finance:income-statement` — generate income statements
- `/finance:journal-entry` — prepare journal entries
- `/finance:reconciliation` — reconcile accounts
- `/finance:sox-testing` — SOX compliance testing
- `/finance:variance-analysis` — analyze variances

Each is a markdown file describing the workflow Claude should follow, what data to gather, and how to format the output.

**2. Skills — knowledge Claude draws on automatically**

Skills activate without the user asking. The sales plugin includes six skills that fire based on context: `account-research`, `call-prep`, `competitive-intelligence`, `create-an-asset`, `daily-briefing`, and `draft-outreach`. When a user mentions preparing for a meeting, the `call-prep` skill activates. When they ask about a competitor, `competitive-intelligence` kicks in.

**3. Agents — specialized sub-agents for complex tasks**

Agents are autonomous workers Claude can spawn for specific subtasks. The `feature-dev` plugin in the official directory defines three agents:

```yaml
---
name: code-architect
description: Designs feature architectures by analyzing existing
  codebase patterns and conventions
tools: Glob, Grep, LS, Read, WebFetch, WebSearch
model: sonnet
---
```

Each agent has its own tool permissions, model override, and system prompt. The `code-architect` agent only gets read-only tools — it can explore the codebase but can't modify it. The `code-reviewer` agent gets a different set. This principle of least privilege means agents only have access to what they need.

**4. Hooks — event handlers that run automatically**

Hooks are scripts triggered by lifecycle events: before a tool is used, after a file is written, when a session starts. The `security-guidance` plugin uses a `PreToolUse` hook to catch security issues before they're written to disk:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py"
          }
        ],
        "matcher": "Edit|Write|MultiEdit"
      }
    ]
  }
}
```

Every time Claude tries to write or edit a file, this hook runs a Python script that checks for dangerous patterns — `eval()`, `innerHTML`, `pickle.loads()`, `os.system()` — and blocks the edit with a warning if it finds one. The hook fires automatically; neither Claude nor the user needs to remember to run a security check.

**5. MCP Servers — tool connections bundled with the plugin**

Plugins include an `.mcp.json` file that defines which external tools they need. When the plugin is installed and enabled, these connections activate automatically. The sales plugin's `.mcp.json` wires up HubSpot, Slack, Gmail, Clay, ZoomInfo, Fireflies, Notion, and more — all the tools a sales team uses daily.

**6. LSP Servers — language intelligence**

LSP (Language Server Protocol) servers give Claude real-time code intelligence: diagnostics, go-to-definition, find-references. The official marketplace includes LSP plugins for TypeScript, Python, Rust, Go, Java, C/C++, PHP, Swift, Kotlin, C#, and Lua. Each is just a few lines of configuration:

```json
{
  "typescript": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact"
    }
  }
}
```

### Real-world plugins in action

Three examples show the breadth of what plugins enable:

**Enterprise knowledge work: the finance plugin.** Five commands and six skills covering the full accounting workflow — journal entry preparation, account reconciliation, financial statement generation, variance analysis, SOX testing, close management, and audit support. A CFO's office can install this plugin, connect it to Snowflake and Microsoft 365, and Claude becomes a finance specialist that understands GAAP, knows how to structure journal entries, and can walk through a reconciliation step by step.

**Scientific research: the bio-research plugin.** Five skills spanning single-cell RNA quality control, scvi-tools integration, Nextflow pipeline development, instrument data conversion, and scientific problem selection. Connectors reach PubMed, bioRxiv, ClinicalTrials.gov, ChEMBL, Open Targets, Synapse, BioRender, and Benchling. The `single-cell-rna-qc` skill includes Python scripts for analysis and a `references/` directory with domain-specific documentation. This isn't a chatbot that knows biology — it's a research assistant with real tools.

**Developer tooling: the feature-dev plugin.** A multi-agent workflow for feature development. The main command orchestrates three specialized agents — `code-explorer` (reads and maps the codebase), `code-architect` (designs the feature architecture), and `code-reviewer` (reviews the implementation) — each with specific tool permissions and model overrides. It's a development team in a plugin.

### Building your first plugin

Creating a plugin takes minutes:

```bash
# Create the directory structure
mkdir -p my-plugin/.claude-plugin
mkdir -p my-plugin/skills/hello

# Create the manifest
cat > my-plugin/.claude-plugin/plugin.json << 'EOF'
{
  "name": "my-plugin",
  "description": "My first plugin",
  "version": "1.0.0"
}
EOF

# Create a skill
cat > my-plugin/skills/hello/SKILL.md << 'EOF'
---
name: hello
description: Greet the user warmly
---

Greet the user and ask how you can help today.
EOF

# Test it
claude --plugin-dir ./my-plugin
```

That's a working plugin. Add commands, agents, hooks, and MCP connections as you need them.

### Distributing through marketplaces

A marketplace is a Git repository with a `marketplace.json` file that lists available plugins. Three official marketplaces already exist:

- **claude-plugins-official** — 40+ plugins including dev tools, LSP servers, and partner-built integrations
- **knowledge-work-plugins** — 11 domain-specific plugins for sales, finance, legal, support, and more
- **agent-skills** — document processing and example skills

Installation is two commands:

```bash
# Add a marketplace
claude plugin marketplace add anthropics/knowledge-work-plugins

# Install a plugin
claude plugin install sales@knowledge-work-plugins
```

Once installed, plugins activate automatically. Skills fire when relevant, commands appear in the slash-command menu, hooks run on their events, and MCP servers connect to their tools.

---

## The Ecosystem

The marketplace model follows proven platform patterns — VS Code extensions, npm packages, browser add-ons — but with a key difference: everything is markdown and JSON. The barrier to building a plugin is writing a few files, not learning an SDK.

Partners are already building. The `knowledge-work-plugins` marketplace includes partner-built plugins from Apollo (sales prospecting), Common Room (GTM intelligence), Slack (team communication), and Tribe AI (brand voice). The official plugin directory includes external contributions from Stripe, Figma, Sentry, Vercel, PostHog, CodeRabbit, HuggingFace, Semgrep, and others.

The enterprise customization story is particularly compelling. These plugins are designed to be forked and adapted. Anthropic's README says it plainly: "These plugins are generic starting points. They become much more useful when you customize them for how your company actually works." The playbook is straightforward:

- **Swap connectors** — edit `.mcp.json` to point at your tool stack
- **Add company context** — drop your terminology, org structure, and processes into skill files
- **Adjust workflows** — modify instructions to match how your team actually does things
- **Build new plugins** — create plugins for roles and workflows not yet covered

The composability story completes the picture. A knowledge worker can install the sales plugin, the productivity plugin, and the Slack plugin. Each adds its own skills and commands, but they all compose through Claude. The same person might run `/sales:call-prep` in the morning, `/productivity:plan-day` after standup, and ask Claude to draft a follow-up email in the afternoon — all powered by different plugins, all working together seamlessly.

---

## What This Means

The three-layer architecture — MCP for connectivity, Skills for competence, Plugins for distribution — creates a complete extensibility stack. Each layer addresses a specific concern, and together they solve the full problem of turning a general-purpose AI into a domain-specific, organization-aware tool.

**For developers:** you can build plugins today using markdown and JSON. No SDKs, no APIs, no build steps. The barrier to extending Claude is intentionally as low as possible. If you can write a markdown file that describes how to do something, you can teach Claude to do it.

**For enterprises:** extensibility means AI adoption scales with your organization. Instead of a generic assistant that requires constant hand-holding, you get domain-specific AI that understands your tools, your processes, your terminology — and it gets better as your teams contribute plugins. The finance team's reconciliation plugin. The legal team's contract review plugin. The sales team's call-prep plugin. Each one encodes institutional knowledge that would otherwise live in people's heads or scattered across wikis nobody reads.

**The flywheel:** as more plugins are built, Claude becomes more useful across more domains. As it becomes more useful, more people build plugins. The knowledge of an entire organization — its processes, its conventions, its hard-won expertise — can be encoded, versioned, shared, and improved over time.

Claude isn't just an AI assistant. It's becoming the interface layer through which organizations access their collective knowledge and execute their workflows. Extensibility — MCP, Skills, and Plugins — is what makes that possible.

---

*Ready to build? Start with the [plugin documentation](https://code.claude.com/docs/en/plugins), browse the [official marketplace](https://github.com/anthropics/claude-plugins-official), or explore the [knowledge-work plugins](https://github.com/anthropics/knowledge-work-plugins) to see what's possible.*
