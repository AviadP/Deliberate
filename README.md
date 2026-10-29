# Deliberate

A Claude Code skill that runs a multi-agent deliberation, modeled on Microsoft's MAI Diagnostic Orchestrator (MAI-DxO, called DxO in the Science paper) but for any question or decision, not medicine. Four Opus 5.5 agents with different jobs work through a question over several rounds. They talk to each other through a shared message board, ask you for facts only you know, and keep going until a validator decides the answer holds up.

| Turn | Agent | Job | Tools beyond reading and writing its file |
|---|---|---|---|
| 1 | Researcher | Carries out the research the panel asked for and reports evidence with sources | Web search, web fetch |
| 2 | Theorist | Ranks hypotheses with probabilities, proposes new ones, and plans the research that would best tell them apart | None; asks the researcher |
| 3 | Critic | Attacks the latest ranking and the research plan, leader first: bias, contradictions, weak sources | Web search, web fetch (spot checks) |
| 4 | Validator | Checks quality and communication, cuts research that couldn't change the answer, picks questions for you, and decides whether to continue or conclude; writes the verdict | Web fetch (checks key sources) |

## What it takes from MAI-DxO

MAI-DxO has one model play a panel of five virtual doctors who debate each case. Each step, the panel decides whether to ask the patient a question, order a test, or commit to a diagnosis. A "gatekeeper" reveals information only when it's explicitly asked for, and every test has a cost. In the Science paper, with GPT-5, it reached 83.2% accuracy on 304 hard NEJM cases, against 36.1% for generalist physicians with reference tools.

The paper names three processes behind the gain: explicit hypothesis tracking, adversarial critique, and cost-aware test selection. Questioning mattered most. The system asked about 25 questions before ordering its first test, against about 6 for physicians. Questions alone, with no tests, took it from about 39% to about 63% accuracy. Its advantage was concentrated in cases where careful history-taking surfaced the deciding clue.

| MAI-DxO | Here |
|---|---|
| Dr. Hypothesis keeps a probability-ranked differential, updated after each finding | The theorist's ranked hypothesis table, updated every round |
| Dr. Test-Chooser picks up to 3 tests that best separate the leading diagnoses | The theorist sends at most 3 research requests per round, each saying which hypotheses it separates and how each possible result would shift them |
| Dr. Challenger: devil's advocate, watches for anchoring and other biases, proposes tests that could disprove the leader | The critic, with the same bias list plus decision-specific ones, and a falsifying check every round |
| Dr. Stewardship cuts low-value tests and prefers asking over testing | The validator's value check: research that couldn't change the answer is cut; a question to you comes before research when the uncertainty is about you |
| Dr. Checklist does quality control and consistency checks | The validator's quality check, which also confirms every message got a reply |
| A consensus step chooses ask, test or diagnose | The validator's decision: ask you, research more, or conclude at 85% or more with no critical or major objection open, or when nothing left could change the answer |
| The gatekeeper reveals findings only when asked, and the panel takes a thorough history before testing | You hold the facts about your own situation. The theorist interviews you before any research starts (up to three batches of questions, following up on your answers), and the panel can ask more between rounds. The web holds the rest |
| Most errors were answers in the right area at the wrong level of specificity | The validator checks that the leading answer is as specific as the evidence supports, no vaguer and no more specific |

The role details come from the preprint and secondary coverage, because the Science paper puts them in its supplementary materials. A few smaller ideas come from the open-source re-implementation of MAI-DxO, not from the paper: the 85% confidence threshold for concluding, the theorist switching to confirming or breaking the leader once it passes 75%, and the validator watching for the panel going in circles.

Not adopted, at least for now:

- **Ensemble mode:** several independent panels whose answers are combined. It gave the paper's highest accuracy, but the gain was small for the cost: from 83.2% to 84.5% on the NEJM cases, while the cost of tests rose by about 65% ($5,431 to $8,945 per case). It could come later as a `--panels N` option.
- **Dollar costs per test:** research here has no real price tag, so the round limits and the value check do that job instead.
- **One model playing every role in a single prompt:** MAI-DxO does it that way. Separate agents with their own context are more independent, which helps against the panel simply agreeing with itself.

## Who asks you questions

No single agent talks to you; the moderator asks you the questions and writes down your answers word for word. The questions come at two points:

1. **Intake, before round 1.** The theorist reads your question, forms first hypotheses from it alone, and interviews you: your situation, goals, constraints, preferences, and what you've already tried. It asks up to five questions at a time, picking the ones that best separate its hypotheses, and follows up on your answers, for at most three batches. It stops as soon as more answers wouldn't change anything. If the answer doesn't depend on you (a purely factual question, say), it asks nothing.
2. **Between rounds.** Any agent can propose a question for you. The validator picks at most three, and only ones whose answer could change the outcome, so you don't get a stream of questions from four agents.

The questions follow fixed rules:
- Each question asks one thing, in plain language.
- When a question offers options, each comes with both its cost and its benefit.
- You never see the panel's notes on how your answer would shift the ranking, so they can't steer you.
- The panel can mark one question per batch as key. If you skip a key question, or an answer could be read two opposite ways, the moderator checks with you once and records the exchange.

## How the agents talk to each other

