# raw/

Immutable source documents. The wiki skill reads from here but never modifies existing files.

Drop source material into the appropriate subdirectory. The `/wiki ingest` command processes files from here into the curated wiki.

## Subdirectories

| Directory | Source | Authority | What goes here |
|---|---|---|---|
| `research/` | `/wiki research` | Trusted provenance | Web research fetched and pre-analyzed for this project. Claims are cited and qualified, not wiki-authoritative. |
| `external/` | `/wiki convert` | Trusted | Operator-provided docs staged verbatim during wiki conversion. Originals preserved as-is. |
| `gsd/` | GSD lifecycle | Trusted | Knowledge packets from GSD phase completion, milestone close, and deferred decisions. Auto-ingested. |
| `advisory/` | External dispatcher | Advisory | Cross-project knowledge another system thinks this project should know about. Wiki evaluates and decides: promote, adapt, flag, or skip. |
| `conversations/` | Manual | Trusted | Exported Claude Code transcripts or meeting notes worth preserving. |

## How to ingest

Run `/wiki ingest` to process any unprocessed files. The skill reads `WIKI-CLAUDE.md` to determine where content belongs in the curated wiki.

Each subdirectory's files are treated according to their source authority level. `raw/advisory/` drops are advisory and will be evaluated against existing wiki state before ingestion.

## File-format tips

- **Markdown / text:** ideal — Claude reads directly.
- **PDF:** Claude can read up to 20 pages per request. Big PDFs may need page-range hints.
- **Images:** Claude can view them. For images with text, say what you want extracted.
- **Web articles:** save as `.md` and drop the markdown file here.
- **Audio / video transcripts:** transcribe first, then drop the transcript.

## Out of scope for this folder

- Anything that belongs in the live project (production code, deployed assets).
- LLM-generated content — those go in `wiki/`.
- Operating instructions (WIKI-CLAUDE.md) — those are project-level, not sources.
