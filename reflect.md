---
description: "Analyze Obsidian notes across time - today, yesterday, 1w, 1m, 1y ago"
---

Time-travel analysis of your Obsidian vault. Surfaces patterns, threads, and insights across 5 time slices. Writes results to an Obsidian reflection note linked from today's daily note.

## Configuration

Before using this command, set these values:

- **VAULT_PATH**: `/path/to/your/obsidian/vault` — the root of your Obsidian vault
- **READING_TAG** (optional): `[[reading]]` — if your vault has a tag for book quotes/reading wisdom, set it here. Leave empty to skip the reading wisdom search.
- **EXCLUDED_FOLDERS**: Folders whose contents are never sent to the API. Default list:
  - `Reflections/`
  - `Private/`
  - `Finance/`
  - `Health/`
  Edit this list to match your vault. Any file whose path contains a listed folder name is invisible to all phases.

**Privacy notice**: Running this command sends note contents from your Obsidian vault to the Claude API for analysis. Notes in `EXCLUDED_FOLDERS` are never read. Notes with `private: true` frontmatter or a `#private` tag are skipped. Sensitive patterns (SSNs, credit cards, etc.) are redacted before analysis. Your original vault files are never modified.

**Vault path**: `$VAULT_PATH`

**Optional argument**: $ARGUMENTS
- If provided, use it as the anchor date instead of today (format: YYYY-MM-DD)
- If empty, use today's date

**CRITICAL: Determining today's date**
Do NOT rely on any date from the system prompt or your training data. Always run `date +%Y-%m-%d` via Bash to get the actual current date. Use this for the anchor date (unless an argument was provided) and for all file naming.

## Phase 1: Gather Date-Based Notes

Calculate the 5 target dates relative to the anchor date:
- **Today** (anchor date)
- **Yesterday** (anchor - 1 day)
- **1 week ago** (anchor - 7 days)
- **1 month ago** (anchor - 1 month)
- **1 year ago** (anchor - 1 year)

For each target date, find notes using these 3 strategies:

**Exclusion**: Skip any files whose path contains a folder listed in `EXCLUDED_FOLDERS`. This applies to **every phase** — Glob results, Grep results, and Read targets. If a file is in an excluded folder, treat it as invisible. This prevents self-referential loops (Reflections), protects sensitive content (Private, Finance, Health), and respects the user's privacy boundaries.

Exception: Phase 4c reads recent reflections for thinker deduplication — this is exempt since it reads only the "Thinkers to Explore" section.

**Private-note exclusion**: Before reading any note's content, batch-check all candidate files for private markers. Use Grep in `files_with_matches` mode (returns only file paths — private note content never enters the API):
1. Check for `^private: true` in frontmatter
2. Check for `#private` anywhere in the note

Remove any matching files from the working set. They are excluded from all subsequent phases.

### Strategy A: Daily notes by filename

Search for files matching these naming patterns:
- `YYYY-MM-DD.md` (e.g., `2025-02-14.md`)
- Legacy format: `Month Dayth, YYYY.md` (e.g., `February 14th, 2025.md`) — check with ordinal suffixes (st, nd, rd, th)

Use Glob to find these files anywhere in the vault.

### Strategy B: Frontmatter created date

Use Grep to search all `.md` files in the vault for frontmatter `created:` fields matching each target date. The field may appear as:
- `created: YYYY-MM-DD`
- `created: "YYYY-MM-DD"`
- `created: YYYY-MM-DDT...` (with time component)

This is fast — just file path matching, no content reading yet.

### Strategy C: Follow wiki links

For every note found in A and B, extract `[[wiki links]]` from their content. Resolve each link to a file in the vault:
- `[[Note Name]]` → find `Note Name.md` anywhere in the vault
- `[[Note Name|display text]]` → use the part before the pipe

