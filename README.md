# golang-html-template — agent skill

Agent skill for Go `html/template`: contextual auto-escaping internals, the `Clone()`-per-page isolation pattern, and the pitfalls of htmx/Alpine custom attributes (`hx-get`, `hx-vals`, `x-data`) that the escaper does not know about.

Works with Claude Code and any agent that reads the [SKILL.md](SKILL.md) format.

## What it covers

- Why `hx-get`/`hx-post`/`x-data` values are **not** URL-escaped like `href`/`src` — and the `| urlquery` fix
- The JSON-inside-attribute double-escaping trap (`hx-vals`)
- `Clone()`-per-page to avoid silent `{{define}}` collisions, and why clone-before-first-execute matters
- `//go:embed` subdirectory gotchas, buffer-then-flush rendering, dev-mode reparse
- Go 1.22+ `ServeMux` `GET /` vs `GET /{$}`

## Install

### Claude Code

```bash
git clone https://github.com/esrid/golang-html-template-skill ~/.claude/skills/golang-html-template
```

Or per-project: clone into `.claude/skills/golang-html-template` at the repo root.

### Other agents

Point your agent at `SKILL.md` — it is self-contained (frontmatter description + instructions, no external files).

## License

MIT
