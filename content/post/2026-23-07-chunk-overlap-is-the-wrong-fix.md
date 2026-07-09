---
layout: post

title: "Chunk Overlap Is the Wrong Fix"
subtitle: "Recover context at retrieval time, where you know what matched, instead of duplicating it at ingest time, where you don't"
author: Jamie Craane
date: 2026-07-23
description: "Standard RAG advice says to overlap your chunks by 10 to 20 percent. My knowledge base uses zero overlap and expands a matched chunk to its enclosing section at query time instead. Here is why, and where that approach breaks down."
image: "/img/posts/chunk-overlap-is-the-wrong-fix.png"
showtoc: false
tags:
- AI
- RAG
- Kotlin
- PostgreSQL
- Retrieval

categories: [ AI ]
URL: "/2026/23/07/chunk-overlap-is-the-wrong-fix/"
---

### Introduction

I run a local-first knowledge base over my own notes. Markdown files and PDFs in a folder get chunked, embedded with OpenAI, and stored in PostgreSQL with pgvector. Semantic search and a RAG chat sit on top.

If you read anything about building one of these, you will be told to overlap your chunks. Ten to twenty percent is the usual number, so that an idea sitting on a chunk boundary is not cut in half. My own improvement notes for the project say exactly this, filed under "Impact: High".

The splitter takes an `overlap` parameter. Every call site leaves it at zero.

That was not laziness. It is a bet that overlap solves the right problem in the wrong place, and the questions worth asking are:

- What is overlap actually insuring you against?
- What does that insurance cost?
- Is there a cheaper policy?

### What Overlap Is For

Chunking cuts a document into pieces small enough to embed. The cut lands somewhere, and sometimes it lands badly: mid-sentence, or between a claim and the sentence that qualifies it. A chunk retrieved from near that boundary is missing the context that makes it useful, and the model sees half a thought.

Overlap is the hedge. Repeat the last 10 to 20 percent of each chunk at the start of the next one, and any idea near a boundary appears whole in at least one chunk. It works. It also has three costs, and only the first one gets talked about:

- **You embed the same text more than once.** Every overlapping window is another API call and another row. On a large vault that is real money and real disk.
- **Near-duplicate chunks compete in the results.** Top-k retrieval has a fixed budget. Two chunks sharing 20 percent of their text will often score similarly, so one slot in your context window gets spent on text you already have. Overlap buys context by *reducing the diversity* of what you retrieve.
- **You have to pick the overlap fraction at ingest time.** That is the real problem. At ingest you do not know what will be asked, so you cannot know how much surrounding context a future match will need. You are guessing a constant, uniformly, for every chunk in the corpus.

That last point is the whole argument. **Ingest time is the moment when you have the least information about what matters.**

### The Alternative: Expand at Retrieval

At retrieval time you know something you did not know before: which chunk matched. That is exactly when you can afford to be generous with context, and exactly when generosity is cheap, because you are dealing with a handful of chunks rather than a whole corpus.

So the pipeline does two things instead of one:

**At ingest**, split on structure rather than on length. For markdown that means splitting on headers, so each chunk is a section with its heading path retained as metadata. The chunks are already semantic units, which is most of what overlap was protecting against. A second pass caps anything pathologically long.

**At retrieval**, take each matched chunk and grow it back out to the section it came from, by re-reading the original document. The database stores each chunk's `content_start_offset` and `content_end_offset` along with the document body, so this is a substring operation, not a re-embedding.

```kotlin
val (sStart, sEnd) = if (headers.isEmpty()) {
    // No markdown structure, so use a neighbor window around the chunk.
    val a = (chunk.startOffset - windowChars).coerceAtLeast(0)
    val b = (chunk.endOffset + windowChars).coerceAtMost(text.length)
    a to b
} else {
    findSection(chunk.startOffset, headers, text.length)
}
```

The embedding is computed on the precise chunk, which keeps the vector focused. The text handed to the model is the surrounding section, which keeps the answer grounded. Those are two different jobs and there is no reason the same span of text has to do both.

### What Stops It From Swallowing the Whole File

An expander with no brakes is just a document loader with extra steps. Two constants apply the brakes, and both exist because of a specific failure I hit.

**A section can be enormous.** A long file with sparse headers has "sections" thousands of characters wide. A two-line match in such a file would drag the entire document into the prompt, which defeats the point of chunked retrieval. So a section wider than `DEFAULT_MAX_SECTION = 4000` characters is not used. Instead the expander falls back to a *neighbour window*: the matched chunk plus the chunk immediately before it and the chunk immediately after it, reconstructed from the stored chunk offsets.

