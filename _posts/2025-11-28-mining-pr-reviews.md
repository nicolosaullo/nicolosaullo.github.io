---
layout: post
title: claude code - mining pr reviews
---

## the challenge

Every development team accumulates valuable knowledge in their PR review comments—mistake patterns, architectural decisions, coding standards, hard-won lessons learned the painful way.

But this knowledge stays scattered across hundreds of PRs, rarely surfacing when developers need it most: before they submit their code.

## the process

After accumulating a backlog of PR feedback — I got curious about what patterns reviewers were actually seeing. Was I making the same mistakes repeatedly? Which categories kept coming up?

Instead of reading through hundreds of review threads manually, I had Claude Code do it, for all users in the repository.

## the prompt

```
Analyze the past 200 PRs in the repository and categorize the patterns from the review comments.
```

One sentence. That's the entire input. What a human analyst would take days to produce manually — reading threads, spotting patterns, writing them up — came back in minutes.

### step 1: bulk data extraction

Claude used the GitHub CLI to fetch 200 merged PRs with their metadata, then iterated through each one to pull every review comment thread.

```bash
# list merged PRs
gh pr list --state merged --limit 200 --json number,title,author,mergedAt > prs.json

# for each PR, extract review comments
gh api repos/{owner}/{repo}/pulls/{number}/comments
```

### step 2: pattern recognition

From hundreds of individual comments, Claude identified 10 major categories:

| Category | Frequency |
|----------|-----------|
| Logic and bug detection | ~25% |
| Code quality and style | ~20% |
| Data integrity issues | ~15% |
| Performance | ~10% |
| Architecture and design | ~8% |
| Testing patterns | ~7% |
| Security | ~5% |
| Database and SQL | ~5% |
| Documentation | ~3% |
| API design | ~2% |

### step 3: pattern extraction

For each category, Claude extracted specific anti-patterns along with concrete fixes.

<details markdown="1">
<summary>view extracted patterns</summary>

**logic and bug detection**

The most common issues were missing null reference guards, boolean operator precedence errors, transaction scope violations, and exception type mismatches.

```csharp
// wrong: intended "(guest or authenticated) AND terms accepted"
// actual: guest always gets access regardless of terms
if (isGuest || isAuthenticated && hasAcceptedTerms)

// correct: explicit grouping enforces the intended logic
if ((isGuest || isAuthenticated) && hasAcceptedTerms)
```

```csharp
// wrong: catches too narrow
catch (JsonReaderException)

// correct: catches all JSON issues
catch (JsonException)
```

**data integrity**

Reviewers frequently caught validation bypass bugs where assignment was used instead of accumulation, along with value propagation errors and serialization breaking changes that lacked versioning.

```csharp
// wrong: overwrites previous validation
valid = ValidateCustomFields();

// correct: accumulates validation failures
valid &= ValidateCustomFields();
```

**performance**

The usual suspects appeared: N+1 query patterns, loading entire collections when only the first item was needed, creating new collections on every property access, and calling feature flags multiple times when once would suffice.

```csharp
// wrong: creates new HashSet on every access
public HashSet<string> ExcludedItems => _config.ToHashSet();

// correct: cache the result
private readonly HashSet<string> _excludedItems;
```

**testing patterns**

Async method stubs returning the wrong types came up repeatedly, along with mock configurations missing for new methods and shared state bleeding between tests.

```csharp
// wrong: test stubs return wrong type for async
_repository.GetByIdAsync(id).Returns(entity);

// correct: async methods need Task wrapper
_repository.GetByIdAsync(id).Returns(Task.FromResult(entity));

// wrong: null async stub
_repository.GetByIdAsync(id).ReturnsNull();

// correct: null wrapped in Task
_repository.GetByIdAsync(id).Returns(Task.FromResult<Entity>(null));
```

**transaction scope**

HTTP calls inside open database transactions that hold locks too long, handlers that write to the database without explicit transactions, and commits that fire before related operations complete.

**security**

Input validation gaps, missing CSRF protection, mass assignment vulnerabilities, and scope mismatches between app registrations.

**database patterns**

Non-deterministic queries from `FirstOrDefault()` without `OrderBy()`, missing index hints on performance-critical queries, and foreign key constraints not handled properly in delete commands.

```sql
-- wrong: non-deterministic
SELECT ... FirstOrDefault()

-- correct: deterministic ordering
SELECT ... ORDER BY Id FirstOrDefault()
```

**api design**

Breaking changes to response keys, payload format changes without versioning, and missing backward compatibility.

</details>

### step 4: knowledge persistence

Using a custom `/learn` skill — a reusable Claude Code skill defined in `.claude/skills/learn/SKILL.md` that appends structured entries to a `lessons.md` file in the repo — these insights get saved to a memory file that Claude Code references in future sessions. The system effectively teaches itself the team's standards over time.

Here's the actual output — seven entries written to `lessons.md`:

<details markdown="1">
<summary>view lessons.md entries</summary>

---

**Entry 1 — logic: boolean operator precedence**

