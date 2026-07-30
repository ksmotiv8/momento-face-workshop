# Exercise: the devil's advocate pass

Agents default to being encouraging about your writing. That is the last
thing a draft needs. This exercise turns the harness into the hostile,
smart reader you will eventually face anyway, plus a voice check and an
LLM-tell detector, before the real audience sees it.

Works best after the voice exercise (`VOICE-EXERCISE.md`): if you have a
`VOICE.md`, the tone review compares against it. Without one, point the
prompt at 2-3 of your writing samples instead.

## The prompt

> Read my draft at [PATH].
>
> Audience: [AUDIENCE]
> Purpose: [WHAT THIS PIECE SHOULD MAKE THEM UNDERSTAND, BELIEVE, OR DO]
> Voice reference: [VOICE.md or samples/]
> Maximum findings per pass: [5-8]
>
> Your job is to pressure-test the draft, not improve it. Do not rewrite
> it. Do not compliment it. No praise sandwich. Do not insult me, and do
> not manufacture objections merely to appear critical. Report only
> defensible findings grounded in the draft, ranked by how much damage
> they could do with the real audience. If a pass produces no meaningful
> finding, say so.
>
> For every finding: quote the exact passage; assign a severity
> (Critical, Major, Minor); state the objection plainly; explain why it
> matters to this audience and purpose; and distinguish "unsupported in
> the draft" from "unclear" from "probably false". Do not call anything
> false unless the evidence establishes it.
>
> PASS 1 - DEVIL'S ADVOCATE. First state the draft's core argument in
> two sentences at its strongest, including what the audience is being
> asked to believe or do. That is the argument you must attack. Then
> respond as the best-informed, fairest, most skeptical person in the
> room. Find: the strongest counterargument the draft ignores; important
> claims with no support in the text; numbers that need a source; where
> it overclaims; where it hedges so heavily it says little; the weakest
> paragraph and why; the hidden assumptions the conclusion depends on;
> and the question I most clearly hope nobody asks. Attack only
> positions the draft actually takes.
>
> PASS 2 - VOICE AND TONE. Compare against the voice reference. Flag
> only drift that conflicts with recurring, well-supported traits in it.
> For each: quote the passage, name what drifted (rhythm, vocabulary,
> formality, directness, hedging, humor), cite the voice-reference rule
> it breaks, and show a one-line rewrite in my voice. The rewrites are
> diagnostic only; do not revise surrounding text. Separately flag where
> my normal voice is wrong for THIS audience and purpose.
>
> PASS 3 - MACHINE DETECTOR. Flag passages that feel generic,
> formulaic, or machine-produced in context: em-dashes, forced groups of
> three, mechanically parallel lists, "It's not just X, it's Y", generic
> transitions, empty intensifiers, uniform sentence rhythm, an ending
> that merely summarizes, sentences that could appear unchanged in
> anyone's essay on this topic. Flag patterns, not any single
> punctuation mark as proof. For each: quote it, name the signal, say
> why it reads generic HERE. Then name the three sentences most likely
> to make a skeptical reader suspect a model wrote this.
>
> FINAL VERDICT. One direct paragraph, no hedging: arguing against me in
> front of [AUDIENCE], which single finding would you lead with, why is
> it the most damaging, and would the piece survive it as written?
> Return the critique only; no revised draft.

## How to read the results

- Pass 1 findings that sting are the ones to fix. The ones that miss
  simply tell you the draft was unclear enough to be misread, which is
  also a finding.
- If Pass 2 flags nothing, your VOICE.md is too vague. Tighten it.
- Pass 3 is not accusing you of using a model. Plenty of humans write
  mechanical prose. The flags mark where readers stop trusting the
  words, whoever typed them.
- Run it twice: once before your own edit, once after. The second run
  should lead with a different finding. If it leads with the same one,
  you dodged it.

## Why each part exists

- "State my argument at its strongest first" prevents the cheap version
  of devil's advocacy: attacking a strawman of your draft.
- "Ranked by damage in front of [AUDIENCE]" plus the Purpose line turns
  a generic critique into a rehearsal of the actual room.
- The findings cap and "if a pass finds nothing, say so" stop the agent
  from inventing nitpicks to look thorough. The anti-manufactured-
  objection rule stops it from performing harshness.
- "Unsupported vs unclear vs probably false" keeps the attack honest:
  an agent must not call your claim false just to sound tough.
- Quoting exact passages keeps every finding actionable; a critique
  without quotes is a vibe.
- "Do not rewrite it" matters: a rewrite lets the agent smuggle its own
  voice back in. You want findings; the fixes stay yours.
- The final forced verdict ("would it survive?") is the anti-hedge: an
  agent allowed to conclude "overall, solid with minor issues" always
  will.
