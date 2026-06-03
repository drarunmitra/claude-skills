---
name: zotero-cite
description: Use when inserting citations in Quarto/markdown documents, searching Zotero for references, verifying citation authenticity, converting manual citations to citekeys, adding papers to Zotero by DOI/search/PDF, or creating Obsidian literature notes from Zotero entries. Triggers on requests involving citations, references, bibliography, citekeys, Zotero, or BibTeX in academic writing contexts.
---

# Zotero Cite

Insert verified citations into Quarto/markdown documents using Zotero and Better BibTeX as the source of truth.

## When to Use

- User wants to cite a paper in a `.qmd` or `.md` file
- User needs a citation for a claim and wants suggestions from their Zotero library
- User wants to convert manually typed citations like `(Author, Year)` to `@citekey` format
- User wants to add a paper to Zotero (by DOI, search, or PDF)
- User asks to verify existing citations (DOI resolution, retraction check, claim accuracy)
- User wants an Obsidian literature note created for a cited paper

**Not for:** Formatting bibliography styles (use CSL files), editing Zotero library metadata directly (use Zotero GUI), or non-academic citation contexts.

## Prerequisites

**Required:** Zotero running with Better BibTeX installed.

```bash
# Verify BBT JSON-RPC is active
curl -s -X POST http://localhost:23119/better-bibtex/json-rpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"api.ready","id":1}'
# Should return: {"jsonrpc":"2.0","result":{"zotero":"...","betterbibtex":"..."},"id":1}
```

**Optional:** Zotero MCP server (provides native Claude tools for richer search).
```bash
# Check if installed
which zotero-mcp
```

## Core API: BBT JSON-RPC

All commands go to `http://localhost:23119/better-bibtex/json-rpc` via POST with `Content-Type: application/json`.

| Method | Parameters | Returns |
|--------|-----------|---------|
| `item.search` | `terms` (string), `library?` | Search results |
| `item.citationkey` | `item_keys` (array) | `{ key: citekey }` mapping |
| `item.export` | `citekeys` (array), `translator` (string) | BibTeX string |
| `item.bibliography` | `citekeys` (array), `format?` | Formatted bibliography |
| `item.notes` | `citekeys` (array) | Notes attached to items |
| `item.attachments` | `citekey` (string) | Attachment data |

### Search Zotero

```bash
curl -s -X POST http://localhost:23119/better-bibtex/json-rpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"item.search","params":["braa sahay health information"],"id":1}'
```

### Get Citekey

```bash
curl -s -X POST http://localhost:23119/better-bibtex/json-rpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"item.citationkey","params":[["ABC123"]],"id":1}'
```

### Export BibTeX Entry

```bash
curl -s -X POST http://localhost:23119/better-bibtex/json-rpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"item.export","params":[["citekey2024"],"Better BibTeX"],"id":1}'
```

### CAYW (Cite As You Write) Endpoint

```bash
# Get Pandoc-formatted citation from Zotero picker
curl -s "http://localhost:23119/better-bibtex/cayw?format=pandoc&brackets=1"
```

### Pull Export (whole collection as .bib)

```bash
# Export a collection by path
curl -s "http://localhost:23119/better-bibtex/collection?PhD.bib"
```

## Five Modes of Operation

### Mode 1: CITE (Known Paper)

User knows which paper to cite. They provide author/title/DOI.

**Workflow:**
1. Search Zotero: `item.search` with author/title terms
2. If not found, check if user wants to ADD (go to Mode 4)
3. Get citekey: `item.citationkey` with the Zotero item key
4. Export BibTeX: `item.export` with the citekey
5. Check if citekey already in project `.bib` file; if not, append the entry
6. VERIFY (run Mode 5 verification pipeline)
7. Insert `@citekey` at the specified location in the `.qmd` file
8. Optionally create Obsidian literature note

**Example interaction:**
```
User: "Cite the Braa and Sahay 2012 book in the introduction paragraph about data use"

1. Search: item.search("braa sahay 2012")
2. Found: key=ABC123 → citekey=braa2012integrated
3. Export BibTeX, append to thesisrefs.bib if missing
4. Verify DOI, check retraction status
5. Insert [@braa2012integrated] at specified location
```

### Mode 2: SUGGEST (Need Citation for a Claim)

User writes a claim and needs supporting references.

