# everything-claude-code

Out of the box, Claude Code is already pretty good. But a tuned `.claude/` directory makes it become **incredibly** more competent for your use case. However, it will become significantly more token-hungry.


> Forked from [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code). All credit to the original author for the foundation. The original repo has ballooned into a multi-agent harness; I distilled it back down to be only relevant to my Claude Code use case.

---

## Easy Set Up

Just clone one of my branches. It has a ready-to-use `.claude/` and a  `.mcp.json` tuned to the project type in the branch name. Replace `.mcp.json`'s place holders with your real keys.

```bash
git clone -b <branch-name> --single-branch https://github.com/waterprooftoaster/everything-claude-code
```

The `main` branch holds the full unfiltered collection to personalize your own. The details are below.

---

## Setting Up Your Own

If you are already familiar with `.claude/`, this section is not for you. If you are not, setting up your own requires some basic understanding of the structure. Claude reads from a `.claude/` directory in your project (or globally from `~/.claude/`). Everything inside shapes how Claude behaves — what it knows, what it does automatically, and how it delegates work. Here's what each piece does:

### `agents/`

Specialized subagents Claude can spin up and delegate tasks to. Each agent is a Markdown file with a YAML frontmatter block defining its name, description, available tools, and model. Claude routes work to agents when a task matches their specialty — e.g., a `code-reviewer` agent gets called during review workflows, a `tdd-guide` agent guides test-first development.

```
agents/
├── architect.md
├── code-reviewer.md
├── tdd-guide.md
├── security-reviewer.md
└── ...
```

### `commands/`

Slash commands that users invoke directly in the Claude Code prompt (e.g., `/tdd`, `/plan`, `/code-review`). Each command is a Markdown file with a `description` frontmatter field. When you type `/tdd`, Claude loads that command's instructions and follows them.

```
commands/
├── tdd.md        → /tdd
├── plan.md       → /plan
├── code-review.md → /code-review
└── ...
```

### `rules/`

Claude always follows these guidelines in every conversation. Think of these as your coding standards, git workflow rules, security policies, and testing requirements baked directly into Claude's behavior. Rules are organized into a `common/` layer (language-agnostic) and language-specific directories that extend it.

```
rules/
├── common/          # universal principles
│   ├── coding-style.md
│   ├── git-workflow.md
│   ├── testing.md
│   └── security.md
├── typescript/      # language specific
├── python/
├── golang/
└── swift/
```

Install only what you need:

```bash
cp -r rules/common ~/.claude/rules/common
cp -r rules/typescript ~/.claude/rules/typescript
```

### `skills/`

Deep reference material for specific tasks. While rules tell Claude *what* to do, skills tell it *how*. Skills are loaded contextually when relevant — e.g., the `tdd-workflow` skill gets pulled in during TDD sessions, `golang-patterns` during Go work. There are skills for everything from `django-tdd` to `liquid-glass-design` to `investor-outreach`.

### `scripts/` and hooks

This is where automation lives. Hooks are shell commands Claude Code runs automatically in response to tool events — before or after Claude edits a file, runs a bash command, or finishes a response.

**Hook triggers:**
| Event | When it fires |
|---|---|
| `PreToolUse` | Before Claude uses a tool |
| `PostToolUse` | After Claude uses a tool |
| `Stop` | After Claude finishes responding |

The hook definitions live in `hooks/hooks.json`. The actual logic runs as Node.js scripts in `scripts/hooks/`:

```
scripts/
├── hooks/
│   ├── post-edit-format.js        # auto-format JS/TS after edits (Biome or Prettier)
│   ├── post-edit-typecheck.js     # TypeScript check after .ts/.tsx edits
│   ├── post-edit-console-warn.js  # warn about console.log after edits
│   ├── post-bash-pr-created.js    # log PR URL after gh pr create
│   ├── post-bash-build-complete.js # async build analysis
│   ├── pre-bash-git-push-reminder.js # pause before git push
│   ├── pre-bash-tmux-reminder.js  # remind to use tmux for long-running commands
│   ├── auto-tmux-dev.js           # auto-start dev servers in tmux
│   ├── check-console-log.js       # scan modified files for console.log on Stop
│   ├── quality-gate.js            # run quality checks after file edits
│   ├── suggest-compact.js         # nudge to compact context at logical intervals
│   └── doc-file-warning.js        # warn when writing non-standard doc files
└── lib/                           # shared utilities
```

Hooks run outside Claude's context — they're executed by the Claude Code harness itself, not by Claude. This means they're reliable, fast, and bypass any instructions Claude might otherwise ignore.

### `settings.json`

The global Claude Code config at `~/.claude/settings.json`. Controls permissions (which tools Claude can use without asking), environment variables, hook registration paths, and other harness-level behavior. This is where you wire up your hooks and set broad defaults.

### `settings.local.json`

Project-scoped overrides that sit in your project root (not committed to git). Use this for project-specific permissions, env vars, or hook configurations that shouldn't apply globally. `settings.local.json` takes precedence over global settings for the project it lives in.

---

## MCPs (Model Context Protocol servers)

MCPs are external servers that extend what Claude can *do*. Think of them as prompt-actuated APIs. A GitHub MCP gives Claude real GitHub API access, a Playwright MCP lets Claude control a browser, and a Supabase MCP lets Claude query your database directly.

### Where to configure them

The original repo recommends putting MCPs in `settings.json`. **I don't do this.** In practice, Claude frequently fails to read MCPs declared in `settings.json` unless you explicitly tell it to look there. The reliable way is `.mcp.json`,

**Use `.mcp.json`, not `settings.json`, for MCPs.**

```json
// .mcp.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your-token-here"
      }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

> **Keep it under 10 MCPs** — each one consumes context window. Realistically, you should only need a few.

---

## Repository structure

```
.
├── agents/          # subagent definitions
├── commands/        # slash commands
├── rules/           # always-on coding standards
├── skills/          # deep reference material
├── scripts/
│   ├── hooks/       # Node.js hook scripts
│   └── lib/         # shared utilities
├── hooks/
│   └── hooks.json   # hook event → script mappings
├── .mpc.json  # MCP reference configs
├── examples/        # example CLAUDE.md files
└── CLAUDE.md        # instructions for Claude in this repo
```
