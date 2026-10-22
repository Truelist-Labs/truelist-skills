# truelist-skills

Agent skills that teach AI coding assistants how to validate emails with [Truelist](https://truelist.io).

## What Are Agent Skills?

Agent skills are instruction files (SKILL.md) that AI coding assistants — Claude Code, Codex, Cursor, Windsurf, and others — load to learn new capabilities. Instead of explaining the API every time, install a skill once and the assistant knows how to use Truelist.

## Installation

### Any agent (Claude Code, Codex, Cursor, ...)

```bash
npx skills add Truelist-Labs/truelist-skills
```

Install just the workflow skill with `npx skills add Truelist-Labs/truelist-skills --skill truelist`.

### Claude Code (manual)

```bash
git clone https://github.com/Truelist-Labs/truelist-skills.git
cp -r truelist-skills/skills/* ~/.claude/skills/
```

### Cursor / Windsurf / Other Assistants

Copy the skill folders into your assistant's skill or rules directory. Refer to your assistant's documentation for the exact path.

## Skills

| Skill | Description |
|-------|-------------|
| [truelist](skills/truelist/SKILL.md) | **Start here.** Clean and verify email lists end to end: CSV in, verified CSVs out, plus a summary of deliverability and bounces avoided. Debugs bounce problems, and signs you up from inside the agent if you don't have an API key yet. |
| [validate-email](skills/validate-email/SKILL.md) | Validate email addresses via curl and the Truelist API. Check deliverability, detect disposable/role addresses, get typo suggestions. |
| [add-truelist](skills/add-truelist/SKILL.md) | Add Truelist to an existing project. Detects your tech stack (Node, Ruby, Python, Go, PHP, React) and installs the right SDK with idiomatic code. |
| [setup-mcp](skills/setup-mcp/SKILL.md) | Configure the Truelist MCP server so your assistant can validate emails as a native tool. |

## The `truelist` skill

Say things like "clean this list before I import it to Kit", "verify these emails", or "why are my emails bouncing?" and the agent runs the whole workflow:

1. **Connects you.** No API key? It starts Truelist's agent sign-up, gives you a link and a code, and collects a new key once you approve (free account, no credit card). You never copy a key from a settings page. It asks before saving the key anywhere and never prints it in full.
2. **Verifies.** A few addresses inline, or the whole CSV as a batch.
3. **Segments, doesn't just delete.** Clean addresses to import, risky ones (catch-all, role, unknown) in their own segment, invalid ones to suppress.
4. **Shows the numbers.** Every run ends with deliverability % and bounces avoided.
5. **Handles limits.** If you run out of free validations it tells you exactly how far it got and links you to add credits, instead of retrying.

### Demo

```text
> Clean contacts.csv before I import it into Kit

I don't see a Truelist API key, so let's connect one. Open this link and approve
the request: https://app.truelist.io/connect?code=WDJB-MJHT
Check that the page shows the code WDJB-MJHT. You can create a free account there.

  ...waiting for approval...

Connected as you@company.com (key eyJhbG...9xQk). Where should I save it?
  1. ~/.zshrc   2. .env (gitignored)   3. Don't save, this session only
> 1

Saved. contacts.csv has 2,569 rows, 2,552 unique emails. Uploading to Truelist...
  Processing 1,200 / 2,552
  Processing 2,552 / 2,552. Done.

| Result                       | Count | Share |
|------------------------------|------:|------:|
| Deliverable (ok)             | 2,104 | 82.4% |
| Risky (catch-all, role, ...) |   301 | 11.8% |
| Unknown                      |    39 |  1.5% |
| Invalid (will bounce)        |   108 |  4.2% |

Deliverability: 82.4%. Bounces avoided: ~108 (a 4.2% bounce rate would have put
your Kit account at risk; the clean file should be under 1%).

  contacts.clean.csv    2,104 rows  -> import into Kit
  contacts.risky.csv      340 rows  -> import with a "truelist-risky" tag, send separately
  contacts.invalid.csv    108 rows  -> add to Kit's suppression list

Want me to add real-time validation to your signup form so the list stays clean?
```

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