**Workflow:**
1. Extract key terms from the claim
2. Search Zotero library with `item.search` using multiple term combinations
3. If Zotero MCP is available, use `zotero_search_items` for richer results
4. For each candidate, get abstract/notes to assess relevance
5. If insufficient results in Zotero, search online:
   - CrossRef API: `https://api.crossref.org/works?query=<terms>&rows=5`
   - PubMed API: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=<terms>`
   - WebSearch as fallback
6. Present candidates with relevance reasoning to user
7. User selects; proceed as Mode 1

**CrossRef search example:**
```bash
curl -s "https://api.crossref.org/works?query=health+information+systems+tribal&rows=5" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); [print(f'{i[\"DOI\"]} | {i[\"title\"][0][:80]} | {i.get(\"published-print\",{}).get(\"date-parts\",[[\"?\"]])[0][0]}') for i in d['message']['items']]"
```

### Mode 3: AUDIT (Bulk Conversion)

User has a `.qmd` file with manually typed citations like `(Author, Year)` or `(Author et al., Year)`.

**Workflow:**
1. Scan the `.qmd` file for patterns matching manual citations:
   - `(Author, Year)` e.g., `(Braa, 2012)`
   - `(Author & Author, Year)` e.g., `(Braa & Sahay, 2012)`
   - `(Author et al., Year)` e.g., `(Mishra et al., 2022)`
   - `(Author1, Year; Author2, Year)` compound citations
2. For each match, search Zotero: `item.search("author year")`
3. Present matches for user confirmation
4. Replace with `@citekey` or `[@citekey]` format
5. Export all matched BibTeX entries and update `.bib` file
6. Report any unmatched citations

**Regex patterns for detection:**
```
\(([A-Z][a-z]+(?:\s(?:&|and)\s[A-Z][a-z]+)?(?:\set\sal\.)?),?\s*(\d{4}[a-z]?)\)
```

### Mode 4: ADD (Paper Not in Zotero)

Paper needs to be added to Zotero before citing.

**By DOI:**
```bash
# Add via Zotero Web API
curl -s -X POST "https://api.zotero.org/users/LIBRARY_ID/items" \
  -H "Zotero-API-Key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '[{"itemType":"journalArticle","DOI":"10.1234/example"}]'
```

Or if Zotero MCP is available: use `zotero_add_by_doi` tool.

**By search query:**
1. Search CrossRef/PubMed for the paper
2. Present candidates to user for confirmation
3. Get DOI from confirmed result
4. Add to Zotero via DOI method above
5. Wait for BBT to generate citekey (poll `item.search` briefly)
6. Proceed as Mode 1

**By PDF on disk:**
1. Identify PDF path
2. Extract metadata: check for DOI in PDF text, or use filename
3. If DOI found, add via DOI method
4. If no DOI, use Zotero's file import:
   ```bash
   open -a Zotero "<path_to_pdf>"
   ```
   Then search for the newly added item after a brief pause

**By online search (autonomous):**
1. User describes the paper vaguely ("that Smith paper about data assemblages")
2. Search CrossRef and PubMed with extracted terms
3. Also use WebSearch for broader academic search
4. Present top candidates with titles, authors, years, DOIs
5. User confirms; proceed with DOI-based addition

### Mode 5: VERIFY (Citation Integrity)

Verification pipeline for each citation. Run automatically in Mode 1, or standalone on request.

**Level 1 - Existence check:**
```bash
# Confirm citekey exists in .bib file
grep -c "citekey" thesisrefs.bib

# Confirm citekey exists in Zotero
curl -s -X POST http://localhost:23119/better-bibtex/json-rpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"item.export","params":[["citekey"],"Better BibTeX"],"id":1}'
```

**Level 2 - DOI resolution:**
```bash
# Check DOI resolves (HEAD request to avoid downloading full page)
curl -sI -o /dev/null -w "%{http_code}" "https://doi.org/10.1234/example"
# 302 = valid redirect, 404 = broken DOI
```

**Level 3 - Retraction check:**
```bash
# Query OpenAlex for retraction status
curl -s "https://api.openalex.org/works/doi:10.1234/example" \
  | python3 -c "import sys,json; w=json.load(sys.stdin); print('RETRACTED' if w.get('is_retracted') else 'OK')"
```

**Level 4 - Claim cross-check:**
1. Get abstract from Zotero: `item.export` or `item.notes`
2. Read the surrounding text in the `.qmd` where the citation appears
3. Assess whether the claim in the text is supported by the abstract/title
4. Flag potential misattributions with specific reasoning

**Output format:**
```
@braa2012integrated
  [OK] Exists in thesisrefs.bib
  [OK] Exists in Zotero
  [OK] DOI resolves (302)
  [OK] Not retracted (OpenAlex)
  [OK] Claim supported: text says "data rich information poor" - matches book title/abstract

