---
layout: post
title: "RRF in Java or in the database? Anatomy of Lore's hybrid search"
date: 2026-07-25
tags: [rag, kotlin, spring-boot, spring-ai, chunking, embeddings, postgres]
---
*Part 4 of a series on building Lore, a local-first RAG system. [Part 1](/posts/introducing-lore/) introduced the project, [part 2](/posts/what-breaks/) covered ingesting real books, [part 3](/posts/lore-chunking-shootout/) measured three chunking strategies.*

After the chunking post went out, a friend I used to work with in my geospatial days — he works on Postgres search now — read it and asked me one question: "Are you doing the RRF in Java, or in database?"

I was using Java originally but by the end of writing this post I had swapped to Postgres, and working out whether that was the better option is what this blog became. I originally was just going to write about the hybrid approach I used. He also made an observation in the same chat that was a really valid point that I had read before but had ignored in the first version of the hybrid search: chunking makes vector search better, but often makes full-text search worse. That sounded testable, so I decided to test it for this post too.

Same methodology as the chunking post: instrument first, capture data against the current behavior, make changes, capture again. Everything below comes from those captures. And the biggest finding wasn't anything I went looking for — it was that the most expensive stage in my whole pipeline, the reranker, was making retrieval worse. Worse in three separate ways: it was returning fewer chunks than it was asked for on most queries, without reporting it; it was costing around ten seconds a call to do so, against a search that otherwise finishes in about fifty milliseconds; and when I finally scored it against a golden query set, retrieval with the reranker found the right chunk half as often as retrieval without it. The full data, and the two very different causes behind that last number, are towards the end.

## What hybrid search is

