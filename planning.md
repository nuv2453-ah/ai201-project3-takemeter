# Project 1 Planning: The Unofficial Guide
> Write this document before you write any pipeline code.
> Your spec and architecture diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Update the Retrieval Approach and Chunking Strategy sections if you change your approach during implementation.
> Update this file before starting any stretch features.
---
## Domain
This guide covers student-generated knowledge about Purdue CS professors — specifically reviews,
opinions, and course advice for professors teaching core CS courses like CS 180, CS 182, CS 251,
and CS 18200. This knowledge is valuable because it reflects real student experiences with grading
style, exam difficulty, lecture quality, and feedback that never appear in official course catalogs
or university websites. It is hard to find otherwise because it is scattered across Rate My
Professors pages, Reddit threads, and informal forums with no unified, searchable interface that
answers plain-language questions like "Does Borkowski curve?" or "Is Bergstrom's CS 180 harder
than other sections?"

---
## Documents
| # | Source | Description | URL or location |
|---|--------|-------------|-----------------|
| 1 | Rate My Professors | Student reviews for Sarah Sellke (CS 18200) | https://www.ratemyprofessors.com/professor/1734941 |
| 2 | Rate My Professors | Student reviews for Michael Borkowski (CS 180, CS 251) | https://www.ratemyprofessors.com/professor/3058603 |
| 3 | Rate My Professors | Student reviews for Tony Bergstrom (CS 180) | https://www.ratemyprofessors.com/professor/2523519 |
| 4 | Rate My Professors | Purdue CS department page — all professor ratings | https://www.ratemyprofessors.com/school/783 |
| 5 | Purdue CS Faculty Directory | Official faculty listing with research areas and contact info | https://www.cs.purdue.edu/people/faculty/index.html |
| 6 | Purdue CS Faculty Page | Tony Bergstrom official profile (courses taught, background) | https://www.cs.purdue.edu/people/faculty/bgstm.html |
| 7 | Purdue CS Faculty Page | Michael Borkowski official profile (courses taught, background) | https://www.cs.purdue.edu/people/faculty/mhborkow.html |
| 8 | Purdue CS PhD Visit Day FAQ | Student-authored FAQ with advice on advisors, professors, and courses | https://www.cs.purdue.edu/gsa/files/purdue_cs_phd_visit_day_faq.pdf |
| 9 | Coursicle | Michael Borkowski course history — which semesters, which courses | https://www.coursicle.com/purdue/professors/Michael+Borkowski/ |
| 10 | r/Purdue (Reddit) | Manually saved Reddit threads mentioning Bergstrom, Borkowski, Sellke — saved as `reddit_purdue_cs_advice.txt` | local file |

