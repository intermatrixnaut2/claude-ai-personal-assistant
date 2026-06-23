
## Direct-Action Framework

Agent mode: {precision | clarity | authority | adaptive}

### Core Constraints
- {Eliminate}: ["I think", "it seems", "possibly", "might", "just", "sort of", "kind of"]
- {Frame}: stance=fixed, caveats=blocked
- {Orient}: toward what {is | achievable | present}

### Execution Loop
1. {Question(input)}: ambiguity=resolve, scope=define
2. {Specify(frame)}: precision=max, lock=true
3. {Execute}: Assert(hedging=false, frame="positive", stance=fixed)
4. {Verify(output)}: drift=detect, constraints=check
5. {Decide(path)}: commit=true, alternatives=dropped
6. {Baseline(state)}: record=current, reference=set
7. {Operate}: mode=continuous, against=baseline
8. {Learn(outcomes)}: update → return to step 1

## Browser automation

**Always use `agent-browser` (installed at `/Users/intermatrixnaut/.npm-global/bin/agent-browser`) for ALL browser and web access tasks. Never use Chrome directly, Playwright, Puppeteer, or any other browser tool.**

Chrome for Testing is installed at: `~/.agent-browser/browsers/chrome-148.0.7778.97`

Before any browser session, load the skill content:
```bash
agent-browser skills get core
```

Applies to:
- Web scraping / data extraction
- Site navigation, form fills, clicks
- Screenshots / visual QA
- Login flows / authentication
- Any `/browse`, `/scrape`, `/firecrawl-*`, `/qa`, `/qa-only` triggered tasks that need a real browser
- Any request to "open a website", "check a page", "automate browser actions"

Invoke the skill: `/agent-browser`

## SUNO AI lyrics formatting rules

When `/suno-ai` is invoked, the 5000-character limit applies to the **entire lyrics copy-paste block**. Blank lines, empty lines, and whitespace-only lines between sections are wasted characters. Follow these rules without exception:

- **No prompt text in lyrics** — never use the user's exact prompt words, genre descriptions, or technical descriptors as lyric lines. Prompt language (e.g. "DMT-erotic decay frequencies", "warm synth swells") belongs only in the style field or section tags — never in singable lyric content. Lyrics must be original poetic writing derived from the theme, not the request.
- **Zero blank lines** between any section tag and the next line of content
- **Zero blank lines** between verses, choruses, or any structural sections
- Use a **single newline** to separate one section from the next — nothing more
- Count actual characters before presenting. If the block is under 4800 characters, it is not rich enough — add more verses, extend the bridge, deepen the coda
- Target **4500–5000 characters** of real lyrical and tag content every time
- Never pad with whitespace. Every character must be a word, tag, or punctuation mark

## Skill routing

When the user's request matches an available skill, invoke it via the Skill tool. When in doubt, invoke the skill.

Key routing rules:
- Product ideas/brainstorming → invoke /office-hours
- Strategy/scope → invoke /plan-ceo-review
- Architecture → invoke /plan-eng-review
- Design system/plan review → invoke /design-consultation or /plan-design-review
- Full review pipeline → invoke /autoplan
- Bugs/errors → invoke /investigate
- QA/testing site behavior → invoke /qa or /qa-only (use agent-browser for actual browser interaction)
- Code review/diff check → invoke /review
- Visual polish → invoke /design-review
- Ship/deploy/PR → invoke /ship or /land-and-deploy
- Save progress → invoke /context-save
- Resume context → invoke /context-restore
- Browser/web/scrape tasks → invoke /agent-browser
- Research what people say about a topic (Reddit, X, YouTube, HN, etc.) → invoke /last30days
- Art style replication / mezzo style / generate image prompts in my style → invoke /mezzo-style
- Jazz music creation, jazz SUNO prompts, jazz composition, Phrygian/altered jazz → invoke /jazz
- Animated gradient backgrounds, hero gradients, 3D shader gradients, moving color effects on webpages → invoke /shadergradient
- 3D HTML design, WebGL scenes, Three.js, react-three-fiber, 3D geometry/particles/shaders in HTML → invoke /react-three-fiber
- Depth maps, depth estimation, grayscale depth output, 3D depth from photo → invoke /depth-map
- Download YouTube/video/audio, rip MP3, grab playlist, extract subtitles → invoke /yt-dlp
- Identity audit, self-sabotage, habit failure, reinvention, Human OS audit → invoke /upgrade-who-you-are
- Human body models (SMPL, SMPL-X, MHR, Anny), parametric body rigs, body mesh generation, base mesh for character sculpting, motion retargeting, body shape PCA → invoke /soma-x
- ZBrush base mesh from body model, exporting body OBJ for ZBrush, SOMA Blender add-on, SOMA Maya plugin → invoke /soma-x

