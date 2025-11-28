---
title: what you can do with claude code pt.1 - mining pr reviews
---

## the challenge

Development teams accumulate valuable knowledge in PR review comments: mistake patterns, architectural decisions, coding standards, hard-won lessons.

This knowledge is scattered across hundreds of PRs, and rarely surfaces when developers need it most: before code submission.

## the process

A single conversation analyzed 200 merged PRs from an enterprise codebase and categorized patterns from review comments.

## the prompt

```
Can you analyze the past 200 PRs in the whole repo
and categorize the patterns from the review comments?
```

### step 1: bulk data extraction

Claude used the GitHub CLI to fetch 200 merged PRs with metadata (number, title, author, merge date), then iterated through each PR to extract review comments.

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

Claude extracted specific anti-patterns with concrete fixes for each category.

**Logic and bug detection**

Null reference guards missing. Boolean operator precedence errors. Transaction scope violations. Exception type mismatches.

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

**Data integrity**

Validation bypass through assignment instead of accumulation. Value propagation errors. Serialization breaking changes without versioning.

```csharp
// wrong: overwrites previous validation
valid = ValidateCustomFields();

// correct: accumulates validation failures
valid &= ValidateCustomFields();
```

**Performance**

N+1 query patterns. Loading entire collections when only first item needed. Creating new collections on every property access. Multiple feature flag calls when one suffices.

```csharp
// wrong: creates new HashSet on every access
public HashSet<string> ExcludedItems => _config.ToHashSet();

// correct: cache the result
private readonly HashSet<string> _excludedItems;
```

**Testing patterns**

Async method stubs returning wrong types. Mock configuration missing for new methods. Shared state between tests.

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

**Transaction scope**

HTTP calls inside open database transactions hold locks. Handlers without unit of work write to DB without explicit transaction. Commits before related operations complete.

**Security**

Input validation gaps. Missing CSRF protection. Mass assignment vulnerabilities. Scope mismatches between app registrations.

**Database patterns**

Non-deterministic queries from `FirstOrDefault()` without `OrderBy()`. Missing index hints for performance-critical queries. Foreign key constraints not handled in delete commands.

```sql
-- wrong: non-deterministic
SELECT ... FirstOrDefault()

-- correct: deterministic ordering
SELECT ... ORDER BY Id FirstOrDefault()
```

**API design**

Breaking changes to response keys. Payload format changes without versioning. Missing backward compatibility.

### step 4: knowledge persistence

Using a custom `/learn` command, insights saved to a structured memory file. 
Claude Code references this in future sessions. 
The system teaches itself the team's standards.

## what this enables

New team members learn institutional knowledge before first PR submission. 
Code reviewers focus on architecture and business logic instead of catching repeated patterns. 
The AI assistant gets smarter about the specific codebase with each session.
Learnings can be leveraged by AI to review the code.

The loop closes when tribal knowledge becomes searchable reference.