---
## Chunking Strategy
RMP reviews are short opinion snippets — usually 2–5 sentences per review. Reddit posts vary:
some are a single paragraph, others are multi-paragraph threads. The key fact in a review is
almost always contained within a single sentence or two ("exams are curved," "lectures are hard
to follow"), so chunks should be small enough to isolate individual claims, but large enough to
carry meaningful context.

**Chunk size:** 300 characters

**Overlap:** 50 characters

**Reasoning:** 300 characters captures roughly 2–4 sentences of review text, which is enough
for a single, self-contained student opinion. A larger chunk (e.g. 800+ characters) would merge
opinions about different topics (grading, attendance, exams) into one embedding, making it hard
to retrieve precisely. A smaller chunk (e.g. 100 characters) would cut sentences mid-thought and
produce fragments with no standalone meaning. 50-character overlap ensures that a claim split
across a chunk boundary is still represented in at least one complete chunk — important for
short reviews where a key fact might fall right at the break point.

---
## Retrieval Approach
**Embedding model:** `all-MiniLM-L6-v2` via `sentence-transformers` (runs locally, no API key,
no rate limits)

**Top-k:** 5

**Production tradeoff reflection:** For a real deployment, I would weigh several factors.
`all-MiniLM-L6-v2` is fast and free but has a 256-token context window, which limits it for
longer documents (not a problem here, but would be for a housing guide corpus). A model like
`text-embedding-3-small` from OpenAI offers better accuracy on domain-specific phrasing and
a larger context window, but costs per token and introduces API latency and a dependency on
external uptime. For a multilingual use case (e.g. international students reviewing professors),
a model like `paraphrase-multilingual-MiniLM-L12-v2` would be necessary. For this project,
local + free wins since latency and cost aren't constraints — but in production with real user
traffic, I would likely switch to a hosted model with better domain accuracy and monitor
retrieval quality with a held-out eval set.

---
## Evaluation Plan
| # | Question | Expected answer |
|---|----------|-----------------|
| 1 | Does Professor Borkowski curve exams in CS 251? | Yes — multiple reviews mention Borkowski curves exams and is student-friendly about grading |
| 2 | What do students say about Professor Sellke's lectures in CS 18200 — are they easy to follow? | Mixed to negative — several reviews say lectures are hard to follow and students need to supplement with outside resources |
| 3 | How hard is CS 180 with Professor Bergstrom compared to other sections? | Harder than average — Bergstrom is tagged "tough grader" with only 32% would-take-again; reviews mention lots of homework and tests |
| 4 | Does Professor Borkowski give useful feedback on assignments? | Yes — reviews specifically mention clear grading criteria and good feedback as top tags |
| 5 | Would students recommend taking CS 18200 with Professor Sellke, and what should they know going in? | Mixed — Sellke is described as nice and approachable but lectures don't explain concepts well; students should expect to learn material independently online |

---
## Anticipated Challenges

1. **RMP scraping may be blocked or return incomplete data.** Rate My Professors uses
   JavaScript rendering, which means a simple `requests` + `BeautifulSoup` scrape will return
   an empty page. I may need to copy review text manually into `.txt` files or find a community
   scraper. If reviews are manually copied, formatting will be inconsistent (no guaranteed
   newlines between reviews), which could cause the chunker to merge multiple reviews into one
   chunk and hurt retrieval precision.

2. **Chunks may split key claims across boundaries.** A review like "Professor Borkowski's
   exams are heavily curved — multiple students confirmed this is consistent across semesters"
   could be split so that "heavily curved" ends up in one chunk and "consistent across
   semesters" in another. Neither chunk alone fully answers a query about his grading. The
   50-character overlap is designed to mitigate this, but short reviews with a single key fact
   are especially vulnerable. I'll inspect chunks manually before embedding to catch this.

---
## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PIPELINE OVERVIEW                           │
└─────────────────────────────────────────────────────────────────────┘

  [1] Document Ingestion          [2] Chunking
  ──────────────────────          ──────────────────────
  Tool: Python + manual .txt      Tool: custom chunk_text()
  • Load .txt files from disk     • chunk_size = 300 chars
  • Strip HTML artifacts          • overlap = 50 chars
  • Normalize whitespace          • attach source metadata
         │                                │
         ▼                                ▼
  [3] Embedding + Vector Store    [4] Retrieval
  ──────────────────────────      ──────────────────────
  Tool: sentence-transformers     Tool: ChromaDB query()
        all-MiniLM-L6-v2          • top-k = 5
  Tool: ChromaDB (local)          • returns chunks +
  • embed each chunk                source filenames
  • store with source metadata           │
         │                               ▼
         └──────────────────────▶ [5] Generation
                                  ──────────────────────
                                  Tool: Groq API
                                        llama-3.3-70b-versatile
                                  • prompt enforces grounding
                                  • cites source documents
                                  • declines out-of-scope queries
                                         │
                                         ▼
                                  [6] Query Interface
                                  ──────────────────────
                                  Tool: Gradio
                                  • text input box
                                  • answer output box
                                  • sources output box
```

---
## AI Tool Plan

**Milestone 3 — Ingestion and chunking:**
I will give Claude my `## Documents` section (listing file types and sources) and my
`## Chunking Strategy` section (chunk size 300, overlap 50, rationale). I will ask Claude to
implement two functions: `load_documents(folder_path) -> list[dict]` that reads all `.txt`
files and returns a list of `{text, source}` dicts, and `chunk_text(doc, chunk_size=300,
overlap=50) -> list[dict]` that splits the text and preserves the source filename in each
chunk's metadata. I will verify the output by printing 5 random chunks and checking that each
is under 300 characters, contains readable text (no HTML artifacts), and has a non-empty
`source` field.

**Milestone 4 — Embedding and retrieval:**
I will give Claude my `## Retrieval Approach` section and the `## Architecture` diagram. I will
ask Claude to implement `embed_and_store(chunks, collection_name) -> None` using
`SentenceTransformer("all-MiniLM-L6-v2")` and ChromaDB, storing each chunk with its source
metadata. I will also ask Claude to implement `retrieve(query, collection, k=5) -> list[dict]`
that returns the top-k chunks and their source filenames. I will verify by running 3 of my
evaluation plan queries and checking that returned chunks visibly relate to each question and
that distance scores are below 0.5.

**Milestone 5 — Generation and interface:**
I will give Claude my grounding requirement ("answer only from retrieved context, cite source
documents, decline out-of-scope queries") and the Gradio skeleton from the project spec. I will
ask Claude to implement `ask(question) -> {answer, sources}` that builds a prompt with the
retrieved chunks as context and calls the Groq API, and to wire it into a Gradio UI with an
input textbox, an answer output, and a sources output. I will verify by testing 2–3 queries
end-to-end and confirming that (1) the answer text references content from my documents, not
generic LLM knowledge, and (2) source filenames appear in the output.
