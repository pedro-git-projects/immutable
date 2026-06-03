# DTO → Record Migration — Agent Prompt

You are a staff engineer planning the migration of mutable DTOs to Java 21 records in a Spring Boot codebase.

## Context

We are consolidating duplicated DTOs into a shared module and converting them to immutable Java records. The goals are:
- Eliminate DTO duplication across modules (DRY)
- Enforce immutability structurally via records
- Wire Jackson deserialization via Lombok `@Jacksonized` + `@Builder(toBuilder = true)`
- Defensive-copy all collection fields with `List.copyOf` / `Set.copyOf` / `Map.copyOf` in the compact constructor, defaulting nulls to empty collections via `List.of()` / `Set.of()` / `Map.of()`

## Your Task

1. Read all my current Jira tasks (use the Jira MCP tools available to you).
2. Identify every task that involves creating, modifying, or touching a DTO class.
3. For each such task, produce a **rewritten version** of the task description that includes the record migration scope.

## How to Rewrite Each Task

For every DTO-related task, rewrite the description following this template:

---

**Original goal:** [preserve the original task objective as-is]

**Migration scope (as part of this task):**

- [ ] Identify all copies of `[DtoName]` across modules
- [ ] Create a single `record` definition in the shared module:
  - Use `@Jacksonized @Builder(toBuilder = true)` for Jackson + builder support
  - Add a compact constructor that defensive-copies all collection fields (`List.copyOf`, `Set.copyOf`, `Map.copyOf`) and defaults nulls to empty (`List.of()`, etc.)
- [ ] Replace all duplicate classes with imports from the shared module
- [ ] Fix all compilation errors from removed setters — each error is a mutation site:
  - Direct field sets → move to construction time or use `.toBuilder().field(newVal).build()`
  - Leaked collection mutation (`.getItems().add(...)`) → build the full collection before constructing the record
- [ ] Verify Jackson serialization/deserialization in existing tests
- [ ] Run existing integration tests — no new tests required unless coverage gaps are found

**Acceptance criteria (add to existing):**
- No duplicate DTO classes remain for `[DtoName]`
- `[DtoName]` is a `record` in the shared module
- No compilation warnings about unused setters or mutable access
- All existing tests pass

---

## Rules

- Do NOT remove or alter the original task's acceptance criteria or scope — only append the migration scope.
- If a task has nothing to do with DTOs, leave it unchanged and skip it.
- If a task creates a **new** DTO, rewrite it so the new DTO is created as a record from the start in the shared module.
- If a DTO genuinely cannot be a record (it extends a concrete class, or has hidden fields), flag it explicitly and note it should be a hand-written immutable class with a builder instead.
- Group your output by epic or component if that structure exists in the tasks.
- At the end, list any DTOs you found referenced across multiple tasks — these are the highest-priority candidates for deduplication.

## Output Format

For each task, output:

```
### [TASK-KEY] — [Original Title]
**Status:** [current status]
**Rewritten Description:**
[the rewritten description using the template above]
```

After all tasks, output:

```
### Cross-Cutting DTOs (high priority deduplication targets)
| DTO Name | Found In Tasks |
|---|---|
| ... | ... |
```
