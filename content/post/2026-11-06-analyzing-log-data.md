---
layout: post

title: "Parsing, Visualizing and Analyzing Log Data with Claude Code"
subtitle: "From a static log file to an interactive analysis tool—and beyond"
author: Jamie Craane
date: 2026-06-11
description: "How I used Claude Code to turn a production slow-query log into an interactive visualization, then let Claude analyze the result and propose fixes."
image: "/img/posts/analyzing-log-data.png"
showtoc: false
tags:
- AI
- Claude Code
- AI-Assisted Development
- Observability
- Tooling

categories: [ AI ]
URL: "/2026/11/06/analyzing-log-data/"
---

### Introduction

A production system feels sluggish, you suspect the database, and you have a log file full of slow query entries staring back at you. The data you need is right there. The hard part is turning a few thousand lines of text into answers.

Recently I had exactly this situation: a log file from production containing slow database queries, and four questions:

- **Which queries are slow?**
- **What is the distribution of the execution time?**
- **When are the queries slow?**
- **What are the most frequently executed queries?**

These are the bread-and-butter questions of database performance work. And yet, answering them well is surprisingly awkward with the tools we usually reach for.

### The Problem: Good Data, Wrong Tools

I had two obvious options, and neither felt right.

**Grafana.** We already ship logs to Grafana, but a dashboard is a fixed lens. I am limited to the panels and queries I have pre-configured. The moment I want to ask a slightly different question—"show me the execution-time distribution bucketed by query shape, but only for the evening peak"—I am fighting the tool instead of exploring the data.

**A script and a notebook.** Flexible and powerful, but slow to set up, it lives on my machine, and it is not something I can hand to a colleague who just wants to drag in a log file and look.

What I wanted was the flexibility of the notebook with the immediacy of a dashboard—reusable for *any* log file, not just this one.

### The Solution: Let Claude Build the Tool

Instead of choosing, I asked Claude Code to build an interactive HTML page. A single self-contained file that lets me **drag in any log file**, **parses** the entries client-side, and renders **visualizations tailored to my exact questions**. No server, no build step, no notebook kernel to keep alive.

This is the key shift: the cost of building a custom, single-purpose tool has collapsed to almost nothing. When a bespoke analysis page takes minutes to generate, you stop bending your question to fit the dashboard and start building the dashboard to fit your question. The tool becomes disposable—you build a perfectly-fitted tool for *this* log and *these* questions, and regenerate it next time.

#### From Log Line to Insight

Suppose the slow-query log looks like this:

```
2026-06-10T19:42:11Z duration=1834ms rows=2 query="SELECT * FROM orders WHERE customer_id = $1"
2026-06-10T19:42:13Z duration=2203ms rows=0 query="SELECT * FROM orders WHERE customer_id = $1"
2026-06-10T19:42:15Z duration=87ms   rows=1 query="UPDATE sessions SET last_seen = now() WHERE id = $1"
2026-06-10T20:11:48Z duration=4120ms rows=512 query="SELECT * FROM line_items WHERE order_id IN (...)"
```

The prompt to Claude Code is essentially the four questions plus the log format. The resulting page gives me, in one view:

1. **A ranked table of slow queries**—normalized by *query shape* (parameters stripped) so the same query with different bind values is grouped together.
2. **A histogram of execution times**—do I have a long tail of occasional 4-second outliers, or a broad smear of consistently-slow queries? This is the question dashboards usually answer worst.
3. **A timeline of slow queries over time**—in the sample above the pain clusters around the evening peak, which is a very different problem from uniform slowness.
4. **A frequency table**—so I can weigh a query that is slow-but-rare against one that is moderately-slow-but-runs-constantly. The second is almost always the better thing to fix first.

### Going Further: Three Stages of Leverage

The HTML page is useful on its own, but it is really just the first stage. Each stage hands Claude more context and asks it to do more of the thinking.

#### Stage 1 — Visualize

Claude writes the interactive page; I do the interpreting. This is where most people stop, and it is already a big win over a static dashboard.

#### Stage 2 — Analyze the Rendered Page

Next, let Claude analyze the parsed data behind the page and reason about what it sees:

> What are the top 3 problems worth tackling, ranked by impact? For each, suggest a likely cause and a concrete fix.

Now Claude is doing the triage. It can notice that one query shape accounts for 60% of total time despite being only 5% of executions, or that the slowness correlates with the evening peak—suggesting a load or locking issue rather than a missing index. I move from *reading charts* to *reviewing recommendations*.

#### Stage 3 — Point Claude at the Code and the Database

The final stage closes the loop: point Claude Code at the application code and the database schema. Now it can connect a slow query back to the function that issues it, check whether the relevant column is indexed, and propose a real change:

- "This `SELECT * FROM orders WHERE customer_id = $1` has no index on `customer_id`; here is the migration."
- "This query is issued in a loop in `OrderService.kt`; it is an N+1 and should be a single batched query."

At this point the log was just the entry point that told Claude where to look. The real value is Claude reasoning across the log, the code, and the schema together to produce actionable fixes.

### Caveats

A few things keep this honest:

- **Verify the parser.** An AI-generated parser can silently mis-handle an edge case in your log format and skew every chart downstream. Spot-check the parsed output against the raw lines.
- **Keep production data in mind.** Logs can contain sensitive values. Dragging a file into a local HTML page is fine; be careful about what you paste into any hosted context.
- **Treat recommendations as hypotheses.** Stage 2 and 3 suggestions are starting points for investigation, not conclusions. Confirm with `EXPLAIN ANALYZE` before you ship an index.

### Conclusion

I started with a log file and four questions, and the conventional answers—a dashboard or a notebook—each forced a compromise. By letting Claude Code build a throwaway, custom analysis page, I got exactly the visualizations I wanted. And by escalating from *visualize* to *analyze the page* to *analyze the code and database*, the same workflow grows from a charting tool into something closer to a junior performance engineer that triages the problem and proposes fixes.

The log was never really the point. It was the doorway. Once Claude is looking through it, the interesting work is everything it can reason about on the other side.
