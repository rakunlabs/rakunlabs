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
| [`rakunlabs-go`](./rakunlabs-go/) | Build a Go service the rakunlabs way: project layout, `main.go` wiring, and quickstarts for `into`, `logi`, `chu`, `tell`, `ada`, `ok`, `cache`, `query`, `bw`, `alan`. |

## Install

Pick the folder that matches your agent and copy the skill into it. Examples
for `rakunlabs-go` (replace with any skill name):

### Claude Code

```sh
mkdir -p ~/.claude/skills
cp -r rakunlabs-go ~/.claude/skills/
```

### opencode

Global (available in every project):

```sh
mkdir -p ~/.config/opencode/skill
cp -r rakunlabs-go ~/.config/opencode/skill/
```

Or per-project (only inside one repo):

```sh
mkdir -p /path/to/project/.opencode/skill
cp -r rakunlabs-go /path/to/project/.opencode/skill/
```

Or leave it where it is and point opencode at this folder in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": { "paths": ["./skills"] }
}
```

### Other agents (`~/.agents`)

```sh
mkdir -p ~/.agents/skills
cp -r rakunlabs-go ~/.agents/skills/
```

## After installing

Restart your agent (or start a new session) so it picks up the new skill. The
agent loads a skill on demand when your request matches the skill's
`description`, so just ask it to build or wire a rakunlabs Go service.

## Authoring notes

- Folder name must equal the `name:` in the skill's frontmatter, lowercase and
  hyphen-separated.
- The `description:` should say *what* the skill does **and** *when* to trigger
  it, front-loading the keywords a user is likely to type — that is what the
  agent matches against.
