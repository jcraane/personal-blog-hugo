---
layout: post

title: "Where Money Bugs Actually Live"
subtitle: "What a correctness review of a real invoicing system turned up, and why the Double migration was the easy part"
author: Jamie Craane
date: 2026-07-16
description: "Money bugs cluster where code recomputes what it should have read. A review of the invoicing paths in my time tracking product found a PDF charging clients the wrong VAT, two implementations of the same discount, and reports summing euros and dollars into one number."
image: "/img/posts/money-bugs-in-invoicing.png"
showtoc: false
tags:
- Kotlin
- Correctness
- Code Review
- PostgreSQL
- AI-Assisted Development

categories: [ Kotlin ]
URL: "/2026/16/07/money-bugs-in-invoicing/"
---

### Introduction

MinuteMint, the time tracking and invoicing tool I run for my own company, sends real invoices to real clients. A while back I sat down to review every line of code that touches money, because a rounding bug in an invoicing product is not a style nit. It is a client being asked to pay the wrong amount.

The review found something I did not expect. The arithmetic in the domain layer was correct. Rounding was uniform, no bare `divide()` calls, everything in `BigDecimal`. The bugs were somewhere else entirely:

- The PDF attached to the invoice email charged 21% VAT regardless of what the invoice actually said.
- Two different files implemented "discount", and they disagreed by a cent.
- The stored invoice lines did not sum to the stored subtotal.
- A report summed euros and dollars into a single number.

Every one of these is a *presentation* bug, or a *duplication* bug. None of them is an arithmetic bug. That turns out to be the pattern.

### The Bug Everybody Talks About

If you have read anything about money in software, you know the first rule: never use floating point. `0.1 + 0.2` is not `0.3`, and a `Double` cannot represent a cent exactly.

MinuteMint had that problem, and I fixed it. Money now crosses the wire as an exact value type:

```kotlin
@Serializable
data class Amount(
    val minorUnits: Long,
    @EncodeDefault(EncodeDefault.Mode.ALWAYS) val currency: String = "EUR",
    @EncodeDefault(EncodeDefault.Mode.ALWAYS) val scale: Int = 2,
)
```

A few decisions in there earn their keep. `minorUnits` is a `Long` rather than an `Int`, because a `NUMERIC(10,2)` column maxes out at 99,999,999.99, which is 9,999,999,999 cents, comfortably past `Int.MAX_VALUE` (it still fits in a JavaScript `Number`, so it can serialize as a plain JSON integer). Every amount carries its own currency, so nothing downstream has to guess. And `Amount` is deliberately **arithmetic-free**: it has no `plus`, no `times`. All calculation stays in the domain layer in `BigDecimal`. `Amount` only carries an exact value across the wire.

Hours and percentages cross as exact decimal strings for the same reason.

This migration was necessary. It was also, in hindsight, the easy part. Not one of the bugs that mattered was fixed by it, because none of them was caused by `Double`.

### Money Bugs Live Where You Recompute

Here is the finding that made me put the review down for a minute.

**The invoice PDF, the one that gets stored and emailed to the client, recomputed the totals from scratch.** It re-summed the line rows and applied a hardcoded VAT rate:

```kotlin
val vatRate = BigDecimal("0.21")   // in the PDF renderer
```

The invoice row in the database has `subtotal`, `vatPercentage`, `vatAmount` and `totalAmount`. The renderer read none of them.

For a normal Dutch invoice at 21% VAT you would never notice. Now create an invoice at 9% VAT, or at 0% for *BTW verlegd* (reverse-charged VAT), with a subtotal of €1,000. The database stores a total of €1,090. The email body renders €1.090,00 from `totalAmount`. The attached PDF, which is the legal document, demands "BTW (21%): 210,00" and "Te betalen: 1.210,00".

The client is asked to pay an amount the ledger does not record. Two documents for the same invoice, disagreeing, and the wrong one is the one with legal standing.

