# reflect — Time-Travel Reflection for Obsidian

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) command that analyzes your Obsidian daily notes across 5 time slices — today, yesterday, 1 week ago, 1 month ago, and 1 year ago — and produces a structured reflection note with theme arc SVGs, trade-off analysis, thinker recommendations, and cross-temporal insights.

## What it produces

Each run creates a reflection note in your vault containing:

- **Theme Arc SVGs** — smooth Catmull-Rom curves showing how 1-3 themes evolved across the time slices, with a combined overview and individual detail views in collapsible callouts
- **Trade-off analysis** — tensions and competing priorities visible in your notes, with evidence and current lean
- **Thinker recommendations** — 2 cross-disciplinary people whose work connects to the themes (deduplicated against your last 3 reflections)
- **5-section analysis** — Snapshots, Threads Over Time, Causal Chains, Contrasts, and Combinatorial Insights
- **Time Capsule Summary** — 3-5 actionable bullet points
- **Sources** — wiki links to every note that was read

The reflection note is automatically linked from your daily note.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- An Obsidian vault with daily notes (named `YYYY-MM-DD.md` or legacy `Month Dayth, YYYY.md` format)

## Installation

1. Copy `reflect.md` to your Claude Code commands directory:

   ```bash
   # Global (available in all projects)
   cp reflect.md ~/.claude/commands/reflect.md

   # Or project-level (available only in one project)
   cp reflect.md .claude/commands/reflect.md
   ```

2. Open `~/.claude/commands/reflect.md` (or `.claude/commands/reflect.md`) and edit the **Configuration** section at the top — set your vault path and optionally your reading tag.

## Usage

```
/reflect              # Reflect on today
/reflect 2025-06-15   # Reflect on a specific anchor date
```

The command runs through 5 phases automatically and writes the result to `{VAULT_PATH}/Reflections/YYYY-MM-DD Reflection.md`.

## Configuration

Open `reflect.md` and set the two values in the Configuration section:

### VAULT_PATH (required)

The absolute path to the root of your Obsidian vault. Replace `/path/to/your/obsidian/vault` with your actual path.

```
/Users/you/Documents/My Vault
```

### READING_TAG (optional)

If your vault uses a tag or wiki link to mark book quotes and reading wisdom (e.g., `[[reading]]` or `#reading`), set it here. The command will search for notes containing this tag combined with keywords from your time slices to surface relevant reading wisdom.

Leave empty to skip this step entirely.

## How it works

1. **Gather** — Finds notes for each of the 5 time slices by filename, frontmatter `created:` date, and one level of wiki link following
2. **Expand** — Extracts keywords (tags, wiki links, proper nouns, distinctive terms) and searches the vault for topically related notes
3. **Organize** — Reads all gathered notes and groups them by time slice
4. **Analyze** — Produces the structured analysis, theme arc SVGs, trade-offs, and thinker recommendations
5. **Write** — Creates the reflection note and links it from today's daily note

## Customization tips

- **Time slices**: Edit the list in Phase 1 to change which dates are analyzed (e.g., add "6 months ago" or remove "yesterday")
- **SVG style**: Modify the colors, dimensions, or viewBox in Phase 4b Step 5. The three theme colors are defined in the color table.
- **Analysis sections**: Add or remove sections in Phase 4. The five default sections (Snapshots, Threads Over Time, Causal Chains, Contrasts, Combinatorial Insights) can be adjusted to your preference.
- **Thinker count**: Change "2 people" in Phase 4c to recommend more or fewer thinkers
- **Output path**: Change the `Reflections/` subfolder in Phase 5 if you prefer a different location in your vault

## License

MIT — see [LICENSE](LICENSE).
