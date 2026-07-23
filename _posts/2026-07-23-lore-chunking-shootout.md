---
layout: post
title: "The Chunking Strategy Shootout: TOKEN vs. STRUCTURAL vs. SEMANTIC"
date: 2026-07-23
tags: [rag, kotlin, spring-boot, spring-ai, chunking, embeddings, postgres]
---

*This is the third post in a series about [Lore](https://github.com/walterdeane/lore), a local-first RAG system for personal documents. Earlier posts: [Introducing Lore]([LINK]) and [What Breaks When You Ingest Real Books]([LINK]).*

I first thought of writing Lore last year, when GenAI really took off and it looked like a great way to solve my cookbook problem. It got sidelined — work was busy, and we were getting evicted so the landlord could renovate and jack up the rent. My previous employer was also slow to adopt meaningful AI work beyond a bit of agentic coding, so I couldn't get any agentic projects approved there. Then I was made redundant along with a bunch of others, and shortly after that I bombed an interview badly enough to realize how little I actually knew about AI outside of AWS's built-in tools. That was the push. I came back to Lore as a way to learn the full stack properly, with a real project I'd actually use at home every day.

Learning RAG was the driver, and when I started reading about it seriously, one thing stood out: the chunking strategy was going to have a much bigger impact than the querying techniques. Garbage in, garbage out was going to be the main issue given the constraints I'd set for myself — everything runs locally through Ollama, as green and as cheap as possible ([the first post]([LINK]) covers those choices, and the optional Claude escape hatch for chat).

I also didn't want this to be purely cookbook-centric, even though cookbooks were the original driver. I have books on woodworking, boatbuilding, permaculture, blacksmithing — and books vary a lot. Some are large chapters of continuous text, some have frequent headers and subheaders. Some are EPUBs, which aren't too bad to work with (it's HTML internally), and some are PDFs, which were designed for printing and are horrible to process — purely positional, no structural tagging. I decided early that I'd need multiple chunking strategies for the different domains and documents. In the current version the strategy is user-selectable per document; I might add a batch agentic feature later where an LLM picks the right one after examining the doc.

A few more deliberate restrictions shaped what you'll see below. I limited chunk size so the context window wouldn't blow out — this runs locally and I have to work within Ollama's limits. The current version is a single request-and-response design, because my use case is finding a direct answer; it doesn't handle conversations yet (I might add that later, if only to learn the memory management). I wanted chunks to be roughly the same size so the vectors would be comparable in granularity — results from different documents should be roughly equivalent. And I wanted the whole thing searchable as a pure ranked search engine without the LLM, so each chunk gets indexed two ways: a full-text index for keyword search, and a dense embedding for semantic search. That's what supports the hybrid search and reranking later in the pipeline.

A naming warning here, because it caught me out while writing this. In the schema, the full-text index lives in a column called `search_vector` — but it isn't a vector in the embedding sense at all. Postgres's `tsvector` type is a list of stemmed words and their positions ('cook', 'meat', 'intimid'), basically a keyword index that happens to have "vector" in its name for reasons that predate the ML meaning of the word by decades. The embedding column is the numeric kind of vector, the one that lives in a 768-dimensional space. Two columns with "vector" in the name, and they have nothing to do with each other. So through this post: when I say full-text or keyword search, I mean the stemmed-word index; when I say embedding or semantic search, I mean the numeric vectors. The next post digs into both kinds of search properly.

So: three chunking strategies, user-selectable, and a design that leans on chunk quality. Which raises the obvious question — does the choice of strategy actually matter? Chunking gets treated as a one-line config choice in most RAG writeups ("we used 512-token chunks with 50-token overlap"), and few show their work on real books. I wanted to know, with data, on mine.

The example that convinced me it matters is a chapter transition in *Thinking, Fast and Slow*. The chapter "Norms, Surprises, and Causes" ends with a set of closing pull-quotes, and "A Machine for Jumping to Conclusions" opens with an anecdote about the comedian Danny Kaye. Same book, same transition, three strategies:

![Bar chart showing where TOKEN, SEMANTIC, and STRUCTURAL each drew chunk boundaries around the chapter transition in Thinking, Fast and Slow. TOKEN's chunk 58 and SEMANTIC's chunk 78 both span across the chapter boundary, fusing the end of "Norms, Surprises, and Causes" with the start of "A Machine for Jumping to Conclusions." STRUCTURAL's chunk 63 starts exactly at the boundary.](/assets/img/posts/lore-chunking-shootout/01-chapter-boundary.png)

Each bar shows the span of text one strategy put into a single chunk; the vertical line is the chapter boundary. TOKEN fused the end of one chapter with the opening of the next — one chunk spanning two unrelated chapters. SEMANTIC did the same thing, which surprised me: both sides of the boundary talk about the same broad topic (how intuitive thinking behaves), so as far as the embeddings were concerned there was no seam there, and the literal `## A Machine for Jumping to Conclusions` heading marker ends up sitting mid-chunk. STRUCTURAL was the only one that started a fresh chunk exactly at the heading.

This shows up in retrieval, not just in the chunk table. I asked the hybrid search endpoint — hybrid meaning keyword and semantic search combined, which the next post covers properly — *why we need causal stories to explain surprising events*, content that sits right at that transition. STRUCTURAL returned the relevant passage at rank 1. TOKEN and SEMANTIC both returned a tangential match about regression to the mean.

The rest of this post runs all three strategies over two of my actual books — *The Meat Hook Meat Book* (a professionally typeset cookbook PDF) and *Thinking, Fast and Slow* (a long-form nonfiction EPUB) — and looks at what the numbers say. The short version: the interesting finding is not "STRUCTURAL wins."

## Why chunking matters more than it looks like it should

A chunk is the unit of retrieval, so every boundary decision has consequences. Get a boundary wrong in one direction and a fact gets split across two chunks — the retriever surfaces half an answer. Get it wrong in the other direction and unrelated content gets merged into one chunk, which drags its embedding toward the average of two topics so it matches neither of them well.

The three strategies make different assumptions about where good boundaries come from. TOKEN cuts on arithmetic: every N tokens, with some overlap so nothing is severed too badly. STRUCTURAL trusts the author: split where the headings are, because the person who wrote the book already divided it into coherent units. SEMANTIC watches the content: embed the text, look for the places where the meaning shifts, cut there.

There's also a reason I put this much effort into chunking specifically, and it's about where in the pipeline mistakes are cheap. Most of the system is easy to change after the fact. Tweak the retrieval ranking and the next query just behaves differently. Swap the chat model and the next answer comes from the new one. But chunking happens at ingestion — the chunks and their embeddings are what's *stored*. Get chunking wrong and the fix isn't a config change; it's re-parsing, re-chunking, and re-embedding every document in the corpus. In the first post I called the embedding model a one-way door for exactly this reason, and chunking sits behind the same door: both are baked into the index. That's why the tuning and the measurement in this post happened early, on two books, rather than later — after my whole library was ingested — when every mistake found would have cost a full re-ingest to fix.

## TOKEN — the baseline

`TokenTextSplitter`, fixed-size chunks, configurable overlap (`token-overlap-chars: 200` by default). It's the cheapest and simplest of the three, and before this experiment I would have described it the way the outline for this post did: "no parsing dependency, nothing to go wrong."

That turned out to be backwards. When I ingested *Thinking, Fast and Slow* for this comparison, TOKEN was the only strategy that failed outright — zero chunks, status FAILED. If you read [the last post]([LINK]) you already know the culprit: the same `<divlity>` tag that opened that post. TOKEN fed the file to Tika, Tika's strict SAX parser threw, and there was no fallback behind it. STRUCTURAL and SEMANTIC ingested the identical file without a hiccup, because they use their own lenient Jsoup-based parsers and only touch Tika as a last resort. The "simple" strategy actually had the hardest parsing dependency of the three. (It has a fallback now — TOKEN degrades to the markdown parsers when Tika throws.)

**TOKEN in short**
- Good: cheap and fast to compute — no extra model calls at ingest, just counting.
- Good: works on any document type; needs no structure or headings to exist.
- Good: predictable, uniform chunk sizes, easy to budget against a context window.
- Bad: blind to structure — will happily fuse a chapter ending with the next chapter's start.
- Bad: boundaries land mid-thought, so some embeddings blur two topics together (overlap softens this, doesn't fix it).

## STRUCTURAL — heading-aware, mostly in name

STRUCTURAL parses the source to markdown, splits on heading markers, and has GENERIC/COOKBOOK/ACADEMIC variants for different document shapes. For PDFs it prefers the document's embedded outline over font-size guessing — [the last post]([LINK]) has that story. When a heading section exceeds a size cap, a token-splitter fallback splits it further.

Now the finding that surprised me most in the whole experiment. Here's STRUCTURAL's chunk sizes next to TOKEN's on the cookbook. A quick note on how to read this, since I'll use the same format throughout: sizes are in tokens (a token is roughly three-quarters of an English word, so 600 tokens is in the neighborhood of 450 words). "Typical" is the median — half the chunks are smaller than it, half bigger. The "middle 80%" range ignores the most extreme 10% at each end and tells you where the bulk of the chunks actually live: a narrow range means the chunks are all about the same size, a wide range means they vary a lot.

![Table comparing STRUCTURAL and TOKEN chunk sizes on the cookbook: STRUCTURAL has 223 chunks with a typical size of 637 tokens (middle 80% range 596–649, biggest 1750); TOKEN has 259 chunks with a typical size of 639 tokens (middle 80% range 623–676, biggest 705). The two distributions are nearly identical.](/assets/img/posts/lore-chunking-shootout/02-structural-vs-token-cookbook.png)

They're nearly identical — same typical size, same narrow band. Think about what STRUCTURAL should look like if it were really cutting at headings: sections of a book vary wildly in length, so you'd expect ragged sizes — some tiny chunks for short sections, some large ones, all over the place. Instead it looks exactly like the strategy that cuts every 600-odd tokens by arithmetic. That's not what a heading-aware strategy should look like, so I traced it. Of STRUCTURAL's 223 chunks, exactly 5 came from headings whose sections fit inside the size cap (`maxChunkChars`, 4,000 characters) — Cover, Title, Copyright, Resources, Acknowledgments. The other 218, which is 98%, were produced by the token-splitter fallback, because every real content chapter (Beef, Pork, Cooking Meat, and so on) blew through it. The PDF's outline is chapter-level only, and the chapters run 20,000–45,000 characters.

The obvious comeback is "so raise the cap." But the cap isn't arbitrary — it exists because of where chunks end up, which is the context budget covered later in this post. Five retrieved chunks have to fit in a local model's prompt alongside the question and the answer; a cap generous enough to hold a 30,000-character chapter would blow that budget on a single chunk. The cap can't chase the chapters. The chapters have to be split.

My first thought was that this was a coarse cookbook and a book with more headings would behave differently. The second book was the test: *Thinking, Fast and Slow* has 47 sections to the cookbook's 20. The result was worse, not better — 3 natural single-chunk sections, 94% of sections needing the fallback, 99.3% of chunks coming from it. More headings didn't help, because Kahneman's chapters are long-form essays that individually still exceed the cap. For ordinary full-length books, I now think this is the default outcome rather than a quirk of one book.

So on this data, STRUCTURAL is TOKEN chunking with a heading label prepended, 98–99% of the time. Which raises the question of how it still won the chapter-transition example at the top of this post. The answer is that the fallback only ever splits *within* a heading section, never across one. However ordinary its chunks look statistically, a STRUCTURAL chunk can never span two chapters — that holds by construction. It's the one property TOKEN can't offer, and the chapter transition is exactly where TOKEN and SEMANTIC both failed.

**STRUCTURAL in short**
- Good: chunks never cross a section boundary — the author's structure is respected by construction.
- Good: that guarantee wins exactly the queries that sit at chapter transitions, where the other two lose.
- Good: still cheap — no extra model calls; parsing markdown structure is the only added work.
- Bad: for real full-length chapters it degrades to TOKEN sizing almost everywhere — don't expect heading-shaped chunks.
- Bad: only as good as the document's structure — a chapter-level-only PDF outline gives it little to work with, and you have to pick the right variant.

## SEMANTIC — embedding-similarity breakpoints

SEMANTIC works by sliding a window over paragraphs, embedding each window, and measuring how far each window's embedding is from the next one's (cosine distance — the standard way of scoring how different two embeddings are). Where the distance spikes, the content has shifted, and that's where it cuts.

"Spikes" is defined by a percentile, and it's worth walking through what that means because it took me a moment too. Every gap between neighboring windows gets a score for how much the topic changed across it. Most gaps score low — the next paragraph usually continues the current thought. Now line all those scores up from smallest to biggest. A percentile of 0.85 says: only the gaps in the top 15% count as real breaks; cut there and nowhere else. So out of every hundred candidate boundaries in a book, roughly the fifteen biggest topic-shifts become chunk boundaries and the other eighty-five get ignored. Raise the percentile to 0.95 and you're only cutting at the five biggest shifts per hundred — fewer, bigger chunks. Lower it and you cut more eagerly — more, smaller chunks. Oversized chunks get the token-splitter fallback as well (`maxChunkChars`, 4000 chars).

The percentile being *per-document* is the part worth understanding. A fixed distance cutoff would assume all books are equally choppy, and they aren't — a recipe collection lurches from topic to topic while a Kahneman chapter glides. Ranking each document's own shifts and cutting at its own top 15% adapts the sensitivity to the book. The defaults (`windowSize=2`, `breakpointPercentile=0.85`, `maxChunkChars=4000`) came from sweeping them against the cookbook.

The numbers show SEMANTIC genuinely doing something different from the other two. Here "small chunks" means chunks under 200 tokens — roughly a solid paragraph or less:

![Table comparing chunk sizes across both books. Cookbook: SEMANTIC has 469 chunks, typical size 192 tokens, 49% small chunks; STRUCTURAL has 223 chunks, typical size 637, ~0% small; TOKEN has 259 chunks, typical size 639, 0% small. Thinking, Fast and Slow: SEMANTIC has 556 chunks, typical size 551, 22.5% small; STRUCTURAL has 427 chunks, typical size 618, 2.3% small; TOKEN has 407 chunks, typical size 617, 0.0% small. SEMANTIC's sizes clearly track the content on both books.](/assets/img/posts/lore-chunking-shootout/03-semantic-both-books.png)

Look at the cookbook column: half of SEMANTIC's chunks are small, tight units, and its typical chunk is a third the size of the other strategies'. That's the shape you'd expect if it really is cutting where the content shifts — a recipe collection changes topic constantly, so it should produce lots of short, focused chunks. On the Kahneman book, where the prose flows for pages on one idea, its chunks grow accordingly. It's the only strategy whose sizes follow the content instead of the configuration. I'll note this cuts against one of my own design goals from the intro — roughly uniform chunks so results across documents are comparable. SEMANTIC trades that uniformity for topical tightness, and I'm still deciding how I feel about the trade.

The percentile does need tuning, though, and getting it wrong does real damage. The outline for this post said to show the tuning rather than assert it, so: I re-ingested the cookbook with `breakpointPercentile=0.95` — a plausible-looking value, "only cut at the top 5% of shifts" — into a throwaway domain. Next to the default:

![Table comparing SEMANTIC's breakpoint percentile settings on the cookbook: p=0.95 produces 282 chunks with a typical size of 497 tokens and 20% small chunks; p=0.85 produces 469 chunks with a typical size of 192 tokens and 49% small chunks. Raising the percentile cuts far less often, producing fewer, bigger chunks.](/assets/img/posts/lore-chunking-shootout/04-percentile-tuning.png)

Read that as: raising the threshold from 0.85 to 0.95 means the splitter cuts far less often, so you get 40% fewer chunks, the typical chunk is two and a half times bigger, and most of those small, focused chunks disappear. Same book, same pipeline, one config value changed — p=0.95 systematically merges content that p=0.85 keeps apart. The concrete example: one p=0.95 chunk contains the tail of one recipe ("The Inevitable Pork Chop with Cheddar Grits"), the entirety of a second ("Rooftop Ribs"), and the headnote of a third ("Chinese Barbecue Pork"). Three recipes in one chunk means one embedding pulled toward the average of all three — so a search for any one of those recipes matches that chunk less well than it should. At p=0.85 the same span falls into five cleanly bounded chunks, each recipe starting fresh.

One wrinkle worth being honest about. My original tuning notes describe p=0.95 producing a single 24,000-character monster chunk. It doesn't anymore — the size-cap fallback now catches it, so the *largest* chunk at p=0.95 looks perfectly healthy (3,993 chars, just under the cap). The failure didn't go away; it went quiet. A badly tuned percentile no longer produces one obviously broken chunk you'd trip over — instead it quietly shifts *every* chunk toward bigger and blurrier, which you only notice if you compare typical sizes across the whole book rather than looking for a single outlier. That's good for production and bad for noticing you need to tune.

**SEMANTIC in short**
- Good: the tightest topical cohesion — chunks start and end where the content actually shifts, so each embedding represents one thing.
- Good: adapts to the document — short focused chunks for choppy books, longer ones for flowing prose.
- Bad: by far the slowest at ingest — re-ingesting the cookbook just now (same file, same machine, single run) took 213s wall-clock for SEMANTIC against 32s for TOKEN and 45s for STRUCTURAL. Nearly all of that time (212 of 213s) is the sliding-window pass finding breakpoints, before the chunks themselves ever get embedded for the index — SEMANTIC pays for two embedding passes where the other two pay for one.
- Bad: the percentile is a genuinely dangerous knob, and a bad setting now fails quietly across the whole distribution.
- Bad: ignores hard section boundaries, and its variable sizes cut against cross-document comparability.

## Head-to-head on both books

Chunk counts and sizes for both books, in the same format as before (tokens are counted with the same tokenizer the TOKEN splitter itself uses, so the comparison is apples to apples):

![Two tables comparing chunk sizes on both books. The Meat Hook Meat Book (PDF, COOKBOOK variant): SEMANTIC 469 chunks, typical 192 tokens, middle 80% range 94–591, biggest 1744; STRUCTURAL 223 chunks, typical 637, range 596–649, biggest 1750; TOKEN 259 chunks, typical 639, range 623–676, biggest 705. Thinking, Fast and Slow (EPUB, GENERIC variant): SEMANTIC 556 chunks, typical 551, range 103–598, biggest 854; STRUCTURAL 427 chunks, typical 618, range 528–640, biggest 795; TOKEN 407 chunks, typical 617, range 595–635, biggest 659. STRUCTURAL and TOKEN track each other closely on both books; SEMANTIC is the outlier with more, smaller, content-adaptive chunks.](/assets/img/posts/lore-chunking-shootout/05-head-to-head.png)

The story the sizes tell: STRUCTURAL and TOKEN are nearly interchangeable on both books, while SEMANTIC lives in a different world — far more chunks, much smaller typical size, and a middle-80% range five to six times wider, because its sizes track the content. (If STRUCTURAL's biggest chunk of 1750 tokens looks odd next to "98% capped fallback pieces" — that outlier is one of the five natural sections, not a fallback chunk.)

The retrieval spot-check: three real queries per book against each strategy's chunks, through the production hybrid endpoint. "Hit, rank 1" means the passage that actually answers the question came back as the top result. Cookbook first:

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

Reading this honestly: four of the six queries are clean three-way sweeps. That's a useful sanity check — retrieval works regardless of strategy for easy questions — but it doesn't discriminate between them. The two hard queries share a property: their answers aren't localized in one tight passage. The causal-stories answer sits at a chapter boundary; the gas-vs-charcoal answer is spread across a whole section. That's where the strategies diverge: nobody cleanly wins gas-vs-charcoal (the failure modes just differ), and STRUCTURAL alone wins the causal-stories query, which is the retrieval consequence of the chapter boundary in the opening exhibit. And six queries is a spot-check, not an evaluation — the proper version would be a golden set of a few dozen queries with known answers, scored on whether the right chunk appears in the top k. That harness is on the future-work list; these six were chosen to probe specific failure modes, and they did.

For balance, the exhibit I expected to lead this post with went the other way. On the cookbook's high-heat-versus-low-heat cooking guidance — a two-part instruction pivoting on the word "Conversely" — TOKEN cut exactly at the pivot, so one chunk ends mid-argument on the high-heat half. But TOKEN's 200-character overlap meant the following chunk carried both halves anyway, and STRUCTURAL's save turned out to be luck: its winning chunk was a fallback piece whose boundary happened to land a few hundred characters later. SEMANTIC was the only one that kept the whole unit together deliberately — its chunk spans the entire how-to-think-about-cooking-meat run and ends where the topic genuinely shifts. Three strategies survived the same passage for three different reasons, and only one of those reasons was on purpose.

## The finding I wasn't looking for: the context budget

While tracing why STRUCTURAL's sizes mirrored TOKEN's, I pulled a thread that had nothing to do with chunking strategy. The fallback splitter — and TOKEN itself — used Spring AI's `TokenTextSplitter` default chunk size of 800 tokens. Nothing in Lore had ever chosen that number. It was inherited from the library.

Meanwhile, at the other end of the pipeline, Lore's chat view puts 5 retrieved chunks into a single prompt for the local model. A model's context window is its working memory — everything it can consider at once: the instructions, the retrieved chunks, your question, and the answer it's writing all have to fit inside it. And `llama3.1:8b`, despite advertising a 128,000-token context on paper, actually loads in Ollama with a 4,096-token window, because Lore never overrides the `numCtx` setting. The model card and the running process are describing two different things: 128K is what the architecture *supports*, but context costs memory — the model keeps a cache for every token in the window, and at 128K that cache alone would eat many gigabytes — so Ollama loads models with a small window by default unless you explicitly ask for more. Lore never asked. I found this by checking Ollama's API for what was actually loaded rather than trusting the model card — worth doing, as it turns out.

What makes this failure mode nasty is that going over the window doesn't produce an error. Ollama truncates the prompt to fit and generates an answer from whatever survived, so part of the retrieved context simply never reached the model, and nothing in the stack said so. The only symptom is answers being subtly worse than the retrieval deserved.

Add it up: 5 chunks × 800 tokens is 4,000 tokens of context, over budget before the system prompt, the question, or a single token of the answer. Every chat turn was losing context this way, and I hadn't noticed. Post 00 called fluent-but-wrong the nastiest failure mode in RAG; this was my own concrete case of it, in my own app, found by accident. The fix was an explicit `tokenChunkSize` (default 600) applied everywhere `TokenTextSplitter` gets used, which puts a 5-chunk turn at roughly 3,000–3,400 tokens with real headroom for everything else.

The general lesson is about defaults whose consequences live somewhere else. Chunk size looks like an ingestion setting, but its real constraint sits two subsystems away in the chat prompt budget. The library default wasn't wrong — it was unexamined against how the app actually uses chunks.

## A note on the data

Gathering the boundary exhibits surfaced two real parser bugs — two-column PDF layouts getting zippered together line by line, and then my first fix for that bisecting full-width prose that had been correct before — and the context-budget discovery forced the chunk-size change. All three fixes landed before the final data pass, so every number in this post is post-fix. Which means the first dataset I collected was quietly measuring parser artifacts as if they were chunking behavior. If I'd drafted this post from that first pass, I would have confidently blamed chunking for bugs that lived in the parser.

One more thing worth pinning down: every number above was captured at the code state tagged [`chunking-shootout-data`](https://github.com/walterdeane/lore/tree/chunking-shootout-data) in the repo. I've since made a further fix to STRUCTURAL's EPUB path — recognizing bold-only paragraphs as headings, not just real `<h#>` tags — which changes the fallback share reported above for EPUB-sourced books specifically (it doesn't touch the cookbook, which is a PDF). Whether that dents the 98–99% figure enough to change the "STRUCTURAL is TOKEN with a label" framing is a good question for a follow-up; I'm not going back to rewrite this post's numbers every time the parser improves.

## Which one, then

The "in short" blocks above cover the properties; the practical guidance is shorter. None of these is simply better — they fail in different places, and on this data the failures line up with what each strategy ignores: TOKEN ignores structure, SEMANTIC ignores the author, STRUCTURAL ignores content size. My own defaults after this experiment: STRUCTURAL where the document has real structure to respect (its guarantee is cheap insurance that pays out exactly on the queries the others lose — but check the heading granularity before expecting more), SEMANTIC where topical tightness matters and I'm willing to watch the distribution after tuning, TOKEN when I just need something format-agnostic that works. And its fixed-size blindness to boundaries is a real cost, not a rounding error: it's what the hard queries punished.

## Strategies I didn't implement

Three strategies is not the whole field, and it's worth being upfront about what Lore doesn't do yet. I went with these three because they had the best capability for comparing approaches — three genuinely different theories of where boundaries come from, each with clear pros and cons, all cheap enough to run over a whole library locally.

The two I'd been mulling beyond them are agentic chunking and small-to-big retrieval. Agentic chunking in the modest form: an LLM examines the document and picks the right strategy and variant, which is currently a manual choice per document. That's one model call per document, so it stays within the green-and-cheap constraints. Small-to-big (also called child-to-parent or parent-document retrieval) means searching against small, tight chunks but returning something larger — the parent section, or the hit plus its neighbors. It's appealing because it would dissolve a tension this post's own data shows: SEMANTIC's small chunks are great for matching and stingy as context, and the context budget section shows the pressure at the other end. Search small, return big gets both. Since every chunk already carries its position in the document, returning neighbors is nearly free to build.

There's also a cheaper improvement I haven't done yet: embedding document metadata into each chunk — prepending something like the book title and chapter to the chunk text before it's embedded, so a chunk about resting meat embeds as meat-cookery advice rather than generic advice. It's a string concatenation at ingest, and STRUCTURAL already carries the heading labels that would make good prefixes. The catch is the same one-way door as everything else in this post: adding it means re-ingesting, which is a good reason to decide before the whole library goes in.

Any of these might turn into a follow-up post or a project of its own, with the same before-and-after measurement this post used — the capture harness is already built.

## Closing: per-document tuning, not a solved problem

Lore resolves a chunking strategy per document (`chunkingStrategyResolver`), with STRUCTURAL variants per document shape and SEMANTIC defaults tuned against several cookbooks. That's hand-tuning, and I'm calling it that. The open question is whether resolution should get smarter — more variants, per-corpus threshold sweeps, or the agentic selection from the last section — or whether hand-tuned defaults plus honest measurement is the right stopping point for a personal tool.

One more thing from the spot-check that I'm saving for next time. I also ran those six queries through the lexical-only endpoint, and one conversational query returned zero results across all three strategies — while hybrid search degraded gracefully on the exact same question. Why keyword search fails completely where hybrid merely wobbles, and how Lore fuses the two legs so each covers the other's blind spots, is the next post.

*Next in the series: [The hybrid search architecture]([LINK]).*