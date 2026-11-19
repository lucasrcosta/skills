---
name: readback
description: Read the user's instruction back to them, unambiguously, before acting on it, and end on a confirmation a plain "yes" answers. Use when the user types /readback, or asks you to read back, repeat back, or confirm what you understood before you start.
---

# Readback

The user gave an instruction and wants to hear what landed before you act on it. Say back what you are about to do, precisely enough that a wrong reading is visible in your wording, then stop and wait for a yes.

This is a pilot's readback: one short exchange. You are not interviewing the user, planning the work, or reviewing whether the idea is good.

## What to read back

The instruction is whatever came with the invocation. If nothing came with it, it is the user's most recent instruction in the conversation. If you cannot tell which instruction they mean, ask that one thing and nothing else.

Read it in its context: what the conversation was about when they said it, what is already decided, what you were working on. The same words mean different things in different places, and the context usually settles which.

## Be critical of your own reading

Before writing, look for the places where what they said and what they meant could differ:

- **Scope words**: everywhere, all, every, always, never, everything, anything, the whole. They almost always carry an unstated boundary from the context. "Do it everywhere" said while working on a desktop sidebar probably means every place in that sidebar, not every platform and screen.
- **Pointers**: it, that, this, the same, the other one, like before. Name the thing each one points at.
- **Unstated edges**: where the work starts and stops, and what sits next to it that you would be tempted to touch as well.
- **Loose verbs**: fix, clean up, update, handle, improve, align. Say the actual change.
- **Earlier decisions**: anything already settled in the conversation or the project that this instruction seems to contradict or quietly reverse.

Where the context settles a question, take that reading and state it as a fact about what you will do. Where it does not, still pick the reading you believe is meant, state it, and name in one clause what you are leaving out, so the user can see the choice and correct it. Never hand the user a list of interpretations to pick from: a readback commits to one.

Use what you already know. Look something up only when a reading hinges on a fact that is one quick check away. Do not start an investigation.

## What to write

Keep it short enough to take in at a glance; a small instruction gets two or three lines.

1. **What you will do**, in concrete terms: the actual things, places, and changes, named in plain words. Do not echo their sentence back; a paraphrase that keeps the ambiguous word proves nothing.
2. **Where it stops**: only the boundaries someone could reasonably have assumed otherwise. Skip this when there are none.
3. **A reading you chose**: only where the instruction could honestly have gone another way. One line each: what you took it to mean, and what that excludes.
4. **What the user might be missing**: only when there is something concrete. See below.
5. **The confirmation**: the last line, always. A single yes-or-no question where "yes" means "go ahead exactly as described".

Leave out: how you will implement it, estimates, alternatives, and your opinion of the idea.

### What the user might be missing

This is the one place you contribute to the instruction instead of mirroring it. Use it when, reading the instruction against its context, you see a clear oversight or a useful aspect it does not cover:

- a case the instruction does not say what to do with, that the work will certainly run into
- something that will break or become inconsistent if the instruction is carried out exactly as given
- a neighbouring thing that plainly belongs to the same change and looks forgotten, not excluded
- an earlier decision the instruction contradicts
- a reason to think the instruction itself is a mistake

Each point is one or two lines: what is missing, and what you would do about it. It has to be specific to this instruction and this context. Generic advice (add tests, consider edge cases, think about performance) does not belong here.

Most instructions are complete. When you have nothing concrete, leave the section out entirely: no heading, no "nothing to add". Do not go looking for something to say.

These points are offers, not part of the work. "Yes" approves the instruction as you read it back, without them; the user takes one by saying so. Word them so that is clear, and never fold one into "what you will do".

## After the answer

- **Yes** (or any plain go-ahead): start the work as described, without the points you offered under "What the user might be missing". No second readback, no further questions about what was just confirmed.
- **Yes, plus one of your points**: include it and start.
- **A correction**: take it and start, unless the correction itself opens a new ambiguity. Then read back only the part that changed.
- Until one of those arrives, do not start the work.

## Example

During work on a desktop sidebar, the user says: "make the hover highlight the same everywhere."

> Every row in the desktop sidebar (songs, setlists, tags, bands) gets the hover highlight the song rows have now.
>
> Sidebar only: I am not touching hover in the main song list or in dialogs. Nothing changes on phone or tablet, which have no hover.
>
> Might be missing: the selected row has its own highlight, and the instruction does not say which wins when you hover it. I would keep the selected look and show no hover on top. Say so if you want that included.
>
> Go ahead with that?
