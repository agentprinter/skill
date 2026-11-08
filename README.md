# AgentPrinter skill

Let your agent send a real letter or postcard to a US address. The agent makes one HTTP call to the [AgentPrinter](https://agentprinter.app) API and gives you a payment link. When you pay, the piece is printed and mailed by USPS First Class.

No account and no API key. You pay per piece through Stripe. Prices are on [agentprinter.app](https://agentprinter.app).

## Install

**Claude Code**, as a plugin:

```
/plugin marketplace add agentprinter/skill
/plugin install agentprinter@agentprinter
```

**Any agent that reads skills** (Claude Code, Codex, Cursor, OpenCode and others), with the [skills](https://skills.sh) CLI:

```
npx skills add agentprinter/skill
```

**By hand:** copy `skills/agentprinter/` into your agent's skills folder, for example `~/.claude/skills/agentprinter/` for Claude Code.

The agent needs to make outbound HTTPS requests (with `curl` or an HTTP tool). Calls from a browser sandbox or canvas are blocked.

## Use it

Ask in plain words:

- "Mail this letter to Jane Smith, 123 Main St, Portland OR 97201."
- "Send a postcard to my mom that says happy birthday."
- "Post this invoice PDF to the client."

The agent asks for anything that is missing, such as your return address, then replies with a link to pay and a link to track the piece.

## What is in this repo

- `skills/agentprinter/SKILL.md`: the skill.
- `skills/agentprinter/reference/examples.md`: more requests (PDF by URL, base64 PDF, postcard layouts, webhooks, cancel).
- `.claude-plugin/`: the Claude Code plugin and marketplace files.

The full API contract is at [agentprinter.app/v1/openapi.json](https://agentprinter.app/v1/openapi.json), and a summary for agents is at [agentprinter.app/llms.txt](https://agentprinter.app/llms.txt).

## Changes and issues

The AgentPrinter team keeps this skill in step with the API, and updates come from the service's own repo. To report a problem with the skill, open an issue here.

## License

MIT. See [LICENSE](LICENSE).
