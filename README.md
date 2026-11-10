# Scope Discipline: Ship Integration, Ask Before Surface

> Ship the integration, ask before exposing the surface. The rule that keeps a small feature request from turning into a public API you have to support forever.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg) [![Available on Agensi](https://img.shields.io/badge/Agensi-%245-ff5a5f.svg)](https://agensi.io/skills/scope-discipline-ship-integration-ask-before-surface)

## What it does

Ship integration; ask before exposing surface.

A working user flow is sacred. When you add a third-party integration, you are adding two things: (1) the integration layer (client, auth, settings) and (2) the surface that exposes it to the user. These are not the same deliverable.

## The failure it prevents

Working this by hand means rediscovering the same edge cases every time. They are written down here instead, with the fix for each one.

## When to use it

- decide whether a change belongs in the public surface
- ship an integration without committing to an API
- keep a small request from becoming a permanent contract

## What's inside

```
plugins/
└── scope-discipline/
    ├── .claude-plugin/plugin.json
    └── skills/scope-discipline/
        ├── SKILL.md
```

## Install

Add this repo as a Claude Code marketplace, then install the skill:

```bash
claude plugin marketplace add Verstige/scope-discipline
claude plugin install scope-discipline@scope-discipline
```

Or copy the skill straight into your skills directory:

```bash
git clone https://github.com/Verstige/scope-discipline.git
cp -R scope-discipline/plugins/scope-discipline/skills/scope-discipline ~/.claude/skills/
```

## Usage

The skill activates on the work it describes. Ask for the outcome, and it
supplies the method:

```
> "decide whether a change belongs in the public surface"
```

## License

MIT. See [LICENSE](LICENSE).

---

**Also available as a paid skill on Agensi** — $5. Install it with one command, no clone required:

[agensi.io/skills/scope-discipline-ship-integration-ask-before-surface](https://agensi.io/skills/scope-discipline-ship-integration-ask-before-surface)

Part of the [Verstige](https://github.com/Verstige) skills portfolio. More at [github.com/Verstige](https://github.com/Verstige).
