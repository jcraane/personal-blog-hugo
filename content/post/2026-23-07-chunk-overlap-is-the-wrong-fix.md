---
layout: post

title: "Chunk Overlap Solves the Right Problem at the Wrong Time"
subtitle: "Split on structure at ingest, and buy the context back at retrieval"
author: Jamie Craane
date: 2026-07-23
description: "Standard RAG advice says to overlap your chunks by 10 to 20 percent. My knowledge base uses zero overlap and expands a matched chunk to its enclosing section at query time instead. Here is why, and where it actually breaks down."
image: "/img/posts/chunk-overlap-is-the-wrong-fix.jpg"
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

I run a local-first knowledge base over my own notes: markdown files and PDFs in a folder get chunked, embedded with OpenAI, and stored in PostgreSQL with pgvector. Semantic search and a RAG chat sit on top.

Read anything about building one of these and you will be told to overlap your chunks. Ten to twenty percent is the usual number, so that an idea sitting on a chunk boundary is not cut in half. My own improvement notes for the project say exactly this, filed under "Impact: High".

The splitter takes an `overlap` parameter. Every call site leaves it at zero.

Two questions are worth answering before turning that number up:

- What is overlap actually protecting you against?
- Is there a cheaper way to get the same protection?

### What Overlap Buys, and What It Costs

Chunking cuts a document into pieces small enough to embed. The cut lands somewhere, and sometimes it lands badly: mid-sentence, or between a claim and the sentence that qualifies it. Retrieve a chunk from near that boundary and the model sees half a thought.

Overlap hedges against that. Repeat the last 10 to 20 percent of each chunk at the start of the next one, and any idea near a boundary appears whole in at least one chunk. It works. It also costs three things, and only the first one gets talked about:

- **You embed the same text more than once.** Every overlapping window is another API call and another row. On a large vault that is real money and real disk.
- **Near-duplicate chunks compete in the results.** Top-k retrieval has a fixed budget. Two chunks sharing 20 percent of their text tend to score similarly, so one slot in your context window gets spent on text you already have. Overlap buys context by reducing the *diversity* of what you retrieve.
- **You pick the fraction at ingest time.** At ingest you do not know what will be asked, so you cannot know how much surrounding context a future match will need. You are guessing one constant for every chunk in the corpus, at the moment you know the least about what matters.

### The Alternative: Expand at Retrieval

At retrieval time you know something you did not know at ingest: which chunk matched. That is when you can afford to be generous with context, because you are dealing with a handful of chunks instead of a whole corpus.

So the pipeline splits the job in two.

**At ingest, split on structure rather than on length.** For markdown that means splitting on headers, so each chunk is a section with its heading path kept as metadata. The chunks are already semantic units, which is most of what overlap was protecting against. A second pass caps anything pathologically long.

**At retrieval, grow each matched chunk back out to the section it came from**, by re-reading the original document. The database stores each chunk's `content_start_offset` and `content_end_offset` alongside the document body, so this is a substring operation, not a re-embedding.

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

The embedding is computed on the precise chunk, which keeps the vector focused. The text handed to the model is the surrounding section, which keeps the answer grounded. Those are two different jobs, and there is no reason the same span of text has to do both.

### Keeping Expansion Bounded

Expansion without limits is a slow way to paste whole files into a prompt. Two constants apply the brakes, and both exist because of a failure I actually hit.

**A section can be enormous.** A long file with sparse headers has "sections" thousands of characters wide, and a two-line match in such a file would drag the entire document into the prompt. So a section wider than `DEFAULT_MAX_SECTION = 4000` characters is not used. The expander falls back to a *neighbor window* instead: the matched chunk plus the chunk before it and the chunk after it, reconstructed from the stored offsets.

**Many chunks in one big file can all match.** Think of a long table where every row resembles every other row. Expand each match to its neighbors and you stitch the file back together one window at a time. So a single document may contribute at most `MAX_TRIMMED_REGIONS_PER_DOC = 1` such trimmed region.

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

Best match first, cap the rest, and deduplicate by region start so two chunks in the same section produce one region instead of two copies of it. That deduplication is worth noting on its own: overlapping chunks hand the model the same paragraph twice, whereas expanded regions get merged.

### The PDF Path Is Not the Counterargument

The obvious objection is that this all depends on documents having structure, and that PDFs do not. My own code looks like it makes that case for me:

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

The doc comment describes the splitter I intended. The code is a blind fixed window of 4000 characters with a step of 4000: no structure and no overlap. When one of those chunks matches, the expander finds no markdown headers and falls back to a neighbor window.

That is a gap in my implementation, not a limit of the approach. Extracted PDF text still has paragraph breaks, and usually heading-shaped lines too: short, title-cased, surrounded by blank lines. Split on those and PDFs get the same structural chunks markdown gets for free, and a match can be expanded back to its paragraph or its enclosing section. That is the fix, and it is the next thing on my list for this project. Turning overlap on would only paper over it.

### Where This Actually Breaks Down

There are two real limits, and neither is "PDFs".

**Some text genuinely has no structure to expand into.** An OCR dump of a scanned page, an auto-generated transcript with no speaker turns, a wall of text with no paragraph breaks. There, a fixed window really is all you have, and there is no section to grow back to. The neighbor window is a partial substitute, but it guesses at a span the same way overlap guesses at a fraction. This is the case where overlap earns its keep.

**Expansion is only cheap if you kept the original.** You need the full document body and each chunk's offsets available at query time. If your pipeline throws the source away after embedding, or stores chunks without offsets, you cannot expand and overlap is your only option. Keeping the offsets is the one decision that made everything else possible, and it costs two integer columns.

### A Few Caveats Keep It Honest

**This is an argument, not a benchmark.** The project has no golden query set and no recall@k harness, so I cannot show you a number where expansion beats overlap. The failure modes the two constants prevent are real and observed, but the comparison itself is a judgment call.

**4000 and 1 are judgment calls too.** They are the values at which the failures I saw stopped happening, not numbers derived from anything.

**Expansion inflates the prompt.** You trade a small precise chunk for a larger region, and a handful of matches across a handful of documents adds up. The two caps are what keep the context window bounded.

### Conclusion

Overlap is bought at ingest time, against a question nobody has asked yet, and paid for in duplicate embeddings and crowded results. Expansion is a decision you make once the question has arrived and the match is already in your hand.

Three things I would keep in any pipeline:

- **Split on structure wherever structure exists.** Headers and paragraphs are free semantic boundaries, and they beat any percentage you could pick. If your splitter is not finding them, fix the splitter.
- **Store the offsets and keep the source document.** They are what let you buy context back later for the price of a substring.
- **Bound the expansion**, per section and per document.

The embedding and the context do not have to be the same text. Once you stop assuming they do, overlap stops being the default and becomes what it should have been all along: the fallback for text with no structure left to recover.

### References

- [pgvector](https://github.com/pgvector/pgvector)
- [LangChain: Text splitters](https://python.langchain.com/docs/concepts/text_splitters/)
