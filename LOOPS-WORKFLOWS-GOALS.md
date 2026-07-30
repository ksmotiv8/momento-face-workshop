# Loops, workflows, and goals

One prompt, one answer is the default shape of a coding agent. It is the
wrong shape for plenty of real work. Claude Code and Codex each grew
primitives past it, and they answer three different questions.

## Claude Code: /loop - "do this again, over time"

`/loop` reruns a prompt on an interval, or lets the model pace itself.
The harness becomes a standing process instead of a conversation.

```
/loop 5m check the deploy; if it failed, diagnose and page me with evidence
```

Reach for it when the work is watching: a CI run, a migration, a queue,
a nightly chore. The prompt does not change; the world does.

## Claude Code: workflows - "do these all at once, in order"

A workflow is a small script that fans work out to many subagents with
control flow the model cannot forget or reorder: run five reviewers in
parallel, adversarially verify every finding, then merge what survives.
Determinism lives in the script; judgment lives in the agents.

This repo was hardened by one. A workflow read an autonomous run of the
workshop, fixed the doc bugs it found, and voted on whether the fixes
deserved another validation run. Two rounds later the syllabus was solid.

Reach for it when you need coverage (many files, many angles) or
confidence (independent reviewers, adversarial checks) that one context
window cannot hold.

## Codex: /goal - "keep going until this is true"

A goal states completion criteria and lets the agent iterate until the
checks pass, with an escape hatch for real blockers:

```
/goal
Implement the face-recognition pipeline.

Continue until:
- the complete probe set passes
- genuine faces clear the calibrated threshold
- strangers remain unmatched
- cargo test and cargo clippy pass

After every attempt, run the checks, inspect the failures, and adjust.
Stop and report evidence if the same external blocker prevents progress
repeatedly.
```

Reach for it when the end state is testable and the path is not. Notice
what makes that goal good: every criterion is checkable by running
something. "Continue until it works well" is not a goal; it is a wish.

## Which one, when

| You are really asking | Primitive |
|---|---|
| Watch this and act when it changes | Claude `/loop` |
| Cover this from many angles at once | Claude workflow |
| Converge on this provable end state | Codex `/goal` |

They rhyme more than they compete. A goal is a loop whose exit is proof.
A workflow is a loop unrolled across agents instead of time. All three
work for the same reason the syllabus keeps repeating: the agent is held
to checks it can run, not to promises it can make.
