# Publishing this plugin to the Claude Code marketplace

There is **no app-store submission** for Claude Code plugins. A "marketplace" is just a
**public git repo** containing a `.claude-plugin/marketplace.json`. Publishing means:
make this repo public, push it, and share the two install commands. That's the whole thing.

This repo is already structured correctly, so publishing is mostly a visibility + push step.

---

## Repo layout (already in place)

```
.claude-plugin/
  marketplace.json                      # declares the "cyborgdb-public" marketplace + its plugins
plugins/
  cyborgdb-skills/
    .claude-plugin/plugin.json          # the plugin manifest (name, version, author)
    skills/
      cyborgdb-core/SKILL.md
      cyborgdb-install/SKILL.md
      cyborgdb-migrate/SKILL.md
      cyborgdb-sdks/SKILL.md
      cyborgdb-service/SKILL.md
README.md                               # user-facing install + usage
PUBLISHING.md                           # this file
```

- `marketplace.json` → `name: "cyborgdb-public"` and lists the `cyborgdb-skills` plugin.
- `plugin.json` → `name: "cyborgdb-skills"`, `version` (bump on each release).

---

## One-time publish checklist

Run these from the repo root.

### 1. Make sure everything is committed and pushed

```bash
git status                 # working tree should be clean
git push                   # push any local commits to origin/main
```

### 2. Make the repo public

The repo currently has **INTERNAL** (org-only) visibility. External users cannot add a
marketplace they can't see, so it must be **public**:

```bash
gh repo edit cyborginc/cyborgdb-claude-plugin \
  --visibility public \
  --accept-visibility-change-consequences
```

Verify:

```bash
gh repo view cyborginc/cyborgdb-claude-plugin --json visibility
# expect: {"visibility":"PUBLIC"}
```

### 3. Smoke-test the install

In a fresh Claude Code session:

```
/plugin marketplace add cyborginc/cyborgdb-claude-plugin
/plugin install cyborgdb-skills@cyborgdb-public
```

Then confirm the skills are available (e.g. ask "set up cyborgdb-core").

Done. Share those two commands with anyone who wants to try it.

---

## How users install it

```
/plugin marketplace add cyborginc/cyborgdb-claude-plugin
/plugin install cyborgdb-skills@cyborgdb-public
```

- `cyborginc/cyborgdb-claude-plugin` — the GitHub repo (owner/name shorthand).
- `cyborgdb-skills` — the plugin name from `plugin.json`.
- `cyborgdb-public` — the marketplace name from `marketplace.json`.

Users can update later with `/plugin marketplace update cyborgdb-public`.

---

## Shipping updates (after the first publish)

1. Edit the skill(s) under `plugins/cyborgdb-skills/skills/`.
2. Bump `version` in `plugins/cyborgdb-skills/.claude-plugin/plugin.json`
   (semver — e.g. `0.10.6` → `0.10.7`).
3. Commit and push to `main`.

That's all — because the marketplace is the git repo, pushing to `main` publishes the update.
Users pick it up via `/plugin marketplace update cyborgdb-public`. No re-submission, no review step.

---

## Gotchas

- **Repo must stay public.** Flipping back to private/internal breaks every user's marketplace.
- **`marketplace.json` name is the marketplace id** users type after `@` — don't rename
  `cyborgdb-public` casually; it changes the install command for everyone.
- **Validate JSON before pushing** — a malformed `marketplace.json` or `plugin.json` makes the
  marketplace fail to load with no obvious error:
  ```bash
  cat .claude-plugin/marketplace.json | python3 -m json.tool > /dev/null && echo OK
  cat plugins/cyborgdb-skills/.claude-plugin/plugin.json | python3 -m json.tool > /dev/null && echo OK
  ```
- **`source` paths are relative** to the repo root in `marketplace.json`
  (e.g. `./plugins/cyborgdb-skills`).
