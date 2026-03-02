# truelist-skills

Agent skills that teach AI coding assistants how to validate emails with [Truelist](https://truelist.io).

## What Are Agent Skills?

Agent skills are instruction files (SKILL.md) that AI coding assistants — Claude Code, Codex, Cursor, Windsurf, and others — load to learn new capabilities. Instead of explaining the API every time, install a skill once and the assistant knows how to use Truelist.

## Installation

### Claude Code

Clone or download this repo, then add the skills directory:

```bash
# Clone into your skills directory
git clone https://github.com/Truelist-io-Email-Validation/truelist-skills.git ~/.claude/skills/truelist-skills

# Or add individual skills
cp -r truelist-skills/skills/validate-email ~/.claude/skills/validate-email
cp -r truelist-skills/skills/add-truelist ~/.claude/skills/add-truelist
cp -r truelist-skills/skills/setup-mcp ~/.claude/skills/setup-mcp
```

### Cursor / Windsurf / Other Assistants

Copy the skill files into your assistant's skill or rules directory. Refer to your assistant's documentation for the exact path.

## Skills

| Skill | Description |
|-------|-------------|
| [validate-email](skills/validate-email/SKILL.md) | Validate email addresses via curl and the Truelist API. Check deliverability, detect disposable/role addresses, get typo suggestions. |
| [add-truelist](skills/add-truelist/SKILL.md) | Add Truelist to an existing project. Detects your tech stack (Node, Ruby, Python, Go, PHP, React) and installs the right SDK with idiomatic code. |
| [setup-mcp](skills/setup-mcp/SKILL.md) | Configure the Truelist MCP server so your assistant can validate emails as a native tool. |

## Truelist Ecosystem

| Package | Description |
|---------|-------------|
| [truelist-openapi](https://github.com/Truelist-io-Email-Validation/truelist-openapi) | OpenAPI specification |
| [truelist-node](https://github.com/Truelist-io-Email-Validation/truelist-node) | Node.js / TypeScript SDK |
| [truelist-mcp](https://github.com/Truelist-io-Email-Validation/truelist-mcp) | MCP server for AI assistants |
| [truelist-cli](https://github.com/Truelist-io-Email-Validation/truelist-cli) | Command-line interface |

SDKs are also available for [Ruby](https://github.com/Truelist-io-Email-Validation/truelist-ruby), [Python](https://github.com/Truelist-io-Email-Validation/truelist-python), [Go](https://github.com/Truelist-io-Email-Validation/truelist-go), [PHP](https://github.com/Truelist-io-Email-Validation/truelist-php), [Java](https://github.com/Truelist-io-Email-Validation/truelist-java), [C#](https://github.com/Truelist-io-Email-Validation/truelist-csharp), [Swift](https://github.com/Truelist-io-Email-Validation/truelist-swift), [Kotlin](https://github.com/Truelist-io-Email-Validation/truelist-kotlin), and [Rust](https://github.com/Truelist-io-Email-Validation/truelist-rust).

## License

MIT
