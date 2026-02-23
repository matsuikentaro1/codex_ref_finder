# codex_ref_finder

Search academic references (primarily via PubMed) using [OpenAI Codex CLI](https://github.com/openai/codex) and export to CSV.

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) custom slash command that delegates literature searching to OpenAI Codex CLI. It searches PubMed and other sources, then saves structured results (PMID, authors, title, journal, DOI, abstract, and annotated findings) into a CSV file.

## Features

- Search PubMed and other academic sources via Codex CLI
- Export results to CSV with standardized columns
- Append new findings to existing CSV without duplicates (DOI-based dedup)
- Annotate each reference with up to 5 context-specific insights (`whats_interesting1`–`5`)

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [OpenAI Codex CLI](https://github.com/openai/codex) (`npm install -g @openai/codex`)
- Python 3 (used internally by Codex for PubMed API calls)

## Installation

Copy the command file into your project's `.claude/commands/` directory:

```bash
# Clone this repository
git clone https://github.com/matsuikentaro1/codex_ref_finder.git

# Copy the command file to your project
cp codex_ref_finder/.claude/commands/codex_ref_finder.md /path/to/your/project/.claude/commands/
```

Or manually download `.claude/commands/codex_ref_finder.md` and place it in your project's `.claude/commands/` directory.

## Usage

In Claude Code, use the slash command:

```
/codex_ref_finder
```

## CSV columns

| Column | Description |
|---|---|
| PubMed_ID | PMID |
| Author | Author names (semicolon-separated) |
| Year | Publication year |
| Title | Article title |
| Journal | Journal name |
| Volume | Volume number |
| Issue | Issue number |
| Pages | Page range |
| doi | DOI |
| abstract | Abstract |
| whats_interesting1–5 | Context-specific findings and annotations |

## License

MIT