**Many chunks in one big file can all match.** Think of a long table, or a watch list, where every row resembles every other row. Expand each match to its neighbours and you stitch the file back together one window at a time. So a single document may contribute at most `MAX_TRIMMED_REGIONS_PER_DOC = 1` such trimmed region.

Ordering makes that cap safe:

```kotlin
// Highest-similarity matches first, so the per-document cap on trimmed regions keeps
// the best one. A large file whose chunks all match (e.g. one long table) would
// otherwise have every chunk neighbor-expanded and stitched back into the whole file.
for (chunk in docChunks.sortedByDescending { it.similarity }) {
    ...
    val oversized = sEnd - sStart > maxSectionChars
    val (rStart, rEnd) = if (oversized) {
        if (trimmedRegions >= MAX_TRIMMED_REGIONS_PER_DOC) continue
        neighborWindow(chunk, bounds, text.length)
    } else {
        sStart to sEnd
    }

    if (!seen.add(rStart)) continue
    ...
}
```

Process the best match first, cap the rest, and deduplicate by region start so two chunks in the same section produce one region rather than two copies of it. Deduplication, incidentally, is a problem overlap *creates* and expansion *solves*: overlapping chunks hand the model the same paragraph twice, while expanded regions get merged.

### Where This Breaks Down

Here is the part that keeps me honest, because the approach is not uniformly better and my own codebase proves it.

The whole design rests on documents having structure to expand into. Markdown does. **PDFs do not**, and the PDF path in this project is embarrassing:

```kotlin
/**
 * Text splitter specifically designed for PDF content.
 * Splits text by paragraphs and headings, preserving document structure.
 */
class GenericTextSplitter : TextSplitter {
    override fun split(content: String): List<TextChunk> {
        if (content.isBlank()) return emptyList()
        return content.windowed(4000, 4000, partialWindows = true).map { ... }
    }
}
```

The doc comment describes a splitter that preserves paragraphs and headings. The code is a blind fixed window of 4000 characters with a step of 4000, which is to say no structure and no overlap. When one of those chunks matches, the expander finds no markdown headers, so it falls back to a plus or minus 600 character neighbour window. PDFs in my vault get neither structural chunking nor overlap. They get the worst of both.

So the honest position is not "overlap is wrong". It is:

- **Where the document has structure, split on it and expand at retrieval.** Overlap is redundant, and it costs you money and result diversity.
- **Where the document has none, overlap earns its keep.** Fixed-size windows over a PDF, an OCR dump, or a meeting transcript will cut ideas in half, and there is no section to grow back into. My improvement notes are right about PDFs and wrong about markdown, which is what happens when advice is written for "chunks" in general.

There is also a precondition people skip. **Expansion is only cheap if you kept the original.** You need the full document body and each chunk's byte offsets available at query time. If your pipeline throws the source away after embedding, or stores chunks without offsets, you cannot expand and overlap is your only option. Keeping the offsets is the single design decision that made all of this possible, and it costs two integer columns.

### A Few Caveats Keep It Honest

**I cannot prove any of this on my own corpus.** There is no evaluation harness in the project: no golden query set, no recall@k, no regression test over ranked results. The reasoning above is sound and the failure modes the constants prevent are real, observed ones. But "expansion beats overlap here" is an argument, not a measurement, and I would rather say so than dress a judgment call up as a benchmark.

**4000 and 1 are judgment calls.** They are the numbers where the failures I saw stopped happening. They are not derived from anything.

**Expansion inflates the prompt.** You are trading a small precise chunk for a larger region, and a handful of matches across a handful of documents adds up. The two caps are what keep the context window bounded, and without them this technique is worse than overlap, not better.

### Conclusion

Overlap is insurance you buy at ingest time against a question nobody has asked yet, paid for in duplicate embeddings and crowded results. Expansion is a decision you make once the question has arrived and the match is in your hand.

Four things I would keep:

- **Split on structure wherever structure exists.** Headers are free semantic boundaries, and they are better than any percentage you could pick.
- **Store the offsets and keep the source document.** They are what let you buy context back later, for the price of a substring.
- **Bound the expansion**, per section and per document, or you have written a very slow way to paste whole files into a prompt.
- **Where there is no structure, use overlap.** It is the right tool for exactly the case where expansion has nothing to expand into.

The embedding and the context do not have to be the same text. Once you stop assuming they do, the overlap question mostly dissolves.
