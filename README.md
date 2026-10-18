# brain

A personal knowledge base you can talk to from any Claude chat.

You save anything: notes, pasted articles, job posts, conversations. The brain keeps the text, makes it searchable by meaning and by keyword, and builds a graph of the people, companies and projects it mentions, plus a list of facts about you. When you ask a question, it returns the passages that answer it, says how strong the evidence is, and can check an answer against the sources it cites.

```mermaid
flowchart TB
    you(["You, in a Claude chat or the command line"])
    you -- "save this" --> ingest["<b>Save</b><br/>store, chunk, summarize,<br/>embed, extract a graph"]
    you -- "ask" --> search["<b>Search</b><br/>five layers, one evidence contract"]
    ingest --> db[("<b>Postgres + pgvector</b><br/>documents · passages · graph · facts · logs")]
    db --> search
    db --> obsidian["<b>Obsidian vault</b><br/>read-only mirror"]
    search -- "passages, facts,<br/>evidence level" --> answer(["An answer with [P1] [F1] citations"])
    answer -- "its claims" --> verify["<b>Verify</b><br/>check each claim against its sources"]
```

- **Nothing is lost.** The raw text is always kept, so even when the smarter layers miss, a literal scan can still find an exact code or figure.
- **Every answer is traceable.** Each passage and fact has an id you can open. Each search is logged and can be replayed.
- **Facts about you come only from your own words.** A saved article by someone else never becomes a fact about you.
- **Costs are capped.** Embedding calls are metered against a hard daily token cap.

Design doc: `docs/superpowers/specs/2026-09-27-knowledge-base-core-design.md`.