The fix is not better arithmetic. **The fix is to delete the arithmetic.** The invoice is a stored fact, not a derived one. Render `invoice.subtotal`, `invoice.vatAmount`, `invoice.vatPercentage`, `invoice.totalAmount`, and nothing else.

### Two Implementations of "Discount" Will Diverge

The same class of bug, one layer over. The domain computes invoice headers in exactly one place:

```kotlin
object InvoiceAmounts {
    fun calculate(
        subtotalLines: BigDecimal,
        discountPercentage: BigDecimal,
        vatPercentage: BigDecimal,
    ): Amounts {
        val subtotal = subtotalLines.setScale(2, RoundingMode.HALF_UP)
        val discountFactor = BigDecimal.ONE.subtract(
            discountPercentage.divide(BigDecimal("100"), 10, RoundingMode.HALF_UP)
        )
        val discountedSubtotal = subtotal.multiply(discountFactor).setScale(2, RoundingMode.HALF_UP)
        val vatAmount = discountedSubtotal.multiply(vatPercentage)
            .divide(BigDecimal("100"), 2, RoundingMode.HALF_UP)
        val totalAmount = discountedSubtotal.add(vatAmount)
        return Amounts(subtotal, discountedSubtotal, vatAmount, totalAmount)
    }
}
```

Note what it does *not* do: it never computes the discount amount directly. It rounds the discounted base, charges VAT on that, and then every renderer derives the discount as `subtotal + vatAmount - totalAmount`. That subtraction is exact at scale 2, so the printed rows always reconcile to the printed total.

The HTML renderer had its own idea. It computed `discountAmount = round2(subtotal × d / 100)` and mixed the result with the *stored* VAT and total.

Take a subtotal of €100.05, a 10% discount and 21% VAT:

```text
domain:  discounted base 90.05   VAT 18.91   total 108.96   discount 10.00
html:    discount 10.01

printed document:  100.05 - 10.01 + 18.91 = 108.95
printed total:                               108.96
```

The document does not add up. A client's bookkeeper will find that, and they will be right. Worse, the same method powered the wizard's preview, so the preview showed one total and the created invoice stored another.

Two implementations of one formula are not redundancy. They are a pending divergence.

### Round Then Sum, or Sum Then Round: Pick One

This one is subtler and I suspect it is very common.

Take three invoice lines, each 1.5 hours at €33.33. That is 49.995 per line, which is not representable in cents.

The invoice lines are persisted into a `NUMERIC(10,2)` column, so Postgres rounds each one to 50.00. Three lines, €150.00. The header subtotal, meanwhile, was computed in memory as `round2(3 × 49.995)`, which is €149.99.

```text
stored lines:     50.00 + 50.00 + 50.00 = 150.00
stored subtotal:  round2(149.985)        = 149.99
```

The invoice detail screen, the HTML render and the PDF all re-sum the lines and disagree with the header by a cent. Then it gets worse: editing the draft (changing nothing but the date) recomputes the subtotal from the *persisted, already-rounded* lines, so the total silently moves from €181.49 to €181.50. Preview, create and edit produce three different numbers for identical input.

The fix is one `setScale(2, RoundingMode.HALF_UP)` on the line total, so lines are rounded before they are summed, matching what the database will store anyway.

The general rule: **decide whether you round per line or per total, apply it everywhere, and make sure the database agrees.** A `NUMERIC(*,2)` column is itself a rounding step, and it will happen whether you planned for it or not.

### An Amount Is Not a Number

Two findings that are really the same finding.

**Invoices did not persist their currency.** It was resolved at read time from the client record. So changing a client's currency retroactively relabelled every historical invoice: a €1.210,00 invoice re-renders as $1,210.00, same digits, different symbol, no conversion. The fix is a migration adding `invoices.currency`, snapshotted at creation. Currency is part of what the invoice *was*, not a property of the client today.

