# College Document AI Assistant

A genuine Retrieval-Augmented Generation (RAG) app: upload college PDFs, ask questions in
plain English, get an answer grounded in the document text — with the exact source
document and page number cited.

## What it does

1. You upload one or more PDFs (regulations, syllabus, scholarship notices, handbooks).
2. The app extracts real text from every page and remembers which page each sentence came from.
3. When you ask a question, it finds the most relevant chunks of text using vector
   similarity search (not keyword matching).
4. Only those chunks are sent to Claude, which is instructed to answer **only** from that
   context — if the answer isn't there, it says so instead of guessing.
5. The answer is shown with its sources: `📄 filename.pdf — 📑 Page 24`.

## What "RAG" means (for your presentation)

RAG = Retrieval-Augmented Generation. Instead of asking an LLM to answer from its own
training data (which risks hallucination and can't know your specific document), you:
- **Retrieve** the most relevant pieces of a real document using vector search
- **Augment** the LLM's prompt with that retrieved text
- **Generate** an answer that's grounded in and traceable to the source

This is the same pattern used in production AI search tools — just built at
student-project scale.

## Architecture

```
PDF upload
  → pdf.js extracts text page-by-page (page numbers preserved)
  → text is cleaned and split into ~700-character overlapping chunks
  → each chunk tagged with { document, page, text, chunkId }
  → TF-IDF vectors built over the whole chunk vocabulary   (the "embeddings")
  → question is vectorized the same way
  → cosine similarity ranks all chunks against the question (the "vector search")
  → top 3-4 chunks passed as context to Claude
  → Claude answers strictly from that context
  → UI displays the answer + the document/page of every chunk used
```

## Technologies (100% free)

- **Frontend**: single-file HTML/CSS/JS (no build step, no server needed)
- **PDF extraction**: [pdf.js](https://mozilla.github.io/pdf.js/) (Mozilla, open source, loaded from a free CDN)
- **Embeddings / vector search**: a hand-rolled **TF-IDF + cosine similarity** engine
  written in plain JavaScript — a real vector-space retrieval model, not exact keyword
  matching. Two questions with zero overlapping words but similar meaning will still
  rank differently based on term weighting across the whole corpus; it's a lightweight
  browser-only stand-in for a sentence-transformer embedding model, so the whole app can
  run with no Python backend and no paid embedding API.
- **Generation**: Claude, called through the Artifact's built-in API bridge — no API key
  needed from you, and nothing to pay for.
- **Fallback**: if the AI generation call is ever unavailable, the app falls back to a
  local **extractive** answer (it picks the most relevant retrieved sentence directly from
  the source text), so the app still works and still never invents facts.

## How PDF processing works

`pdf.js` opens the PDF in the browser and calls `getTextContent()` on every page, which
returns the actual embedded text (not an image render). That text is joined per page, so
every chunk downstream keeps a `page` number attached. Scanned/image-only PDFs without an
embedded text layer are detected and reported as an error rather than silently faked.

## How chunking + embeddings work

Each page's cleaned text is split into ~700-character windows with a 120-character overlap
(so an answer that spans a chunk boundary isn't lost). A TF-IDF vocabulary is built across
every chunk in the current session: term frequency (how often a word appears in a chunk) ×
inverse document frequency (how rare that word is across all chunks) gives each chunk a
vector. Common/stop words are filtered out first.

## How vector search works

The question is turned into the same kind of TF-IDF vector, then compared against every
chunk vector with cosine similarity (the angle between the two vectors — 1.0 means
identical direction, 0 means unrelated). The top 3-4 chunks above a small similarity
threshold are kept; if nothing clears the threshold, the app tells you it couldn't find
the answer instead of guessing.

## How source citations work

Every chunk carries its origin (`{ document, page }`) from the moment it's created during
chunking, all the way through vectorization, retrieval, and the final prompt. The UI reads
those same fields back off the retrieved chunks to render the "Sources Used" cards — the
page numbers are never invented, they're the same metadata used to build the prompt.

## How to run it

It's a single HTML file — no install, no server, no build step.
1. Open `college-document-ai.html` in any modern browser (Chrome/Edge/Firefox).
2. That's it. It runs entirely client-side.

## How to test it

1. Click **Load Demo Document** (no PDF needed) to seed a sample regulations document.
2. Type or click the example question *"What is the minimum attendance required?"*
3. Confirm the answer appears with a **Sources Used** card showing `VTU_Regulations.pdf (Demo) — Page 24`.
4. Try *"What are the scholarship eligibility criteria?"* and confirm it cites **Page 12** instead.
5. Upload a real PDF (e.g. your own college handbook) via **Upload PDF**, wait for
   "✓ Ready", then ask a question about its actual content — verify the cited page number
   matches the real PDF page.

## Free-tier limitations

- TF-IDF is a real, legitimate vector retrieval technique, but it is a lexical/statistical
  model, not a deep-learning sentence embedding — it works very well for keyword-rich
  regulation/policy text like this project targets, but won't catch fully paraphrased
  questions with zero shared vocabulary as well as a model like `all-MiniLM-L6-v2` would.
- Scanned/image PDFs with no embedded text layer aren't supported (no OCR is bundled, to
  keep the app dependency-free and instant to load).
- All data lives in memory for the current browser tab only — nothing is saved to a
  server or database, so refreshing the page clears uploaded documents (by design: no
  backend, no persistence infrastructure, matching the "keep it simple" brief).
