---
name: golang-html-template
description: "Go html/template — contextual auto-escaping internals, the Clone()-per-page isolation pattern, and the specific pitfalls introduced by htmx/Alpine-style custom attributes (hx-get, hx-post, hx-vals, x-data). Use when writing or reviewing server-rendered HTML in Go (html/template, embed.FS templates), building or reviewing HTMX/Alpine.js integrations on top of html/template, debugging blank pages or wrong content across multiple templates ({{define}} collisions), or investigating why a custom (non-standard) HTML attribute isn't being escaped as expected. Not for text/template (no auto-escaping, different risk profile) or for client-side JS template engines."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang's html/template.
metadata:
  author: esrid (https://github.com/esrid/golang-html-template-skill — not part of upstream samber/cc-skills-golang)
  version: "1.0.0"
  openclaw:
    emoji: "🧩"
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*)
---

**Persona:** You are a Go engineer who has been bitten by silent template bugs before — a blank page with no error, a broken JSON attribute that only shows up in production, a login form that stops working after adding a second page. You reach for the contextual auto-escaper's actual behavior, not folk wisdom about it.

## 1. html/template vs text/template

Always `html/template` for anything served to a browser. It auto-escapes based on where a value lands (HTML body, attribute, URL, JS, CSS) — `text/template` has none of this and is an XSS risk for web output. Same syntax, different safety guarantee.

## 2. The contextual escaper only knows attributes it was told about

`html/template` parses the surrounding HTML at template-compile time and picks an escaper per position. It has a **hardcoded list** of attributes it treats as URLs: `href`, `src`, `action`, `formaction`, and a few others. It also recognizes `<script>` bodies and `on*` event handler attributes (`onclick`, `onchange`...) as JS context.

**It does NOT recognize custom attributes** — `hx-get`, `hx-post`, `hx-vals`, `hx-target`, Alpine's `x-data`, `x-on:click`, or any `data-*` attribute. A value placed there gets only generic HTML-attribute escaping (quote → `&#34;`), never URL-percent-encoding, never the `javascript:` scheme defanging that `href`/`src` get for free.

