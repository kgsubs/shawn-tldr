---
name: shawn-tldr
description: "Use this skill whenever the user wants an executive brief, verification, or worth-reading assessment of an article or video. Triggers include \"/shawn-tldr\", \"tldr this article\", \"tldr this\", \"is this worth reading\", \"is this worth watching\", \"executive brief on\", \"verify this article\", \"fact check this piece\", or any time the user provides article text, a file path to an article, a URL, or a video link and asks for a summary plus verification. Produces a structured brief with verdict, bottom line, claim-by-claim verification against two or more independent sources, flagged concerns, novel insights, actionable data, and cross-industry relevance. Do not trigger for general summarization of non-article content (code, docs, internal memos) or for fact-checking single claims outside an article or video context."
---

# shawn-tldr: Executive Intelligence Brief


## MODEL ROUTING

Default to Sonnet. Escalate to Opus when the article or video is dense (financial modeling, legal analysis, scientific methodology, multi-layered causal claims) or when verification surfaces significant disputes requiring judgment.

Do not drop to Haiku: verification reasoning is too deep. If running on Haiku, stop and recommend Sonnet with a one-line reason before proceeding.


## INPUT TYPES

Accept any of the following:

- Pasted article text
- File path to an article (read it)
- Article URL (fetch with WebFetch)
- Video URL (YouTube, Vimeo, or similar)

For article URLs, use this fallback chain in order. Stop at the first tier that yields the full article body (not just metadata, an intro blurb, or a subscribe wall):

Tier 1 -- WebFetch:
  `WebFetch <url>` with prompt: "Return the full article text, all claims, statistics, quotes, expert names, and any named studies. Do not summarize."
  If the returned content includes the full article body, proceed.
  Signs of a partial/paywalled result: content ends abruptly, contains "subscribe to read more" or "upgrade to paid", or is under ~500 words for a substantive article.

Tier 2 -- Firecrawl (JS-rendered, bypasses some soft paywalls):
  Check if $FIRECRAWL_API_KEY is set. If not, run `pass show firecrawl/api-key` to get it.
  POST https://api.firecrawl.dev/v1/scrape with body {"url": "<url>", "formats": ["markdown"]}
  Authorization: Bearer <key>
  Firecrawl renders JS and can bypass soft paywalls (metered access, cookie-gated content).
  It cannot bypass hard paywalls (subscription-required content with server-side enforcement).

Tier 3 -- smry.ai (paywall bypass for major publishers):
  `WebFetch https://smry.ai/<original_url>` (prepend smry.ai/ to the original URL).
  smry.ai works for many major publishers (WSJ, NYT, Bloomberg, The Atlantic, etc.).
  Known NOT to work: every.to (server blocks the iframe). If smry.ai returns "is blocked" or "refused to connect", skip to Tier 4.
  If smry.ai returns full article text, proceed.

Tier 4 -- agent-browser (interactive, last resort):
  1. `agent-browser open <url>`
  2. `agent-browser snapshot -i`
  3. Extract visible article text from the snapshot
  4. `agent-browser close`

If all four tiers fail to return the full article body, state clearly in the brief which tiers were tried, what each returned, and that the brief is based on partial content only. Do not write a verdict without disclosing the limitation.

For video URLs, use this fallback chain in order. Stop at the first tier that yields usable content:

Tier 1 -- yt-dlp (best):
  Check if ~/.config/yt-dlp/cookies.txt exists. If it does, run:
    `yt-dlp --remote-components ejs:github --cookies ~/.config/yt-dlp/cookies.txt --write-auto-sub --skip-download --sub-format ttml -o /tmp/shawn-tldr "<url>"`
  If cookies.txt does not exist, run without --cookies:
    `yt-dlp --remote-components ejs:github --write-auto-sub --skip-download --sub-format ttml -o /tmp/shawn-tldr "<url>"`
  If successful, read the resulting .ttml or .vtt file for the full transcript.
  Use this tier first. It retrieves the actual caption file without page rendering.
  If yt-dlp returns a "Sign in to confirm" error, note it and move to Tier 2 -- do not retry.

Tier 2 -- Firecrawl (JS-rendered page scrape):
  If yt-dlp is not installed or fails, call the Firecrawl API:
  Check if $FIRECRAWL_API_KEY is set. If not, run `pass show firecrawl/api-key` to get it.
  POST https://api.firecrawl.dev/v1/scrape with body {"url": "<video_url>", "formats": ["markdown"]}
  Authorization: Bearer <key>
  Firecrawl renders JS, so it will capture the video description, chapter markers, pinned comments,
  and any transcript text embedded in the DOM. It cannot click UI elements, so it will not return
  a YouTube transcript that requires the "Show transcript" button.

Tier 3 -- agent-browser (interactive, last resort):
  If Tiers 1 and 2 both fail or yield only metadata, use agent-browser to interact with the page:
  1. `agent-browser open <url>`
  2. `agent-browser snapshot -i` -- identify the transcript button or equivalent control
  3. Click the control, re-snapshot, extract the full transcript text
  4. `agent-browser close`
  Use for YouTube ("Show transcript"), Loom, and platforms where the transcript is behind a UI click.

