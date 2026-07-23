---
layout: post
title: "The Chunking Strategy Shootout: TOKEN vs. STRUCTURAL vs. SEMANTIC"
date: 2026-07-23
tags: [rag, kotlin, spring-boot, spring-ai, chunking, embeddings, postgres]
---

*This is the third post in a series about [Lore](https://github.com/walterdeane/lore), a local-first RAG system for personal documents. Earlier posts: [Introducing Lore](/posts/introducing-lore/) and [What Breaks When You Ingest Real Books](/posts/what-breaks/).*

I first thought of writing Lore last year, when GenAI really took off and it looked like a great way to solve my cookbook problem. It got sidelined — work was busy, and we were getting evicted so the landlord could renovate and jack up the rent. My previous employer was also slow to adopt meaningful AI work beyond a bit of agentic coding, so I couldn't get any agentic projects approved there. Then I was made redundant along with a bunch of others, and shortly after that I bombed an interview badly enough to realize how little I actually knew about AI outside of AWS's built-in tools. That was the push. I came back to Lore as a way to learn the full stack properly, with a real project I'd actually use at home every day.

Learning RAG was the driver, and when I started reading about it seriously, one thing stood out: the chunking strategy was going to have a much bigger impact than the querying techniques. Garbage in, garbage out was going to be the main issue given the constraints I'd set for myself. I wanted the whole system to be as green and as cheap as possible — everything runs locally through Ollama. Lore does let you use Claude as the chat model, but even then your embeddings are all done locally; Claude is only there as an option for someone who wants better answers at query time, while ingestion can stay slow and local.

I also didn't want this to be purely cookbook-centric, even though cookbooks were the original driver. I have books on woodworking, boatbuilding, permaculture, blacksmithing. That introduced a complexity: cookbooks tend to share similar presentation formats — similar headers, similar layouts — but other books are completely different. Some are large chapters of continuous text, some have frequent headers and subheaders. And I have both EPUBs and PDFs. EPUB isn't terrible to work with — it's HTML internally. PDF was designed for printing and was horrible to process; it's purely positional, with no semantic or structural tagging. I decided early that I'd need multiple chunking strategies for the different domains and documents. In the current version the strategy is user-selectable per document; I might add a batch agentic feature later where an LLM picks the right one after examining the doc.

A few more deliberate restrictions shaped what you'll see below. I limited chunk size so the context window wouldn't blow out — this runs locally and I have to work within Ollama's limits. The current version is a single request-and-response design, because my use case is finding a direct answer; it doesn't handle conversations yet (I might add that later, if only to learn the memory management). I wanted chunks to be roughly the same size so the vectors would be comparable in granularity — results from different documents should be roughly equivalent. And I wanted the whole thing searchable as a pure ranked search engine without the LLM, so each chunk gets indexed two ways: a full-text index for keyword search, and a dense embedding for semantic search. That's what supports the hybrid search and reranking later in the pipeline.

A naming warning here, because it caught me out while writing this. In the schema, the full-text index lives in a column called `search_vector` — but it isn't a vector in the embedding sense at all. Postgres's `tsvector` type is a list of stemmed words and their positions ('cook', 'meat', 'intimid'), basically a keyword index that happens to have "vector" in its name for reasons that predate the ML meaning of the word by decades. The embedding column is the numeric kind of vector, the one that lives in a 768-dimensional space. Two columns with "vector" in the name, and they have nothing to do with each other. So through this post: when I say full-text or keyword search, I mean the stemmed-word index; when I say embedding or semantic search, I mean the numeric vectors. The next post digs into both kinds of search properly.

So: three chunking strategies, user-selectable, and a design that leans on chunk quality. Which raises the obvious question — does the choice of strategy actually matter? Chunking gets treated as a one-line config choice in most RAG writeups ("we used 512-token chunks with 50-token overlap") and nobody shows their work. I wanted to know, with data, on my actual books.

The example that convinced me it matters is a chapter transition in *Thinking, Fast and Slow*. The chapter "Norms, Surprises, and Causes" ends with a set of closing pull-quotes, and "A Machine for Jumping to Conclusions" opens with an anecdote about the comedian Danny Kaye. Same book, same transition, three strategies:

```
                      "Norms, Surprises, and Causes" ends │ "A Machine for Jumping..." begins
                                                          │
TOKEN      chunk 58    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┿━━━━━━━━━━━━━━━━━━━━━━━━━━━
SEMANTIC   chunk 78    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┿━━━━━━━━━━━━━━━━━━━
STRUCTURAL chunk 63                                       ┝━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                                          │
                                                   chapter boundary
```

TOKEN fused the end of one chapter with the opening of the next — one chunk spanning two unrelated chapters. SEMANTIC did the same thing, which surprised me: its breakpoint detection saw no big semantic jump at the boundary (both sides are System-1-adjacent content), so the literal `## A Machine for Jumping to Conclusions` heading marker ends up sitting mid-chunk. STRUCTURAL was the only one that started a fresh chunk exactly at the heading.

This shows up in retrieval, not just in the chunk table. I asked the hybrid search endpoint *why we need causal stories to explain surprising events* — content that sits right at that transition. STRUCTURAL returned the relevant passage at rank 1. TOKEN and SEMANTIC both returned a tangential match about regression to the mean.

The rest of this post runs all three strategies over two of my actual books — *The Meat Hook Meat Book* (a professionally typeset cookbook PDF) and *Thinking, Fast and Slow* (a long-form nonfiction EPUB) — and looks at what the numbers say. The short version: the interesting finding is not "STRUCTURAL wins."

## Why chunking matters more than it looks like it should

A chunk is the unit of retrieval, so every boundary decision has consequences. Get a boundary wrong in one direction and a fact gets split across two chunks — the retriever surfaces half an answer. Get it wrong in the other direction and unrelated content gets merged into one chunk, which drags its embedding toward the average of two topics so it matches neither of them well.

The three strategies make different assumptions about where good boundaries come from. TOKEN cuts on arithmetic: every N tokens, with some overlap so nothing is severed too badly. STRUCTURAL trusts the author: split where the headings are, because the person who wrote the book already divided it into coherent units. SEMANTIC watches the content: embed the text, look for the places where the meaning shifts, cut there.

## TOKEN — the baseline

`TokenTextSplitter`, fixed-size chunks, configurable overlap (`token-overlap-chars: 200` by default). It's the cheapest and simplest of the three, and before this experiment I would have described it the way the outline for this post did: "no parsing dependency, nothing to go wrong."

That turned out to be backwards. When I ingested *Thinking, Fast and Slow* for this comparison, TOKEN was the only strategy that failed outright — zero chunks, status FAILED. If you read [the last post](/posts/what-breaks/) you already know the culprit: the same `<divlity>` tag that opened that post. TOKEN fed the file to Tika, Tika's strict SAX parser threw, and there was no fallback behind it. STRUCTURAL and SEMANTIC ingested the identical file without a hiccup, because they use their own lenient Jsoup-based parsers and only touch Tika as a last resort. The "simple" strategy actually had the hardest parsing dependency of the three. (It has a fallback now — TOKEN degrades to the markdown parsers when Tika throws.)

## STRUCTURAL — heading-aware, mostly in name

STRUCTURAL parses the source to markdown, splits on heading markers, and has GENERIC/COOKBOOK/ACADEMIC variants for different document shapes. For PDFs it prefers the document's embedded outline over font-size guessing — [the last post](/posts/what-breaks/) has that story. When a heading section exceeds a size cap, a token-splitter fallback splits it further.

Now the finding that surprised me most in the whole experiment. Here's STRUCTURAL's chunk-size distribution next to TOKEN's on the cookbook:

```
strategy         n    mean    p10    p25    p50    p75    p90    p95    p99    max
STRUCTURAL     223   621.6    596    626    637    644    649    653    699   1750
TOKEN          259   642.5    623    632    639    648    676    691    700    705
```

They're nearly identical — same tight band, same shape. That's not what a heading-aware strategy should look like, so I traced it. Of STRUCTURAL's 223 chunks, exactly 5 came from headings whose sections fit inside the size cap — Cover, Title, Copyright, Resources, Acknowledgments. The other 218, which is 98%, were produced by the token-splitter fallback, because every real content chapter (Beef, Pork, Cooking Meat, and so on) exceeded the cap. The PDF's outline is chapter-level only, and the chapters run 20–45K characters.

My first thought was that this was a coarse cookbook and a book with more headings would behave differently. The second book was the test: *Thinking, Fast and Slow* has 47 sections to the cookbook's 20. The result was worse, not better — 3 natural single-chunk sections, 94% of sections needing the fallback, 99.3% of chunks coming from it. More headings didn't help, because Kahneman's chapters are long-form essays that individually still exceed the cap. For ordinary full-length books, I now think this is the default outcome rather than a quirk of one book.

So on this data, STRUCTURAL is TOKEN chunking with a heading label prepended, 98–99% of the time. Which raises the question of how it still won the chapter-transition example at the top of this post. The answer is that the fallback only ever splits *within* a heading section, never across one. However ordinary its chunks look statistically, a STRUCTURAL chunk can never span two chapters — that holds by construction. It's the one property TOKEN can't offer, and the chapter transition is exactly where TOKEN and SEMANTIC both failed.

## SEMANTIC — embedding-similarity breakpoints

SEMANTIC works by sliding a window over paragraphs, embedding each window, measuring cosine distance between neighbors, and cutting where the distance spikes past a percentile threshold. Oversized chunks get the token-splitter fallback as well (`maxChunkChars`, 4000 chars).

The percentile being *per-document* is the part worth understanding. A fixed distance cutoff assumes all books are equally choppy, and they aren't — a recipe collection lurches from topic to topic while a Kahneman chapter glides. Ranking each document's own boundary candidates and cutting at the top 15% adapts the sensitivity to the document. The defaults (`windowSize=2`, `breakpointPercentile=0.85`, `maxChunkChars=4000`) came from sweeping them against the cookbook.

The distributions show SEMANTIC genuinely doing something different from the other two:

```
                        cookbook                          Thinking, Fast and Slow
strategy         n     p50    under-200-tok         n     p50    under-200-tok
SEMANTIC       469     192       49%              556     551       22.5%
STRUCTURAL     223     637       ~0%              427     618        2.3%
TOKEN          259     639        0%              407     617        0.0%
```

Half the cookbook's SEMANTIC chunks are under 200 tokens — small, topically tight units cut at genuine content shifts rather than at a ceiling. It's the only strategy whose sizes are governed by the content instead of the configuration. I'll note this cuts against one of my own design goals from the intro — roughly uniform chunks so results across documents are comparable. SEMANTIC trades that uniformity for topical tightness, and I'm still deciding how I feel about the trade.

The percentile does need tuning, though, and getting it wrong does real damage. The outline for this post said to show the tuning rather than assert it, so: I re-ingested the cookbook with `breakpointPercentile=0.95` — a plausible-looking value, "only cut at the top 5% of shifts" — into a throwaway domain. Next to the default:

```
strategy             n    p10    p25    p50    p75    p90
SEMANTIC p0.95     282    113    244    497    595    694
SEMANTIC p0.85     469     94    120    192    348    591
```

Median chunk size nearly triples, and the under-200-token share drops from 49% to 20%. Same book, same pipeline, one config value changed — p=0.95 systematically merges content that p=0.85 keeps apart. The concrete example: one p=0.95 chunk contains the tail of one recipe ("The Inevitable Pork Chop with Cheddar Grits"), the entirety of a second ("Rooftop Ribs"), and the headnote of a third ("Chinese Barbecue Pork"). Three recipes in one chunk, one embedding pulled toward the average of all of them. At p=0.85 the same span falls into five cleanly bounded chunks, each recipe starting fresh.

One wrinkle worth being honest about. My original tuning notes describe p=0.95 producing a single 24K-character monster chunk. It doesn't anymore — the size-cap fallback now catches it, so the *largest* chunk at p=0.95 looks perfectly healthy (3,993 chars, just under the cap). The failure didn't go away; it went quiet. A badly tuned percentile no longer produces an obviously broken artifact you'd trip over — it shifts the whole distribution instead, which you only notice if you look at the quantiles rather than the max. That's good for production and bad for noticing you need to tune.

## Head-to-head on both books

Chunk counts and token quantiles (tokens counted with `cl100k_base`, the same tokenizer `TokenTextSplitter` uses):

```
The Meat Hook Meat Book (PDF, COOKBOOK variant for STRUCTURAL)
strategy         n    mean    p10    p50    p90    p99    max
SEMANTIC       469   274.0     94    192    591    933   1744
STRUCTURAL     223   621.6    596    637    649    699   1750
TOKEN          259   642.5    623    639    676    700    705

Thinking, Fast and Slow (EPUB, GENERIC variant)
strategy         n    mean    p10    p50    p90    p99    max
SEMANTIC       556   427.1    103    551    598    778    854
STRUCTURAL     427   588.5    528    618    640    653    795
TOKEN          407   614.8    595    617    635    651    659
```

The retrieval spot-check: three real queries per book against each strategy's chunks, through the production hybrid endpoint. Cookbook first:

| Query | TOKEN | STRUCTURAL | SEMANTIC |
|---|---|---|---|
| how long to braise a beef shank | hit, rank 1 | hit, rank 1 | hit, rank 1 |
| gas vs. charcoal difference | no real hit | partial — top hits contrast both fuels | partial — both fuels named, generically |
| kosher salt | hit, rank 1 | hit, rank 1 | hit, rank 1 |

And *Thinking, Fast and Slow*:

| Query | TOKEN | STRUCTURAL | SEMANTIC |
|---|---|---|---|
| what does WYSIATI stand for | hit, rank 1 | hit, rank 1 | hit, rank 1 |
| why we need causal stories for surprises | miss — tangential | **hit, rank 1** | miss — tangential |
| anchoring effect | hit, rank 1 | hit, rank 1 | hit, rank 1 |

Reading this honestly: four of the six queries are clean three-way sweeps. That's a useful sanity check — retrieval works regardless of strategy for easy questions — but it doesn't discriminate between them. The two hard queries are both boundary-spanning questions, and that's where the strategies diverge: nobody cleanly wins gas-vs-charcoal (the failure modes just differ), and STRUCTURAL alone wins the causal-stories query, which is the retrieval consequence of the chapter boundary in the opening exhibit.

For balance, the exhibit I expected to lead this post with went the other way. On the cookbook's high-heat-versus-low-heat cooking guidance — a two-part instruction pivoting on the word "Conversely" — TOKEN cut exactly at the pivot, so one chunk ends mid-argument on the high-heat half. But TOKEN's 200-character overlap meant the following chunk carried both halves anyway, and STRUCTURAL's save turned out to be luck: its winning chunk was a fallback piece whose boundary happened to land a few hundred characters later. SEMANTIC was the only one that kept the whole unit together deliberately — its chunk spans the entire how-to-think-about-cooking-meat run and ends where the topic genuinely shifts. Three strategies survived the same passage for three different reasons, and only one of those reasons was on purpose.

## The finding I wasn't looking for: the context budget

While tracing why STRUCTURAL's sizes mirrored TOKEN's, I pulled a thread that had nothing to do with chunking strategy. The fallback splitter — and TOKEN itself — used Spring AI's `TokenTextSplitter` default chunk size of 800 tokens. Nothing in Lore had ever chosen that number. It was inherited from the library.

Meanwhile, at the other end of the pipeline, Lore's chat view puts 5 retrieved chunks into a single prompt for the local model. And `llama3.1:8b`, despite supporting 128K context on paper, actually loads in Ollama with a 4,096-token window, because Lore never overrides `numCtx`. I found this by checking Ollama's API for what was actually loaded rather than trusting the model card — worth doing, as it turns out.

Add it up: 5 chunks × 800 tokens is 4,000 tokens of context, over budget before the system prompt, the question, or a single token of the answer. Chunks were being silently truncated on every chat turn, and I hadn't noticed, because the answers still read fine. Post 00 called fluent-but-wrong the nastiest failure mode in RAG; this was my own concrete case of it, in my own app, found by accident. The fix was an explicit `tokenChunkSize` (default 600) applied everywhere `TokenTextSplitter` gets used, which puts a 5-chunk turn at roughly 3,000–3,400 tokens with real headroom for everything else.

The general lesson is about defaults whose consequences live somewhere else. Chunk size looks like an ingestion setting, but its real constraint sits two subsystems away in the chat prompt budget. The library default wasn't wrong — it was unexamined against how the app actually uses chunks.

## A note on the data

Gathering the boundary exhibits surfaced two real parser bugs — two-column PDF layouts getting zippered together line by line, and then my first fix for that bisecting full-width prose that had been correct before — and the context-budget discovery forced the chunk-size change. All three fixes landed before the final data pass, so every number in this post is post-fix. Which means the first dataset I collected was quietly measuring parser artifacts as if they were chunking behavior. If I'd drafted this post from that first pass, I would have confidently blamed chunking for bugs that lived in the parser.

## One more thread: bold paragraphs that were never headings

All the numbers above were already final when a question about this very section stopped me: *Thinking, Fast and Slow* has recurring "Speaking of System 1 and System 2" boxes at the end of several early chapters — short recap sections with a clear visual subhead. Print readers see them as structure. I'd assumed STRUCTURAL saw them the same way; it's part of why I picked this book for the series. It hadn't split on a single one of them.

I opened the raw EPUB XHTML to find out why, and found this:

```html
<p class="calibre25"><span class="bold1">Speaking of System 1 and System 2</span></p>
```

A `<p>`, not an `<h#>`. The subhead's visual weight comes entirely from a CSS class — `.bold1 { font-weight: bold }`, defined in the chapter's stylesheet — that this EPUB's conversion pipeline applied instead of a semantic heading tag. `EpubMarkdownParser` only ever looked at tag names (`h1`–`h6` become `#`–`######`; everything else is body text), so a bold paragraph was invisible to it no matter how heading-like it looked rendered. Every "Speaking of..." box in the book, plus the "Two Systems" part-title that opens the chapter, had been quietly flattened into surrounding body text.

The fix: `EpubMarkdownParser` now reads each chapter's linked stylesheet, resolves which CSS classes render `font-weight: bold`, and promotes a `<p>` to a `##` heading when its entire visible text is wrapped by a bold element (tag or resolved class) and it's shaped like a title — short, no sentence-ending punctuation. Same idea as the PDF font-size fallback from [the last post](/posts/what-breaks/): when a format doesn't state its structure explicitly, infer it carefully, and only as a fallback.

Rerunning STRUCTURAL on the same book with the fix in place, through the same splitter that produced the numbers above (character counts this time, not the `cl100k_base` token pass — I didn't have that harness wired up for this follow-up):

```
                                            before        after
n                                            427           510
mean chars                                  2880          2368
p10                                         2361           675
p50                                         3055          2802
p90                                         3290          3208
p99                                         3442          3839
max                                         3491          3926
chunks from the oversized-chunk fallback  424 (99.3%)   370 (73%)
```

The fallback dependency I flagged above — STRUCTURAL degrading to TOKEN's chunking 99% of the time — drops to 73%. Still the majority, but before I called this fixed I wanted to know what was actually driving that drop, not just trust the percentage.

It's not one box. "Speaking of System 1 and System 2" turned out to be one instance of a running feature: every chapter in this book ends with one of these, and the fix catches all 36 of them — "Speaking of Priming," "Speaking of Anchors," "Speaking of Losses," "Speaking of Regression to Mediocrity," all the way to "Speaking of Thinking About Life" in the closing chapter. Every one of them was invisible to STRUCTURAL before this fix; every one now gets its own chunk.

And the fallback percentage actually understates the effect, because it's a ratio over a growing denominator — more total chunks dilutes it even before you ask whether anything got better. The number I trust more: chunks that never touch the oversized-chunk fallback at all — genuinely heading-bounded, no arithmetic splitting — went from **3 to 140**. Three natural chunks in the entire book, to 140.

Is a natural chunk actually better, or just different? I pulled the text. The new `## Speaking of System 1 and System 2` chunk, in full:

```
## Speaking of System 1 and System 2

"He had an impression, but some of his impressions are illusions."
"This was a pure System 1 response. She reacted to the threat before she recognized it."
"This is your System 1 talking. Slow down and let your System 2 take control."
```

Three sentences, one purpose, nothing else — a clean retrieval unit. Compare that to the section it used to live inside, `## Two Systems` (8,700 characters), which still needs the fallback and still comes out as 7 arbitrary slices of about 3,000 characters each. One slice ends mid-sentence on "...it has also learned skills such as reading and understanding nuances of social situations," and the next opens by repeating that same fragment before continuing — the 200-character overlap doing its job of softening a cut that has no relationship to the content on either side of it. That's not a flaw specific to this slice; it's `TokenTextSplitter` doing exactly what it does everywhere else in this post. The difference is that a natural chunk doesn't need it.

The honest ceiling: 73% is still the majority, and it isn't going lower without more author-marked structure to exploit. The prose between headings can be long even after this fix — the section titled `## Mental Effort`, between "Attention and Effort" and its own "Speaking of..." box, runs 17,746 characters on its own, an order of magnitude past the 4,000-character cap. There's no further subhead in there for the parser to find, bolded or otherwise; Kahneman just writes a long section sometimes. The fix recovers exactly the structure the author actually marked — chapter, named subsection, recap box — and stops there. It can't invent boundaries inside an unbroken argument that has none.

Three more honest caveats. First, this only helps because this EPUB's converter happened to encode its subheads consistently as bold-only paragraphs — a book that bolds things for emphasis *inside* a sentence, or uses italics or small caps for its subheads instead, won't be caught by this at all, and a converter careless enough to drop heading tags might have dropped other structure too. Second, I checked the full heading list for false positives — places where the detector might have split a continuous paragraph run that just happened to contain a short bold phrase — and found none; every promotion was a real "Speaking of..." box. Third, I haven't re-run the retrieval spot-check against these new chunk boundaries — that's the next thing to check before this fix ships, not something to assume from chunk boundaries alone. I picked *Thinking, Fast and Slow* assuming its visible subheads would matter to STRUCTURAL. They didn't, until I checked what the ebook actually encoded instead of what the page renders — and it took a second pass, past the summary percentage, to find out how much of the book that blind spot actually covered.

## When to reach for which

What the data actually supports:

**TOKEN** is a reasonable default. Overlap papers over some of its bad cuts, and for easy queries it retrieves as well as anything. But simple didn't mean robust here — it was the only strategy to fail an ingest — and its blindness to structure is exactly what boundary-spanning queries punish.

**STRUCTURAL** is the right choice more often than its own statistics suggest. Expect it to behave like TOKEN for the bulk of any full-length book, because real chapters exceed any sensible size cap. But its never-spans-sections guarantee is cheap insurance that pays out precisely on the queries the other two lose. Check your document's heading granularity before expecting more from it than that.

**SEMANTIC** is the only strategy that adapts chunk sizes to the content, and on the cookbook it shows. It's also the only one with a genuinely dangerous tuning knob, and its failure mode is now quiet — watch the distribution, not the max. And it shares TOKEN's disregard for hard section boundaries: semantic continuity across a chapter break is real continuity as far as the embedding is concerned, even when the author disagrees.

None of these is simply better. They fail in different places, and on this data the failures line up with what each strategy ignores: TOKEN ignores structure, SEMANTIC ignores the author, STRUCTURAL ignores content size.

## Closing: per-document tuning, not a solved problem

Lore resolves a chunking strategy per document (`chunkingStrategyResolver`), with STRUCTURAL variants per document shape and SEMANTIC defaults tuned against one cookbook. That's hand-tuning, and I'm calling it that. The open question is whether resolution should get smarter — more variants, per-corpus threshold sweeps, maybe an agentic pass where an LLM examines the document and picks — or whether hand-tuned defaults plus honest measurement is the right stopping point for a personal tool.

One more thing from the spot-check that I'm saving for next time. I also ran those six queries through the lexical-only endpoint, and one conversational query returned zero results across all three strategies — while hybrid search degraded gracefully on the exact same question. Why keyword search fails completely where hybrid merely wobbles, and how Lore fuses the two legs so each covers the other's blind spots, is the next post.

*Next in the series: [The hybrid search architecture]([LINK]).*
