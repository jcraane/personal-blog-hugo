---
layout: post

title: "Multi-Tenancy With Postgres Row-Level Security in Ktor"
subtitle: "Enforcing tenant isolation in the database instead of in every query"
author: Jamie Craane
date: 2026-07-09
description: "How I use Postgres Row-Level Security, a non-owner application role, and a small Kotlin transaction wrapper to guarantee tenant isolation in a production Ktor and Exposed application."
image: "/img/posts/multi-tenancy-postgres-rls.jpg"
showtoc: false
tags:
- Kotlin
- Ktor
- PostgreSQL
- Multi-Tenancy
- Exposed
- Security

categories: [ Kotlin ]
URL: "/2026/09/07/multi-tenancy-postgres-rls/"
---

### Introduction

Moxie IT (the company I co-own) developmed a time tracking and invoicing tool called MinuteMint as an alternative to Harvest. It runs in production and supports multi-tenancy from the first migration: every client, project, time entry and invoice belongs to exactly one tenant.

That raises a question you have to answer on day one, because it is expensive to answer later:

- Where does tenant isolation actually live?
- What happens when a developer forgets to apply it?
- How do you prove, in a test, that a tenant cannot see another tenant's rows?

The default answer in most codebases is "in the query". I think that is the wrong place.

### The Problem: `WHERE tenant_id = ?` Is a Promise You Have to Keep

The obvious approach is to add `tenant_id` to every table and filter on it in every query:

```kotlin
ClientsTable.selectAll().where {
    (ClientsTable.tenantId eq tenantId) and (ClientsTable.active eq true)
}
```

This works right up until it does not. The properties of this approach are worth stating plainly:

- **It is unenforceable.** Nothing in the type system, the ORM, or the database stops you from writing a query without the filter.
- **It fails open.** A forgotten `tenant_id` predicate does not throw. It quietly returns *more* rows, which is precisely the failure mode you least want.
- **It scales with surface area.** Every new query, every new report, every ad-hoc join is another chance to leak. Code review is the only guardrail, and code review gets tired.

A cross-tenant leak in an invoicing product is not a bug, it is an incident. We need the guarantee that it cannot be forgotten.

### The Solution: Push Isolation Into Postgres

Postgres has had Row-Level Security since 9.5. You enable it per table and attach a policy that decides which rows a query may see. The policy reads a session variable that the application sets per transaction:

```sql
ALTER TABLE clients ENABLE ROW LEVEL SECURITY;
ALTER TABLE clients FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON clients
    USING (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::UUID);
```

Each of those three statements does something distinct, and it is worth being precise about which.

**`ENABLE ROW LEVEL SECURITY`** switches the feature on for the table. From that point on, Postgres silently appends the `USING` expression of every applicable policy to each `SELECT`, `UPDATE` and `DELETE` that touches `clients`. Rows for which the expression is not `true` simply do not exist as far as the query is concerned: not returned, not updated, not deleted, not counted. Inserts (and the new version of an updated row) are checked against the policy's `WITH CHECK` expression instead, which defaults to the `USING` expression when you omit it, so a row with the wrong `tenant_id` cannot be written either. A table with RLS enabled and no policy at all is a deny-all: it looks empty.

**`FORCE ROW LEVEL SECURITY`** removes the one built-in exemption. By default a table's owner bypasses that table's own policies, on the reasoning that the owner could drop the table anyway. `FORCE` says the owner gets filtered like everybody else. Superusers, and any role granted `BYPASSRLS`, still ignore policies entirely, and no statement you write on the table changes that.

**`CREATE POLICY`** supplies the predicate. Two details in it carry the safety property.

**`current_setting('app.tenant_id', true)`.** The second argument is `missing_ok`. Without it, a query that runs before the variable is set raises an error instead of returning nothing.