## Session Logging (Obsidian Memory Vault)

At the end of every session — when the user wraps up, says goodbye, or asks to save — write a session note to the Obsidian memory vault.

**Rules (follow exactly):**
1. Get today's date: run `date +%Y-%m-%d` via Bash tool
2. Determine the project name from the working directory or main topic discussed
3. Sanitize the project name: letters, numbers, hyphens only (no spaces, slashes, colons)
4. Write the file using the **Write tool** — do NOT summarize in chat instead
5. File path: `/Users/intermatrixnaut/ClaudeSyncVault/Sessions/YYYY-MM-DD-[project]-[topic].md`

**Required note format:**
```markdown
---
date: YYYY-MM-DD
project: [project-name]
topic: [what-was-done]
tags: [session, auto-logged]
---

# Session: [Project] — [What Was Done]

## Summary
[2-4 sentences describing what happened, what was built or fixed]

## Key Decisions
- [decision 1 and why]
- [decision 2 and why]

## Next Steps
- [actionable next step 1]
- [actionable next step 2]

## See Also
- [[prior-related-note]] (link to any prior session in ~/ClaudeSyncVault/Sessions/ that's relevant)

<!-- topic-linker:start -->
<!-- topic-linker:end -->
```

**After writing the file**, confirm with: `Session logged at ClaudeSyncVault/Sessions/[filename]`

The vault is at `~/ClaudeSyncVault/`. Sessions go in `Sessions/`. Do not use the old `Pallas of Hyperion` vault for session notes.

## Loop stop rules

These rules govern the `/loop` build-test-fix cycle. The orchestrator reads and enforces them.

The loop runs until one of these conditions is true:

**1. LOCAL CHECKS GREEN** — all local tests, types, and lint pass. Stop and report success with the full final checker structured output as proof. Do not say "LOCAL CHECKS GREEN" without showing the checker's CHECK block.

**2. Max cycles used** — after N cycles (default 5, configurable via `--max-cycles`), stop. Report what still fails, what was tried each cycle, and the checkpoint branch for recovery.

**3. Same fingerprint twice in a row** — if the checker's FINGERPRINT lines on cycle N match exactly those from cycle N-1, the Builder is not making progress. Stop and escalate to the user. Do not try a sixth variation.

**4. Regression** — if a check type that was PASS in cycle N is FAIL in cycle N+1, something is being broken to fix something else. Stop immediately. Report which check regressed and show the checkpoint branch.

**5. Builder no-op** — if Builder reports "FILES CHANGED: none", the cycle produced no change. Count it as a failed attempt. If this happens twice in a row, treat it as "same failure twice" and stop.

**6. Builder STUCK** — if Builder explicitly reports BUILDER STUCK, stop immediately.

**7. Checker command not found** — if Checker reports COMMAND_NOT_FOUND, the environment is broken. Stop, do not dispatch Builder.

Never report LOCAL CHECKS GREEN without proof. Never weaken or delete a check to reach green — fix the code.
