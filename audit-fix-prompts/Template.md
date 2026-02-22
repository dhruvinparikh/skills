<!--
  TEMPLATE FOR AI FIX PROMPTS
  ============================
  Copy this file to the repo root and rename it:  {auditor}-ai-prompt.md
  Replace every {PLACEHOLDER} with the actual value for your finding(s).
  Delete any HTML comments before committing.

  Reference: ./SKILLS.MD
-->

You are fixing all findings in this scan.

Address each finding below in order of severity.

---

<!--
  ┌──────────────────────────────────────────────────────┐
  │  Repeat this entire section for each finding.        │
  │  Order: Critical → High → Medium → Low → Info.      │
  │  Use a unique ID per finding (e.g., PROJ-1, PROJ-2).│
  └──────────────────────────────────────────────────────┘
-->

## {FINDING_ID} — {Short Descriptive Title}

You are fixing a security finding.

Finding: {Short Descriptive Title}

<!--
  SUMMARY
  Write a self-contained technical explanation. A reader unfamiliar with the
  codebase should understand: what the code does, what it should do, why the
  gap is exploitable, and the concrete impact.

  Tips:
  - Name the functions, variables, and storage slots involved.
  - Explain the attacker's control over inputs.
  - Quantify impact where possible ("drain X from Y", "bypass Z invariant").
  - Avoid vague language like "could be problematic".
-->

Summary:

{Detailed technical explanation of the vulnerability. Cover:
 - What the vulnerable code does and why it is wrong.
 - The specific mechanism by which the issue is triggered or exploited.
 - The asset, invariant, or protocol property at risk.
 - Why existing guards (if any) are insufficient.}

<!--
  CONTEXT FILES
  Include every file the agent needs to locate and understand the vulnerability
  without searching further. Multiple context blocks are encouraged.

  Guidelines:
  - Use relative paths from the repo root.
  - Keep snippets focused but include enough surrounding code for
    unambiguous identification (function signatures, key state variables).
  - For large files, excerpt only the relevant section.
  - Show the full caller → callee chain when the vulnerability spans
    multiple functions.
-->

Context Files:

**File:** `{relative/path/to/file.ext}`
**Language:** {solidity|typescript|rust|...}
**Focus lines:** {start}-{end}

```{language}
{Relevant code snippet. Include:
 - Full function signatures
 - Key variable declarations / storage reads
 - The vulnerable logic with surrounding context
 - Use "// ..." to elide irrelevant lines}
```

<!-- Add additional context files as needed: callers, callees, interfaces,
     type definitions, storage layouts, test helpers, configuration. -->

**File:** `{relative/path/to/another_file.ext}`
**Language:** {language}
**Focus lines:** {start}-{end}

```{language}
{Code snippet}
```

<!--
  CROSS-FINDING DEPENDENCIES (optional)
  If this finding overlaps with another (shared root cause, shared code path,
  or fix that addresses both), note it here so the agent produces a single
  cohesive change rather than conflicting patches.
-->

<!-- Example:
Note: This finding overlaps with {OTHER_ID} ({brief description}). The fix
for {shared mechanism} should address both issues in a single change.
-->

<!--
  PROOF OF CONCEPT
  A numbered sequence of concrete actions with parameter values and
  observable outcomes. Include preconditions, actor (user / attacker /
  keeper), function calls, and the resulting state.

  Bad: "An attacker could potentially misuse this function."
  Good: "1. Attacker calls foo(bar=0xdead, amount=1e18). 2. ..."
-->

Proof of Concept:

{Setup: describe initial state, balances, configuration, roles.}

1. {Actor} calls `{function}({param}={value}, ...)`.
2. {Internal step}: `{code path}` executes because {reason}.
3. {Observable outcome}: {state change, balance delta, revert, etc.}.
4. **Result:** {Impact statement — e.g., "attacker receives X tokens they did not own", "message fails on destination, stranding user funds"}.

<!--
  RECOMMENDED FIX GUIDANCE
  Be prescriptive. The agent should be able to implement the fix without
  ambiguity. Include code snippets for non-trivial changes. Provide a
  primary recommendation and optional defense-in-depth measures.

  Bad: "Consider validating the input somehow."
  Good: "In _payNative(), replace _nativeFee with the cached endpoint fee.
         Add a require(endpointFee > 0) guard. See code snippet below."

  If multiple valid approaches exist, rank them and explain trade-offs so
  the agent picks the best one for the codebase.
-->

Recommended Fix Guidance:

**Primary fix:**

- {Precise instruction: WHAT to change, WHERE to change it, and WHY.}
- {Include code snippets for non-trivial changes:}

```{language}
// In {function}(), replace:
{old code}

// With:
{new code}
```

- {Additional changes needed in related files, if any.}

**Defense-in-depth (optional):**

- {Secondary measure: e.g., add a require guard, emit an event, add a view helper.}
- {Test guidance: specific scenarios to add or update.}

<!--
  ASSUMPTIONS
  List the assumptions the finding depends on.
  Annotate each with [holds], [does not hold], or [needs verification]
  based on your review of the current codebase at HEAD.

  The agent uses these to confirm the exploit path is valid and the fix
  addresses the actual root cause.
-->

Assumptions:

- [{holds | does not hold | needs verification}] {Assumption statement — e.g., "The function is callable by any EOA via the authority config."}
- [{holds | does not hold | needs verification}] {Another assumption.}

<!--
  CONSTRAINTS
  Always include the standard three constraints.
  Add finding-specific constraints as needed (e.g., do not modify vendored
  code, preserve gas efficiency, maintain upgrade compatibility).
-->

Constraints:

- Do not introduce behavior regressions.
- Preserve intended business logic.
- Add or update tests for changed behavior.
<!-- Add finding-specific constraints below: -->
- {e.g., Do not modify vendored code.}
- {e.g., Fix must apply identically.}

---

<!--
  ┌──────────────────────────────────────────────────────┐
  │  Copy everything between the --- separators above    │
  │  for each additional finding. Adjust the ID, title,  │
  │  and all content.                                    │
  └──────────────────────────────────────────────────────┘
-->
