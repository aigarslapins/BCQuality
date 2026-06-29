---
kind: action-skill
id: al-quality-review
version: 1
title: AL code quality review
description: "Reviews AL source changes for structural quality: single responsibility, separation of concerns, cyclomatic complexity, strict typing, input completeness, naming, consistency with surroundings, comment hygiene, and duplication."
inputs: [pr-diff, file-path]
outputs: [findings-report]
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# AL code quality review

Reviews AL source changes for structural code quality concerns and emits a findings report. This is a leaf action skill: it invokes no sub-skills. It is composable by a super-skill reviewing AL code.

An orchestrator invokes this skill with either a `pr-diff` (the standard PR-review entry point) or a `file-path` (single-file review). The skill produces a single JSON document conforming to the DO output contract.

## Source

Read the BCQuality knowledge index once and take the entries whose `domain` is `code-quality` as this skill's candidate set across every enabled layer. Do not open the individual article files at this step. Open an article's full body only once it enters the Worklist below.

## Relevance

Apply the frontmatter matching rules defined in READ (*Frontmatter matching semantics*) against the task context:

- `bc-version` — the target BC version from the PR branch's `app.json` or the orchestrator-supplied version. If unavailable, the dimension is `unknown`.
- `technologies` — `[al]`.
- `countries` — the countries declared in the consuming app's `app.json`. Default to the orchestrator's configured context; if absent, `unknown`.
- `application-area` — the union of application areas declared by the changed objects. If the area cannot be determined, the dimension is `unknown`.

Discard files that are not applicable. Retain conditionally applicable files (any dimension `unknown`) only when the orchestrator's configuration permits them; findings derived from those files MUST have `confidence` no higher than `medium`, and the finding's `message` MUST name the unknown dimension(s).

## Worklist

Narrow the relevant knowledge files to the subset that applies to the changes under review. For each relevant file, compute overlap against the changed AL object names, types, procedure signatures, and tokens from the diff.

A knowledge file enters the candidate worklist when its `keywords` intersect the extracted tokens, or its topic matches a changed object type or procedure pattern. Read an article's full body only after it makes the worklist; candidate selection uses the index alone.

Once the candidate worklist is known, resolve layer-precedence conflicts per READ. Drop lower-precedence files whose normative guidance directly contradicts a higher-precedence candidate and record each in `suppressed` with `reason: "layer-precedence"`. Files suppressed by a disabled layer are recorded with `reason: "configuration"`.

When the post-conflict worklist is empty, the skill continues to the agent evaluation in the Action step and does NOT emit `outcome: "no-knowledge"` — the agent-level rules in this skill apply regardless of the knowledge corpus state.

## Action

For each worklist entry, evaluate the diff against the file's `## Best Practice` and `## Anti Pattern` sections and emit findings per the DO severity and confidence rules.

After evaluating knowledge files, evaluate every changed procedure and trigger against the following quality rules. Each violation is emitted as an agent finding with `references: []`, an `id` prefixed with `agent:`, `confidence` capped at `medium`, and `severity` capped at `minor`. A finding's `message` MUST be self-contained: it names the violation, gives a concrete location reference, and states a specific remedy.

**Rule 1 — Single responsibility.** Flag any procedure or trigger whose description requires the word "and" to name what it does. Name both concerns in the finding and propose a concrete split — a new procedure name for each extracted responsibility.

**Rule 2 — Separation of concerns.** Flag pure logic (a calculation, a validation, a transformation) that directly calls `Record.FindSet`, `Record.Find`, `Record.Get`, `HttpClient.Send`, an event publisher, or a BC REST client function inside the same procedure body. The finding's message must name the I/O call, the line it appears on, and how to push it to the caller.

**Rule 3 — Cyclomatic complexity.** Compute the complexity of every changed procedure and trigger: start at 1, then add 1 for each `if`, `elseif`, `and`, `or`, named `case` value (each label counts), `for`, `foreach`, `while`, and `repeat`. Flag every procedure whose count exceeds 4 (hard ceiling, not a target). In the finding's `message`, give the computed count, the line number of the worst-scoring branch, and a concrete remedy: extract a named helper procedure, replace a branch chain with a dispatch table, or introduce an early `exit` guard. Example: "complexity 6 at lines 12–29: extract the stamp-parse branch into `ParseStamp()` to reduce it to 3."