If all three tiers fail, extract title, description, chapter markers, and visible metadata only.
State clearly in the brief which tier succeeded and which fields are based on metadata only.

Apply the same claim-extraction and verification protocol to video content as to written articles.
Note the video runtime in the BOTTOM LINE.

If any fetch or read fails, stop and report the error. Do not proceed on partial content.


## EXECUTION PROTOCOL

1. Accept input. If file path, read it. If video URL, follow video input steps above. If article URL, follow the article fallback chain above -- do not skip to Firecrawl or smry.ai until WebFetch has been tried first. If fetch or read fails at all tiers, stop and report.
2. Read the entire article or full transcript before writing anything.
3. Identify every weight-bearing claim: statistics, causal assertions, forecasts, named studies, quoted experts, financial figures, dates, attributions.
4. For each weight-bearing claim, run a minimum of two independent web searches against authoritative sources (primary documents, peer-reviewed research, government data, original reporting, SEC filings, named experts). Aggregators and SEO content do not count toward the two-source minimum.
5. If the article or video cites a study or report, fetch that source directly. Do not rely on the article's characterization of it.
6. Tag every factual claim in the output with explicit confidence: high, moderate, low, unknown. State the confidence at the start of the claim.
7. Never fabricate. If verification fails or sources conflict, say so explicitly.
8. Write the brief to disk as ~/Downloads/shawn-tldr-[slug]-[YYYY-MM-DD].md where slug is a short kebab-case identifier from the article or video title.


## OUTPUT STRUCTURE

Deliver the brief in this exact order. Plain text headers. No decorative formatting inside the deliverable.

---

VERDICT
One line: GOLD, OKAY, MEH, or TRASH. Followed by a one-sentence justification.

  GOLD  -- must consume; high signal, novel, or directly actionable
  OKAY  -- worth reading/watching when you get to it; solid but not urgent
  MEH   -- worth it but nothing groundbreaking, or covers ground already known
  TRASH -- skip entirely; low signal, misleading, or not worth the time

BOTTOM LINE
Three to five sentences. Core argument, why it matters cross-industry, what action or awareness it warrants. College-senior reading level. No jargon unless defined. For video, include runtime.

KEY CLAIMS AND VERIFICATION
Numbered list. For each claim:
  Claim: [restated in plain English]
  Confidence: [high / moderate / low / unknown]
  Verification: [two or more named independent sources, one-line summary of what each says]
  Status: [CONFIRMED, PARTIALLY CONFIRMED, DISPUTED, UNVERIFIED, NOVEL]

FLAGGED CONCERNS
Bullets. Claims that appear incorrect, unsupported, or presented as fact without evidence. Be direct. Cite contradicting sources.

NOVEL AND USEFUL INSIGHTS
Bullets. Distinguish truly novel (no prior coverage found) from synthesizing-but-useful (known parts assembled usefully).

ACTIONABLE DATA
Bullets. Specific numbers, frameworks, decisions an executive could apply or escalate. Skip section if none.

CROSS-INDUSTRY AND CROSS-REGION RELEVANCE
Two to four bullets. Sectors and regions where this matters most and why. If narrow, say so.

SOURCES
Numbered list. Original article or video first, then every verification source with URL and one-line description.

---


## RULES

- Lead with the strongest counterargument to the article's or video's thesis before supporting it. If the piece is flawed, say so in the VERDICT.
- No praise openers. Start with the verdict.
- Do not anchor to the article's framing. Generate your own assessment of significance independently.
- If two verification sources disagree, present both and explain the disagreement.
- For platform, product, regulatory, or current-event claims, search current sources. Training data is not acceptable verification.
- ASCII only. No em-dashes, en-dashes, smart quotes, or emoji. Use [OK], [FAIL], -> where status markers are needed.
- Italics for commentary only, never inside the deliverable.
- Output file is Markdown.
- The full brief goes to the output file only. Do not reproduce the full brief in the chat response.
- Commentary, caveats about verification difficulty, or notes to the user go in an italicized block after the in-session output, not before.


## DELIVERABLE HANDOFF

After writing the file, output exactly three things in the chat response -- nothing else:

1. VERDICT line (one line, exactly as written in the file: GOLD / OKAY / MEH / TRASH + one-sentence justification).

2. A plain-English summary of what the article or video is about and why it is or is not worth the reader's time. 100 words maximum. No headers, no bullets, no jargon. Write it for someone who has not seen the content.

3. One question, on its own line: "Want me to add this to the brain?"

The VERDICT line must also appear in the VERDICT section of the output file, using the same GOLD / OKAY / MEH / TRASH label. This ensures the verdict is searchable in both the brain vault and the Downloads file.

Do not include the file path, claim counts, flagged concern counts, or token report in the chat response. Those are in the file. If the user asks for those details, read the file.

If the user answers yes to the brain question: create a TLDR project entry in the brain vault at ~/brain/tldr/briefs/ (customize this path to match your vault) with full Obsidian frontmatter including verdict, date, source URL, channel/author, and tags. Tags must include: tldr, the content domain (e.g. ai-video, finance, policy), and the verdict as a tag (gold, okay, meh, or trash). Commit after writing. Do not push.

If the user answers no or does not respond to the brain question: do nothing further.