When you search in Lore, two completely different search engines run against the same chunks. The lexical leg is Postgres full-text search: stemmed keyword matching against an index of the actual words. The vector leg is [pgvector](https://github.com/pgvector/pgvector): your query gets embedded into the same 768-dimensional space as the chunks, and "relevant" means "nearby." Each leg returns a ranked list, the two lists get merged into one (that merge is the RRF my friend asked about), and the merged list is what you see — or, in the chat flow, what gets handed to the model as context. The whole post is about those three parts: each leg, the merge, and what happens after.

## The two things Postgres calls a vector

Before the legs, I want to clarify the naming confusion I had with vector, because it confused me at first and it's worth getting your head around. Lore's chunk table has two columns with "vector" in the name, and they have nothing to do with each other.

The column called `search_vector` is a [`tsvector`](https://www.postgresql.org/docs/current/datatype-textsearch.html), and a tsvector is not a vector in the embedding sense at all. It's a sorted list of stemmed words with their positions. Run `to_tsvector('english', 'Cooking meat can be intimidating')` and you get `'cook':1 'intimid':5 'meat':2` — the words, normalized down to their stems, with stop words dropped. It's an index of vocabulary, the same family of structure that Lucene builds. The "vector" in the name comes from the older, ordered-collection sense of the word — the same broad usage that gave search its classic term-weight vectors long before anyone was training embeddings. The embedding sense, a dense learned vector where distance means similarity, only became the default meaning of "vector" for most developers in the last decade or so. Same word, two eras, two completely different structures in one table. I almost ran up against this a while ago when I was doing a POC for a remote debugging agent and wanted to store the code in a datastore. In the end, I used an Abstract Syntax Tree for the source code which seemed to be a better fit for that project, so I accidentally avoided the issue and never really got the difference at the time since the implementation was different.

The embedding column is the numeric kind — a point in space, produced by the embedding model, where distance means semantic similarity.

You'll sometimes see lexical search described as "sparse vector search" in RAG writing, and there's a real idea behind that: imagine a vector with one dimension per word in the whole vocabulary, almost all zeros. It's legitimate information-retrieval framing. But in current usage "sparse" strongly implies BM25 or learned-sparse models, which is not what Postgres Full Text Search (FTS) is, so I avoid the term partly because I keep using it wrong anyway. Through this post: lexical means the tsvector leg, vector means the embedding leg.

## Leg one: lexical search

The lexical leg is three pieces of Postgres machinery. [`plainto_tsquery`](https://www.postgresql.org/docs/current/textsearch-controls.html#TEXTSEARCH-PARSING-QUERIES) turns your query text into a tsquery — stemmed terms with operators between them. The `@@` operator matches that tsquery against each chunk's tsvector. And [`ts_rank_cd`](https://www.postgresql.org/docs/current/textsearch-controls.html#TEXTSEARCH-RANKING) scores the matches, mostly by how often and how close together the terms appear.

An honest footnote before going further: this is not [BM25](https://en.wikipedia.org/wiki/Okapi_BM25). BM25 — the ranking function behind Lucene, Elasticsearch, and most serious search engines — weighs terms by how rare they are across the whole corpus and saturates repeated terms so the fiftieth occurrence counts less than the second. `ts_rank_cd` does neither; it only looks at the chunk in front of it. There are extensions that bring real BM25 into Postgres now ([pg_search](https://github.com/paradedb/paradedb), [pg_textsearch](https://github.com/timescale/pg_textsearch) — my friend works on the former, which is how this conversation started), and if Lore's corpus were bigger or the ranking mattered more I'd look hard at them. At my scale, with small chunks and rank-based fusion smoothing over score quality, plain FTS has been good enough. Particularly since I'm limiting the size of chunks I didn't think using BM25 was worth the extra complexity. The point of this post is the hybrid architecture, not the perfect lexical leg.

Now the interesting failure. In the chunking post's retrieval spot-check, one query returned zero lexical results across all three chunking strategies: *what is the difference between cooking with gas versus charcoal*. Not bad results — none. I promised an autopsy, and the explain endpoint I built for this post makes it a one-liner. Here's what `plainto_tsquery` actually produces for that query:

```
'differ' & 'cook' & 'gas' & 'versus' & 'charcoal'
```

The stop words ("what," "is," "the," "between," "with") are stripped, the survivors are stemmed — and then every one of them is ANDed together. All five terms must appear in a single chunk. Four of them are fine; grilling content is full of differences and cooking and gas and charcoal. The killer is `versus`. Nobody writing about grilling technique uses the word "versus." That one incidental word — a word that's about how I phrased the question, not about the topic — zeroes out the entire search. The failure isn't rare vocabulary. It's that AND semantics have zero tolerance for a single word of your phrasing not appearing in the text.

Postgres does have machinery aimed at this class of problem, and it's worth knowing it exists: [full-text search dictionaries](https://www.postgresql.org/docs/current/textsearch-dictionaries.html). You can add a synonym dictionary to the text search configuration that maps words onto other words — versus onto vs, barbecue onto bbq — or a thesaurus dictionary that handles whole phrases, and the mapping gets applied to queries and indexed text alike. I looked at it a while back when someone was asking about using AI for processing KYC forms, which were prone to input errors in names and data, and I was wondering about "cheaper" and more deterministic approaches to solving some of the heavy lifting rather than throwing things at AI and hoping for the best.

Every mapping is hand-curated configuration, and it only fixes the words you thought to list — my failing word was one I would never have predicted. The OR fallback later in this post fixes the entire class of failure without me maintaining a word list. If Lore ever narrows into a domain where the synonyms are actually knowable — cuts of beef and their regional names, say — a dictionary would be worth revisiting, because it fixes the problem at index time rather than by loosening the query.

For contrast, *how do I keep meat from drying out* — just as conversational — survives, because after stop-word stripping and stemming it reduces to `'keep' & 'meat' & 'dri'`, three words that genuinely appear in cooking writing, and 15 chunks satisfy all three. The line between "works fine" and "returns nothing" isn't natural language vs. keywords. It's whether your phrasing happens to include one word the author never used.

And the misspelling case is even simpler. I searched for `brisqet` — brisket with one transposed letter. Zero results, necessarily: FTS has no fuzzy matching at all. A human reads "brisqet" as "brisket" without noticing; Postgres searches for the literal lexeme `brisqet`, which does not exist in the book or the English language. The vector leg, meanwhile, shrugged and returned meat-adjacent chunks anyway — embeddings degrade gracefully where exact matching fails absolutely.

## Leg two: vector search

Shorter, because the earlier posts carry the background. The query gets embedded at search time with the same model that embedded the chunks at ingestion ([`nomic-embed-text`](https://ollama.com/library/nomic-embed-text), locally, always — that same-model requirement is the one-way door from the first post). pgvector's `<=>` operator finds the nearest chunk embeddings by cosine distance, and similarity is one minus that distance. Good at meaning across different wording; blind to exact strings, which is precisely what the lexical leg is for.

## Where each leg wins

I ran a fixed set of nine typed queries — exact recipe titles, jargon, paraphrases, natural questions, one deliberate misspelling — against both books through the explain endpoint, capturing each leg's top 10 and the fused top 10. Every fused result carries a tag: LEXICAL if only the lexical leg found it, EMBEDDING if only the vector leg did, BOTH if both did independently.

The distribution across all 90 fused top-10 slots:

```
found by            slots   share
vector leg only        54     60%
both legs              29     32%
lexical leg only        7      8%
```
At first glance, the lexical leg looks like it doesn't contribute much and could be dropped. The vector leg is the workhorse — it always returns a full candidate list, and for queries where phrasing and text share no vocabulary it's the only leg standing.

But the interesting column is BOTH, and it's not evenly spread. Two queries — `anchoring effect` and `regression to the mean` — scored 100% BOTH: every single one of their top-10 results was found independently by both legs. Those are established technical terms that Kahneman uses repeatedly and consistently around one concept, so keyword matching and semantic matching converge on the same chunks. At the other extreme, the gas-versus-charcoal and misspelling queries scored 0% BOTH, because the lexical leg had nothing to contribute. It is also important that all of these tests are based on a single book for each category. (If I had 200 books on BBQ and asked for a charcoal vs gas I probably would have more results just not from this book.)

Worth admitting at this point: while writing Lore initially, I built the lexical search and display first and did not use the LLM at all, and it was returning great results from my cookbooks that were very satisfactory. I implemented the embeddings and LLM chat after, so even though the embeddings seem better, the FTS was still great and probably good enough — and did not require an LLM at all. But Lore was largely about learning and over-engineering, so here we go.

That BOTH column is the whole argument for fusion, so let's look at the merge itself.

## Merging the two lists with RRF

Reciprocal Rank Fusion is a very simple idea, it comes from a short 2009 paper by Cormack, Clarke, and Büttcher (["Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods"](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)), which showed that this simple formula combined search results better than much fancier voting and machine-learning methods. It's become the default way to merge ranked lists in hybrid search, and Lore uses it straight out of the paper.

Here's the whole mechanism. Imagine two judges each hand you their top-ten list, and you want one combined list. Give every item points based on where it appeared on each list: an item at rank r earns 1/(k + r) points from that judge, with k = 60. So rank 1 is worth 1/61, about 0.0164 points; rank 2 is worth 1/62; rank 10 is worth 1/70; a gently flattening curve where higher ranks are worth more but never overwhelmingly more. An item's final score is its points from judge one plus its points from judge two — and being on *both* lists is the only way to score twice. That's the entire algorithm.

The k = 60 looks arbitrary, and honestly it mostly is — it's the constant that worked best in the original paper's experiments, and everyone has copied it since, me included. What it's *for* is softening the cliff between adjacent ranks. With no constant at all, rank 1 would be worth double rank 2 (1/1 versus 1/2), so topping one list would dominate everything. With k = 60, rank 1 beats rank 2 by about 1.6% — the curve rewards being near the top without letting any single placement crush the rest.

A real worked example from the captures, the query `Rooftop Ribs`. Chunk `2b67a42e` came back at position 1 in the lexical list and position 2 in the vector list. Its fused score is 1/61 + 1/62 = 0.0164 + 0.0161 = 0.0325. Chunk `882f2099` was lexical position 3 and vector position 1: 1/63 + 1/61 = 0.0323. So `2b67a42e` edges it — two decent placements beat one great and one middling. Any chunk found by both legs at reasonable ranks outscores a chunk that topped one leg and missed the other, which is exactly the behavior you want: independent corroboration wins.

Why ranks instead of scores? Because the two legs' scores are incomparable. `ts_rank_cd` gave me values from 0.0002 to 0.64 across the capture set; similarities sat between 0.47 and 0.75. There is no principled way to add those numbers. Ranks throw away the magnitudes — which costs something, a runaway winner and a marginal one are both just "rank 1" — but in exchange nothing needs calibrating, which for a system without labeled relevance data is the right trade.

The captures also handed me a subtlety I hadn't thought about. One query's top fused hit came back tagged LEXICAL, which looked like the lexical leg winning — until I looked closer. That query's lexical leg found exactly one chunk, which the vector leg didn't have anywhere in its top 50. The vector leg's own top chunk was completely different. Both chunks got the *identical* RRF score — rank 1 in one leg, absent from the other, 1/61 each. The tie was broken by insertion order in the Kotlin merge: lexical results went into the map first, so the lexical chunk sorted first. Not relevance. A LinkedHashMap artifact. Fusion ties happen whenever the two legs' top candidates are disjoint, and my tie-break was an accident of implementation. The SQL version below makes the tie-break explicit and deterministic — a small thing, but the kind of small thing you only find by looking at real output.

## Moving the fusion into the database

Back to my friend's question. The Java answer meant Lore's hybrid search was three round trips: one SQL query for the lexical leg, one for the vector leg, the Kotlin `fuse()` merge in memory, then a third query to hydrate the winning chunks with their content and document metadata. I don't actually know what was behind the question — it may have just been professional curiosity from someone who lives in Postgres — but it sent me looking at whether the whole pipeline could be one SQL statement. It can: two CTEs producing the ranked lists, a full outer join on chunk id, the 1/(k + rank) arithmetic right in the select, hydration joined into the same statement, one round trip. He was right to ask, because the exploring led me to a better design.

The refactor itself was easy — the RRF math translates to SQL almost line for line, with [`ROW_NUMBER()`](https://www.postgresql.org/docs/current/tutorial-window.html) supplying the ranks. The Kotlin `fuse()` function didn't die, though: it stays as the executable specification, and a parity test asserts the SQL fusion produces the same ordering and scores. When the refactor landed, the parity spot-check across three queries came back byte-identical — all 30 positions, chunk ids and RRF scores both.

Did it get faster? Honestly: a little. Here's the thing the before-instrumentation made obvious that I'd never actually measured. Per-stage medians across five queries, before the refactor:

```
stage             typical ms
embed the query      33 – 40
lexical SQL           0 – 7
vector SQL            8 – 10
fuse (Kotlin)         0
hydrate SQL           4 – 5
```

Embedding the query dominates — 60 to 70% of total latency on every single query. The three-round-trip structure I was replacing was never where the time went. After the refactor, the SQL side (everything except embedding) dropped 15–25% — real, consistent, and small: total search latency went from roughly 50–58ms to 50–64ms medians, with the difference mostly noise around the embedding call. I moved the fusion into the database because it's a better design — one round trip, one place where ranking logic lives, a deterministic tie-break, and the database doing set operations, which is what databases are for. I did not move it because it was slow, and I'd have known that earlier if I'd measured earlier. (One more honest number: the first embedding call after app start took 595ms — cold-start model loading. Every call after that, 33–45ms. Medians hide that; your first search each session doesn't.)

## Fixing the zero-result query

The autopsy said the disease was AND semantics, so the fix is a fallback. The lexical leg now tries [`websearch_to_tsquery`](https://www.postgresql.org/docs/current/textsearch-controls.html#TEXTSEARCH-PARSING-QUERIES) first — same behavior as before for plain queries, but it understands quoted phrases and OR syntax if you type them. If that returns nothing, it retries with the same stemmed terms ORed together: any term can match, and `ts_rank_cd` still ranks chunks with more of the terms higher. Behind a config flag, defaulting on.

Before and after for the gas-versus-charcoal query:

```
                         before                after
tsquery                  'differ' & 'cook'     'differ' | 'cook'
                         & 'gas' & 'versus'    | 'gas' | 'versus'
                         & 'charcoal'          | 'charcoal'
lexical results          0                     50
fused top-5 tags         all EMBEDDING         all BOTH
```

That last row is the actual payoff, and it's better than "lexical returns something now." The same chunks the vector leg had already put on top — the grilling-technique chunks — now get independently corroborated by the lexical leg, so RRF's both-legs bonus applies to this query for the first time. The fix didn't replace the vector leg's answer; it seconded it.

And the regression check: for the exact-term queries, results after the fix are identical to before, to six decimal places of RRF score. `websearch_to_tsquery` behaves exactly like `plainto_tsquery` on plain unquoted text, and the OR fallback only fires when the primary query returns nothing — so the fix, by construction, only changes behavior for queries that previously failed completely. The nicest kind of change to verify.

## Chunking and full-text search

His observation, roughly: full-text search at document level rewards concentration — a chapter that mentions "postgresql tuning" fifty times outranks a passing mention. Chunk the documents and that signal flattens; every chunk competes alone, and the fact that five of them come from one intensely-relevant chapter is invisible.

To test it I picked `regression to the mean` — Kahneman devotes a whole arc of chapter sections to it, and the term barely appears elsewhere. Then I compared what Lore actually returns (chunk-level lexical ranking) against a chapter-level aggregation of the same hits.

At chunk level, the top hits scatter across headings — "Understanding Regression" twice, "Talent and Luck," "A Defense of Extreme Predictions?", and the section literally titled "Regression to the Mean" ranks sixth, with a single matching chunk. Twenty-one matching chunks spread across eighteen distinct headings. Aggregate by chapter instead — count the hits and sum the ranks per heading — and "Understanding Regression" jumps clear of the field, 0.294 to the runner-up's 0.114, and the whole regression cluster snaps into focus as obviously the place to look.

So he's right: the concentration signal exists in the corpus and per-chunk ranking cannot see it. A reader asking "where should I go in this book for regression to the mean" is better served by the aggregated answer.

But — and I want to state both halves, because the data gave me both — for this query, the flattening didn't actually hurt retrieval. Six of the top seven chunk-level hits already come from the regression cluster; the neighborhood was found even though its density was invisible. And the *fused* top 10 was more concentrated still: seven of ten chunks from the core cluster, including one the lexical leg hadn't surfaced at all, pulled in by the vector leg. Fusion reinforced an already-decent lexical signal rather than rescuing a failure.

So where did I land on his claim? It holds — chunking really does destroy the "this chapter is dense with your topic" signal, and if Lore ever grows a "which chapter should I read" feature, it will need chapter-level aggregation, which the data model already supports. But I mostly dodged the damage, and not through foresight — through a constraint I already had. My small local chat model forced me into aggressive chunking, with a hard maximum size on every chunk no matter which strategy produced it. Small, capped chunks mean the lexical leg is ranking a lot of little competitors instead of a few giant ones, so no single chunk can hoard the term-frequency signal in the first place, and fusion stitches the topical cluster back together at the top of the results. If I'd been running big chunks, his point would have landed much harder. It's the same reason I could get away without BM25: the context-window constraint that made chunking painful in the last post quietly made lexical search more forgiving in this one.

## The reranker

To see why reranking exists at all, follow the numbers through the chat flow. Hybrid search returns up to fifty candidate chunks, roughly ordered by the fused score. The chat model's context window — its working memory, the thing the chunking post's budget arithmetic was all about — only has room for five of them alongside the system prompt, your question, and the answer it writes. So somewhere, fifty has to become five, and you'd like it to be the *best* five, not just the first five. That final, careful cut is what reranking is.

Why not just trust the fused order and take the top five? Because of how the vector leg judges relevance. Every chunk was embedded at ingestion time — before your question existed — so retrieval can only compare your question against each chunk's general-purpose summary of itself. It's like judging whether a book answers your question by reading its back-cover blurb. Fast, works in bulk, but lossy: the truly best chunk can easily sit at rank seven. The standard fix is a second model called a cross-encoder, which reads your question and one chunk *together* and scores how well they actually match — the difference between judging by the blurb and reading the chapter with your question in mind. Careful reading is slow, which is exactly why you only do it to a shortlist and not to the whole library. Wide cheap net first, careful judge second: that's the textbook two-stage design, and Lore's shape follows it.

Except for one problem: Ollama has no way to run a cross-encoder. There's no reranking endpoint in [its API](https://github.com/ollama/ollama/blob/main/docs/api.md). So Lore improvises with the only careful reader available locally — the chat model itself. The fifteen best fused candidates go into a single prompt as a numbered list, with an instruction: respond with ONLY a JSON array of passage numbers, most relevant first. Like handing a stack of fifteen photocopied pages to an assistant and asking for the numbers of the five most useful ones, in order.

The code that reads the model's answer assumes, correctly, that the model won't reliably follow the format — it pulls out every integer in the response in order of appearance, ignores everything else, and if nothing usable comes back at all, falls back to the original fused order. Defensive, sensible, and it turns out the defense has a hole.

Across the capture runs, the model *always* returned parseable indices — the empty-response fallback never fired once. What it didn't reliably do is return *enough* of them. Asked to rank 15 passages, it answered with lists of 3, 2, 6, 5, and 3. Only one response in five had at least the 5 entries needed. And here's the hole: a 3-element parsed order isn't empty, so the fallback doesn't trigger; the code takes what it got, and the chat model receives 3 chunks of grounding instead of 5. Silently. Nothing in the response distinguishes "the reranker judged only 3 passages relevant" from "the model got bored." I confirmed it in the actual rendered chat page: three sources listed where there should be five. The fallback-rate metric I'd planned to report — zero, a clean bill — was true and completely misleading. The metric that matters is what I've started calling the shrinkage rate: responses shorter than requested. Four out of five.

The first post in this series talked about fluent-but-wrong answers as the failure mode I worry about most in RAG. This is the same kind of problem in the plumbing. The reranker failed partially, the output looked normal, and my safeguard never fired. I had written it for the wrong failure.

Then there's the cost. The rerank call took 7.5 to 12 *seconds* per query. Not milliseconds. The entire hybrid search — both legs, fusion, hydration — takes about 55ms; the reranker multiplies the pre-generation latency by well over a hundred. That would be defensible if the judgment were good. So is it?

## The golden set

The retrieval check in the chunking post was six queries. I called it a spot-check and promised a proper eval later. This is it, in a small form.

Twenty queries across both books. For each one I picked the target chunk before running anything, by searching the raw content for a distinctive phrase — recipe titles for the cookbook, concept headings for Kahneman. The measure is [recall](https://en.wikipedia.org/wiki/Precision_and_recall)@5: is the target chunk in the top five results? I ran five configurations: lexical with the old AND-only behavior, lexical with the new fallback, vector only, hybrid, and hybrid with reranking.

```
configuration            recall@5
lexical (old, AND-only)     40%
lexical (new fallback)      45%
vector only                 55%
hybrid                      60%
hybrid + rerank             30%
```

Three things stand out.

First, the expected one: hybrid beats both of its own legs. Not dramatically — 60 against 55 and 45 — but consistently, and the per-query detail shows why: the legs fail on *different* queries. Vector search missed `anchoring effect`, `availability heuristic`, and `Two Systems` — exact terms the lexical leg nailed. Lexical missed the paraphrases and natural questions the vector leg handled. Fusion's margin is exactly the union of two different competences.

Second, the fix earned its keep, barely and precisely: the new fallback added one hit — the gas-versus-charcoal query, the exact query it was built for — and lost nothing. A small, clean, honest win.

Third: reranking made things worse. Half the recall of plain hybrid. The most expensive stage in the pipeline, the one adding ten seconds per question, actively hurt retrieval quality on this data. I want to be precise about why, because "reranking bad" is not the finding. Of the six queries where hybrid had the target in its top 5 and reranking lost it, three were the shrinkage bug — the target was sitting in the candidate list and the model returned a 3-element ranking that didn't include it. Those are a parsing-and-prompting bug wearing a relevance costume. But the other three were full five-element rankings where the model looked at the candidates and actively ranked the target out. Same failure in the scoreboard, completely different disease: one is fixable with "return exactly 5 indices, pad if unsure" and validation; the other is a question about whether an 8-billion-parameter model is a competent relevance judge at all, and on this evidence the answer is not yet, not at this size.

Collapsing those into one number would misattribute half the damage. Keeping them apart is the whole reason the golden set exists.

Worth saying plainly: the absolute numbers are modest across the board. Six of the twenty queries — the hard paraphrases, the misspelling, several bare concept terms like `loss aversion` and `planning fallacy` — weren't found by *any* configuration. Twenty queries against two books with hand-picked targets is a relative comparison tool, not an IR benchmark, and what it's for is ranking my own configurations, which it did decisively.

So the reranker is off by default now. The honest current state: hybrid retrieval straight into the context window, 60% recall@5, ~55ms. The reranking idea isn't dead — the shrinkage bug is fixable, and a real cross-encoder as a small sidecar ([bge-reranker](https://huggingface.co/BAAI/bge-reranker-v2-m3) and friends run fine on modest hardware) is the version worth building, because it attacks the actual weakness, judge quality, instead of the location. That's a future post if the data turns out interesting. [PostgresML](https://postgresml.org/) would even put a cross-encoder inside Postgres itself, which would make my friend's question recursive — but adopting a whole ML-in-database platform for a personal tool is a bit too far considering how over-engineered this already is. 

I might revisit this as claude only toggle or with the bge-reranker which I haven't really looked too deeply into yet. Pivoting to using Hypothetical Document Embeddings (HyDE) might have a bigger impact and I really need to do this kind of testing with a much larger doc set. Single documents are good for catching missing hits that should be found but searching with a lot of results should uncover other bugs. All of my measurements are based on my own laptop which was running multiple apps including IntelliJ at the same time on a dedicated workstation with more memory this should be a lot faster. I also didn't test with alternative models locally or remotely as I was trying to keep that as a constant.

## What changed

Same closing ledger as the chunking post, because the pattern held. The instrumentation existed for the blog; the blog was supposed to describe the system; instead the data changed it. The fusion moved into the database and picked up a deterministic tie-break on the way. The lexical leg grew an OR fallback that turned a dead query into a corroborated one. The reranker — the component I expected to describe in a flattering paragraph about pragmatic workarounds — got measured, caught silently shrinking the context on most queries, benchmarked at half the recall of doing nothing, and switched off. And the latency numbers relieved me of a belief I'd never examined: the round trips were never the cost; the embedding call is, and now I know by how much.

My friend's question has a proper answer now. The fusion belongs in the database — it's set arithmetic over ranked lists, and the database does it in one statement with less machinery than my three round trips. The judgment doesn't belong anywhere yet: not in Java, where I had it, and not in the database either, until there's a judge worth hosting. Knowing which is which cost me an explain endpoint, twenty golden queries, and the willingness to find out that a thing I built made things worse.

Next post: [TESTING — the suite that catches all of this without mocking the LLM.]

---

## Sources and further reading

Postgres full-text search: [text search types (`tsvector`/`tsquery`)](https://www.postgresql.org/docs/current/datatype-textsearch.html), [controlling text search — query parsing, ranking, and headlines](https://www.postgresql.org/docs/current/textsearch-controls.html), [dictionaries, synonyms, and thesauruses](https://www.postgresql.org/docs/current/textsearch-dictionaries.html), and the [window functions tutorial](https://www.postgresql.org/docs/current/tutorial-window.html) behind the `ROW_NUMBER()` ranking.

Vector search: [pgvector](https://github.com/pgvector/pgvector) and the [nomic-embed-text model card](https://ollama.com/library/nomic-embed-text) on Ollama.

Ranking and fusion: Cormack, Clarke & Büttcher, ["Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" (SIGIR 2009, PDF)](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf); [Okapi BM25](https://en.wikipedia.org/wiki/Okapi_BM25) for what `ts_rank_cd` isn't; [precision and recall](https://en.wikipedia.org/wiki/Precision_and_recall) for the eval metric.

BM25-in-Postgres extensions mentioned: [pg_search (ParadeDB)](https://github.com/paradedb/paradedb) and [pg_textsearch (Tiger Data)](https://github.com/timescale/pg_textsearch).

Reranking: the [Ollama API docs](https://github.com/ollama/ollama/blob/main/docs/api.md) (note the absence of a rerank endpoint), the [bge-reranker-v2-m3 model card](https://huggingface.co/BAAI/bge-reranker-v2-m3) as the cross-encoder I'd reach for, and [PostgresML](https://postgresml.org/) for the in-database version.

---

*Lore is open source at [github.com/walterdeane/lore](https://github.com/walterdeane/lore).*