Verified behavior (`{{.ID}}` = `` a"b/c?d=1&e<f> ``):
```
href="/x/{{.ID}}"          → href="/x/a%22b/c?d=1&amp;e%3cf%3e"        (URL-encoded)
hx-get="/x/{{.ID}}"        → hx-get="/x/a&#34;b/c?d=1&amp;e&lt;f&gt;"  (NOT URL-encoded, just HTML-escaped)
onclick="f('{{.ID}}')"     → onclick="f('a"b\/c?d=1&e<f>')" (JS-escaped, correct)
```

**Consequence:** any dynamic value inside `hx-get`/`hx-post`/`hx-vals`/`hx-target` (or any other htmx/Alpine attribute) must be escaped explicitly with the `urlquery` builtin function when it's meant to be a URL segment or query value:
```go
hx-post="/admin/menu/{{.ID | urlquery}}/toggle"
```
Don't rely on the attribute name alone to get you safety — only `href`/`src`/`action`/`formaction` do.

## 3. JSON embedded inside an HTML attribute is a double-escaping trap

`hx-vals='{"id":"{{.ID}}"}'` looks reasonable but is broken for any `.ID` containing a quote. The browser HTML-decodes the attribute value *first* (`&#34;` → `"`) before HTMX ever parses it as JSON. `html/template` only tracks **one** escaping context (HTML attribute) — it has no idea the attribute's content is meant to be valid JSON, so it never applies JSON string-escaping (`\"`) underneath the HTML-escaping layer. The result: a literal unescaped quote lands inside the JSON string and breaks parsing.

**Fix:** don't interpolate raw Go values into hand-written JSON inside a template attribute. Put dynamic identifiers in the URL path/query instead (which the fix in #2 already handles safely), and reserve `hx-vals` for static JSON or values built server-side with `encoding/json.Marshal` and injected as a single `template.JS`-typed field **only if the source is already trusted/sanitized** — `template.JS` bypasses escaping entirely.

## 4. The Clone()-per-page pattern — and why the order matters

Every `{{define "name"}}` in a parsed template set lives in one global namespace. Two files defining `{{define "content"}}` collide silently — the last one parsed wins for *all* pages using that name. Symptom: some pages render blank, others show the wrong content, and there's no error.

```go
// 1. Parse the shared layout ONCE, containing {{block "content" .}}{{end}}
base := template.Must(template.New("").ParseFS(fsys, "layout.html"))

// 2. For EACH page, Clone the layout BEFORE it has ever been executed,
//    then parse that page's own {{define "content"}} into the clone.
for _, page := range pages {
    t := template.Must(base.Clone())
    pages[page] = template.Must(t.ParseFS(fsys, page))
}

// 3. Execute the clone, never the shared `base` itself.
pages["dashboard.html"].ExecuteTemplate(&buf, "layout", data)
```

Internals that make this work (verified against Go's source, `html/template/template.go` + `text/template/template.go`):
- `Clone()` fails if the template has already been executed with an escaping error — always clone from an unexecuted base.
- `Clone()` creates a **new, empty** definition set — it does not carry over `{{define}}` blocks from the parent, only the parsed tree + FuncMap. This is what gives each page isolation.
- Escaping is **lazy and mutates the parse tree**: the first `ExecuteTemplate` call on a given clone rewrites `{{.}}` into `{{. | urlescaper | attrescaper}}` (etc.) in place. Every later `Execute` on that *same* clone reuses the already-escaped tree — cheap and correct. Clone **before** that first execution, or the clone inherits an already-baked (and possibly wrong-context) tree.
- If two branches of an `{{if}}` leave the parser in different HTML/JS/URL states, compilation fails with `ErrBranchEnd` at first-execution time — a real error, not silent corruption. `{{range}}` inside a `<script>` block can similarly fail with `ErrRangeLoopReentry`.

## 5. `//go:embed` does not flatten subdirectories

```go
//go:embed templates/*.html
var fsys embed.FS
```
This glob only matches files directly under `templates/` — files under `templates/admin/*.html` are invisible to it and any `ParseFS`/`Clone`+`ParseFS` call referencing them will panic with `pattern matches no files`. Add a second pattern explicitly:
```go
//go:embed templates/*.html templates/admin/*.html
```
Symmetrically, when serving embedded static assets directly (not through html/template) — `//go:embed static` preserves the `static/` prefix inside the FS tree; `http.FileServer(http.FS(fsys))` will 404 everything unless you `fs.Sub(fsys, "static")` first.

## 6. Buffer, then flush — never execute straight to `http.ResponseWriter`

```go
var buf bytes.Buffer
if err := t.ExecuteTemplate(&buf, "layout", data); err != nil {
    http.Error(w, "Internal Server Error", http.StatusInternalServerError)
    return
}
w.Write(buf.Bytes())
```
A template error mid-execution against a live `ResponseWriter` leaves a half-rendered page with a 200 status already sent — the client sees broken HTML with no error. Buffering catches the error before anything is written.

## 7. Parse once, cache forever — except in dev

```go
func (c *Templates) Instance() *template.Template {
    if c.dev {
        return c.parse() // reparse every request — instant feedback while iterating
    }
    return c.cached // parsed once at startup
}
```
`Clone()` is ~100x faster than re-`Parse()`-ing from scratch, but neither matters if you're not caching at all in production. Gate re-parsing behind a dev-mode flag (env var), not a hardcoded always-reparse — forgetting this either makes local iteration painful (restart per template edit) or wastes CPU per request in production.

## 8. Unrelated but adjacent: Go 1.22+ `ServeMux` exact-match

`mux.HandleFunc("GET /", handler)` is a catch-all for every unmatched GET request in Go 1.22+'s pattern syntax — it is not "the home page." Use `GET /{$}` for an exact match on `/`. This bites HTMX-heavy apps particularly hard because partial-page routes (`GET /admin/menu`, `GET /admin/orders`) can get silently swallowed by an overly broad `GET /` registered earlier.

## Checklist before shipping a template that uses htmx/Alpine attributes

- [ ] Every `{{...}}` inside `hx-get`/`hx-post`/`hx-vals`/`hx-target`/`x-data`/any `data-*` attribute is either static or passed through `| urlquery` (if it's a URL segment) — never assume the attribute name gives you automatic escaping.
- [ ] No dynamic Go value is interpolated into hand-written JSON inside an attribute (`hx-vals`, `x-data="{...}"`) — move the data into the URL, or marshal server-side into `template.JS` only from a trusted source.
- [ ] Every page sharing a layout was produced via `Clone()` of an **unexecuted** base template.
- [ ] `//go:embed` patterns list every subdirectory that actually contains templates.
- [ ] All renders go through a buffer, not directly to `w`.
- [ ] A dev-mode flag reparses templates per-request; production caches.
