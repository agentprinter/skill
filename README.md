# AgentPrinter skill

Let your agent send a real letter or postcard to a US address. The agent makes one call to [AgentPrinter](https://agentprinter.app), with its MCP tools or its HTTP API, and gives you a payment link. When you pay, the piece is printed and mailed by USPS First Class.

No account and no API key. You pay per piece through Stripe. Prices are on [agentprinter.app](https://agentprinter.app).

## Install

**Claude Code**, as a plugin (the skill and the MCP server):

```
/plugin marketplace add agentprinter/skill
/plugin install agentprinter@agentprinter
```

**Any agent that reads skills** (Claude Code, Codex, Cursor, OpenCode and others), with the [skills](https://skills.sh) CLI:

```
npx skills add agentprinter/skill
```

**By hand:** copy `skills/agentprinter/` into your agent's skills folder, for example `~/.claude/skills/agentprinter/` for Claude Code.

**The MCP server alone**, in any MCP client: add `https://api.agentprinter.app/mcp` as a remote server (Streamable HTTP). No account or key. In Claude Code: `claude mcp add --transport http agentprinter https://api.agentprinter.app/mcp`.

With the skill alone, the agent needs to make outbound HTTPS requests (with `curl` or an HTTP tool). Calls from a browser sandbox or canvas are blocked.

## What it runs and sends

The plugin is a skill (instructions for the agent) and the address of AgentPrinter's remote MCP server, `https://api.agentprinter.app/mcp` (in `.mcp.json`). It installs nothing and runs no code on your machine: no downloads, no scripts, no hooks.

When you ask to send mail, the agent calls AgentPrinter's MCP tools, or makes HTTPS requests to `api.agentprinter.app` with `curl` or its own HTTP tool. Both reach the same service. A request to send a piece carries:

- the document: your text, a link to a PDF, or the PDF itself (for a PDF on your computer, the agent reads the file and sends it),
- the recipient's name and address, and your return address,
- the options you choose, and a webhook URL if you give one. AgentPrinter then sends status updates for the piece to that URL.

Other requests price a piece without creating it, change an unpaid piece, read the status of a piece, cancel it, or pass on feedback you ask the agent to send. AgentPrinter uses Lob to verify addresses and to print and mail, and Stripe for payment. The [privacy policy](https://agentprinter.app/privacy) says what is kept and for how long.

## Use it

Ask in plain words:

- "Mail this letter to Jane Smith, 123 Main St, Portland OR 97201."
- "Send a postcard to my mom that says happy birthday."
- "Post this invoice PDF to the client."

The agent asks for anything that is missing, such as your return address, then replies with a link to pay and a link to track the piece.

## What is in this repo

- `skills/agentprinter/SKILL.md`: the skill.
- `skills/agentprinter/reference/examples.md`: more requests (PDF by URL, base64 PDF, postcard layouts, webhooks, cancel).
- `.mcp.json`: the MCP server the plugin adds.
- `.claude-plugin/`: the Claude Code plugin and marketplace files.

The full API contract is at [agentprinter.app/v1/openapi.json](https://agentprinter.app/v1/openapi.json), and a summary for agents is at [agentprinter.app/llms.txt](https://agentprinter.app/llms.txt).

## Changes and issues

The AgentPrinter team keeps this skill in step with the API, and updates come from the service's own repo. To report a problem with the skill, open an issue here.

## License

MIT. See [LICENSE](LICENSE).