@mishra2022mapping
  [OK] Exists in thesisrefs.bib
  [OK] Exists in Zotero
  [OK] DOI 10.4103/ijph.ijph_1672_21 resolves
  [OK] Not retracted
  [WARN] Claim says "69 national data sources" - verify exact count matches paper
```

## BibTeX File Management

### Adding entries to project .bib

When inserting a citation, always:
1. Export the BibTeX entry from BBT: `item.export` with translator `"Better BibTeX"`
2. Check if citekey already exists in the project `.bib` file
3. If not present, append the entry to the `.bib` file
4. Never duplicate entries

### Regenerating .bib from all citations

```bash
# Extract all citekeys from .qmd files
grep -ohE '@[a-zA-Z][a-zA-Z0-9_:-]+' *.qmd | sort -u | sed 's/^@//'

# Export all as BibTeX from BBT
# Use item.export with the full list of citekeys
```

### Multiple .bib files

Some projects use multiple `.bib` files (e.g., `thesisrefs.bib` + `grateful-refs.bib`). Check `_quarto.yml` or the document YAML for the `bibliography:` key to determine which file to update. Add thesis/article references to the primary `.bib`, not auto-generated ones.

## Obsidian Literature Note Creation

When a citation is inserted, optionally create a literature note.

**Location:** `~/Obsidian/PhD/Literature/{citekey}.md`

**Template:**
```markdown
---
title: "{title}"
authors: "{authors}"
year: {year}
citekey: "{citekey}"
doi: "{doi}"
tags: [literature-note]
---

## {title}

**Authors**: {authors}
**Year**: {year}
**Journal**: {journal}
**DOI**: [{doi}](https://doi.org/{doi})
**Zotero**: [Open in Zotero](zotero://select/library/items/{zotero_key})

### Abstract
{abstract}

### Key Findings


### Relevance to Thesis


### Notes

```

**Creating the note:**
```bash
# Use obsidian-cli if available and Obsidian is running
obsidian vault=PhD create name="Literature/{citekey}" content="..." --overwrite

# Or write directly to disk
# Write to ~/Obsidian/PhD/Literature/{citekey}.md
```

**Linking from fleeting notes:**
When citing in Obsidian notes (not Quarto), use wikilink format:
```markdown
This connects to the data use literature [[@braa2012integrated]].
```

## Zotero Connection Details

**BBT JSON-RPC:** `http://localhost:23119/better-bibtex/json-rpc` (POST, JSON)
**BBT CAYW:** `http://localhost:23119/better-bibtex/cayw` (GET)
**BBT Pull Export:** `http://localhost:23119/better-bibtex/collection?<path>.<format>`
**Zotero Web API:** `https://api.zotero.org/users/{library_id}/items`
**Zotero MCP:** Available as `zotero_*` tools if the MCP server is running

## User's Project Configuration

When working in the PhD thesis project (`~/PhD_3.0`):
- **Primary .bib:** `thesisrefs.bib`
- **R packages .bib:** `grateful-refs.bib` (auto-generated, do not edit)
- **CSL:** `vancouver.csl` (numeric in-text citations)
- **Citation format in text:** `[@citekey]` (Pandoc syntax)
- **Multiple citations:** `[@key1; @key2; @key3]`
- **HARD RULE — Citation placement:** The `[@citekey]` MUST be placed at the END of the sentence, before the full stop. Never mid-sentence. Author names may appear inline in the narrative text, but the bracketed reference goes at the end. Example: `Sahay et al. argued that data availability does not equal data use [@sahay2017public].` NOT: `Sahay et al. [@sahay2017public] argued that...`

When working in paper subdirectories (e.g., `papers/data_sources_paper_1/src/`):
- Check for local `references.bib` or `bibliography.bib`
- Check the document's YAML `bibliography:` key

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Citing without checking Zotero first | Always search Zotero before adding new entries |
| Adding duplicate .bib entries | Check if citekey exists before appending |
| Wrong .bib file | Check document YAML for `bibliography:` key |
| Hardcoding citation text | Always use `@citekey` format, never manual `(Author, Year)` |
| Not verifying DOI | Run Level 2 verification for every new citation |
| Trusting citekey without checking content | Always verify the paper supports the claim |
| Creating literature notes without abstract | Fetch abstract from Zotero before creating note |
| Placing `[@citekey]` mid-sentence | HARD RULE: `[@citekey]` goes at end of sentence only. Author names inline, reference at the end. |
| Using narrative `@citekey` (no brackets) | In Vancouver style, `@citekey` renders as bare `(N)` without author name. Always use `Author [@citekey]` format with author name written explicitly in text. |