**`NULLIF(..., '')::UUID`.** When the setting is absent, `current_setting` returns an empty string, which is not a valid UUID and would blow up the cast. `NULLIF` turns it into `NULL`, and `tenant_id = NULL` is `NULL`, which is not `true`, so **no rows match**. The policy fails *closed*. That is the entire safety property, and it is worth writing a test for (I do, below).

#### The Owner Bypass

That owner exemption is the part people miss, and it is the one that bites. If your Ktor app connects to Postgres as the same user that ran the migrations, and you wrote only `ENABLE`, you have turned RLS on and changed nothing at all. Every policy is skipped and every query returns every tenant's rows. Nothing errors, nothing warns, and the tests you run over that connection pass happily.

There are two ways out, and I use both, because the local and production environments differ:

1. `FORCE ROW LEVEL SECURITY` makes the policies apply to the table owner too. This covers production, where the app connects as a dedicated, non-superuser database user.
2. A **non-login application role** that the connection switches into for the duration of a transaction. This covers local development, where it is convenient to connect as `postgres`, and superusers ignore even `FORCE`.

```sql
-- V4__App_role.sql
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'minutemint_app') THEN
    CREATE ROLE minutemint_app NOLOGIN;
  END IF;
END $$;

GRANT USAGE ON SCHEMA public TO minutemint_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO minutemint_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO minutemint_app;
```

The role has no password and cannot log in. It exists only to be switched into.

#### Setting the Tenant Context Per Transaction

The application side is one function. It opens an Exposed transaction, drops into the `minutemint_app` role, sets the tenant, and runs your block:

```kotlin
suspend fun <T> Database.withTenant(tenantId: TenantId, block: suspend () -> T): T =
    newSuspendedTransaction(db = this) {
        val safeUuid = UUID.fromString(tenantId.uuid.toString()).toString()
        exec("SET LOCAL ROLE minutemint_app")
        exec("SET LOCAL app.tenant_id = '$safeUuid'")
        block()
    }
```

`SET LOCAL` is the critical keyword. It scopes both settings to the current transaction, so they revert automatically on commit or rollback. That matters enormously with a connection pool: a plain `SET` would persist on the pooled connection and hand the next request, possibly for a different tenant, a live tenant context. `SET LOCAL` makes the leak structurally impossible.

The awkward `UUID.fromString(...).toString()` round-trip is not decoration. `SET LOCAL` does not accept bind parameters, so the value has to be interpolated into the statement text. Re-parsing it as a UUID guarantees that whatever reaches the string is a UUID and nothing else.

With this in place, repository code stops mentioning tenants entirely:

```kotlin
db.withTenant(tenantId) {
    ClientsTable.selectAll().where { ClientsTable.active eq true }
}
```

That query looks single-tenant. It *is* single-tenant, because the database says so.

#### Composability: Nesting Without Losing Atomicity

A real business operation touches several repositories. Creating an invoice reads projects, reads time entries, writes invoice lines, and marks entries as invoiced. If each repository opened its own `withTenant` transaction, the operation would not be atomic, and each call would pay for a connection checkout and two `SET LOCAL` statements.

So `withTenant` joins an already-open transaction when it is for the same tenant, and refuses when it is not:

```kotlin
private val TENANT_MARKER = Key<UUID>()

suspend fun <T> Database.withTenant(tenantId: TenantId, block: suspend () -> T): T {
    val openTenant = TransactionManager.currentOrNull()?.getUserData(TENANT_MARKER)
    if (openTenant != null) {
        if (openTenant != tenantId.uuid) {
            throw IllegalStateException(
                "Nested withTenant requested tenant ${tenantId.uuid} but the open transaction " +
                    "is bound to tenant $openTenant, refusing to cross tenant boundaries."
            )
        }
        return block()
    }
    return newSuspendedTransaction(db = this) {
        // set role + tenant, then:
        putUserData(TENANT_MARKER, tenantId.uuid)
        block()
    }
}
```

Nesting for the same tenant is free and atomic. Nesting for a *different* tenant throws before any context is established. Isolation only ever gets stronger as you nest, never weaker.

