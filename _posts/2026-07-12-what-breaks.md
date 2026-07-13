---
layout: post
title: "What Breaks When You Ingest Real Books Instead of Test Fixtures"
date: 2026-07-12
tags: [rag, kotlin, spring-boot, spring-ai, tika, pdf, epub, ingestion]
---

*This is the second post in a series about [Lore](https://github.com/walterdeane/lore), a local-first RAG system for personal documents. The [first post](/posts/introducing-lore/) covers what Lore is and how it's put together.*

My copy of *Thinking, Fast and Slow* killed the ingestion pipeline with a single tag. There's some irony there — it's probably the book that's made me better at almost everything, to the point that I used to buy copies just to give to friends and coworkers. Now it was the first book to take my code down.

Somewhere in one of the EPUB's 70 content files sat `<divlity>` — a malformed fragment, probably a scraping or conversion artifact, one broken tag in 1.16 million characters of otherwise fine XHTML. Tika's strict SAX parser hit it and threw, the exception propagated up, and the whole import failed. Nothing else about the book was wrong. One tag.

I'll admit the failures caught me off guard. When I started writing Lore I thought the hard part would be learning Spring AI — but Spring AI turned out to be well put together and easy to use. I'd done my deep dives into chunking strategies and RAG techniques and figured the rest would be easy to knock over. The first few imports backed me up: search worked, a simple chat worked, I was a genius. Then I started importing more books to grow the corpus, and the failures piled up — malformed tags, invalid EPUBs, PDFs broken in ways I'd never seen.

Every RAG tutorial I read while building Lore ingests a clean PDF or two and moves on to the interesting parts — embeddings, retrieval, prompts. This post is about what happens before any of that: pointing an ingestion pipeline at real files — EPUBs of varying provenance, archive.org scans, professionally typeset cookbooks — and watching it break in ways no fixture would have predicted. Four failures, in the order they found me. The first two happen to be EPUBs and the last two PDFs, but don't read too much into that — strict-vs-lenient parsing, bad structural assumptions, and print artifacts can bite in either format.

## The setup

Just enough context for the failures to land. Lore's ingestion pipeline is: extract text from the EPUB or PDF, split it into chunks using one of three chunking strategies (fixed-size token, heading-aware structural, or embedding-similarity semantic — the next post compares them properly), embed each chunk, store everything in Postgres. Extraction is Apache Tika by default, but the structural and semantic strategies have their own Jsoup-based markdown parsers that understand document structure, and only fall back to Tika when their own extraction comes back empty.

That fallback arrangement is where the first failure lived.

## Failure 1: the strict parser vs. the lenient one

The `<divlity>` tag raised `TIKA-237: Illegal SAXException` and failed the import. My first instinct was to blame the file — and the file *was* malformed. But then I noticed something odd: `EpubMarkdownParser`, the Jsoup-based extractor that the structural strategy actually uses, handled the same book perfectly. Jsoup is lenient by design — it's built for the web, where malformed markup is the norm — and it extracted all 1.16 million characters, broken tag and all.

So why did the import fail? Because `DocumentIngestionService` was calling Tika *unconditionally*, before branching on chunking strategy — even for strategies that only use Tika as a fallback. The strict parser ran first, threw, and the lenient parser that would have succeeded never got the chance. The real bug wasn't in the file; it was eager work sitting in front of a branch that mostly didn't need it.

The fix was to make the Tika read lazy — pass a supplier instead of a value:

```kotlin
val reader = TikaDocumentReader(FileSystemResource(document.sourcePath))

val splitDocuments = when (strategy) {
    ChunkingStrategy.TOKEN ->
        tokenOverlapChunker.applyOverlap(tokenSplitter.split(reader.get()), ...)

    ChunkingStrategy.STRUCTURAL ->
        structuralTextSplitter.split(document.sourcePath, document.sourceType, reader::get, variant)

    ChunkingStrategy.SEMANTIC ->
        semanticTextSplitter.split(document.sourcePath, document.sourceType, reader::get)
}
```

`reader.get()` is only called eagerly for the TOKEN strategy, which genuinely needs it. The other two receive `reader::get` as a fallback they may never invoke — so a file that Tika chokes on no longer fails ingestion for a strategy that never needed Tika in the first place.

The general lesson: unconditional eager work in front of a branch is a classic footgun, and it hides well. The code *looked* reasonable — "extract the text, then chunk it" is the obvious reading order. It took a malformed file to reveal that extraction and chunking strategy were more entangled than they should have been.

## Failure 2: a "valid enough" file that isn't valid

The next casualty was an archive.org-scanned EPUB that threw `EpubZipException` from inside Tika's own EPUB parser. The cause this time was structural: the EPUB spec requires the `mimetype` entry to be the first file in the zip container, stored uncompressed. This file didn't honor that. Apparently plenty of real EPUBs don't — the spec requirement exists so a reader can identify the format by looking at fixed byte offsets, and files that violate it mostly still open fine in lenient reader apps, so the problem goes unnoticed until something strict looks at them.

This one I didn't fix. The container was genuinely malformed — not "malformed but recoverable" like a bad tag inside otherwise-parseable XHTML, but broken at the level of the format's own identification mechanism. I rejected the fixture and moved on.

That's a line worth drawing deliberately: which failures do you engineer around, and which do you declare out of scope? My rule of thumb after this: recover from damage *inside* the content (bad markup, weird encoding, junk characters), reject damage to the *container* (invalid zip structure, wrong format claims). Content damage is universal and unavoidable; container damage means the file is lying about what it is, and code that second-guesses that tends to grow into a parallel implementation of the format spec.

## Failure 3: headings that aren't headings

Before the last two failures, a word about PDFs, because both failures live there. PDFs are fundamentally a pain to work with, and it's not the format's fault so much as its purpose: a PDF isn't meant to be read by software, it's meant to be *printed*. The text isn't stored linearly — it's a series of drawing commands placing glyphs at positions, in fonts, at sizes. "Extracting the text" means reverse-engineering a document out of what is essentially a plotter instruction stream — anyone who's tried to cut and paste from a PDF into a word processor has met this firsthand. And the spec is flexible enough to allow a dozen ways of doing the same thing, some of them broken, so extraction code has to be prepared for anything. I've generated and parsed PDFs at various points in my career and this project still had me doing more googling — and leaning on Claude more — than everything EPUB-related combined.

Which brings us to headings. PDFs don't have them, structurally speaking. The common heuristic for recovering structure is font size: if a line's font is much bigger than the body text, call it a heading. Lore's structural chunking strategy leaned on this so chunks could align to the document's actual sections.

Then a professionally typeset PDF turned up with a print-shop production stamp on it, plus a scattering of stray larger-font glyphs, and the heuristic dutifully turned them all into headings. Body text fragmented at every false boundary — dozens of tiny chunks, each cut mid-thought, none of them a coherent retrieval unit.

The fix wasn't a better heuristic; it was noticing the heuristic shouldn't run at all when better information exists. Plenty of PDFs carry an embedded outline — the bookmarks panel in your PDF reader — which is the *author's* structure, not a guess. Spring AI's `ParagraphPdfDocumentReader` reads exactly that. So the parser now prefers the embedded outline whenever one exists and falls back to the font-size heuristic only when it doesn't. (`ParagraphPdfDocumentReader` throws at construction when a PDF has no outline, which makes the fallback decision pleasantly unambiguous.)

The pattern generalizes: when a document format lets authors state structure explicitly, trust that first and infer only in its absence. Inference should be the fallback, never the default.

## Failure 4: running headers, three times

The failure that kept coming back. Print books reprint things at the top of every page — the book title, the chapter name — and when you extract a PDF's text, those running headers land in the middle of your content, over and over. Left alone, they leak into chunks and poison retrieval: search for the book's title and you match every page of it.

**Pass one** handled the obvious cases. Book-wide running headers repeat on nearly every page, so a frequency threshold catches them: any short line appearing in more than 30% of sections is a header, strip it. Chapter-scoped headers don't clear a book-wide bar, so a pattern match (`^chapter\s+\d+`) catches those.

**Pass two** came courtesy of a different book — *The Meat Hook Meat Book* — which baked the page number directly into the header line: `THE MEAT HOOK MEAT BOOK 14`, `THE MEAT HOOK MEAT BOOK 15`, and so on. Every occurrence was a *distinct string*. None individually cleared the frequency threshold, so none were stripped, and 71 of the book's 436 chunks leaked the header mid-sentence. The fix: strip trailing page numbers before counting, so all the variants collapse into one key that sails past the threshold.

**Pass three** was the same book, still leaking. The PDF's text extraction goes through PDFBox's region-based layout stripper, and its column handling leaves irregular *internal* spacing on the same logical line: `THE MEAT  HOOK  MEAT  BOOK` in one place, `THE  MEAT HOOK  MEAT  BOOK` in another. Different strings again, different count keys again, each variant under the threshold again. So the normalization now collapses whitespace runs *first*, then strips the trailing page number:

```kotlin
internal fun normalizeForHeaderDetection(line: String): String {
    val collapsed = line.replace(Regex(" {2,}"), " ").trim()
    val stripped = collapsed.replace(Regex("""\s+\d{1,4}$"""), "").trim()
    val wordCount = stripped.split(Regex("""\s+""")).count { it.isNotBlank() }
    return if (stripped.length >= 12 && wordCount >= 3) stripped else collapsed
}
```

The part of this saga that stung most wasn't any of the three passes — it was the diagnostic that lied to me between passes two and three. I'd written a verification check to confirm headers were gone, and it reported all clear while chunks were still leaking. It compared raw, unnormalized lines: the exact same blind spot as the detection code it was supposed to be checking. The tooling you build to verify a fix can inherit the bug's assumptions wholesale — if the fix needed normalization, odds are the verifier does too.

## What this adds up to

None of these were exotic edge cases hunted down for blog material. They were the *first few books* I fed the pipeline — a bestselling paperback's EPUB, an archive.org scan, two nicely typeset cookbooks. Four failures from a handful of inputs, and every one of them invisible to the test fixtures, because fixtures encode the assumptions you already have. Real files encode everyone else's.

The practical takeaway: budget ingestion hardening proportional to how varied your input corpus actually is, not how clean your fixtures are. Testing against fixtures tells you the code runs. Testing against the wild tells you where your assumptions live — and the wild is cheaper to consult early, when each discovery reshapes one function, than late, when it invalidates a corpus.

Lore only handles EPUBs and PDFs today; more formats will come as I need them or someone asks. I'm also being deliberate about what I *won't* fix: no scanned PDFs, no OCR, no DRM'd books, no files that are simply broken. I'd rather handle the vast majority of real books well than chase every pathological file. Given everything above, I suspect each new format will bring its own chapter to this failure log.

These four failures are also why Lore's test suite grew an unusual shape: integration tests that run the real pipeline against real fixture files — including some of the exact books above — with real Postgres and real Ollama behind them, no mocks. How that works, what it costs, and the bug that Testcontainers' documentation practically dared me to write is the next post but one. First, though: the chunking shootout.

*Next in the series: [The chunking strategy shootout]([LINK]).*
