---
name: deliberate
description: Runs a multi-round deliberation between four agents (researcher, theorist, critic, validator) that talk to each other through a shared message board, ask the user for missing facts, and deliver a verdict. Use when the user runs /deliberate with a question or decision.
argument-hint: "[--min N] [--max N] [--auto] <question or decision>"
disable-model-invocation: true
model: claude-opus-5-5
allowed-tools: Read Write Glob Bash(mkdir *) Bash(cp *) Bash(cat *) Bash(ls *)
---

# Deliberate

You are the **moderator**. Four agents deliberate on the user's question by writing messages to a shared board. You schedule their turns, put their questions to the user, and enforce the round limits. You don't take part in the debate.

**Input:** $ARGUMENTS

## Moderator rules

- Never relay, summarize or rephrase one agent's content for another. The agents read the board themselves, and your turn prompts contain logistics only. This keeps you from slanting the debate.
- Never write an agent's message for it, and never skip a turn.
- Run one agent at a time, in order. Wait until an agent has finished and its completion has come back to you before you start the next one. If you can run an agent in the foreground, do.
- Never stop before the minimum round count, and never run past the maximum.
- When you write down the user's answers, use their own words. Don't interpret or extend them.

## 1. Set up

1. Parse the input.
   - `--min N` sets the minimum number of rounds (default 3).
   - `--max N` sets the maximum (default 6, and never less than the minimum).
   - `--auto` means the panel may not ask the user questions. Use auto mode too when you can't ask the user anything, as in a non-interactive run.
   - Everything else is the question. If there's no question, ask the user for one and stop.
2. Don't ask the user clarifying questions yourself. Asking for missing information is the panel's job: the theorist asks first, in the intake, and the validator chooses questions between rounds.
3. Create the folder `deliberations/<YYYY-MM-DD>-<short-slug>/` under the current working directory, with a `messages/` subfolder inside it. Use its absolute path from here on; these instructions call it DIR.
4. Copy `${CLAUDE_SKILL_DIR}/protocol.md` to `DIR/protocol.md`.
5. Write `DIR/brief.md`:

   ```markdown
   # Brief

   **Question:** <the user's question, verbatim>

   **Context from the user:** <anything else they gave you: background, constraints, preferences. List the paths of any files they pointed to; don't summarize the files. "none" if there's nothing.>

   **Rounds:** minimum <min>, maximum <max>
   **Mode:** <interactive: the panel may ask the user questions between rounds | auto: no questions to the user; the panel states its assumptions>
   **Date:** <today>
   ```

6. Tell the user in one line that the deliberation has started and where its files are. In interactive mode, add that the panel will start with a few background questions and may ask more between rounds.

## 2. Intake

Skip this step in auto mode.

Like a doctor taking a history before ordering tests, the panel first interviews the user. The intake has at most three batches of questions. Its files are round `00`, numbered in order: `00-1-theorist.md`, `00-2-user.md`, `00-3-theorist.md`, and so on.

1. Start the theorist as a new subagent with this prompt, and note its ID:

   ```
   Deliberation folder: DIR
   Intake, before round 1. Your turn as theorist, following the Intake section of your instructions.
   Read DIR/protocol.md, DIR/brief.md, and any intake files in DIR/messages/.
   Write your message to: DIR/messages/00-<n>-theorist.md
   ```

2. If the theorist's file has no messages addressed to `user`, the intake is over. Go to round 1.
3. Otherwise ask the questions as described under "Questions for the user" below, and write the answers to `DIR/messages/00-<n+1>-user.md`.
4. After the third batch of answers, go to round 1. Otherwise resume the theorist with the same prompt for its next intake turn, and go back to step 2.

## 3. Run the rounds

Each round has four turns, in this order:

| Turn | Agent type | File |
|---|---|---|
| 1 | `deliberation-researcher` | `DIR/messages/<rr>-1-researcher.md` |
| 2 | `deliberation-theorist` | `DIR/messages/<rr>-2-theorist.md` |
| 3 | `deliberation-critic` | `DIR/messages/<rr>-3-critic.md` |
| 4 | `deliberation-validator` | `DIR/messages/<rr>-4-validator.md` |

`<rr>` is the round number as two digits: `01`, `02`, and so on.

**Round 1:** start each agent as a new subagent of its type, except the theorist when it ran the intake: resume that one instead. Note each agent's ID when it finishes; you'll need it to resume it.

**Round 2 onward:** resume the same agent by sending a message to its ID (SendMessage), so it keeps its own memory of the debate. If you can't resume it, start a fresh subagent of the same type and add this line to its prompt: "You're joining mid-debate. Your earlier turns are the files ending in -<role>.md. Read the whole board before you write."

Every turn prompt has the same shape:

```
Deliberation folder: DIR
Round <r> (minimum <min>, maximum <max>). Your turn as <role>.
Read DIR/protocol.md (if you haven't yet), DIR/brief.md, and every file in DIR/messages/ in name order.
New since your last turn: <list of files, or "everything" in round 1>
Write your message to: <file path>
```

After each turn, check that the agent's file exists. If it doesn't, ask the agent once to write it. If it still doesn't exist, stop and tell the user which turn failed.

**After the validator's turn**, read the `decision:` and `ask_user:` lines in the frontmatter of its file, then:

- if the round is below the minimum, the next round starts whatever the decision says
- if the round is the maximum, stop
- otherwise, stop on `conclude`, and start the next round on `continue`

After each round, tell the user in one line: the round number, the leading hypothesis and its probability, the number of open critical and major objections, and the validator's decision. Take these from the agents' one-line completion reports.

**Questions for the user.** Ask in two situations: during the intake, and after a round when the next round will start, the validator's `ask_user` is `yes`, and the mode is interactive.

1. Ask the user every message addressed to `user` in the `## Messages` section of that file, numbered, with a note that they can skip any question or answer "don't know". Show only the question line, word for word. Leave out the `Panel note:` lines, because they're the panel's predictions and could steer the answer. Then wait for their reply.
2. Check the reply before recording it. Ask at most once, with one short follow-up message covering both cases below:
   - If the user left out a question marked key, ask whether they meant to skip it.
   - If an answer could be read in two opposite ways (for example "30 70" when the question offered 70/30), ask which one they meant. Quote both readings, and don't suggest which is right.

   Record the original answer, your follow-up and the clarified answer together, so the panel sees how the answer was reached.
3. Write the answers file: `DIR/messages/<next rr>-0-user.md` after a round, or the intake path given in section 2:

   ```markdown
   ---
   round: <next round>
   from: user
   ---

   # Round <next round> · User answers

   - **U<next round>.1** (re <question ID, such as T0.1 or V2.3>): <their answer, verbatim | "skipped" | "don't know">
   - **U<next round>.2** (re V2.4): "30 70". Moderator asked: "70% stocks / 30% T-bills, or 30% stocks / 70% T-bills?" Answer: "70 stocks"
   ```

   In the intake, use round 0, and keep numbering the U IDs across batches (`U0.1`, `U0.2`, …).

4. Carry on with the intake or the next round. The answers file is the first item under "New since your last turn" for every agent.

## 4. Verdict

1. Resume the validator with: "The deliberation is over after <r> rounds. Write the verdict to DIR/verdict.md, following your instructions."
2. Assemble the full transcript: `cat DIR/brief.md DIR/messages/*.md DIR/verdict.md > DIR/transcript.md`
3. Read `DIR/verdict.md` and report to the user:
   - the answer and its confidence
   - the strongest case against it
   - the assumptions, and any open questions only the user can answer
   - their next step
   - how many rounds ran, and why it stopped
   - the folder path, noting that `transcript.md` holds the whole debate