#### The Escape Hatch: Global Access

Login breaks the model. Someone posts an email address and a password, and at that moment there is no tenant to set: working out which tenant they belong to is the *point* of the lookup. A policy keyed on `app.tenant_id` matches nothing, the user row stays invisible, and nobody logs in.

So `users` carries a second policy, gated on a different setting:

```sql
CREATE POLICY global_access ON users
    USING (current_setting('app.global_access', true) = 'true');
```

Permissive policies are ORed together. A transaction that sets `app.global_access` sees every row in `users`; one that does not still falls back to `tenant_isolation`. Only one function sets it:

```kotlin
suspend fun <T> Database.withGlobalAccess(block: suspend () -> T): T =
    newSuspendedTransaction(db = this) {
        exec("SET LOCAL ROLE minutemint_app")
        exec("SET LOCAL app.global_access = 'true'")
        block()
    }
```

Four call sites use it: login, password reset, email verification, and the cross-tenant numbers on our admin dashboard. Inside the block, RLS is off. The name is long and awkward to type on purpose, so that a fifth call site is hard to add without someone noticing it in the diff.

### Testing That It Actually Works

How do you test Row Level Security within unit test and actually verify that it works?. The trap here is that RLS does not exist on H2, so tests against an in-memory database will happily pass while proving nothing.

MinuteMint's tests run against real Postgres via Testcontainers, and the setup deliberately mirrors production: migrations run as the owner, then a **separate non-owner user** is created for the application connection pool.

```kotlin
transaction(ownerDb) {
    exec("CREATE USER app_user WITH PASSWORD 'app_pass'")
    exec("GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user")
    exec("GRANT minutemint_app TO app_user")
}
```

That lets me assert the property directly, including the one that matters most:

```kotlin
@Test
fun `given a transaction without tenant context when querying an RLS table then it fails closed returning zero rows`() =
    runBlocking {
        val tenantA = insertTenant("Tenant Fail Closed")
        clientRepo.save(buildClient(tenantA, "Hidden Client"))

        // No SET LOCAL app.tenant_id, so the tenant_isolation policy matches nothing.
        val rows = newSuspendedTransaction(db = db) {
            exec("SET LOCAL ROLE minutemint_app")
            ClientsTable.selectAll().count()
        }
        assertEquals(0L, rows)
    }
```

Forget the tenant context and you get zero rows, not everybody's rows. Sibling tests cover the rollback of joined transactions and the rejection of cross-tenant nesting.

### A Few Caveats Keep It Honest

**RLS is not free.** Every policy is a predicate appended to every query on that table. It is cheap, but it is not nothing, and it can affect plans. Index your `tenant_id` columns.

**New tables are opt-in.** A migration that creates a table without `ENABLE`, `FORCE`, and a policy has created an unprotected table. Nothing warns you. This is the weakest link in the whole design, and the honest mitigation is a checklist plus a test that enumerates tables and asserts each one has RLS enabled.

**`SET LOCAL` cannot be parameterized.** You are interpolating a string into SQL. Validate the type first, as above, and never let a raw request value near it.

**Global access is a real bypass.** Every `withGlobalAccess` call site deserves scrutiny, because inside that block RLS is off.

### Conclusion

Where should tenant isolation live? Not in a filter you have to remember, but in the layer that cannot be talked out of it.

The pieces are small: a `tenant_id` column, a policy that fails closed, `FORCE ROW LEVEL SECURITY`, a non-login role, and one Kotlin function called `withTenant`. Together they change the failure mode from "a forgotten `WHERE` clause leaks another tenant's invoices" to "a forgotten `withTenant` returns nothing and the feature is visibly broken in development".

I would rather debug an empty list than explain a data breach.

### References

- [PostgreSQL: Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [PostgreSQL: SET](https://www.postgresql.org/docs/current/sql-set.html)
- [Exposed](https://github.com/JetBrains/Exposed)