- **A shared board.** Every turn is a file in `messages/`. Every agent reads the whole board before writing, so each sees what the others said directly, with no summarizing in between.
- **Addressed messages, mandatory replies.** Each turn ends with messages addressed to specific agents. The addressee must reply to every one in its next turn: answer, concede, push back or explain why it can't. Ignoring a message breaks the rules.
- **The validator audits it.** Each round, the validator lists any unanswered or dodged messages and re-sends them as its own directives.
- **The moderator stays out of it.** The main Claude session only schedules turns and passes questions to you. It's told never to relay or rephrase content.
- **Agents keep their memory.** The same four agents are resumed every round, so the critic remembers its objections and can check whether they were answered.

Here's what a few turns look like on the board:

```
00-1-theorist      Intake. T0.1 → user: Is this for a team of 5 or of 50? Separates H1 from H2.
00-2-user          U0.1 (re T0.1): about 12 people, growing to 30 next year
01-2-theorist      T1.1 → researcher: Is there independent data behind E2? If yes, H1 +10; if no, H2 +15.
01-3-critic        O1 [major] on H1: rests on E2, a single vendor-funded study.
01-4-validator     decision: continue. V1.3 → user: What's the most you could spend per month?
02-0-user          U2.1 (re V1.3): about $400
02-1-researcher    re T1.1: two independent studies (E7, E8). E7 agrees; E8 finds a smaller effect.
02-2-theorist      H1 55% → 45% (E8, U2.1). H3 now 30%.
```

## Rounds

There are at least 3 rounds and at most 6 by default. Before round 3, the validator can't stop the debate; if it already looks settled, it uses the remaining rounds to stress-test the leading answer. From round 3 on, it concludes when the leader reaches 85% with no critical or major objection open and its key sources check out, or when nothing left to research or ask could change the answer.

## Install

Clone the repo, then copy the files into `~/.claude/` to use the skill in every project (or into one project's `.claude/` folder to use it only there):

```bash
git clone https://github.com/AviadP/deliberate.git
cd deliberate
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r .claude/skills/deliberate ~/.claude/skills/
cp .claude/agents/deliberation-*.md ~/.claude/agents/
```

Start a new Claude Code session afterwards so it picks them up. To update later, run `git pull` in the clone and the two `cp` lines again.

Cloud sessions started on this repo load the skill automatically, because it lives in the repo's `.claude/` folder.

## Use

```
/deliberate Should we move our Postgres workload to managed Aurora?
/deliberate --max 8 What's the most likely cause of the drop in signups since March? Context is in notes/signups.md
/deliberate --auto Is it worth refinancing a 4.1% mortgage at 3.4% with $6k closing costs?
```

Put any background in the question itself, or point to files; the researcher reads them. The panel starts with a short interview (up to three batches of up to five questions) and may ask up to three more questions between rounds. You can skip any of them or answer "don't know". With `--auto`, the panel asks nothing and states its assumptions instead, and the verdict lists them.

Each run writes to `deliberations/<date>-<slug>/` in the current directory:

| File | Contents |
|---|---|
| `brief.md` | The question, your context, the round limits and the mode |
| `protocol.md` | The rules the agents follow |
| `messages/` | One file per turn, such as `02-2-theorist.md`, plus the intake interview (files starting `00-`) and your answers, such as `02-0-user.md` |
| `verdict.md` | Answer, confidence, strongest case against, assumptions, open questions, your next step, how the debate moved |
| `transcript.md` | All of the above in one file |

## Permissions

The agents write their files without prompting (`permissionMode: acceptEdits`). Web search and web fetch may still ask for approval. For runs that don't stop to ask, allow them in `.claude/settings.json` or `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": ["WebSearch", "WebFetch"]
  }
}
```

## Cost and time

A minimum run is 13 Opus 5.5 turns (4 agents × 3 rounds, plus the verdict), many of them with web research, plus one to three short intake turns. Expect tens of minutes per run, plus however long you take to answer the panel's questions. A run uses up your plan's limits much faster than a normal chat.

## Tuning

- **Rounds:** pass `--min` and `--max`, or change the defaults in `.claude/skills/deliberate/SKILL.md`.
- **Model and effort:** set in each agent file's frontmatter (`model: claude-opus-5-5`, `effort: high`). Use `model: opus` to follow future Opus releases automatically, or `effort: xhigh` for harder questions.
- **Roles:** edit the agent files in `.claude/agents/`. The rules all four agents share are in `.claude/skills/deliberate/protocol.md`.

## Sources

- Nori et al., "Sequential diagnosis with language models", [Science 394 (6822), 423–428](https://www.science.org/doi/10.1126/science.aeh7930), 22 October 2026. Preprint: [Microsoft Research](https://www.microsoft.com/en-us/research/publication/sequential-diagnosis-with-language-models/), arXiv [2506.22405](https://arxiv.org/abs/2506.22405)
- Microsoft, [SDBench](https://github.com/microsoft/sdbench): the official benchmark code (the case database, the gatekeeper, the scoring "judge" and a baseline agent, but not the orchestrator itself)
- Microsoft AI, [The Path to Medical Superintelligence](https://microsoft.ai/news/the-path-to-medical-superintelligence/)
- The Swarm Corporation, [Open-MAI-Dx-Orchestrator](https://github.com/The-Swarm-Corporation/Open-MAI-Dx-Orchestrator), an open-source re-implementation

## License

MIT. See [LICENSE](LICENSE).
