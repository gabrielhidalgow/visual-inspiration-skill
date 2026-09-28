# visual-inspiration

**Quick, credited visual references for the design you're working on, shown right in the session.** An
agent skill: plain markdown instructions, not tied to one vendor.

Say *"find inspiration for a minimalist coffee brand logo"* and it:

1. reads the brief (medium, style, subject) without interviewing you;
2. picks the **3–4 curated sources that fit the medium**: Fonts In Use, typo/graphic posters, BP&O,
   Brand New, Identity Designed, The Dieline, Logobook, Typewolf, Awwwards, SiteInspire, Minimal Gallery
   or Are.na;
3. collects around 20 real projects with title, creator and link, and screens them;
4. **looks at them** and picks the 9 strongest, covering 2–3 distinct directions;
5. shows **one numbered contact sheet** in the conversation, with a short reply: 2–3 directions, a
   palette, a type note, and `n — title (creator)` links to the originals.

It writes **nothing into your project**. Everything stays in the session scratchpad. Say
**"more like 4"** to search around a reference.

Free sources only: web search, web fetch and curl. No paid APIs, no API keys, no logins.

## Requirements

An agent with web search and web fetch tools and a shell: **bash**, `curl`, `jq`, `file`, and **`uv`**
(or any Python 3 with Pillow). SVG logos are rasterised with macOS `qlmanage` or `rsvg-convert`.

## Install

**Any agent, one command** (Claude Code, Cursor, Codex, GitHub Copilot, Windsurf, Gemini CLI, OpenCode…):

```bash
npx skills add gabrielhidalgow/visual-inspiration-skill
```

**Claude Code, natively:**

```bash
/plugin marketplace add gabrielhidalgow/visual-inspiration-skill
```

Then run `/plugin install visual-inspiration@visual-inspiration-skill`.

## Hacking on it

```bash
git clone https://github.com/gabrielhidalgow/visual-inspiration-skill.git ~/code/visual-inspiration-skill
ln -s ~/code/visual-inspiration-skill ~/.claude/skills/visual-inspiration
```

Keep test runs and backups **outside** both the clone and `~/.claude/skills/`. Every subdirectory there
registers as a skill, and anything inside the clone is read as part of this one.

## What it won't do

- **Get past bot protection.** Behance, Dribbble and Land-book block automated access; the skill skips
  them.
- **Invent credits.** If a creator isn't stated in a retrieved page, it says so.
- **Write files into your project, or publish anything.** The sheet shows other people's work.
- **Duplicate Mobbin or Pinterest.** For app screens and flows, use the Mobbin MCP or refero-design. For
  Pinterest, use the [pinterest skill](https://github.com/gabrielhidalgow/pinterest-skill).

## Not usable in Claude chat

Agent Skills there run in a sandbox with no network access.

## Scope

References are for inspiration and direction. They are third-party copyrighted work, never traced,
never shipped inside a deliverable, and never fed to an image generator as a style target.
