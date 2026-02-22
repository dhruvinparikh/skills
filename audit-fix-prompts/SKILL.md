# Generating AI Fix Prompts from Security Audit Findings

> Guidelines for transforming raw security audit findings (e.g., from Spearbit, Trail of Bits, OpenZeppelin, Code4rena) into structured AI-agent prompts that reliably produce correct, test-backed code fixes.

---

## When to Use

Use this skill when you have one or more security findings from an audit report and need to produce a single markdown prompt file that an AI coding agent can consume to implement all fixes autonomously.

---

## Output Format

The output is a single markdown file (e.g., `spearbit-ai-prompt.md`) placed at the repo root. The file contains:

1. A **preamble** instructing the agent to fix all findings in severity order.
2. One **finding section** per issue, each self-contained with enough context for the agent to locate, understand, and fix the vulnerability without additional searching.

---

## Prompt Structure

Use the template at TEMPLATE.MD as the starting point for every new prompt file. Copy it to the repo root, rename it `{auditor}-ai-prompt.md`, and fill in each `{PLACEHOLDER}`.

The template encodes the exact section order, field names, and comment guidance described below. Refer to it for the canonical structure; the rest of this document explains the reasoning behind each section.

### Preamble

```markdown
You are fixing all findings in this scan.

Address each finding below in order of severity.
```

Keep it terse. The agent should understand it is implementing code changes, not writing a report.

### Per-Finding Section

Each finding uses the structure defined in TEMPLATE.MD:

| Section | Purpose |
|---------|---------|
| **Header** (`## {ID} — {Title}`) | Unique ID + descriptive title |
| **Summary** | Self-contained technical explanation of the vulnerability |
| **Context Files** | Every file in the call chain the agent needs to understand and fix the issue |
| **Cross-Finding Dependencies** | (Optional) Notes when findings overlap and should share a fix |
| **Proof of Concept** | Concrete numbered exploit steps with parameter values |
| **Recommended Fix Guidance** | Prescriptive instructions with code snippets |
| **Assumptions** | Annotated `[holds]` / `[does not hold]` / `[needs verification]` |
| **Constraints** | Standard + finding-specific constraints |

---

## Authoring Guidelines

### 1. Severity Ordering

Order findings from highest to lowest severity within the prompt. The agent processes them sequentially; fixing critical issues first prevents cascading conflicts.

Typical ordering: Critical > High > Medium > Low > Informational.

### 2. Context File Selection

Include **every file in the call chain** needed to understand the vulnerability. The agent should not need to search the codebase to comprehend the finding. Prioritize:

- The vulnerable function itself (with full signature and body)
- Callers that pass attacker-controlled input
- Callees that execute the dangerous operation
- Related storage structures or type definitions
- Test helpers that configure the vulnerable path (e.g., authority/role setup)

Use the `Focus lines:` field to direct the agent's attention to the specific lines that matter. For large files, include only the relevant excerpt with enough surrounding code for unambiguous identification.

### 3. Summary Depth

The summary must be **self-contained**. Write it so someone unfamiliar with the codebase can understand:

- What the code is supposed to do
- What it actually does
- Why the gap is exploitable
- What the impact is (e.g., "steal escrowed assets", "drain keeper ETH")

Avoid vague language. Replace "could be problematic" with "allows an attacker to extract X from Y by doing Z."

### 4. Proof of Concept Specificity

A good PoC is a numbered sequence of **concrete actions** with parameter values:

```markdown
1. Attacker chooses _shares = 100 (≤ SHARE.balanceOf(vault)).
2. Attacker calls instantRedeemWithPermit2(100, attackerAddr, attackerAddr, _permit2=EOA, 0, type(uint256).max, "").
3. _permit2.permitTransferFrom(...) succeeds as a no-op (EOA has no code).
4. Vault burns 100 shares from its own escrowed balance and sends assets to attackerAddr.
```

Bad PoC: "An attacker could potentially misuse this function."

### 5. Fix Guidance Precision

The recommended fix should be implementable without ambiguity:

**Good:**
```markdown
- Store PERMIT2 as an immutable in the constructor. Remove the _permit2 parameter from all Permit2 functions.
- In instantRedeemWithPermit2, add a balance-delta check:
  uint256 pre = SHARE.balanceOf(address(this));
  PERMIT2.permitTransferFrom(...);
  require(SHARE.balanceOf(address(this)) - pre >= _shares, "transfer failed");
```

**Bad:**
```markdown
- Consider validating the permit2 address somehow.
- Maybe add some checks.
```

### 6. Assumption Annotations

Mark each assumption with `[holds]`, `[does not hold]`, or `[needs verification]` based on your verification of the codebase. This tells the agent which assumptions it can rely on and which need further investigation:

```markdown
- [holds] The authority configuration makes this function callable by arbitrary EOAs.
- [does not hold] The OFT enforces options inspection (no msgInspector is configured).
- [needs verification] Whether ISwapRouter exposes swapExactAmountIn.
```

Only include assumptions that are **relevant to the exploit path**. The agent uses these to validate that the fix addresses the actual root cause.

### 7. Constraints Block

Always include the standard constraints block at the end of each finding. Add finding-specific constraints when needed:

```markdown
Constraints:
- Do not introduce behavior regressions.
- Preserve intended business logic.
- Add or update tests for changed behavior.
- Do not modify the LayerZero vendored code in src/layerzero/.
```

### 8. Cross-Finding Dependencies

When findings overlap (e.g., two findings both involve sanitizing `SendParam` in the same relay path), note this explicitly so the agent can implement a single cohesive fix:

```markdown
Note: This finding overlaps with {OTHER_ID} ({brief description}). The fix for
{shared mechanism} should address both issues.
```

---

## Workflow

### Step 1: Triage the Audit Report

Read the full audit report. For each finding:
- Confirm severity classification
- Verify the finding against the current codebase (the code may have changed since the audit snapshot)
- Mark assumptions as `[holds]` or `[does not hold]`
- Skip findings that are already fixed, accepted risks, or disputed

### Step 2: Gather Context

For each confirmed finding, collect:
- The vulnerable code (exact file paths and line numbers in the current repo)
- The full call chain from entry point to exploit
- Relevant configuration (authority roles, constructor params, storage layout)
- Existing tests that exercise the vulnerable path

### Step 3: Draft the Prompt

Copy TEMPLATE.MD to the repo root and rename it `{auditor}-ai-prompt.md`. Fill in each finding section following the template's inline guidance. Review for:
- **Completeness:** Can the agent fix this without searching the codebase?
- **Precision:** Are file paths and line numbers correct for the current HEAD?
- **Actionability:** Does the fix guidance tell the agent exactly what to do?

### Step 4: Order and Finalize

- Sort findings by severity (Critical → Informational)
- Check for cross-finding dependencies and add notes
- Verify the preamble is present
- Remove all HTML comment instructions from the template
- Save the final file at the repo root

### Step 5: Execute and Verify

Run the prompt through the AI agent. After fixes are applied:
- Verify all tests pass (`forge test`)
- Verify the build is clean (`forge build`)
- Review each fix against the original finding to confirm the root cause is addressed
- Create a branch, commit, and push for review

---

## Common Pitfalls

| Pitfall | Consequence | Prevention |
|---------|-------------|------------|
| Stale file paths / line numbers | Agent edits wrong code or fails to locate target | Verify paths against current HEAD before finalizing |
| Vague fix guidance | Agent invents a plausible but incorrect fix | Include code snippets and specific variable/function names |
| Missing call-chain context | Agent doesn't understand how attacker-controlled input reaches the vulnerable code | Include caller → callee chain as separate Context Files |
| Overlapping findings without cross-references | Agent implements conflicting fixes for related issues | Add explicit cross-finding dependency notes |
| Missing test guidance | Agent skips test updates or writes superficial tests | Mention specific test scenarios in fix guidance (e.g., "add a test where _permit2 is an EOA") |
| Assumptions not verified | Agent fixes a non-issue or misses the real root cause | Always verify `[holds]` / `[does not hold]` against the actual codebase |

---

## Reference

- **Template:** [TEMPLATE.md](./TEMPLATE.md) — Copy this file to start every new prompt.
