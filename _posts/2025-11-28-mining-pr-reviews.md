---
layout: post
title: what you can do with claude code pt.1 - mining pr reviews
---

## the challenge

Every development team accumulates valuable knowledge in their PR review comments—mistake patterns, architectural decisions, coding standards, hard-won lessons learned the painful way. But this knowledge stays scattered across hundreds of PRs, rarely surfacing when developers need it most: before they submit their code.

## the process

In a single conversation, I had Claude Code analyze 200 merged PRs from an enterprise codebase and categorize the patterns hiding in the review comments.

## the prompt

```
Can you analyze the past 200 PRs in the whole repo
and categorize the patterns from the review comments?
```

### step 1: bulk data extraction

Claude used the GitHub CLI to fetch 200 merged PRs with their metadata—number, title, author, merge date—then iterated through each one to extract every review comment.

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

**logic and bug detection**

The most common issues were missing null reference guards, boolean operator precedence errors, transaction scope violations, and exception type mismatches.

```csharp
// wrong: && binds tighter than ||
if (a && b || c && d)

// correct: explicit grouping
if ((a && b) || (c && d))
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

### step 4: knowledge persistence

Using a custom `/learn` command, these insights get saved to a structured memory file that Claude Code references in future sessions. The system effectively teaches itself the team's standards over time.

## what this enables

New team members can absorb institutional knowledge before their first PR submission. Code reviewers can focus on architecture and business logic instead of catching the same patterns over and over. The AI assistant grows smarter about the specific codebase with each session, and these learnings can be leveraged to have AI review the code proactively.

The loop closes when tribal knowledge becomes searchable reference.