Only follow **one level** of links (don't recurse). Add these linked notes to the collection.

**After Phase 1**: Report what you found:
> "Found [N] date-based notes and [M] linked notes across [X] time slices. Proceeding to keyword expansion..."

## Phase 2: Keyword Expansion

### Step 1: Read the Phase 1 notes
Read all notes gathered so far. Keep them organized by which time slice they came from. After reading, apply sensitive-pattern redaction (see Phase 3) to the in-memory content before extracting keywords — this prevents sensitive patterns from becoming search terms.

### Step 2: Extract key terms
From the content of Phase 1 notes, extract:
- **Tags**: All `#tag` references
- **Wiki links**: All `[[...]]` references (these are strong topic signals)
- **Proper nouns**: People names, project names, company names, place names
- **Distinctive terms**: Technical concepts, unusual phrases — skip generic stopwords

Aim for 10-30 keywords total across all time slices.

### Step 3: Search the vault
Use Grep to search all `.md` files in the vault for each keyword (or batch them). Find topically related notes regardless of creation date. **Filter out any results in `EXCLUDED_FOLDERS` before proceeding.**

### Step 3b: Search the reading wisdom archive (optional)

If a READING_TAG is configured above, search for notes containing that tag combined with keywords from Step 2. This surfaces book quotes and reading notes that connect to the current time slices. Include the 2-3 most relevant reading wisdom notes in the final set. **Filter out any results in `EXCLUDED_FOLDERS` before proceeding.**

If no READING_TAG is configured, skip this step.

### Step 3c: Private-note check for new files
Batch-check all newly discovered keyword-matched files (from Steps 3 and 3b) for private markers using the same Grep `files_with_matches` check as Phase 1: `^private: true` frontmatter and `#private` tags. Remove matches before proceeding.

### Step 4: Cap and deduplicate
- Rank results by how many keywords each file matches
- Take only the **top 10-15 additional notes** (not already in the Phase 1 set)
- Combine with Phase 1 notes and deduplicate

**After Phase 2**: Report the inventory:
> "Total notes gathered: [N]. [breakdown by source: date-based, wiki-linked, keyword-matched]. Reading all notes now..."

## Phase 3: Read & Organize

Read all gathered notes. After reading each note, apply **sensitive-pattern redaction** to the in-memory content (original vault files are never modified). Regex-replace these patterns with `[REDACTED]`:
- **SSNs**: `\d{3}-\d{2}-\d{4}`
- **Credit card numbers**: `\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}`
- **Account/routing numbers**: 8-12 digit sequences preceded by "account", "routing", or "acct" (case-insensitive)
- **Phone numbers**: Common formats — `(###) ###-####`, `###-###-####`, `+1##########`, etc.
- **Email addresses**: `\S+@\S+\.\S+`
- **IP addresses**: `\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}`

Organize the redacted content into groups:
1. **By time slice**: Today, Yesterday, 1 Week, 1 Month, 1 Year
2. **Keyword-matched**: Note which keywords connected them and each note's date (from filename or frontmatter).

### Step 3a: Identify temporal clusters

Look at the dates of keyword-matched notes. If 2+ keyword-matched notes cluster around a date that falls outside the 5 fixed time slices (e.g., several notes from ~2 months ago), mark that cluster as a **dynamic time slice**.

Dynamic slices represent a **period of activity**, not a single date. Use the cluster's median date for x-axis positioning, but label it as an approximate period (e.g., "~Aug 2025" rather than "2025-08-15"). When scoring themes for a dynamic slice in Phase 4b, score based on the *entire cluster of notes* in that period — this measures intensity of engagement across the window, not a point-in-time snapshot.

Present a brief inventory table to the user showing all notes grouped by time slice (including any dynamic slices), with a 1-line summary of each note's topic.

## Phase 4: Analyze

Produce a structured analysis with these 5 sections:

### Snapshots
For each time slice that has notes, summarize: What were you thinking about? What projects, concerns, ideas were active?

### Threads Over Time
What topics appear across multiple time slices? How have they evolved? What persisted, what faded, what transformed?

### Causal Chains
Where do earlier notes contain seeds of later developments? Decisions that led to outcomes? Ideas that materialized? Identify specific cause-and-effect links across time.

### Contrasts
What's surprisingly different between time slices? Shifts in focus, tone, priorities, or worldview? What did you care about then that you've forgotten? What do you care about now that didn't exist then?

### Combinatorial Insights
The most valuable section. Cross-pollinate across time slices:
- Old idea + current project = opportunity?
- Past concern + recent learning = resolution?
- Abandoned thread + new context = worth revisiting?
- Pattern across time slices the user might not have noticed?

Be specific and cite the actual notes. Don't be generic — the value is in concrete, actionable connections.

### Kairos Moments
The Greek concept of *kairos* — the opportune moment, as distinct from *chronos* (sequential time). Scan the preceding analysis for three types of signal that something is ripe for action:

- **Convergence**: Multiple independent threads (from different time slices or unrelated notes) pointing at the same idea or action. When 3+ unconnected sources suggest the same thing, the moment may be ripe.
- **Readiness**: Accumulated preconditions — skills learned, resources gathered, blockers removed, context shifted. Something that wasn't feasible before may be feasible now.
- **Recurrence**: Ideas or intentions that keep resurfacing across time slices but haven't been acted on. The pattern of recurrence itself is a signal — this thing keeps wanting to happen.

For each kairos signal found, name it concretely: what's ripe, what evidence supports it, and what the first move would be. If no genuine signals exist, say so — don't fabricate urgency.

**Inline linking in Snapshots:** In the Snapshots section, put the wiki link in the time-slice heading itself — not in the body text. Use display text "link" and the note's filename (without `.md`) as the target. Examples:

- `### 1 Week Ago (2026-02-08) ([[2026-02-08|link]])`
- `### 1 Year Ago (2025-02-15) ([[2025-02-15|link]])`

For non-daily notes referenced within a snapshot (e.g. a book or topic note), put the link parenthetically after the title:
- `**Josh Waitzkin / The Art of Learning** ([[TheArtofLearning|link]]): "Disappointment is..."`

This makes the time-slice headings clickable in Obsidian so the reader can quickly open the original daily note.

End with a "Time Capsule Summary" — 3-5 bullet points of the most striking or actionable insights.

## Phase 4b: Theme Arcs & Trade-Offs

After completing the analysis, produce two visual elements: **theme arcs** (1-3 SVG curves showing how themes evolved) and **trade-offs** (1-3 tensions or competing priorities visible across the notes). Lead with whichever is most notable.

### Step 1: Identify themes and trade-offs

**Themes (1-3):** Pick the most dynamic themes from your Phase 4 analysis — themes with the biggest range of intensity across time slices. A theme that was absent in some slices and dominant in others is ideal. Only include themes that are genuinely dynamic; don't pad to 3 if only 1-2 have real movement. The themes should be distinct from each other.

**Trade-offs (1-3):** Identify tensions or competing priorities that the notes reveal. A trade-off is two things the author values that pull against each other — not a problem to solve, but a tension to navigate. Examples:
- Building features vs. shipping now
- Authenticity vs. marketing effectiveness
- Deep focus vs. public visibility
- Ambition vs. infrastructure maintenance

For each trade-off, write:
- **The tension**: Side A vs. Side B (short labels, 2-4 words each)
- **Evidence**: Which notes show each side pulling? Be specific.
- **Current lean**: Which side is winning right now, and is that intentional?

Only include trade-offs with real evidence in the notes. Don't invent tensions that aren't there.

### Step 2: Decide the lead

Compare the most notable trade-off against the combined theme arc. **Lead with whichever delivers more insight at a glance.** If the trade-offs reveal a tension the author may not have noticed, lead with trade-offs. If the theme arcs tell a more striking visual story, lead with arcs. Use your judgment.

### Step 3: Format trade-offs

Each trade-off is a short block:

```markdown
**Trade-off: [Side A] vs. [Side B]**
[2-3 sentences of evidence from the notes, citing specific notes. End with the current lean.]
```

If there are 2-3 trade-offs, present them sequentially. If trade-offs lead, they go above the SVG. If arcs lead, trade-offs go below the collapsed callouts.

### Step 4: Score each time slice for each theme

For each theme (1-3), assign an intensity score from 0.0 to 1.0 at each time slice — including any dynamic time slices identified in Phase 3 Step 3a:
- **0.0** = completely absent, not mentioned at all
- **0.3** = faintly present, a passing mention
- **0.5** = moderately present, discussed but not central
- **0.7** = significant focus, a major topic
- **1.0** = dominant, the primary concern

If a time slice had no notes found, score it 0.0 and label it "no data".

For each point on each theme, write a short label (2-4 words) describing what was happening with that theme at that moment.

### Step 5: Generate the combined overview SVG

Build one SVG with all theme curves overlaid on shared axes. If only 1 theme, use a single-curve SVG (no legend needed). If 2-3, use the multi-curve format.

**Layout:**
- `viewBox="0 0 800 400"`
- Margins: left 80, right 40, top 80, bottom 80
- Chart area: x from 80 to 760, y from 80 to 300 (height 220)
- Y maps: 1.0 → y=80 (top), 0.0 → y=300 (bottom)

**X-axis positions**: Place all time slices (the 5 fixed slices plus any dynamic slices from Phase 3 Step 3a) on a proportional timeline. The x-axis spans from x=80 (oldest date) to x=760 (today). Position each slice proportionally by its actual date:

```
x = 80 + (days_from_oldest / total_day_span) × 680
```

If no dynamic slices exist, the 5 fixed slices use these default positions:
| Slice | x position |
|-------|-----------|
| 1 Year Ago | 80 |
| 1 Month Ago | 250 |
| 1 Week Ago | 420 |
| Yesterday | 590 |
| Today | 760 |

If dynamic slices exist, recalculate all positions proportionally. Ensure x-axis labels don't overlap — if two slices are close together, stagger their labels vertically.

**Y coordinate formula:** `y = 300 - (score × 220)`

**Three theme colors:**
| Theme | Curve color | Point color | Label |
|-------|-----------|------------|-------|
| Theme 1 (most dynamic) | `#6366f1` (indigo) | `#6366f1` | Indigo |
| Theme 2 | `#e11d48` (rose) | `#e11d48` | Rose |
| Theme 3 | `#f59e0b` (amber) | `#f59e0b` | Amber |

**Catmull-Rom smooth curve** between points (convert to cubic Bezier):
For each segment i → i+1:
```
cp1x = p1.x + (p2.x - p0.x) / 6
cp1y = p1.y + (p2.y - p0.y) / 6
cp2x = p2.x - (p3.x - p1.x) / 6
cp2y = p2.y - (p3.y - p1.y) / 6
```
(Clamp p0 and p3 at array boundaries.)

**Style elements:**
- **Title**: "Theme Arcs" centered at top. Font 16px bold, fill `#818cf8`
- **Legend**: Below the title, centered. Three items: colored circle + theme name, spaced apart. Font 12px, fill matches each theme color. Use `<circle>` and `<text>` elements at roughly y=55.
- **Axes**: Light gray lines (`#d1d5db`), stroke-width 1
- **Y-axis labels**: "1.0" at top, "0.5" at middle, "0.0" at bottom. Font 12px, fill `#6b7280`. Italic sub-labels: "Dominant" near top, "Absent" near bottom (font 10px, fill `#9ca3af`)
- **X-axis labels**: Time slice names below x-axis (font 11px, fill `#6b7280`). Date in parentheses below (font 10px, fill `#9ca3af`)
- **Midline**: Dashed at y=190 (0.5), stroke `#9ca3af`, dasharray `6 4`
- **Gradient fill**: Only Theme 1 (indigo) gets area fill — gradient from `rgba(99, 102, 241, 0.10)` to `rgba(99, 102, 241, 0.02)`. Themes 2 and 3 have no fill (just curves).
- **Curves**: Each theme's color, stroke-width 3 for Theme 1, stroke-width 2 for Themes 2 and 3. Round linecap/linejoin.
- **Points**: Circles at each data point. Theme 1: r=5, Themes 2-3: r=4. All white stroke, stroke-width 2.
- **No point labels** on the combined view (too cluttered with 15 labels). Labels go on the individual views.
- **Font**: `-apple-system, BlinkMacSystemFont, sans-serif`

Wrap the SVG in a `<div>`.

### Step 6: Generate individual detail SVGs in collapsed callouts

Below the combined SVG, add 1-3 collapsed Obsidian callouts (one per theme). Each contains a single-theme SVG with point labels.

Format:
```markdown
> [!abstract]- Theme 1: [THEME NAME]
> <div><svg>...</svg></div>

> [!abstract]- Theme 2: [THEME NAME]
> <div><svg>...</svg></div>
```

Each individual SVG uses the same coordinate system as the original single-theme spec, with the same proportional x-axis positioning (including dynamic slices):
- `viewBox="0 0 800 360"`
- Chart area: x 80-760, y 60-280
- Y formula: `y = 280 - (score × 220)`
- The theme's own color for curve, points, and labels
- Gradient area fill in the theme's color (10-15% opacity top, 2-3% bottom)
- **Point labels included**: short descriptor above/below each point
- Title: "Theme: [NAME]" in the theme's color

**Gradient IDs must be unique** across all 4 SVGs (e.g., `themeArcGrad1`, `themeArcGrad2`, `themeArcGrad3`, `themeArcGradOverview`).

Do the coordinate math carefully. Double-check that each point's y value matches the formula.

## Phase 4c: Thinker Recommendations

Recommend **2 people** (thinkers, researchers, practitioners, historical figures) whose work could help deepen or extend the themes surfaced in this reflection. These should be people the user might not already know well — surprising, cross-disciplinary connections are ideal.

### Step 1: Check recent reflections for duplicates

Read the last 3 reflection files in `$VAULT_PATH/Reflections/` (by filename date, descending). Extract any previously recommended people from their "Thinkers to Explore" sections. Do NOT recommend anyone who appears in those.

### Step 2: Choose 2 people

For each person, consider:
- How does their work connect to the themes in this reflection?
- Would the user gain a useful new lens, framework, or counterpoint?
- Prefer cross-disciplinary picks (e.g., a physicist for a business theme, a philosopher for a technical theme)

### Step 3: Format

Place this immediately below the Theme Arc SVG, before the Snapshots section:

```markdown
**Thinkers to Explore:**
[Person Name](https://en.wikipedia.org/wiki/Wiki_Page_Name) — 1-sentence reason this person's work connects to the themes above
[Person Name](https://en.wikipedia.org/wiki/Wiki_Page_Name) — 1-sentence reason
```

Use real, correct Wikipedia URLs. Verify the person exists and the URL slug is plausible (standard format: underscores for spaces, proper capitalization).

## Phase 5: Write to Obsidian

**IMPORTANT: Do NOT ask for permission before writing. Write the reflection file and link it from the daily note immediately — no confirmation prompts. The user expects to initiate this skill and come back to a completed file.**

**First-run setup:** If this is the first time the user has run `/reflect`, check whether their `~/.claude/settings.local.json` already contains `Write` and `Edit` permissions for `$VAULT_PATH`. If it does NOT, ask:

> "To let /reflect write files without prompting you each time, I can add write permissions for your Obsidian vault to your Claude Code settings. Want me to do that?"

If they say yes, add these two entries to the `permissions.allow` array in `~/.claude/settings.local.json` (creating the file if needed):
- `Write(file_path:$VAULT_PATH/*)`
- `Edit(file_path:$VAULT_PATH/*)`

Then continue with the reflection. Only ask this once — if the permissions are already present, skip silently.

### Create the reflection note

Write the full analysis to a new note in the vault at:
`$VAULT_PATH/Reflections/YYYY-MM-DD Reflection.md`

(where YYYY-MM-DD is the anchor date)

Create the `Reflections/` folder if it doesn't exist. The note should contain:
- Frontmatter with `created: YYYY-MM-DD` and `tags: [reflection]`
- The combined Theme Arc SVG and 3 collapsed individual arcs from Phase 4b (immediately after frontmatter, before any text)
- The Thinker Recommendations from Phase 4c (immediately below the SVG)
- A "Related Notes" collapsible callout immediately below the Thinker Recommendations, listing keyword-matched notes (from Phase 2) grouped by date. For each note, include its wiki link and a brief note on which keywords connected it. This section makes the keyword-expansion trail navigable — the reader can follow threads the reflection surfaced. Use: `> [!note]- Related Notes (keyword-matched)`
- The full analysis from Phase 4
- A "Sources" section at the bottom listing all notes that were read, as wiki links

### Link from today's daily note

Find today's daily note at `$VAULT_PATH/YYYY-MM-DD.md` (using the actual current date from `date +%Y-%m-%d`, NOT the anchor date if an argument was provided).

- If the daily note **exists and has content**, append `[[YYYY-MM-DD Reflection]]` on a new line at the end
- If the daily note **exists but is empty**, write `[[YYYY-MM-DD Reflection]]` as its content
- If the daily note **does not exist**, create it with `[[YYYY-MM-DD Reflection]]` as its content

Tell the user: "Reflection written to [[YYYY-MM-DD Reflection]] and linked from today's daily note."