`&&` binds tighter than `||`. When both appear in the same condition without parentheses, the evaluation order is likely not what the developer intended — and in access-control logic, it becomes a security bug. Always use explicit grouping.

```csharp
// ❌ Reads like "(guest or authenticated) AND terms accepted"
// Evaluates as: isGuest || (isAuthenticated && hasAcceptedTerms)
// Guests bypass the terms check entirely
if (isGuest || isAuthenticated && hasAcceptedTerms)

// ✅ Parentheses enforce the intended logic
if ((isGuest || isAuthenticated) && hasAcceptedTerms)
```

---

**Entry 2 — logic: exception type too narrow**

Catching `JsonReaderException` misses other failure modes in the `System.Text.Json` and `Newtonsoft.Json` hierarchies. Catch the base `JsonException` unless there is a deliberate reason to handle a specific subtype.

```csharp
// ❌ Misses serialization, path, and writer exceptions
catch (JsonReaderException ex)

// ✅ Catches the full JSON exception hierarchy
catch (JsonException ex)
```

---

**Entry 3 — data integrity: validation accumulation**

Using `=` to collect validation results silently discards earlier failures — the last assignment wins. Use `&=` so any failure propagates through to the final result.

```csharp
// ❌ Overwrites — earlier failures are lost
valid = ValidateRequiredFields();
valid = ValidateCustomFields();

// ✅ Accumulates — any failure is preserved
valid &= ValidateRequiredFields();
valid &= ValidateCustomFields();
```

---

**Entry 4 — performance: HashSet allocated on every property access**

A property getter that calls `.ToHashSet()` constructs a new collection on every read. If the property is accessed in a loop this creates significant allocation pressure. Cache the result in a backing field.

```csharp
// ❌ New HashSet allocated on every access
public HashSet<string> ExcludedItems => _config.Items.ToHashSet();

// ✅ Constructed once in the constructor
private readonly HashSet<string> _excludedItems;
public MyService(Config config) {
    _excludedItems = config.Items.ToHashSet();
}
public HashSet<string> ExcludedItems => _excludedItems;
```

---

**Entry 5 — testing: async NSubstitute stubs must wrap in Task**

NSubstitute stubs for `async` methods must return `Task.FromResult(value)`, not the value directly. Passing the entity compiles but the stub will not resolve correctly. Null returns also need explicit wrapping — `ReturnsNull()` does not work for async methods.

```csharp
// ❌ Wrong — compiles but stub doesn't resolve
_repository.GetByIdAsync(id).Returns(entity);
_repository.GetByIdAsync(id).ReturnsNull();

// ✅ Correct — Task wrapper required for both cases
_repository.GetByIdAsync(id).Returns(Task.FromResult(entity));
_repository.GetByIdAsync(id).Returns(Task.FromResult<Entity>(null));
```

---

**Entry 6 — transaction scope: HTTP calls inside open transactions**

Making HTTP calls while a database transaction is open holds the connection and any acquired locks for the duration of the network round-trip. This is a common cause of lock contention and deadlocks under load. Perform all external calls before opening the transaction, or after committing.

```csharp
// ❌ HTTP call holds the DB transaction open
using var tx = await _db.BeginTransactionAsync();
var result = await _httpClient.PostAsync(endpoint, payload); // locks held here
await _db.SaveChangesAsync();
await tx.CommitAsync();

// ✅ External call completed before transaction opens
var result = await _httpClient.PostAsync(endpoint, payload);
using var tx = await _db.BeginTransactionAsync();
await _db.SaveChangesAsync();
await tx.CommitAsync();
```

---

**Entry 7 — database: FirstOrDefault without OrderBy is non-deterministic**

SQL Server does not guarantee row order without an explicit `ORDER BY`. `FirstOrDefault()` on an unordered query returns stable results in development but can vary in production as index state changes. Always pair `First`/`FirstOrDefault` with explicit ordering.

```csharp
// ❌ Non-deterministic — depends on index state
var latest = _db.Invoices
    .Where(x => x.ClientId == id)
    .FirstOrDefault();

// ✅ Deterministic — explicit ordering
var latest = _db.Invoices
    .Where(x => x.ClientId == id)
    .OrderByDescending(x => x.CreatedAt)
    .FirstOrDefault();
```

---

</details>

## what this enables

Manually going through 200 PRs — reading every thread, extracting the pattern, writing it up — is days of work that almost never gets done. The value is invisible until the knowledge is already structured, so it stays trapped in review history that nobody reads.

This ran in one conversation.

For me personally, seeing the categories laid out was clarifying in a way that individual review comments never are. A comment saying "wrap this in a transaction" is easy to fix and forget. A chart showing that transaction scope issues made up a significant slice of all feedback over months of PRs is harder to ignore.

The compounding benefit is subtler: every future session that references `lessons.md` means the assistant already knows the specific failure modes reviewers have flagged. It starts catching them before a human reviewer ever sees the code.

The loop closes when tribal knowledge becomes searchable reference.
