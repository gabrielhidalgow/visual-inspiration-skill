# visual-inspiration

**Credited visual references and a reference board for any design brief.** An agent skill: plain
markdown instructions, not tied to one vendor.

Say *"find inspiration for a bold, editorial-style flyer for a Sydney design meetup"* and it:

1. reads the brief (medium, style, industry, constraints) without interviewing you;
2. picks the **3–5 curated sources that fit the medium**: Fonts In Use, typo/graphic posters, BP&O,
   Brand New, Identity Designed, The Dieline, Logobook, Typewolf, Awwwards, SiteInspire, Minimal Gallery
   or Are.na;
3. collects real projects with title, creator and link, downloads the previews and screens them (format,
   resolution, near-duplicates);
4. **looks at every image** and keeps the 12–20 strongest, most on-brief ones;
5. writes `./inspiration/<slug>/board.html`: a responsive grid of credited cards, plus an analysis of
   recurring patterns, 3–5 creative directions linked to the references that support them, a sampled
   palette and a type direction.

Free sources only: web search, web fetch and curl. No paid APIs, no API keys, no logins.

## Requirements

An agent with web search and web fetch tools and a shell: **bash**, `curl`, `jq`, `file`, and **`uv`**
(or any Python 3 with Pillow).

## Install

**Any agent, one command.** Works for Claude Code, Cursor, Codex, GitHub Copilot, Windsurf, Gemini CLI,
OpenCode and others:

```bash
npx skills add gabrielhidalgow/visual-inspiration-skill
```

Add `-g` to install it for every project rather than just the current one.

**Claude Code, natively.** Register this repo as a plugin marketplace:

```bash
/plugin marketplace add gabrielhidalgow/visual-inspiration-skill
```

Then run `/plugin install visual-inspiration@visual-inspiration-skill`.

## Hacking on it

Clone it and symlink the clone, so edits are live with no sync step:

```bash
git clone https://github.com/gabrielhidalgow/visual-inspiration-skill.git ~/code/visual-inspiration-skill
ln -s ~/code/visual-inspiration-skill ~/.claude/skills/visual-inspiration
```

Keep test runs and backups **outside** both the clone and `~/.claude/skills/`. Every subdirectory there
registers as a skill, and anything inside the clone is read as part of this one.

## What it won't do

- **Get past bot protection.** Behance, Dribbble and Land-book block automated access. The skill logs
  them and moves on, and reaches that work through Are.na saves that link back to the original.
- **Invent credits.** If a creator isn't stated in a retrieved page, the card says so.
- **Publish the board.** It embeds other people's work, so it stays a local file.
- **Duplicate Mobbin or Pinterest.** For app screens and flows, use the Mobbin MCP or refero-design. For
  Pinterest, use the [pinterest skill](https://github.com/gabrielhidalgow/pinterest-skill).

## Not usable in Claude chat

Agent Skills there run in a sandbox with no network access, so nothing can be fetched or downloaded.

## Scope

References are for inspiration and direction. They are third-party copyrighted work, never traced,
never shipped inside a deliverable, and never fed to an image generator as a style target. The skill
stays at research scale: a few sources, one pass, a few dozen candidates.
