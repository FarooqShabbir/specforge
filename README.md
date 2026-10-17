# SpecForge

A Claude Code plugin that builds complex software end to end, following Anthropic's recommended workflow:
explore → plan → independent plan review → test-first build → verify → gated deploy.

It sizes the request, writes the spec and milestone plan, has a second Claude review the plan,
sets up CLAUDE.md, permissions and hooks, and creates only the subagents the project needs
(plan-reviewer, code-reviewer, security/infra/UI/migration reviewers as required).
Irreversible steps (production deploys, `terraform apply`, live payments, DNS) always wait for your approval.

## Install

```bash
claude plugin marketplace add FarooqShabbir/specforge
claude plugin install specforge@specforge-marketplace
```

## Use

In Claude Code, run `/specforge:specforge` followed by what you want built, e.g.

```
/specforge:specforge Build a full-stack e-commerce app with Stripe checkout, deployed on AWS with SSL
```

Claude also picks it up automatically for large build requests.

## Update

```bash
claude plugin update specforge@specforge-marketplace
```

## Privacy

SpecForge does not collect, store, or send any data. It contains only instructions (a skill file) that run inside your own Claude Code session. It has no hooks, scripts, MCP servers, analytics, or network calls of its own. Any commands run while using it are run by Claude Code on your machine, under your own permission settings.

## License

MIT
