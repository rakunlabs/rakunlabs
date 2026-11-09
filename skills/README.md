# rakunlabs agent skills

Portable agent **skills** for working with the rakunlabs Go ecosystem.

Each skill is a self-contained folder with a `SKILL.md` file. The `SKILL.md`
format (YAML frontmatter with `name` + `description`, followed by a Markdown
body) is shared across coding agents — Claude Code, opencode, and any agent
that scans `~/.agents/skills/`. To install a skill you just copy its folder
into the directory your agent watches.

## Available skills

| Skill | What it does |
| --- | --- |
| [`rakunlabs-go`](./rakunlabs-go/) | Build a Go service the rakunlabs way: project layout with an embedded UI, `main.go` wiring, quickstarts for `into`, `logi`, `chu`, `gofret`, `tell`, `ada`, `ok`, `cache`, `query`, `muz`, `types`, `bw`, `alan`, `hardloop`, `tummy`, and the **ada Auth Kit** (login, users, roles, admin page, MCP OAuth). |

## Install

### With the `skills` CLI (recommended)

The [`skills`](https://github.com/vercel-labs/skills) CLI detects the coding
agents you have (Claude Code, opencode, Codex, Cursor, …) and installs the
skill for each of them:

```sh
npx skills add rakunlabs/rakunlabs --skill rakunlabs-go -g
# or
pnpx skills add rakunlabs/rakunlabs --skill rakunlabs-go -g
```

- Drop `-g` to install into the current project instead of your home directory.
- Pick agents explicitly with `-a`, e.g. `-a claude-code -a opencode`.
- List what this repo offers with `npx skills add rakunlabs/rakunlabs --list`.
- Update later with `npx skills update`.

### Manual copy

Pick the folder that matches your agent and copy the skill into it. Examples
for `rakunlabs-go` (replace with any skill name):

#### Claude Code

```sh
mkdir -p ~/.claude/skills
cp -r rakunlabs-go ~/.claude/skills/
```

#### opencode

Global (available in every project):

```sh
mkdir -p ~/.config/opencode/skills
cp -r rakunlabs-go ~/.config/opencode/skills/
```

Or per-project (only inside one repo):

```sh
mkdir -p /path/to/project/.opencode/skills
cp -r rakunlabs-go /path/to/project/.opencode/skills/
```

Or leave it where it is and point opencode at this folder in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": { "paths": ["./skills"] }
}
```

#### Other agents (`~/.agents`)

```sh
mkdir -p ~/.agents/skills
cp -r rakunlabs-go ~/.agents/skills/
```

## After installing

Restart your agent (or start a new session) so it picks up the new skill. The
agent loads a skill on demand when your request matches the skill's
`description`, so just ask it to build or wire a rakunlabs Go service.

## Authoring notes

- Keep `SKILL.md` an index: frontmatter, when to use, a table of
  `references/*.md`, and the short conventions. Put each topic in its own
  reference file so the agent loads only what the task needs. Link files
  relatively (`references/x.md`; inside `references/`, `x.md`).

- Folder name must equal the `name:` in the skill's frontmatter, lowercase and
  hyphen-separated.
- The `description:` should say *what* the skill does **and** *when* to trigger
  it, front-loading the keywords a user is likely to type — that is what the
  agent matches against.
