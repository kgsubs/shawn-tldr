# shawn-tldr

A Claude Code skill that reads an article or video, verifies its claims against independent sources, and tells you whether it's worth your time. You get a one-word verdict (GOLD / OKAY / MEH / TRASH) plus a structured brief.

## What it does

Point it at a URL, paste article full text, give it a file path, or drop a YouTube link. It fetches the content, extracts every factual claim, and verifies each one against at least two independent authoritative sources. Then it writes a brief in this structure:

- VERDICT -- GOLD, OKAY, MEH, or TRASH, with a one-sentence justification
- BOTTOM LINE -- three to five sentences on the core argument and why it matters
- KEY CLAIMS AND VERIFICATION -- each claim, its confidence level, sources checked, and status (CONFIRMED / DISPUTED / UNVERIFIED / etc.)
- FLAGGED CONCERNS -- claims that appear incorrect or unsupported, with citations
- NOVEL AND USEFUL INSIGHTS -- what was actually new vs. useful-but-familiar
- ACTIONABLE DATA -- specific numbers or frameworks you can act on
- CROSS-INDUSTRY RELEVANCE -- where else this matters and why
- SOURCES -- everything consulted, with URLs

The chat response gives you just the verdict, a plain-English summary, and a prompt to save it to your notes. The full brief goes to `~/Downloads/shawn-tldr-[slug]-[YYYY-MM-DD].md`. 

For articles, it works through a four-tier fallback stack: plain WebFetch first, then Firecrawl (JS rendering + soft paywall bypass), then smry.ai (major publisher paywall bypass), then agent-browser as a last resort. For video, it tries yt-dlp first for a full transcript, then Firecrawl, then agent-browser.

## Requirements

**Required:**
- [Claude Code](https://claude.ai/code)

That is all you need to run it on plain-HTML articles.

**Optional** (each unlocks an additional capability):

| Tool | What it adds |
|---|---|
| Firecrawl API key | JS-rendered pages, soft paywall bypass. Set as `$FIRECRAWL_API_KEY` or store via `pass` at `firecrawl/api-key`. |
| yt-dlp | Full transcript extraction for YouTube, Vimeo, and similar. Install: `pip install yt-dlp` or `brew install yt-dlp`. |
| agent-browser | Interactive page access for hard cases (paywalled video platforms, sites that block scrapers). Comes with Claude Code. |

## How to install

```bash
git clone https://github.com/kgsubs/shawn-tldr ~/.claude/skills/shawn-tldr
```

That is the entire install. Claude Code loads skills from `~/.claude/skills/` automatically on next launch.

**Use it:**

```
/shawn-tldr https://example.com/some-article
```

Or paste a URL or article text and ask "tldr this" -- the skill triggers on natural language too.

## Customization

**Different slash command:** rename the cloned directory and update the `name:` field at the top of `SKILL.md`. For example, to use `/tldr`:

```bash
mv ~/.claude/skills/shawn-tldr ~/.claude/skills/tldr
# then edit SKILL.md: change `name: shawn-tldr` to `name: tldr`
```

**Brain vault path:** if you want the "add to brain" step to save somewhere other than `~/brain/tldr/briefs/`, edit that path in the DELIVERABLE HANDOFF section of `SKILL.md`.

## License

MIT