**Rule 4 — AL strict, no escapes.** Flag:
- A `Variant` type declared for a parameter or variable where a specific AL type would suffice.
- A compiler-warning suppression comment (for example `// AL0604` or `#pragma warning disable`).
- An `HttpClient` method call that sends a request where the return value is not checked and no error handling surrounds the call, leaving network or HTTP failures to silently continue.

**Rule 5 — Input completeness and loud failure.** Flag a procedure that accepts a parameter value but silently ignores an invalid or out-of-range input without calling `Error()`, `FieldError()`, or otherwise terminating the flow early. The target anti-pattern is silent fall-through: the procedure reaches its end with unhandled input state and returns as if it succeeded.

**Rule 6 — Readability, naming, and simplicity.** Flag:
- A procedure name that is not a PascalCase verb phrase (for example `ProcessOrder`, `ValidateEntry`, `GetCustomer` are correct; `processorder`, `Validate_Entry`, `Customer` are not).
- A local variable name that is not a camelCase noun (for example `salesHeader`, `lineCount`, `isPosted` are correct).
- A construct that achieves a simple goal through needless indirection when a straightforward AL expression is available. Apply the precision bar strictly: flag only clearly needless cleverness, not a valid alternative approach.

**Rule 7 — Consistency with surroundings.** Flag object naming, procedure casing, or local variable declaration placement that diverges from the pattern established by neighbouring objects visible in the diff or file. Local variable declarations must appear at the top of the procedure body; declarations intermixed with statements are a violation.

**Rule 8 — Comment hygiene.** Flag:
- A decorative comment (a divider, a banner, a section header that adds no information).
- A comment that restates what the immediately following line of code does rather than explaining why it does it.

**Rule 9 — Duplication.** Flag two or more procedure bodies in the diff that perform the same logical operation — the same sequence of record lookups, the same validation pattern, the same transformation — and could be replaced by a single shared helper procedure. Name the duplicated procedures and propose a helper name.

Apply the agent-finding precision bar from `skills/do.md` to every candidate: steelman the code as written before emitting, and omit any finding that is merely stylistic preference, speculative, or dependent on code outside the diff.

Return `not-applicable` when the input contains no AL changes. Return `completed` otherwise, including when `findings` is empty.

## Output

Output conforms to the DO output contract. A populated example:

```json
{
  "skill": { "id": "al-quality-review", "version": 1 },
  "outcome": "completed",
  "summary": {
    "counts": { "blocker": 0, "major": 0, "minor": 3, "info": 0 },
    "coverage": { "worklist-size": 0, "items-evaluated": 0 }
  },
  "findings": [
    {
      "id": "agent:cyclomatic-complexity",
      "severity": "minor",
      "message": "Procedure `PostSalesDocument` has complexity 7 (line 34 is the worst branch: a nested if inside a case). Extract the inner validation into `ValidateShipmentDate()` to reduce it to 4.",
      "location": {
        "file": "src/Sales/SalesPost.Codeunit.al",
        "line": 22,
        "range": { "start-line": 22, "end-line": 61 }
      },
      "references": [],
      "confidence": "high"
    },
    {
      "id": "agent:separation-of-concerns",
      "severity": "minor",
      "message": "Procedure `CalculateDiscount` calls `Customer.Get(customerNo)` directly at line 88. Pure discount logic should receive the customer record as a parameter; the caller owns the I/O.",
      "location": {
        "file": "src/Pricing/DiscountMgmt.Codeunit.al",
        "line": 88
      },
      "references": [],
      "confidence": "medium"
    },
    {
      "id": "agent:comment-hygiene",
      "severity": "minor",
      "message": "Comment on line 14 restates what the next line does (`// Get the customer`). Remove it; the identifier is self-explanatory.",
      "location": {
        "file": "src/Pricing/DiscountMgmt.Codeunit.al",
        "line": 14
      },
      "references": [],
      "confidence": "high"
    }
  ],
  "suppressed": []
}
```
