# Exercise: the devil's advocate pass

Agents default to being encouraging about your writing. That is the last
thing a draft needs. This exercise turns the harness into the hostile,
smart reader you will eventually face anyway, plus a voice check and an
LLM-tell detector, before the real audience sees it.

Works best after the voice exercise (`VOICE-EXERCISE.md`): if you have a
`VOICE.md`, the tone review compares against it. Without one, point the
prompt at 2-3 of your writing samples instead.

## The prompt

> Read my draft at [PATH]. Your job is to attack it, not improve it. Do
> not rewrite it. Do not compliment it. No praise sandwich: findings
> only, ranked by how much they would hurt in front of my real audience:
> [AUDIENCE].
>
> PASS 1 - DEVIL'S ADVOCATE. First state my core argument in two
> sentences, at its strongest, so I know you attacked the real thing.
> Then argue against the piece the way the sharpest skeptic in the room
> would: the strongest counterargument I ignored; every claim that has
> no support in the text; every number with no source; where I
> overclaim; where I hedge so much I say nothing; the weakest paragraph
> and why; the question I clearly hope nobody asks. Quote the exact
> passage for every finding. Only attack what the text says; do not
> invent positions I did not take.
>
> PASS 2 - VOICE AND TONE. Compare against my `VOICE.md` [or: my
> samples in samples/]. Quote every passage that does not sound like me,
> name what drifted (rhythm, word choice, formality, hedging), and show
> a one-line rewrite in my voice. Say where the tone mismatches the
> audience and purpose, even if it matches my usual voice.
>
> PASS 3 - THE MACHINE DETECTOR. Flag everything a reader might smell
> as AI-written: em-dashes, groups of three, mechanically parallel
> lists, "It's not just X, it's Y", generic transitions, empty
> intensifiers, uniform sentence rhythm, a summary ending that repeats
> the piece, any sentence that could appear unchanged in anyone's essay
> on this topic. Quote each, say which tell it is. Then name the three
> sentences MOST likely to make a reader think a model wrote this.
>
> End with one paragraph, no hedging: if you were arguing against me in
> front of [AUDIENCE], which single finding would you lead with, and
> would the piece survive it?

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
- "Ranked by how much it would hurt in front of [AUDIENCE]" turns a
  generic critique into a rehearsal of the actual room.
- Quoting exact passages keeps the attack honest and actionable; a
  critique without quotes is a vibe.
- "Do not rewrite it" matters: a rewrite lets the agent smuggle its own
  voice back in. You want findings; the fixes stay yours.
- The final forced verdict ("would it survive?") is the anti-hedge: an
  agent allowed to conclude "overall, solid with minor issues" always
  will.
