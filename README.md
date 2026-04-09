# ZeroTrusted.ai Guardrails API — Claude Code Skill

A [Claude Code](https://claude.ai/claude-code) skill for integrating [ZeroTrusted.ai](https://zerotrusted.ai) guardrails into any application. Provides AI-powered PII detection, anonymization, deanonymization, hallucination checking, compliance validation, and audit logging.

## What is a Claude Code Skill?

Claude Code skills are markdown files that give Claude deep knowledge of specific APIs, patterns, and workflows. When loaded into a Claude Code session, this skill enables Claude to generate correct, production-ready ZeroTrusted.ai guardrails integration code for your project.

## Features Covered

- **PII Detection** — Scan text for 50+ entity types (SSN, credit cards, emails, names, addresses, etc.)
- **Anonymization** — Replace PII with realistic fake values, preserving text structure
- **Deanonymization** — Restore original values using stored mappings
- **Hallucination / Reliability Check** — Score AI responses for factual accuracy (0-100)
- **Compliance Check** — Validate against GDPR, CCPA, HIPAA, PCI DSS, and more
- **Audit Logging** — Log complete interaction events for compliance trails
- **Model-Assisted Detection** — Use LLMs to enhance PII detection accuracy

## Quick Start

### 1. Get your API key

Sign up at [zerotrusted.ai](https://zerotrusted.ai) and get your API key.

### 2. Add to your environment

```bash
# .env or .env.local
ZT_GUARDRAILS_API_KEY=zt-your-api-key-here
ZT_ENCRYPTED_PROVIDER_KEY=your-encrypted-key  # For hallucination checks
```

### 3. Load the skill in Claude Code

Copy `zt-guardrails-api.md` into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp zt-guardrails-api.md .claude/skills/
```

Then ask Claude Code to integrate ZeroTrusted.ai guardrails into your project.

## Supported Languages & Frameworks

The skill includes implementation examples for:

- **TypeScript / JavaScript** — Next.js API routes, Express.js
- **Python** — requests library

The patterns are framework-agnostic and can be adapted to any HTTP client.

## API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /detect-sensitive-keywords` | Detect PII entities in text |
| `POST /anonymize-sensitive-keywords` | Replace PII with fake values |
| `POST /deanonymize-sensitive-keywords` | Restore original values |
| `POST /responses/evaluate-reliability` | Score response reliability |
| `POST /responses/check-compliance` | Check regulatory compliance |
| `POST /audits/log-event` | Log audit trail event |

## Related

- [zt-guardrails-copilot-skill](https://github.com/www-zerotrusted-ai/zt-guardrails-copilot-skill) — Claude Code skill for adding guardrails UI to AI copilots

## License

MIT
