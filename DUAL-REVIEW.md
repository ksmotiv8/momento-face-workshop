# Exercise: Claude Code launches Codex as a second reviewer

Two models reviewing the same diff catch more than one model reviewing it
twice: different training, different blind spots. This pattern runs the
review inside Claude Code, with Claude as reviewer one and the installed
Codex CLI as reviewer two, then forces a reconciliation where agreement
is a signal but never proof.

## Setup

- Claude Code in the repo you want reviewed.
- Codex CLI installed and authenticated (`codex --help` works).
- Uncommitted or branch changes to review.

## The prompt

> Review the current code changes using two independent reviewers:
> yourself and the installed Codex CLI.
>
> Scope:
> - Review the diff against: <BASE_BRANCH, default origin/main>
> - Focus on correctness, regressions, security, data loss, concurrency,
>   error handling, API compatibility, performance, and missing tests.
> - Read and follow the repository's AGENTS.md/CLAUDE.md instructions.
> - Preserve all existing user changes. Do not modify files yet.
>
> Process:
>
> 1. Establish the review scope
>    - Inspect git status, the diff, and relevant surrounding code.
>    - Identify the correct merge base with <BASE_BRANCH>.
>    - Note any generated files, vendored code, or unrelated changes that
>      should be excluded.
>
> 2. Perform your own review first
>    - Complete an independent review before invoking Codex.
>    - Record each potential issue with:
>      - priority: P0, P1, P2, or P3
>      - file and tight line range
>      - concrete failure scenario
>      - why the current behavior is wrong
>      - suggested correction
>      - confidence: high, medium, or low
>    - Do not read or anticipate Codex's conclusions yet.
>
> 3. Run an independent Codex CLI review
>    - First inspect `codex --help` and, if available,
>      `codex review --help`, so you use the installed CLI correctly.
>    - Run Codex non-interactively against the same base, diff, and
>      review criteria.
>    - Tell Codex to report only actionable defects, not style
>      preferences.
>    - Ask Codex for file/line evidence, failure scenarios, severity,
>      and suggested tests.
>    - Do not let Codex edit files.
>    - Capture its complete output for comparison.
>    - If Codex cannot run, report the exact blocker and continue with
>      your own review.
>
> 4. Reconcile the reviews
>    - Compare the findings as:
>      - agreement between Claude and Codex
>      - Claude-only findings
>      - Codex-only findings
>      - conflicting conclusions
>    - Independently inspect the code for every finding from both
>      reviewers.
>    - Reject false positives, duplicates, speculative concerns without
>      a plausible failure path, and issues outside the diff unless the
>      change clearly exposes them.
>    - Do not accept a finding merely because both reviewers reported it.
>
> 5. Prioritize the verified findings
>    Use:
>    - P0: catastrophic or release-blocking
>    - P1: likely serious correctness, security, or data-loss problem
>    - P2: real defect with narrower impact or an important regression
>      risk
>    - P3: minor but actionable defect
>    Rank by user impact, likelihood, blast radius, and ease of
>    detection, not reviewer agreement alone.
>
> 6. Propose fixes
>    - For each verified finding, propose the smallest safe fix.
>    - Explain affected behavior and relevant tradeoffs.
>    - Specify the tests that should prove the fix and prevent
>      regression.
>    - Identify dependencies between fixes and recommend an
>      implementation order.
>    - Do not implement anything until I approve the proposed plan.
>
> Final response format:
>
> ## Verdict
> A short assessment of whether the change is safe to merge.
>
> ## Verified findings
> For each finding:
> - `[P#] Title`
> - Source: Claude, Codex, or both
> - Evidence: `path:line`
> - Failure scenario
> - Recommended fix
> - Required test
> - Confidence
>
> If there are no verified defects, say so explicitly.
>
> ## Rejected or duplicate findings
> Briefly list material findings you rejected and why.
>
> ## Prioritized fix plan
> An ordered, minimal implementation plan.
>
> ## Review coverage
> State what was inspected, what tests or static checks were run, and
> any areas that could not be validated.
>
> Be concise, evidence-driven, and willing to disagree with either
> reviewer.

## Why it works

- Ordering enforces independence: Claude finishes its own review before
  Codex runs, so neither anchors the other.
- Agreement is treated as correlation, not confirmation. Both models can
  share a blind spot, so every finding gets re-inspected against the
  code, and "both reported it" is explicitly not sufficient.
- The rejected-findings section makes the reconciliation auditable. A
  reviewer that never rejects anything did not reconcile; it collated.
- Codex reviews but never edits, and no fix lands without approval. Two
  reviewers, one writer, and the writer is you.
- Graceful degradation: if Codex cannot run, you still get one full
  review plus the exact blocker, not a silent fallback.