**Contents:** [How it works](#how-it-works) · [Setup](#setup) · [Using it](#using-it) · [Reading a search result](#reading-a-search-result) · [Reference](#reference)

---

## How it works

### 1. Saving a document

When you save something, the brain answers as soon as the text is stored and split into passages. The rest runs in the background, one stage at a time. A document's stage is recorded, so an interrupted document picks up where it stopped.

```mermaid
flowchart TB
    A["<b>stored</b> · raw text kept"] --> B["<b>chunked</b> · sections and passages"]
    B -. "brain_ingest returns here" .-> R(["you get the document id"])
    B --> C["<b>summarized</b> · Claude writes a title and summary"]
    C --> D["<b>embedded</b> · Voyage vectors for each passage and the summary"]
    D --> E["<b>extracted</b> · Claude reads each section: entities, relations, facts"]
    E --> F["<b>resolved</b> · entities matched to existing graph nodes"]
    F --> G(["done"])
    classDef bg fill:#eef3ff,stroke:#7a8cc2,color:#1a1a1a
    class C,D,E,F bg
```

Blue stages run in the background.

| Stage | What happens | Calls |
|---|---|---|
| stored | The raw text, its kind (note, news, job_description…), author, date and origin are saved. | none |
| chunked | Split into **sections** (~1,500 tokens, used for extraction) and **passages** (~400 tokens with 15% overlap, used for search), on markdown headings and sentence boundaries. Each passage keeps its heading path. | none |
| summarized | A title, a one-line summary and a longer summary. | Claude |
| embedded | Each passage is embedded with a short context prefix (document title, summary line and heading path), and so is the summary. | Voyage |
| extracted | People, organizations, places, projects, events, artifacts and concepts; relations between them; facts about you, each with a verbatim quote. | Claude, once per section |
| resolved | Each entity is matched to an existing node or creates one; relations become edges and facts are written, each pointing at the passage that supports it. | Voyage (entity names) |

```mermaid
flowchart TB
    doc["Document"] --> s1["Section 1<br/>~1,500 tokens"] & s2["Section 2"]
    s1 --> p1["Passage<br/>~400 tokens"] & p2["Passage"] & p3["Passage"]
    s2 --> p4["Passage"] & p5["Passage"]
    s1 -. "read by the extractor" .-> x(["entities, relations, facts"])
    p2 -. "embedded and indexed" .-> q(["what search returns"])
```

At most 2 documents process in the background at once, so a slot stays free for new saves. If the process running a document exits partway, `brain_status` lists it as stalled and resumes it (see [Pipeline and stalls](#pipeline-and-stalls)).

### 2. Asking a question

Every search runs five layers and returns one structure, the **evidence contract**. That same structure is what the model reads, what you read, and what is logged.

```mermaid
flowchart LR
    Q(["Your<br/>question"]) --> layers
    subgraph layers["Five layers"]
        direction TB
        L1["<b>1. Hybrid passages</b><br/>vector + keyword, fused, then reranked"]
        L2["<b>2. Document summaries</b><br/>the same two branches"]
        L3["<b>3. Graph expansion</b><br/>entities named in the question, and their neighbours"]
        L4["<b>4. Facts about you</b><br/>sharing a term or pointing at a named entity"]
        L5["<b>5. Literal scan</b><br/>exact codes and figures, only when the rest is weak"]
        L1 ~~~ L2 ~~~ L3 ~~~ L4 ~~~ L5
    end
    layers --> C["<b>Evidence contract</b><br/>retrieval id · mode<br/>scores · how found<br/>author · evidence level"]
    C --> M["shown to the<br/>model and to you"]
    C --> LOG[("retrieval_log")]
    LOG --> EX["brain_explain<br/>replays the ranking"]
    LOG --> VF["brain_verify<br/>checks an answer"]
```

Layer 1 is where most answers come from:

```mermaid
flowchart LR
    q(["question"]) --> v["Vector search<br/>HNSW over Voyage<br/>embeddings, top 60"]
    q --> k["Keyword search<br/>GIN index over stems of<br/>text, headings and title, top 60"]
    v & k --> f["Reciprocal rank<br/>fusion (k = 60)"]
    f --> r["Voyage reranker<br/>score 0 to 1"]
    r --> t{"best score<br/>≥ 0.56?"}
    t -- yes --> s(["evidence: strong"])
    t -- no --> w(["evidence: weak<br/>the base may not<br/>hold the answer"])
```

If the query embedding fails or the daily cap refuses it, the search falls back to keyword-only. If the rerank fails, results keep their fused order. Either way, the result says so.

No model ranks results or checks answers. Ranking is Postgres plus Voyage; checking is Postgres stemming plus fixed rules. Claude is called only to summarize and extract at save time, to write answers in `brain ask`, and to draft eval questions.

### 3. Checking an answer

After a model answers from a search, it can pass its claims, each with the `[P1]`/`[F1]` labels it cites, to `brain_verify`. Each claim is checked only against what it cites.

```mermaid
flowchart LR
    claim(["Claim + its cites"]) --> c{"Any cites?"}
    c -- no --> U["uncited<br/><i>the model's own</i>"]
    c -- "cites nothing<br/>in that search" --> B["bad_citation"]
    c -- yes --> chk["Compare with the cited text:<br/>content words · numbers, dates, codes ·<br/>negation · polarity words"]
    chk --> S["supported<br/>≥ 60% of words, all numbers,<br/>negation agrees"]
    chk --> P["partial<br/>≥ 30%, or one check fails"]
    chk --> N["unsupported<br/>< 30%"]
```

It checks **vocabulary, not logic**. "Supported" means the cited text contains the claim's words and numbers, not that the claim is true. A swapped entity or a reversed relation can pass. Every rule and every known limit is listed in [Answer checking](#answer-checking).

### 4. Who wrote it decides what becomes a fact

```mermaid
flowchart LR
    d(["Saved document"]) --> a{"author?"}
    a -- owner --> yes["Facts about you and<br/>relations from you<br/>are written"]
    a -- "other / unknown" --> no["Kept in the stored extraction<br/>but not written,<br/>and counted in brain_status"]
    yes --> sv{"single-valued fact?<br/>lives_in, visa_status,<br/>current_employer…"}
    sv -- yes --> sup["The newer value supersedes<br/>the older one; a value you set<br/>by hand always wins"]
    sv -- no --> add["Added alongside<br/>existing facts"]
```

Notes, pastes, conversations and resumes default to `owner`. News, papers, job descriptions and emails default to `other`. Pass `author: "other"` when you save someone else's writing as a note.

---

## Setup

1. `npm install`
2. `cp .env.example .env` and fill in `VOYAGE_API_KEY`.
   - Model calls go through your local Claude Code CLI (`claude`) on your Claude subscription by default (`BRAIN_LLM=claude-code`). Make sure `claude -p hi` works. Each call runs with no tools and `--strict-mcp-config`, so none of your MCP servers start for it.
   - `BRAIN_LLM=api` with `ANTHROPIC_API_KEY` is optional, for per-token API billing.
   - The `backfill` command (Message Batches API) always needs `ANTHROPIC_API_KEY`.
3. `npm run db:start` (Docker), then `npm run db:reset` to apply migrations locally.

Local Supabase uses ports 55320-55329 (set in `supabase/config.toml`), because the default ports may be taken by another local Supabase project. The local connection string is `DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:55322/postgres`.

To use a hosted Supabase project instead: create the project, run `supabase link --project-ref <ref>` and `supabase db push`, then set `DATABASE_URL` in `.env` to the project's direct connection string.

Before you run code that needs a new migration (for example `20261003000012_verification_log.sql`, which `brain_search` and `brain_verify` write to), apply it to the real database. Otherwise searches fail on the missing column or table:

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f supabase/migrations/<file>
```

---

## Using it

### From a Claude chat (MCP)

```mermaid
flowchart LR
    subgraph mac["Your Mac"]
        cc["Claude Code"] -- stdio --> s1["brain MCP server"]
        cd["Claude Desktop"] -- stdio --> s1
        s1 --> db[("local Postgres")]
    end
    subgraph remote["Optional: hosted"]
        web["claude.ai, ChatGPT,<br/>phone"] -- "HTTPS +<br/>bearer token" --> s2["brain MCP server<br/>(Fly.io)"]
        s2 --> sdb[("hosted Supabase")]
    end
```

**Claude Code.** Register once at user scope, so every chat gets the `brain_*` tools:

```bash
claude mcp add --scope user --transport stdio brain -- /Users/frankfu/Documents/GitHub/brain/node_modules/.bin/tsx /Users/frankfu/Documents/GitHub/brain/src/mcp/stdio.ts
```

Say "save this to my brain" in any chat to save something. Ask anything, and the model calls `brain_search` when it needs your material. The stdio server stops when Claude Code closes it.

**Other clients (HTTP).** See [Hosting the server](#hosting-the-server).

**The eleven tools:**

| Tool | What it does | Read-only |
|---|---|---|
| `brain_orient` | What the base holds: counts, recent documents, facts about you, and which tool to use. Call it first. | ✓ |
| `brain_search` | The five-layer search above. `verified_only` filters facts and neighbours only. | ✓ |
| `brain_get_document` | Fetch one document. | ✓ |
| `brain_get_node` | An entity and its neighbours. The graph keeps one edge and one fact per supporting passage; each relationship and fact is listed once, with how many passages support it. | ✓ |
| `brain_get_facts` | List current facts. | ✓ |
| `brain_status` | Pipeline progress, failures and stalled documents (a writable server resumes them). | ✓ |
| `brain_explain` | Replay a logged search from its retrieval id: mode, candidate counts, timings, and each passage's ranks and score. | ✓ |
| `brain_verify` | Check an answer's claims against the passages and facts they cite. No model call. | ✓ |
| `brain_ingest` | Save text: a note, pasted article or conversation. A URL is recorded as the origin, not fetched. Pass `author: "other"` for anything you did not write. | |
| `brain_add_fact` | Record a fact. | |
| `brain_supersede_fact` | Replace a fact with a corrected one. | |

With `BRAIN_MCP_READONLY=1`, only the eight read-only tools are exposed. `brain_verify` counts as read-only: it changes nothing in the knowledge base and only writes its audit row, just as `brain_search` writes `brain.retrieval_log`.

**What the server tells the model.** On connect, the server sends instructions that the client places in the model's system prompt:
1. Call `brain_orient` once per session.
2. Call `brain_search` with the question in plain words.
3. Cite passages and facts as `[P1]`, `[F1]`.
4. Answer only what a passage states when the evidence is weak.
5. Run `brain_verify` and flag anything not supported.

Every tool call is logged to `brain.tool_calls`, with its client, arguments (saved text as its length only), outcome and duration, so you can check whether a session followed that order:

```bash
psql postgresql://postgres:postgres@127.0.0.1:55322/postgres -c "select created_at, client, tool, ok, args from brain.tool_calls order by created_at desc limit 20;"
```

### From the command line

| Task | Command |
|---|---|
| Save a file, folder, URL or stdin | `npm run brain -- ingest <file\|dir\|url\|-> [--kind note] [--author owner\|other\|unknown] [--title T] [--occurred-at 2026-01-01] [--meta k=v] [--until chunked]` |
| Pipeline progress | `npm run brain -- status` |
| Resume unfinished documents | `npm run brain -- retry [--stage embedded]` |
| Search | `npm run brain -- search "<query>" [--kind news note] [--since 2026-01-01] [--until 2026-12-31] [--verified] [-k 10] [--json]` |
| Ask (search, answer, check) | `npm run brain -- ask "<question>"` |
| Replay a search | `npm run brain -- explain <retrieval-id>` |
| Check claims | `npm run brain -- verify <retrieval-id> --claims <claims.json>`<br/>`npm run brain -- verify <retrieval-id> --claim "<text>" [--cite P1 --cite F2]` |
| Inspect an entity | `npm run brain -- node "<name or id>"` |
| List facts | `npm run brain -- facts [--all]` |
| Mark a fact as true | `npm run brain -- verify-fact <id>` |
| Fix a document's author | `npm run brain -- set-author <document-id> <owner\|other\|unknown>` |
| Voyage usage | `npm run brain -- usage [--days 30]` |
| Rebuild the Obsidian mirror | `npm run brain -- project-obsidian` |
| Bulk extraction via the Batches API | `npm run brain -- backfill [--limit 500] [--poll 30]` |

Eval commands are under [The eval program](#the-eval-program).

---

## Reading a search result

```
retrieval 6f1c2a0e-… · mode: hybrid · 3 passages

[P1] 0.76 rerank · vector#2 keyword#5 · note · author: other · "Databricks costs" · 2026-09-29 (doc 31f1…, chunk 5ec6…)
  What a runaway Databricks bill taught me
Three months later the invoice had nearly quadrupled, from about $41,000 a month to $157,000…

[P2] - · graph via Acme Corp · note · author: owner · "Acme notes" · undated (doc 8b0d…, chunk 77a1…)
Priya confirmed Acme sponsors H-1B and has done it for two analysts on her team…

[P3] - · fallback "X-90" · news · author: other · "Zorblax news" · 2026-08-02 (doc 4c3e…, chars 0–260)
…the X-90 replaces the older model in all Texas plants…

Documents by summary: Databricks costs [note] (doc 31f1…); Acme notes [note] (doc 8b0d…)
Entity organization: Acme Corp (node 2d1e…, matched "acme corp") — Priya Natarajan (person), Austin, TX (place)
Facts about the owner:
[F1] visa_status: F-1 OPT (unverified · from note 9a2e…)
[F2] lives_in: Denver (unverified · stated by owner)
```

Each passage is a provenance line, then its heading path (indented, when it has one) and its full text. `brain search` on the command line prints the text as one line of at most 240 characters instead. After the passages come the documents whose summaries matched, the entities named in the query with their neighbours, and the facts about you.

| Part | Meaning |
|---|---|
| `retrieval <id>` | The log row. `brain_explain` or `npm run brain -- explain <id>` replays the search without searching again. |
| `mode` | `hybrid`: vector and keyword candidates, reranked. `keyword-only`: the query embedding failed or the cap refused it, so nothing was reranked. `fused-order`: the rerank failed or was refused, so passages are in fusion order. A degraded search says why on the next line. |
| `(evidence: weak — …)` | Printed under the header when the best rerank score is below 0.56 (`config.retrieval.answerThreshold`), or nothing was reranked. The base may not hold the answer, so answer only what a passage states outright. A bare code lookup (a query made only of terms like `X-90`) is judged instead by whether every term appears literally. A degraded search has no rerank score, so its evidence is `unknown`. See [Abstention](#abstention). |
| Score | `rerank`: relevance from 0 to 1, higher is stronger. `rrf` (about 0.008 to 0.033): only orders one degraded search, and can't be compared with rerank scores. `-`: unscored, found through the graph or the literal scan. |
| How found | `vector#n` / `keyword#n`: rank among each branch's up to 60 candidates. `graph via <entity>`: the passage mentions an entity named in the query; a ranked passage can show both, and keeps its score. `fallback "<term>"`: a window of the raw document containing an exact term, so it has a character range instead of a chunk id. |
| `author` | Who wrote the document. A passage by someone else says what they wrote, not what is true of you. |
| Facts | `verified` once you confirm with `verify-fact`. `from <kind> <doc id>`: the extractor read it from that document. `stated by owner`: recorded on your word, with no source passage. `confirmed by owner`: verified, with no stored source passage. `extracted; source passage no longer stored`: its passage was re-chunked or deleted. |

A strong score is not proof either. The reranker measures relevance, so a passage about the right person can score high and still lack the detail asked for. In an answer, anything without a `[P…]` or `[F…]` cite is the model's own. `brain ask` prints its sources under the answer and checks each sentence. `brain_verify` does the same for any MCP client.

<details>
<summary>What the evidence contract and the log contain</summary>

Every search (`brain_search`, `brain search`, `brain ask`) returns one structure, `SearchResult` in `src/retrieve/contract.ts`, and logs it to `brain.retrieval_log`. The structure holds the retrieval id, the mode and which parts degraded, candidate counts per branch, and stage timings. For every passage it holds the score and score kind, the layers that found it, its vector, keyword and rerank ranks, and its document's author, source kind, date and ids. `brain_search` also returns this structure as `structuredContent`, without each passage's text, which the text content already carries. `brain search --json` prints the full structure, text included.

Size: on the largest possible result (k=30 plus 25 graph passages at the 1,600-character passage cap, `test/unit/search-output-size.test.ts`), repeating the text in `structuredContent` would bring one call to about 235 KB. Without it, the call is about 146 KB, of which the text is about 106 KB. That is more than some clients show from one tool call (Claude Code's default limit is 25,000 tokens). At the default k=10, the largest result stays under 100 KB (about 99 KB).

`brain.retrieval_log` keeps, per search:
- the query, filters, client and time
- `mode`, and `degraded` (`embedding`, `rerank`, `capReached`)
- `candidates` (`vector`, `keyword`, `fused`)
- `timings` (`embedMs`, `sqlMs`, `rerankMs`, `graphMs`, `totalMs`)
- `k`, and `results`: every returned passage in rank order, with everything above except its text

Rows logged before migration 011 have only chunk ids, layers and a top score (which may be an RRF value), and explain says "logged before evidence v2". From migration 012, each row also keeps `facts`, the facts the search returned in order (index 0 is F1), which `brain_verify` resolves F labels from. From migration 013, each row also keeps `evidence` (`level` strong, weak or unknown; `basis` rerank, literal or no_rerank; and the `threshold` in force), which explain prints.

</details>

---

## Reference

### Search layers in detail

![A query runs through five layers into one evidence contract, which is shown to the client, logged, replayed by brain_explain and used by brain_verify](docs/retrieval.svg)

`search()` in `src/retrieve/search.ts` runs these layers for each query:

1. **Hybrid passages.** A vector branch (HNSW over Voyage embeddings of each passage) and a keyword branch each return up to 60 candidates. The keyword branch is a GIN index over the stems of each passage's text, heading path and title; it matches any query stem and ranks by how many distinct stems match. The two lists are fused with reciprocal rank fusion (k = 60) and reranked by Voyage. The top k (default 10, at most 30) are returned with the rerank score, 0 to 1.
2. **Document summaries.** The same two branches over each document's summary, fused, listed after the passages.
3. **Graph expansion.** Entities named in the query (any case, names up to six words, at most 5) are matched to graph nodes. Each adds up to 20 neighbours and up to 5 passages that mention it, including mentions on merged nodes. A graph passage has no score; a ranked passage the graph also reached keeps its score and gains the entity.
4. **Facts about the owner.** Current facts that share a stem with the query or point at a named entity, at most 10, entity-linked first.
5. **Literal scan.** When the query holds an exact-string term (a code such as `X-90`, a figure such as `$115k`, a version), and the best rerank score is below 0.3 or the search was degraded, documents containing the term are scanned and a window around the match is returned.

The parameters are in `src/config.ts` (`retrieval`, `graph`) and `src/retrieve/fuse.ts`. `test/integration/search-plan.test.ts` fails if a query plan stops using the HNSW or GIN index.

### Pipeline and stalls

`brain_ingest` returns once the document is stored and chunked. Summary, embeddings and extraction continue in the background, at most 2 pipelines at once. Each pipeline holds a per-document lock while it runs, so two processes never work on the same document.

A document is **stalled** when it is unfinished, has no error, no process holds its lock, and it has made no progress for 10 minutes. That happens, for example, when the server running it exited. `brain_status` lists stalled documents, and a writable server resumes them. Unfinished work also resumes on the next save, or with `npm run brain -- retry`.

### Facts and authorship

- **Unverified until you confirm.** Facts written by an agent stay unverified until you run `npm run brain -- verify-fact <id>`. A correction supersedes the old fact; nothing is deleted.
- **Author rule.** Every document records who wrote it: `owner`, `other` or `unknown`. Only documents you wrote produce facts about you or relationships from you. For any other document, the extractor's statements about you are kept in its stored extraction but not written, and `brain status` / `brain_status` show how many per document.
- **Defaults.** Without `--author` (CLI) or `author` (MCP), resume, note, conversation and paste default to `owner`; news, paper, job_description and email to `other`; anything else to `unknown`. Saving text that is already stored keeps its author (the CLI and `brain_ingest` say so when you asked for a different one).
- **Fixing a wrong author.** If someone else's post was saved as yours, run `npm run brain -- set-author <document-id> other`. It removes the facts and relationships that document produced (facts you verified are kept and listed), applies the rule again and refreshes the Obsidian mirror. While the document is being processed it refuses (try again in a moment); for a document not yet resolved it only records the author, which the resolve stage then applies.
- **Single-valued facts.** `lives_in`, `visa_status`, `targeting_role`, `pursuing_degree`, `employment_status`, `current_employer`, `phone` and `email` keep one current value. When a document of yours states a different value, dated no earlier than the current one, the old fact is superseded. An older statement resolved later is recorded as already superseded.
- **Your values win.** Values you set by hand (`brain_add_fact`, `brain_supersede_fact`) or verified always win over extraction. A document stating a different value, whatever its date, is recorded as already superseded by yours. Facts you corrected are kept as history (`set-author` lists them), and no document can state a value you corrected away from again.
- **Audit trail.** Every supersession, restoration and removal is logged in `brain.fact_events`.
- **Consistent fact names.** The extractor is given a list of fact predicates to reuse (`factPredicates` in `src/config.ts`). It is told that a predicate names the relation, not its object, and that values are written in full ("Fall 2026", not "F26"). Names it has used for a listed predicate are mapped onto it (`predicateAliases`).
- **Leak check.** The eval measures leaks directly: the number of facts about the owner, and edges from the owner, whose evidence lies in a document the owner did not write. It must be 0.

### Answer checking

`brain_verify` (MCP), `npm run brain -- verify` and `brain ask` check an answer claim by claim against the passages and facts each claim cites. The check is deterministic: no model is called, so it costs nothing per query and gives the same verdict every time. The code is in `src/verify/` (`terms.ts` for the extraction, `verify.ts` for the method, `resolve.ts` for looking up cites). Every verification is logged to `brain.verification_log`.

<details>
<summary>How a claim is checked, step by step</summary>

1. **Cites are resolved through the search the answer came from.** `P3` is the third passage of that retrieval: its text is read from `brain.chunks`, and a fallback passage is cut from the document's raw text by its character window. `F1` is the first fact that search returned, as it was then. A chunk id or fact id also works. A passage's cited text is its heading path plus its content; a fact's is its predicate (underscores as spaces) plus its value.
2. **Numbers, dates and codes are read from the claim and normalised.**
   - Numbers: `1,000` is `1000`; `~11%`, `11 %` and `11 percent` are `11%`; `$115k`, `$115K`, `$115,000` and `115,000 dollars` are `$115000` (k, m, b, thousand, million and billion scale the number).
   - Dates: `Sep 29, 2026`, `29 September 2026` and `2026-09-29` are `2026-09-29`; `September 2026` is `2026-09`; `October 6` is `--10-06`; `3rd` is `3rd`.
   - Number words: `two` to `ninety` and `two hundred` are numbers. `one` and `first` are not, since they are usually not counts.
   - Codes: a token with a letter and a digit (`H-1B`, `F-1`, `ZX-9000`, `v2.5`) is a code, compared as written.
   - Exact forms: a leading minus stays with its number (`-5%` is not `5%`). Numeric dates (`3/4/2026`), dotted versions (`3.12.1`), digit groups (`555-1234`) and percent ranges (`20-30%`) are compared exactly as written, never as loose numbers. A range of two years (`2019-2023`) is two years.
   - Every one of them must appear in the cited texts. A full date in a source also states its month, year and month-day; a sum of money also states the bare amount.
3. **Content terms** are the rest of the claim's words, stemmed by Postgres (`to_tsvector('english', …)`, the same stemming as the keyword index), without stopwords, negation words, or answer words such as "yes", "also" and "however". Support is the share of the claim's distinct content terms that appear among the cited texts' stems.
4. **Negation.** The words not, no, never, without, none, neither, nor, cannot, and anything ending in n't are read from the raw text. The claim is compared with its anchor sentences: every cited sentence holding at least half of the claim's matched terms, or, when none does, the sentence or sentences holding the most. There is a mismatch when the claim has a negation word and no anchor sentence has one, or the claim has none and an anchor sentence has one. A negation in a cited sentence that shares only a word or two with the claim does not count.
5. **Polarity words.** up, down, before, after, over, under, above, below, more, most, less, fewer, few, all, some, only, against, will, would, might, must, can, could and should are Postgres stopwords, so stemming alone would let "went up 11%" match "went down 11%". They are read from the raw text (`won't` counts as will, `cannot` as can), and every one in the claim must appear as a whole word in the cited texts.
6. **Counting words.** When a content term the cited texts lack is a number or ordinal word (`one`, `first`, `dozen`, `tenth`), the claim states a count or rank the source does not.

</details>

Verdicts, from the first rule that applies:

| Verdict | Rule |
|---|---|
| `bad_citation` | Every cite names nothing in that search (`P9` when it returned 5 passages; `F1` on a search logged before facts were recorded; a passage since re-chunked). Bad cites next to a good one are listed but do not change the verdict. |
| `uncited` | No cites: the model's own statement. |
| `supported` | Support at least 0.6, every number and polarity word present, no negation mismatch, and no missing term that is a number or ordinal word. A claim with no content terms ("Yes [P1].") is supported when it has no numbers, no negation mismatch and no missing polarity word, since it states nothing the source could contradict. With numbers it is at most partial, because nothing says what the figure measures. |
| `partial` | Support at least 0.3, or support at least 0.6 with a missing number, a missing polarity word, a missing number or ordinal word, or a negation mismatch. |
| `unsupported` | Support below 0.3. A failed check never raises a verdict. |

**What it does not check.** Read `supported` as "the cited text contains this claim's words and numbers", not as proof:

- It checks word overlap, not who did what to whom: reversed relations ("Acme led Beta Ventures' round") and swapped entities pass.
- A number only has to appear somewhere in the cited text, not attached to the same thing ("Acme employs 40 people" passes against "grow its teams by 40 people").
- All cited texts are pooled, so citing unrelated passages together can support a claim that neither supports alone.
- Negation is compared only with the anchor sentences, per sentence, not per clause.
- Antonyms ("halved" for "quadrupled") and words like "former" are not detected.
- Up to 40% of a claim's content words can be absent at `supported`, so one added detail in a short claim can pass.
- A bare amount ignores its unit (20 minutes matches 20 hours), though `%` and percentage points are told apart.
- A claim made only of stopwords ("Yes.", "They did.") is vacuously supported.
- A correct paraphrase in other words scores partial or unsupported (the safe direction).
- A raw fact id is checked against the fact as stored now; when that fact has been superseded, the verification notes it.
- Reasoning, sarcasm and certainty words outside the polarity list ("may", "probably") are not checked.

<details>
<summary>Worked example</summary>

A search returned the compensation section of `eval/corpus/job_description--acme-senior-data-analyst.md` as P1 ("Base salary range $115,000 to $140,000. Acme sponsors H-1B for this role. Hybrid, three days a week in the Austin office. …", under the heading "Compensation and visa"). The answer was: "Acme sponsors H-1B visas for this role [P1]. The base salary is $115k to $150k [P1]. The role is not hybrid [P1]. The company will pay for relocation to Austin [P1]. It looks like a strong fit."

```
verification 12bf8876-… · retrieval a4cf4454-… · 5 claims
✓ supported 1.00 — "Acme sponsors H-1B visas for this role." [P1]
~ partial 1.00 — "The base salary is $115k to $150k." [P1]
    missing numbers: $150000
~ partial 1.00 — "The role is not hybrid." [P1]
    negation differs from the cited text
✗ unsupported 0.25 — "The company will pay for relocation to Austin." [P1]
    missing terms: company, pay, relocation · missing polarity words: will
○ uncited - — "It looks like a strong fit."
    no citation: nothing from the knowledge base backs this
Summary: 1 supported, 2 partial, 1 unsupported, 1 uncited
```

Each line shows the verdict, the support (`-` when there is none), the claim and its cites. The line under a claim says what its cited text lacks: terms in the claim's own words, numbers in their normalised form. "visas" in the first claim matched the heading. `$115k` matched `$115,000`; `$150k` did not. Present anything not `supported` as the model's own or as weakly supported; the server instructions ask MCP clients to do exactly that after calling `brain_verify`.

</details>

**Limits on input.** `brain_verify` takes at most 50 claims of at most 2,000 characters, with at most 20 cites each. `brain ask` splits its own answer into sentences: a line break, or `.` `!` `?` followed by a word that does not start in lower case, never after `e.g.`, `Dr.`, `U.S.` or an initial, and never at a decimal point. It cites the `[P#]`/`[F#]` labels inside each sentence and prints the check under its sources.

**How well it works.** It is measured on `eval/verifier.jsonl`: 104 claims quoted against the eval corpus (and, for the number and polarity forms the corpus lacks, written out as facts). The set includes 27 known limits:
- 21 false claims the method passes: reversed relations, swapped entities, numbers attached to another thing, pooling across cites, negation on the wrong clause, antonyms and "former", bare amounts with another unit, stopword-only claims, certainty and one added detail.
- 6 true claims it cannot recognise: four paraphrases in other words, a bare figure the source states, and "No." to "Is the role fully remote?".

Every item is labelled with its true verdict, so each known limit the verifier gets wrong counts as an error.

| View | What it measures | Precision of `supported` | Recall | Accuracy | n |
|---|---|---|---|---|---|
| Regular | Only the cases the method is designed for | 1.00 (40 of 40) | 0.98 | 0.97 | 77 |
| Full | Everything, including the known limits: the honest figure for real answers | 0.67 (40 of 60) | 0.85 | 0.73 | 104 |

Measured 2026-10-03. In the full view, 20 of the 21 false known-limit claims are still marked supported, and none of the 6 true ones is.

`npm run brain -- eval verifier` prints each item, a confusion matrix for each view, both summary lines and the known-limits count; `eval run` prints the two summary lines. `--gate` fails when:
- regular precision of `supported` is below 0.9 (a claim wrongly marked supported is worse than one wrongly flagged)
- full precision falls more than 0.02 below `eval/verifier-baseline.json`
- the set of items changed since that baseline, including an item edited in place ("verifier set changed; review and run `eval verifier --accept`")
- there is no baseline

`npm run brain -- eval verifier --accept` records the baseline (full precision, recall, accuracy, each item's id and sha256, and the commit), and refuses, exiting 1, when the regular bar fails.

A claim labelled `supported` is always one its cited text really supports, and a true claim is labelled `supported` even when the method cannot see it. These labels were written by an agent (`labelled_by: "agent:claude"`), not by the owner: review them, and add your own with `labelled_by: "owner"`.

### Voyage spending cap

```mermaid
flowchart LR
    req(["Voyage request"]) --> est["Estimate tokens<br/>characters ÷ 4"]
    est --> chk{"today's total +<br/>estimate > cap?"}
    chk -- yes --> refuse["Refused, never sent<br/>search: keyword-only<br/>ingest: stops before embedding"]
    chk -- no --> send["Sent"] --> rec["Record Voyage's<br/>real token count"]
```

Every Voyage request (passage and summary embeddings, entity-name embeddings during resolve, query embeddings, reranking) is recorded in `brain.provider_usage` and counted against a hard daily cap: `BRAIN_VOYAGE_DAILY_TOKEN_CAP` tokens per UTC day, default 5,000,000.
- **Before each request,** the client reserves its estimated tokens (characters / 4) under a database lock. It refuses the request without sending it when today's total plus the estimate would pass the cap.
- **After the response,** it records Voyage's own `usage.total_tokens`. A request Voyage answers with an error status is recorded at 0 tokens. A request that times out, or whose connection drops, counts at its estimate, since Voyage may have processed and billed it. A retry is a request of its own.
- **`0` blocks every call.** There is no setting that turns the cap off, and a value that is not a whole number stops the program at startup, so a typo can never lift it.

When the cap is reached:
- **Ingestion** stores, chunks and summarizes as usual, then stops before embedding (or before resolving). `brain status` shows `spend_cap: …` on those jobs, and no retry attempt is used up. After the first refusal, the rest of a batch stops before its Voyage stages without asking again. `brain retry` (or the next `brain_ingest`, which resumes stalled jobs) finishes them after 00:00 UTC, or at once after raising the cap.
- **Search** returns keyword-only results and says "Voyage daily cap reached; keyword-only results" (or "…; results in fused order" when only the rerank was refused).

`npm run brain -- usage [--days 30]` prints requests, tokens, refused calls and errors per UTC day and operation, plus today's tokens against the cap. It adds an estimated cost once `BRAIN_VOYAGE_PRICE_PER_MTOK_EMBED` and `BRAIN_VOYAGE_PRICE_PER_MTOK_RERANK` are set from Voyage's pricing page (default 0: tokens only). `brain_orient` shows today's tokens against the cap (or "Voyage ledger unavailable" on a database without migration 010).

<details>
<summary>How far past the cap a day can go, and how the eval database is capped</summary>

Settled calls count their real tokens, so the only overshoot comes from calls in flight when the cap is reached, which were admitted at their characters / 4 estimate. On measured data that estimate was within about 4% for English; for non-English or code-heavy text it can undercount 2–4×. The worst case is roughly (number of concurrent callers) × (largest batch estimate) × (estimate error): tens of thousands of tokens for English, more for other text. The overshoot never grows over the day, because every later reservation sees the settled totals.

The ledger lives in each database: the real knowledge base and `brain_eval` each count and cap their own calls. The eval database has its own cap, `BRAIN_EVAL_VOYAGE_DAILY_TOKEN_CAP` (default 1,000,000, read the same way). Any database whose name ends in `_eval` is held to it, whichever command opens it. Voyage bills the account, so **the account-wide daily ceiling is `BRAIN_VOYAGE_DAILY_TOKEN_CAP` plus `BRAIN_EVAL_VOYAGE_DAILY_TOKEN_CAP`** (6,000,000 tokens with the defaults), plus the in-flight overshoot and the cap of any other database using the same key. Running processes read the caps when they start, so restart the MCP server after changing them. A request in flight when its process dies stays counted at its estimate for the rest of the day.

A refused query embedding or rerank makes a search degraded, which the result says and the eval gate rejects.

Timeouts: each Voyage request times out after 8 seconds for search and 120 seconds for ingestion. Search sends at most 3 HTTP requests per Voyage call (rate limits, server errors and timeouts share that count) and waits at most 10 seconds in total between them, then falls back. Ingestion waits out rate limits (up to 6 attempts, at most 60 s per wait).

</details>

### Hosting the server

The hosted server needs:
- `DATABASE_URL` pointing at the Supabase project (run `supabase db push` first so the schema exists there)
- `VOYAGE_API_KEY`
- `BRAIN_TOKENS` as `name:token` pairs. Each token must be at least 32 characters (`openssl rand -hex 32`) and is sent as `Authorization: Bearer <token>`. The server refuses to start with no tokens or with short ones.

The image runs as the non-root `node` user with `BRAIN_LLM=api`, so ingestion over HTTP also needs `ANTHROPIC_API_KEY`. Set `BRAIN_MCP_READONLY=1` to expose only the read tools and skip that key.

Fly.io example (adapt names and region):

```bash
fly launch --no-deploy --name brain-mcp --region sjc
fly secrets set DATABASE_URL='postgresql://...' VOYAGE_API_KEY='...' BRAIN_MCP_READONLY=1 \
  BRAIN_TOKENS="claude-desktop:$(openssl rand -hex 32),chatgpt:$(openssl rand -hex 32)"
fly deploy
curl https://brain-mcp.fly.dev/healthz
```

Then register the remote server in Claude Code as a second entry (example):

```bash
claude mcp add --transport http brain-remote https://brain-mcp.fly.dev/mcp --header "Authorization: Bearer <token>"
```

Claude Desktop and ChatGPT take the same URL and header in their connector settings. To skip hosting, run `npm run mcp:http` on the Mac and expose the port through Tailscale or a Cloudflare Tunnel.

### Tests

| Command | Needs | What it does |
|---|---|---|
| `npm run test:unit` | nothing | Unit tests |
| `npm run test:int` | `npm run db:start` | Recreates a separate `brain_test` database from the migrations and runs there with fakes for Claude and Voyage, so your real knowledge base is never touched. The test helper refuses any database whose name does not end in `_test`. |
| `npm run eval:run` / `npm run eval:gate` | an `_eval` database | Retrieval eval and its regression gate. See below. |

### The eval program

```mermaid
flowchart LR
    subgraph public["Public, committed"]
        corp["eval/corpus<br/>25 fictional documents<br/>with planted traps"] --> ev[("brain_eval")]
        gold["eval/golden.jsonl<br/>152 questions"]
    end
    subgraph private["Private, gitignored"]
        real[("your real base")] -- "eval sync<br/>read-only copy" --> rev[("brain_real_eval")]
        goldr["eval/golden-real.jsonl"]
    end
    ev & gold --> run["eval run"]
    rev & goldr --> run
    run --> m["recall · MRR · nDCG ·<br/>abstention · latency ·<br/>leaks · verifier"]
    m --> g{"gate: worse than<br/>the baseline?"}
    g -- yes --> fail(["exit 1"])
    g -- no --> pass(["pass"])
```

The retrieval eval runs only against databases whose names end in `_eval`. Each run checks the live connection before any write.

**Two corpora, two databases.**
- `brain_eval` holds `eval/corpus`: 25 fictional documents (job posts, owner notes, recruiter emails, a meeting transcript, paper abstracts, news, a course note, a cover letter, a project retro, a saved article by someone else). They were written for this eval with shared people and companies, so graph questions have something to traverse. They also contain planted traps:
  - a near-duplicate job post with a different salary and visa policy
  - near-duplicate requisition codes (`REQ-4471`, `REQ-4417`)
  - product codes one digit apart (`ZX-9000`, `ZX-9100`)
  - two retrieval papers with different numbers
  - an interview date that a later email moves
  - an owner note that says where the owner lives, superseded by a later one

  `test/unit/corpus-fixtures.test.ts` pins every planted sentence. Create the database with `npm run eval:prepare`, then `npm run brain -- eval ingest`.
- `brain_real_eval` is a copy of the real knowledge base, made by `npm run eval:prepare-real` and then `npm run brain -- eval sync`. It copies documents, chunks with their embeddings, the graph and facts, with no model or Voyage call; the source is only read, through a read-only snapshot. `eval run --corpus real` runs the real-base items against it.

`eval:prepare` only creates a database. To bring an existing one up to date after a new migration, apply that migration file to it with `psql .../brain_eval -v ON_ERROR_STOP=1 -f <file>`.

**Public and private items.** This repository is public, so the two corpora are kept apart:
- Fixture items quote only the fictional corpus. They are committed in `eval/golden.jsonl`, with their baseline in `eval/baseline.json`.
- Real-base items quote the owner's own documents. They live in `eval/golden-real.jsonl`, with their drafts and review sheets in `eval/drafts-real.jsonl` and `eval/review/real/`. All of these are gitignored (`.gitignore` also catches any `*-real.jsonl` and any `real/` directory under `eval/`) and stay on the owner's machine.
- Real golden ids are opaque (`d-` and 10 hex digits, or `c-` and 8), so `eval/baseline-real.json`, which is committed, holds metrics, ids and per-item hashes but no question or quote text.

**The golden set.**
- `eval/golden.jsonl` holds 152 items on the fixture corpus: 78 approved by the owner and 74 written by an agent (the original fixture items, the ones that pin the corpus traps, and 48 held-out items, below). 41 are negative and 16 are attribution items. The 104 items other than the held-out ones are the calibration split.
- `eval/golden-real.jsonl` holds 8 more on the real base, all approved by the owner.
- How the two fixture review sheets were approved: an agent proposed `keep`, `edit` or `reject` for every draft, with a reason, and the owner accepted those recommendations as they stood. Of the 99 drafts on the two sheets, 62 were kept as drafted, 16 kept with edits and 21 rejected. The recommendation and reason for each item are in `eval/review/*.notes.md`, next to the sheets.
- Each item has a question, its kind (keyword, semantic, graph, filter, fallback, attribution, negative), the expected documents with an optional verbatim answer quote, optional paraphrases, its source (`fixture`, `generated`, `captured`) and who approved it.

Items come from three places:

```mermaid
flowchart LR
    fx["fixture<br/>written with the corpus"] --> G[("golden set")]
    dr["eval draft<br/>Claude proposes 2–3<br/>questions per document"] --> auto{"automatic checks:<br/>quote verbatim, in one passage,<br/>not a duplicate, valid schema"}
    auto -- pass --> sheet["review sheet<br/>owner marks keep,<br/>edit or reject"]
    sheet -- "eval approve<br/>(re-checks)" --> G
    cap["eval capture<br/>recent real searches"] --> lab["eval label<br/>owner names the<br/>expected document"] --> G
```

- `fixture`: written with the corpus, labelled `approved_by: "agent"`.
- `generated`: `npm run brain -- eval draft` makes one Claude call per document for 2 to 3 questions across kinds, each with a verbatim answer quote and two paraphrases, plus one question nothing in the corpus answers. Every draft is checked automatically:
  - the document must be in the eval database
  - the quote must appear verbatim in the document (whitespace aside), and inside one passage, since the eval matches quotes per passage
  - the question must not contain its quote
  - it must not duplicate a golden item or another draft (same normalised text, or Postgres stem sets with Jaccard overlap of at least 0.8)
  - the item must be valid under the golden schema

  Passing drafts go to `eval/drafts.jsonl` and a review sheet, `eval/review/<date>-<n>.md`, where the owner marks each `keep`, `edit` (changing question, quote, kind or paraphrases in place) or `reject`. `npm run brain -- eval approve --sheet <file>` checks every kept and edited item again, and only then adds them with `approved_by: "owner"`, the date, and whether they were edited. Agents never approve.
- `captured`: `npm run brain -- eval capture` lists recent real searches from the knowledge base's log (opened read-only). `npm run brain -- eval label <retrieval id> --expect <document id> [--quote "…"]` turns one into an item approved by the owner, after checking that the document is in `brain_real_eval` (run `eval sync` first; it keeps document ids).

**Eval commands.**

```
npm run brain -- eval ingest [dir]
npm run brain -- eval sync
npm run brain -- eval run [--corpus fixtures|real] [--golden eval/golden.jsonl] [--baseline eval/baseline.json] [--verifier eval/verifier.jsonl] [--verifier-baseline eval/verifier-baseline.json] [--compare] [--gate] [--accept] [--json]
npm run brain -- eval verifier [--file eval/verifier.jsonl] [--baseline eval/verifier-baseline.json] [--gate] [--accept] [--json]
npm run brain -- eval draft [--corpus fixtures|real] [--since 2026-09-01] [--limit 10] [--doc <file name or id>...] [--force]
npm run brain -- eval drafts
npm run brain -- eval approve --sheet eval/review/<date>-<n>.md
npm run brain -- eval reject --id <draft id>...
npm run brain -- eval capture [--corpus real|fixtures] [--since 2026-09-01] [--client mcp-stdio] [--limit 20]
npm run brain -- eval label <retrieval-id> (--expect <document> [--quote "<verbatim span>"] [--kind semantic] | --negative) [--corpus real|fixtures]
```

**Metrics** (`src/eval/metrics.ts`, `src/eval/run.ts`; no model call):

| Metric | Definition | Over |
|---|---|---|
| Recall@k, k = 1, 5, 10 | Share of an item's expected documents found among the documents of its top k passages, averaged | positive items |
| MRR | Mean of 1 / rank of the first expected document, ranks counted over distinct documents; 0 on a miss | positive items |
| nDCG@10 | Binary relevance per passage: a stored passage of an expected document that contains the quote (whitespace aside); the ideal ranking puts every such passage in the corpus first, up to 10 | items with quotes |
| Paraphrase consistency | Share of paraphrases whose top 10 finds the same set of expected documents as the original question | positive items with paraphrases |
| Abstention rate | Share of negative items whose search judged its evidence weak | negative items, overall and per split |
| False-answer rate | Share of negative items whose search judged its evidence strong (a negative with unknown evidence, from a degraded search, is neither) | negative items, overall and per split |
| Answerable items judged weak | Share of positive items whose search judged its evidence weak: the answer is there, and the search said it may not be | positive items, per split |
| Degraded fraction | Share of searches (questions and paraphrases) that ran without the embedding or the rerank | all searches |
| Latency | Nearest-rank p50 and p95 of each search's own total time, and per stage (embed, SQL, rerank, graph) | all items |
| Attribution leaks | Facts about, and edges from, the owner whose evidence is in a document the owner did not write | whole database |
| Voyage | Tokens, requests and refused calls of the run, from the eval database's ledger | the run |
| Verifier | Precision and recall of `supported`, and accuracy, on `eval/verifier.jsonl`, regular and full views | verifier set |

Every run prints these overall, per kind, per source (`fixture`, `generated`, `captured`), per split (`calibration`, `heldout`) for abstention, and the count of items per approver.

**The gate.** `npm run eval:gate` exits 1 when any of these is true:
- recall@10 or MRR falls more than 0.02 below `eval/baseline.json` (or `eval/baseline-real.json` for `--corpus real`)
- the abstention rate falls
- the share of answerable items judged weak rises more than 0.02 in a split
- negative items disappear
- any search ran degraded
- the golden set changed since the baseline (an item added or removed, or edited in place: the baseline keeps each item's sha256)
- there is no baseline
- any attribution leak exists
- the verifier fails its gate

**Current numbers.** The baseline was recorded on 2026-10-04 (UTC) over the fixture corpus: 152 items, 111 positive and 41 negative. `eval/baseline.json` records commit `86346d0`, the code and golden set the run used. `npm run eval:gate` was then run against it, and passed.

| | n | Recall@1 | Recall@5 | Recall@10 | MRR | nDCG@10 |
|---|---|---|---|---|---|---|
| All positive items | 111 | 0.84 | 0.97 | 0.99 | 0.96 | 0.97 |
| keyword | 35 | 0.94 | 1.00 | 1.00 | 0.97 | 0.97 |
| semantic | 36 | 0.93 | 0.97 | 1.00 | 0.96 | 0.98 |
| graph | 17 | 0.35 | 0.91 | 0.95 | 0.94 | 0.90 |
| filter | 4 | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 |
| fallback | 3 | 0.67 | 0.89 | 0.89 | 0.83 | - |
| attribution | 16 | 0.94 | 1.00 | 1.00 | 0.97 | 0.98 |
| source `fixture` (agent) | 48 | 0.75 | 0.94 | 0.98 | 0.92 | 0.95 |
| source `generated` (owner) | 63 | 0.91 | 1.00 | 1.00 | 0.99 | 0.98 |

| Other metric | Value |
|---|---|
| Paraphrase consistency | 1.00 over 126 paraphrase searches |
| Degraded | 0% |
| Latency | p50 335 ms, p95 421 ms; per-stage p50: embed 134 ms, SQL 24 ms, rerank 162 ms, graph 9 ms |
| Attribution leaks | 0 |
| Voyage tokens per full run | about 2.4 million |

**Graph items.** Every graph item lists each corpus document that says something about its person or company, so MRR (0.94, up from 0.80 when each listed one document) is the figure to read. Recall@1 (0.35) is low by construction: the top passage belongs to one document, so an item with n expected documents scores at most 1/n at rank 1, and most graph items now expect two to nine. Recall@10 is below 1 for the same reason: "What do I know about Northwind Robotics?" expects nine documents, more than ten passages usually cover.

**On the real base** (8 private items, 6 positive and 2 negative, in the gitignored `eval/golden-real.jsonl`; baseline `eval/baseline-real.json`): recall@1 1.00 and MRR 1.00. Under the Phase 7 rule both negatives are flagged weak (abstention 1.00, up from 0.00), and none of the 6 answerable items is. Eight items are too few to be more than a smoke test. The committed baseline holds each real item's sha256 alongside its id; the hashed line includes the document's UUID, so the hash cannot be matched against guessed questions.

**Cost.** One full fixture run uses about 2.4 million Voyage tokens in 556 requests, mostly reranking: every question and paraphrase reranks every candidate passage. That is more than the eval's default daily cap of 1,000,000, so the cap is raised for the day when recording a baseline (to 8,000,000 on 2026-10-04, for the accepting run and the gate run after it).

**Reproduce.**

```bash
npm run db:start
npm run eval:prepare                                  # creates brain_eval from the migrations
OBSIDIAN_AUTO=0 npm run brain -- eval ingest          # 25 documents: a summary and an extraction call each, plus Voyage embeddings
BRAIN_EVAL_VOYAGE_DAILY_TOKEN_CAP=4000000 npm run eval:gate   # every metric above, compared with eval/baseline.json
npm run brain -- eval verifier                        # the verifier alone, item by item, no Voyage call
npm run eval:prepare-real && npm run brain -- eval sync && npm run brain -- eval run --corpus real --compare   # needs your own eval/golden-real.jsonl
```

A rebuilt `brain_eval` re-runs summarisation and entity extraction, which are model output and can differ between runs, so graph-dependent ranks can move slightly; the baseline is tied to the database it was recorded on. Without a raised `BRAIN_EVAL_VOYAGE_DAILY_TOKEN_CAP`, the default 1,000,000 a day refuses calls partway through a run, which then says it ran degraded, and the gate fails.

### Abstention

`brain_search` says when the knowledge base may not hold the answer. When the best passage's rerank score is below the answer threshold, 0.56, it prints `(evidence: weak — …)` under the header (`src/retrieve/evidence.ts`; the line, its meaning and the exceptions are under [Reading a search result](#reading-a-search-result)).

**How the threshold was chosen.** On the Phase 6 baseline, the only such signal was the 0.3 threshold that triggers the literal scan, and with it the system abstained on 1 of 17 unanswerable questions. Phase 7 (`docs/superpowers/plans/2026-10-04-phase-7-abstention.md`) chose a new threshold on the 104 existing items (the calibration split), from the scores already logged by the Phase 6 run. 0.56 is the midpoint between the lowest answerable item's score (0.574) and the highest unanswerable one below it (0.551). Combining the score with how many question terms appear in the passages was tried and dropped: paraphrased questions match few terms by design, so it flagged answerable items as often as unanswerable ones. The rule and the threshold were committed (`ccc9cc4`, `154ae31`) before any held-out item was searched.

**The held-out split.** 48 items (24 unanswerable, 24 answerable), marked `split: "heldout"`, were written by an agent that was given the corpus and the golden schema but not the rule, the threshold or any score. Their numbers are the ones to quote. Once reported, they must not be used to change the threshold.

| Split | Unanswerable n | Abstained | Answered falsely | Answerable n | Answerable judged weak |
|---|---|---|---|---|---|
| calibration | 17 | 0.65 | 0.35 | 87 | 0.00 |
| held-out | 24 | 0.42 | 0.58 | 24 | 0.00 |

**What it catches and what it does not.** The held-out unanswerable items are of two kinds:
- **Off-topic questions**, about things the corpus never mentions (a credit score, a tax return, a sister's birthday), were all flagged weak: 8 of 8, with top scores from 0.23 to 0.40.
- **Near-miss questions**, about a person, company or document the corpus covers but asking for a detail it never states (the bonus percentage of a role, the authors of a paper, the founding year of a company), were flagged 2 times in 16. Their passages are about the right subject, so their top scores ran from 0.46 to 0.89, and only the two lowest fell below 0.56.

A relevance score cannot tell "about the right thing" from "states the answer"; only reading the passage can. So a weak line is reliable (no answerable item was flagged on either split), but the absence of one is not evidence that the answer is there. For near-miss questions the safeguards are the reader's: the server instructions tell clients to answer only what a passage states, and `brain_verify` marks a claim that its cited passages do not support.

**End to end, that reading works on this set.** `brain ask` (Claude Opus through Claude Code, the configured backend) was run once on the 16 held-out near-miss questions and the 24 held-out answerable ones:
- It declined all 16 near-miss questions, each time saying what the passages do state and that the detail asked for is not in the knowledge base. That includes the 14 whose search had judged its evidence strong.
- It answered all 24 answerable ones from the passages. Some answers add the model's own commentary (an inference or a suggestion), which `brain_verify` marks partial or uncited.

The answers, the evidence level of each search and the verifier's verdicts are in `eval/ask/2026-10-04-heldout.jsonl`. Whether each answer declined or answered was judged by an agent reading it (`judged_by`), and `scripts/eval-ask.ts` reruns it. This is one run of one model on 40 items of a fictional corpus, with model output that varies between runs: evidence that the safeguard works, not a rate to quote. It is not part of the gate, which makes no model call.

### Obsidian

```mermaid
flowchart LR
    save(["save"]) --> db[("database<br/>the retrieval layer")]
    db -- "debounced refresh<br/>a few seconds after<br/>the last save" --> vault["Obsidian vault<br/>Brain/ folder<br/>read-only mirror"]
    vault --> gv["Obsidian graph view"]
```

Every save writes twice: to the database, which is the retrieval layer, and to a read-only markdown mirror in your Obsidian vault, which is there for reading and for Obsidian's graph view. The vault and folder come from `OBSIDIAN_VAULT_PATH` and `OBSIDIAN_FOLDER`. The mirror only flows one way: edits made in Obsidian never reach the database.

- **When it refreshes.** Saves from a Claude chat (MCP) or the CLI refresh the mirror automatically. A document's note appears with its raw text as soon as the document is stored and chunked; its summary, entities, relationships and facts follow when enrichment finishes, usually a minute or two later. Refreshes are debounced (one run a few seconds after the last save in a burst) and never overlap. A failed refresh never fails the save; it is logged to stderr as `brain: obsidian refresh failed: ...`. CLI commands write the mirror before they exit, and the stdio MCP server does so when it shuts down.
- **Turning it on and off.** The automatic refresh is on whenever `OBSIDIAN_VAULT_PATH` names an existing directory. Turn it off with `OBSIDIAN_AUTO=0`. If the path does not exist, the server logs that once to stderr and leaves the refresh off.
- **Rebuilding on demand.** `npm run brain -- project-obsidian` rebuilds the whole mirror, for example after changing the folder or editing the database by hand. Override the vault and folder with `--vault <path>` and `--folder <name>`, keep it fresh on a timer with `--watch <minutes>` (runs never overlap, and a tick is skipped while the previous run is still going), or print the vaults Obsidian knows about with `--list-vaults`.

Layout inside the folder (`Brain` by default):

| Path | Contents |
|---|---|
| `Brain/<your name>.md` | The self node |
| `Brain/nodes/<type>/` | One note per entity, grouped by type |
| `Brain/documents/<year>/` | One note per document |
| `Brain/README.md` | Counts and graph view tips |

Only files whose frontmatter starts with `---` and contains `brain_managed: true` are ever rewritten or deleted. Symlinks are never followed. Your own notes placed in the folder are left alone and listed in the run output.

Graph view: filter `path:Brain`, then add color groups by `path:Brain/nodes/person`, `path:Brain/nodes/organization`, `path:Brain/documents`, or `tag:#brain/unverified`.

### Layout

See the file structure section of `docs/superpowers/plans/2026-09-27-knowledge-base-core.md`.