**The reports blended currencies.** `/reports/by-user` and `/reports/summary` summed billable value across all clients and labelled it with the tenant's default currency. A tenant with one EUR client and one USD client got a number that means nothing at all.

There is no FX conversion anywhere in the product, and there should not be one hiding implicitly in a `sum()`. The frontend helper now returns `null` for a mixed-currency sum rather than a plausible-looking number, and the backend partitions totals per currency.

Related, and my favourite in a bleak way: a missing hourly rate fell back to `BigDecimal.ZERO`, which produced a perfectly valid €0.00 invoice line. Billable hours, silently unbilled, on an invoice you can send. **Zero is a real amount, so it is a terrible way to signal "I don't know."**

### Why the Tests Did Not Catch Any of This

The test suite was not thin. It ran against real PostgreSQL via Testcontainers, and it asserted totals. It caught nothing here, for two reasons.

**The numbers were too clean.** Every test used 10 hours at €100. Ten times one hundred is a thousand, at any rounding policy, in any order of operations. A test only exercises rounding if the inputs force a rounding decision: 1.5 × 33.33, VAT on 33.33, values landing exactly on a x.xx5 boundary.

**The PDF tests asserted layout, not arithmetic.** They checked that the document rendered, that the locale formatting was Dutch, that the fields were present. The hardcoded 21% was invisible to every one of them.

So: **test money with ugly numbers**, assert that rendered rows sum to the rendered total, and assert that preview and created invoice agree for identical input. Also assert `BigDecimal` with `compareTo` rather than `equals`, because `2.5` and `2.50` are not equal and one stray scale difference in a `groupBy` will ruin your afternoon.

### How I Actually Found Them

I did not read the code top to bottom. I wrote a review prompt, checked it into `docs/reviews/`, and pointed Claude Code at it across five parallel passes (types and currency, calculation correctness, rounding and boundaries, tenancy, test coverage).

The prompt is not "find bugs in my money code". It encodes the architecture (money flows `NUMERIC(10,2)` → `BigDecimal` → `Amount` → JSON), names the files that duplicate the calculation, states the invariants that must hold, and demands a specific output: severity, location, the concrete input that triggers the bug, the fix, and the test that would have caught it. It also insists on verifying claims against the code rather than assuming.

The findings became 23 numbered tickets, which I then let run through the build loop. The commit log reads `feat(PHASE66-008): render persisted invoice totals in PDF; stop recomputing with hardcoded 21% VAT` and so on down the list.

The reusable artifact here is the prompt, not the report. A review prompt that names your invariants is worth keeping in the repository, because the next review is a week of work otherwise.

### A Few Caveats Keep It Honest

**Perfect parity across display modes is impossible.** Grouping the same time entries by person or by task rounds at different places, so a summary screen and an invoice can legitimately differ by a cent. That is a tolerance to document, not a bug to chase.

**`HALF_UP` is a policy choice, not a law of nature.** It is the right default for Dutch invoicing. Confirm it for your jurisdiction, then apply it uniformly, with no stray `HALF_EVEN` or bare `divide()`.

**A review is a snapshot.** These findings were true on one branch on one day. The value came from writing the invariants down, which is what makes them checkable again later.

### Conclusion

I went looking for money bugs in the arithmetic and found them in the renderers. The domain layer, the part I had been careful about, was fine. The damage was done by code that helpfully recalculated a number it should have simply read, and by an amount that had lost track of its own currency.

Four rules, then:

- **Compute once, store the result, render the stored result.** An invoice is a fact, not a formula.
- **A renderer that does arithmetic is a second implementation.** It will drift.
- **An amount is a number, a currency, and a scale.** Drop any of the three and you have a bug waiting for a second client.
- **Test with ugly numbers.** 1.5 hours at €33.33, not 10 at €100.

None of this is about `Double`. Getting off floating point is table stakes. The money bugs that actually reach a client come from somewhere much more ordinary: two places in the codebase that both think they know what the total is.
