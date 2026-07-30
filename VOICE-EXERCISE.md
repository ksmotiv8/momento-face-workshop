# Exercise: teach the agent to write like you

LLMs have a house style. You have yours. This exercise builds a reusable
`VOICE.md` so everything the agent writes for you sounds like you wrote
it, then proves the calibration worked.

## Step 1: gather 5 samples

Pick 5 pieces of YOUR real writing: emails, docs, blog posts, long Slack
messages. Representative beats polished; include at least one where you
were annoyed or in a hurry, because that is where voice lives. Put them
in a `samples/` directory.

## Step 2: the prompt

> Read the five writing samples in `samples/`. They are examples of my
> writing, not source material: never copy their sentences and never
> import their facts into another task.
>
> PHASE 1 - LEARN. Analyze the recurring patterns: tone and formality;
> sentence length, rhythm, and variation; vocabulary; paragraph length
> and structure; how I open, transition, explain, and conclude; how
> direct, opinionated, or funny I am; punctuation, fragments, questions,
> emphasis; what I consistently avoid. Separate stable traits of my
> voice from artifacts of a particular topic. Preserve my patterns of
> THOUGHT and explanation, not just my vocabulary. Support every
> important claim with at least two short quotes from the samples. Where
> the samples disagree, say so and pick the dominant pattern. Where the
> evidence is thin, write "insufficient evidence" and default to
> restrained natural prose rather than inventing a trait. Save all of
> this as `VOICE.md`, written as practical instructions to a writer.
>
> PHASE 2 - GROUND RULES. Append these to `VOICE.md` as hard rules:
> 1. Write like the person in the samples, not like an assistant.
> 2. Never invent personal experiences, opinions, facts, or stories on
>    my behalf. Every personal claim must come from something I supplied.
> 3. No em-dashes.
> 4. No "delve", "crucial", "robust", "seamless", "leverage",
>    "transformative", "unlock", "landscape", "Furthermore",
>    "It's important to note", "In conclusion", "With that said".
> 5. No "It's not just X, it's Y". No groups of three unless my samples
>    make them. No mechanically parallel lists.
> 6. No restating the prompt, no announcing what you are about to do,
>    no redundant closing summary.
> 7. No bullets, headers, or bold unless my samples use them for that
>    kind of content.
> 8. Do not explain obvious implications. Stop when the point is made.
> 9. Vary sentence length the way I do. Keep the imperfections; they
>    are the style.
> 10. Never sacrifice accuracy or clarity to imitate a quirk.
>
> PHASE 3 - PROVE IT. Write a 150-word note in my voice: [PICK SOMETHING
> REAL: a project delay, a launch note, declining a meeting]. Then list
> the three places you were most tempted to fall back on model habits,
> and what you did instead. I want to see the temptations this one time;
> in future sessions, load `VOICE.md` and return only the finished
> writing.

## Step 3: read the proof like a reviewer

The 150-word note tells you if it learned; the three confessed
temptations tell you if the rules landed. If the note reads like a press
release, your samples were too polished. Add an angry email and rerun.

## Why each part exists

- "Not source material" and rule 2 are the firewall: without them the
  agent quotes your old work and invents your opinions.
- The two-quotes-per-claim quota forces evidence over flattery. Without
  it you get a horoscope: "you write with clarity and warmth."
- Hard bans beat "do not overuse". A model can rationalize one more
  em-dash under "overuse"; it cannot under "no em-dashes."
- The visible temptation list is the calibration check. A silent
  self-revision pass may work, but you cannot verify it.
- `VOICE.md` is the point: learn once, load it in every future session,
  like any other skill.
