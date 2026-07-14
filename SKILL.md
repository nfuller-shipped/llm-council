---
name: council
description: "Run any question, idea, decision, OR design/code through a council of AI advisors who independently analyze it, critique each other, and synthesize a final verdict. Based on Karpathy's LLM Council methodology. Two modes — DECISION (5 generalist personas, peer-ranking) and RED-TEAM (adversarial lenses that read the actual artifact, refutation-based verify). MANDATORY TRIGGERS: 'council this', 'run the council', 'war room this', 'pressure-test this', 'stress-test this', 'debate this'. DECISION-MODE TRIGGERS (a choice with stakes): 'should I X or Y', 'which option', 'what would you do', 'is this the right move', 'validate this', 'I can't decide', 'I'm torn between'. RED-TEAM TRIGGERS (an artifact that could be wrong): 'red-team this', 'find the holes', 'find the bugs', 'attack this', 'review this design/plan/code/spec/PR', 'what breaks', 'poke holes in this'. Do NOT trigger on simple yes/no questions, factual lookups, or casual 'should I' without a meaningful tradeoff. DO trigger when the user presents a genuine decision with stakes OR a concrete artifact (design doc, architecture, code, plan) they want broken before committing."
---

# LLM Council

## two modes — pick the right one first

The council has two modes. Before anything else, decide which the question needs:

| | **Decision mode** | **Red-team mode** |
|---|---|---|
| The question | "Should I do X or Y?" — a choice with stakes and options | "Where does this break?" — an artifact that could be wrong |
| The target | a decision, strategy, positioning, a plan of action | a design doc, architecture, plan, spec, code, a PR |
| The lenses | 5 generalist *personas* (Contrarian, First Principles, Expansionist, Outsider, Executor) | adversarial *lenses* chosen for the artifact, that **read the actual files** |
| The middle step | anonymized **peer-ranking** (which take is strongest?) | adversarial **refutation** (try to break each finding; default refuted) |
| The output | a verdict + a recommendation + one first step | severity-ranked surviving findings, each with a scenario + a fix |

Rule of thumb: **choosing** → decision mode; **checking** → red-team mode. If the user says "should I build it this way or that way," that's still a decision (run decision mode on the two designs). If they hand you a design and say "is this right / what did I miss," that's red-team. When genuinely ambiguous, ask one clarifying question.

The rest of this file: decision mode is documented first (the original five advisors), then **red-team mode** in its own section near the end. Steps 5–6 (HTML report + saved transcript) are shared by both modes.

---

# Decision mode — the five advisors

You ask one AI a question, you get one answer. That answer might be great. It might be mid. You have no way to tell because you only saw one perspective.

The council fixes this. It runs your question through 5 independent advisors, each thinking from a fundamentally different angle. Then they review each other's work. Then a chairman synthesizes everything into a final recommendation that tells you where the advisors agree, where they clash, and what you should actually do.

This is adapted from Andrej Karpathy's LLM Council. He dispatches queries to multiple models, has them peer-review each other anonymously, then a chairman produces the final answer. We do the same thing inside Claude using sub-agents with different thinking lenses instead of different models.

## when to run the council

The council is for questions where being wrong is expensive.

Good council questions:
- "Should I launch a $97 workshop or a $497 course?"
- "Which of these 3 positioning angles is strongest?"
- "I'm thinking of pivoting from X to Y. Am I crazy?"
- "Here's my landing page copy. What's weak?"
- "Should I hire a VA or build an automation first?"

Bad council questions:
- "What's the capital of France?" (one right answer, no need for perspectives)
- "Write me a tweet" (creation task, not a decision)
- "Summarize this article" (processing task, not judgment)

The council shines when there's genuine uncertainty and the cost of a bad call is high. If you already know the answer and just want validation, the council will likely tell you things you don't want to hear. That's the point.

## the five advisors

Each advisor thinks from a different angle. They're not job titles or personas. They're thinking styles that naturally create tension with each other.

### 1. The Contrarian
Actively looks for what's wrong, what's missing, what will fail. Assumes the idea has a fatal flaw and tries to find it. If everything looks solid, digs deeper. The Contrarian is not a pessimist. They're the friend who saves you from a bad deal by asking the questions you're avoiding.

### 2. The First Principles Thinker
Ignores the surface-level question and asks "what are we actually trying to solve here?" Strips away assumptions. Rebuilds the problem from the ground up. Sometimes the most valuable council output is the First Principles Thinker saying "you're asking the wrong question entirely."

### 3. The Expansionist
Looks for upside everyone else is missing. What could be bigger? What adjacent opportunity is hiding? What's being undervalued? The Expansionist doesn't care about risk (that's the Contrarian's job). They care about what happens if this works even better than expected.

### 4. The Outsider
Has zero context about you, your field, or your history. Responds purely to what's in front of them. This is the most underrated advisor. Experts develop blind spots. The Outsider catches the curse of knowledge: things that are obvious to you but confusing to everyone else.

### 5. The Executor
Only cares about one thing: can this actually be done, and what's the fastest path to doing it? Ignores theory, strategy, and big-picture thinking. The Executor looks at every idea through the lens of "OK but what do you do Monday morning?" If an idea sounds brilliant but has no clear first step, the Executor will say so.

**Why these five:** They create three natural tensions. Contrarian vs Expansionist (downside vs upside). First Principles vs Executor (rethink everything vs just do it). The Outsider sits in the middle keeping everyone honest by seeing what fresh eyes see.

## how a council session works

### step 1: frame the question (with context enrichment)

When the user says "council this" (or any trigger phrase), do two things before framing:

**A. Scan the workspace for context.** The user's question is often just the tip of the iceberg. Their Claude setup likely contains files that would dramatically improve the council's output. Before framing, quickly scan for and read any relevant context files:

- `CLAUDE.md` or `claude.md` in the project root or workspace (business context, preferences, constraints)
- Any `memory/` folder (audience profiles, voice docs, business details, past decisions)
- Any files the user explicitly referenced or attached
- Recent council transcripts in this folder (to avoid re-counciling the same ground)
- Any other context files that seem relevant to the specific question (e.g., if they're asking about pricing, look for revenue data, past launch results, audience research)

Use `Glob` and quick `Read` calls to find these. Don't spend more than 30 seconds on this. You're looking for the 2-3 files that would give advisors the context they need to give specific, grounded advice instead of generic takes.

**B. Frame the question.** Take the user's raw question AND the enriched context and reframe it as a clear, neutral prompt that all five advisors will receive. The framed question should include:

1. The core decision or question
2. Key context from the user's message
3. Key context from workspace files (business stage, audience, constraints, past results, relevant numbers)
4. What's at stake (why this decision matters)

Don't add your own opinion. Don't steer it. But DO make sure each advisor has enough context to give a specific, grounded answer rather than generic advice.

If the question is too vague ("council this: my business"), ask one clarifying question. Just one. Then proceed.

Save the framed question for the transcript.

### step 2: convene the council (5 sub-agents in parallel)

Spawn all 5 advisors simultaneously as sub-agents. Each gets:

1. Their advisor identity and thinking style (from the descriptions above)
2. The framed question
3. A clear instruction: respond independently. Do not hedge. Do not try to be balanced. Lean fully into your assigned perspective. If you see a fatal flaw, say it. If you see massive upside, say it. Your job is to represent your angle as strongly as possible. The synthesis comes later.

Each advisor should produce a response of 150-300 words. Long enough to be substantive, short enough to be scannable.

**Sub-agent prompt template:**

```
You are [Advisor Name] on an LLM Council.

Your thinking style: [advisor description from above]

A user has brought this question to the council:

[framed question]

Respond from your perspective. Be direct and specific. Don't hedge or try to be balanced. Lean fully into your assigned angle. The other advisors will cover the angles you're not covering.

Keep your response between 150-300 words. No preamble. Go straight into your analysis.
```

### step 3: peer review (5 sub-agents in parallel)

This is the step that makes the council more than just "ask 5 times." It's the core of Karpathy's insight.

Collect all 5 advisor responses. Anonymize them as Response A through E (randomize which advisor maps to which letter so there's no positional bias).

Spawn 5 new sub-agents, one for each advisor. Each reviewer sees all 5 anonymized responses and answers three questions:

1. Which response is the strongest and why? (pick one)
2. Which response has the biggest blind spot and what is it?
3. What did ALL responses miss that the council should consider?

**Reviewer prompt template:**

```
You are reviewing the outputs of an LLM Council. Five advisors independently answered this question:

[framed question]

Here are their anonymized responses:

**Response A:**
[response]

**Response B:**
[response]

**Response C:**
[response]

**Response D:**
[response]

**Response E:**
[response]

Answer these three questions. Be specific. Reference responses by letter.

1. Which response is the strongest? Why?
2. Which response has the biggest blind spot? What is it missing?
3. What did ALL five responses miss that the council should consider?

Keep your review under 200 words. Be direct.
```

### step 4: chairman synthesis

This is the final step. One agent gets everything: the original question, all 5 advisor responses (now de-anonymized so you can see which advisor said what), and all 5 peer reviews.

The chairman's job is to produce the final council output. It follows this structure:

**COUNCIL VERDICT**

1. **Where the council agrees** — the points that multiple advisors converged on independently. These are high-confidence signals.
2. **Where the council clashes** — the genuine disagreements. Don't smooth these over. Present both sides and explain why reasonable advisors disagree.
3. **Blind spots the council caught** — things that only emerged through the peer review round. Things individual advisors missed that other advisors flagged.
4. **The recommendation** — a clear, actionable recommendation. Not "it depends." Not "consider both sides." A real answer. The chairman can disagree with the majority if the reasoning supports it.
5. **The one thing you should do first** — a single concrete next step. Not a list of 10 things. One thing.

**Chairman prompt template:**

```
You are the Chairman of an LLM Council. Your job is to synthesize the work of 5 advisors and their peer reviews into a final verdict.

The question brought to the council:

[framed question]

ADVISOR RESPONSES:

**The Contrarian:**
[response]

**The First Principles Thinker:**
[response]

**The Expansionist:**
[response]

**The Outsider:**
[response]

**The Executor:**
[response]

PEER REVIEWS:
[all 5 peer reviews]

Produce the council verdict using this exact structure:

## Where the Council Agrees
[Points multiple advisors converged on independently. These are high-confidence signals.]

## Where the Council Clashes
[Genuine disagreements. Present both sides. Explain why reasonable advisors disagree.]

## Blind Spots the Council Caught
[Things that only emerged through peer review. Things individual advisors missed that others flagged.]

## The Recommendation
[A clear, direct recommendation. Not "it depends." A real answer with reasoning.]

## The One Thing to Do First
[A single concrete next step. Not a list. One thing.]

Be direct. Don't hedge. The whole point of the council is to give the user clarity they couldn't get from a single perspective.
```

---

# Red-team mode — adversarial lenses over a real artifact

Use this mode when the user hands you something concrete that could be **wrong** — a design doc, an architecture, a plan, a spec, code, a PR — and wants it broken before they commit. The decision-mode personas are the wrong tool here: they reason *about* an idea but they don't open the files, so they miss defects that only exist in the code (a grader that a constant stub passes, an iframe with no `sandbox`, a timeout that ignores its own setting). Red-team lenses read the source.

Two things change from decision mode: the **lenses** are adversarial and artifact-specific (not fixed personas), and the middle step is **refutation, not ranking** — each finding is independently attacked by a skeptic who defaults to "refuted," so only findings that survive cross-examination reach the synthesis. This is what stops a confident-but-wrong finding from surviving on eloquence.

### step 1 (red-team): scope the target and pick the lenses

**A. Read the artifact and its neighborhood.** Identify the file(s) under review and the code they touch. Read them. Read any `CLAUDE.md`, plan/spec docs, and the immediate dependencies the artifact calls into — a finding is only as good as the reviewer's grasp of what actually runs. Note where the trust boundary is (untrusted input, external surface, who consumes the output).

**B. Choose the lens-pack.** Start from this default adversarial set and keep the ones that fit the artifact; drop the irrelevant:

1. **Correctness & logic** — where does it produce a wrong result for a valid input? Off-by-ones, wrong precedence, state that desyncs.
2. **Failure modes & edge cases** — what input or sequence of events breaks it? Empty, huge, concurrent, out-of-order, retried, mid-restart.
3. **Security & adversarial input** — untrusted input, injection, resource exhaustion, blast radius, what runs with what authority. (Include this whenever the artifact touches user input, the network, or a shared surface.)
4. **Integration & regression** — what existing behavior does this couple to or silently change? What relied on the thing being removed?
5. **Operational reality** — humans-in-the-loop who go silent, state that leaks or rots, metrics that drift, "what does this look like after v0 when nobody's watching."
6. **First principles** — is this even the right design, or is it machinery around one worked example / solving the wrong problem? (The one lens that's allowed to say "don't build this.")

**Add project-specific lenses** when the domain has known recurring failure modes — read the workspace's `CLAUDE.md`/memory and add them by name (e.g. for a fleet with a history of over-fitting to one example: a "worked-example trap" lens; for a system with self-graded metrics: a "metric integrity / Goodhart" lens). 4–7 lenses is the right range. More lenses ≠ better; a lens with nothing to attack returns nothing and wastes a slot.

### step 2 (red-team): convene the lenses (parallel sub-agents)

Spawn one sub-agent per lens, **in parallel**. Each gets: its lens, the artifact path(s), the neighborhood to read, and a demand for *concrete* findings. Insist on falsifiability and honesty — an empty list is a respected answer, and inventing findings to fill a quota is a failure.

**Lens sub-agent prompt template:**

```
You are red-teaming an artifact through ONE lens. Find real weaknesses that would
bite in production. Be adversarial and concrete.

ARTIFACT UNDER REVIEW: [path(s)]
READ FIRST: [artifact + the specific neighboring files/docs it touches]

YOUR LENS: [lens name + its one-paragraph description from the pack]

A finding counts ONLY if you can name a specific scenario — inputs, sequence of
events, or state — where the artifact produces a wrong or harmful outcome. No
vague "consider whether…". For each finding give: a one-line defect, the concrete
triggering scenario, the SMALLEST fix that removes it, and a severity
(blocker / major / minor / nit).

Do NOT invent problems to hit a number. An empty findings list is a valid,
respected answer if the artifact is sound on your lens. Rank your findings by
severity, most severe first.
```

Collect all findings across lenses into one list, each tagged with its lens.

### step 3 (red-team): refute each finding (parallel sub-agents)

This is the analogue of decision-mode's peer review, and the heart of the mode. For **each** finding, spawn a skeptic sub-agent (in parallel) whose job is to *refute* it — not to admire it. The default verdict is "refuted"; a finding earns "confirmed" only if its scenario is concretely reachable given what the code and design actually say.

**Refuter sub-agent prompt template:**

```
You are a skeptical verifier. A red-team reviewer claims this weakness in [artifact]:

DEFECT: [defect]
SCENARIO: [scenario]
CLAIMED SEVERITY: [severity]
PROPOSED FIX: [fix]

Try to REFUTE it. Read [artifact + relevant source] yourself. Decide:
(a) Is the scenario actually reachable, or does something in the code/design
    already prevent it?
(b) Is the claimed severity honest, or inflated?
(c) Is the proposed fix sound, or does it introduce a worse problem?

Default to verdict "refuted" if the scenario is not concretely reachable or the
artifact already handles it. Be fair — a real defect survives. Return a verdict
(confirmed / partially-confirmed / refuted), an adjusted severity, and your
reasoning.
```

Keep only findings whose verdict is not "refuted." These survivors are what the chairman sees. For anything claimed to be a *live* defect (already true in shipped code, not just in the plan), spot-verify the one or two highest-severity claims yourself by opening the file — the refuter is a sub-agent, and a live security/correctness claim earns a direct look before it reaches the user.

### step 4 (red-team): chairman synthesis of findings

One agent gets the artifact, all surviving findings with their verdicts, and the refuter reasoning. It produces the red-team verdict:

**Chairman prompt — produce this structure:**

```
## Verdict
[One paragraph: is this artifact sound, sound-with-fixes, or does it need rework?
Say it plainly.]

## Findings (severity-ranked)
[Each surviving finding: title · severity · the concrete scenario · the smallest fix.
Most severe first. If a finding is LIVE (true in shipped code today) vs LATENT
(only bites once the design ships), label it.]

## Convergence
[Findings that MULTIPLE independent lenses hit from different angles. These are the
highest-confidence signals — call them out explicitly.]

## Fix list, by leverage
[The fixes ordered by bang-for-buck — the one-line structural fixes that close whole
classes of finding go first. Not one fix per finding if one fix closes several.]

## What the artifact gets right
[Briefly — so the fixes don't regress the sound parts. Also honest: if a whole lens
returned nothing, that's a real clean bill on that axis, say so.]
```

The chairman may downgrade or drop a survivor whose fix costs more than the defect, and must not smooth over a blocker to sound balanced. The point of red-team mode is an honest map of where the thing breaks and what to do first — not reassurance.

Then continue to the shared report/transcript steps below, adapting the section names to findings (Verdict / Findings / Convergence / Fix list) instead of advisor verdicts.

---

### step 5: generate the council report

After the chairman synthesis is complete, generate a visual HTML report and save it to the user's workspace.

**File:** `council-report-[timestamp].html`

The report should be a single self-contained HTML file with inline CSS. Clean design, easy to scan. It should contain:

1. **The question** at the top
2. **The chairman's verdict** prominently displayed (this is what most people will read)
3. **An agreement/disagreement visual** — a simple visual showing which advisors aligned and which diverged. This could be a grid, a spectrum, or a simple breakdown showing advisor positions. Keep it clean and scannable.
4. **Collapsible sections** for each advisor's full response (collapsed by default so the page isn't overwhelming, but available if the user wants to dig in)
5. **Collapsible section** for the peer review highlights
6. **A footer** showing the timestamp and what was counciled

Use clean styling: white background, subtle borders, readable sans-serif font (system font stack), soft accent colors to distinguish advisor sections. Nothing flashy. It should look like a professional briefing document.

**Red-team mode report:** same file, same clean style, but the content is a findings briefing, not a debate. Put the **Verdict** at the top, then a **severity-ranked findings table** (title · severity · LIVE/LATENT · one-line scenario) as the scannable centerpiece in place of the agreement/disagreement visual, then the **Fix list by leverage**, then collapsible sections for each finding's full scenario+fix and for the refuted findings (so the user can see what was raised and dismissed, and why). Color-code severity (blocker/major/minor). Footer names the artifact reviewed and the lens-pack used.

Open the HTML file after generating it so the user can see it immediately.

### step 6: save the full transcript

Save the complete council transcript as `council-transcript-[timestamp].md` in the same location.

For **decision mode** it includes: the original question, the framed question, all 5 advisor responses, all 5 peer reviews (with anonymization mapping revealed), and the chairman's full synthesis.

For **red-team mode** it includes: the artifact reviewed, the lens-pack chosen (and why), every raw finding per lens, each finding's refutation verdict + reasoning (including the refuted ones — the dismissals are part of the record), any live claims you spot-verified yourself, and the chairman's findings synthesis.

This transcript is the artifact. If the user wants to run the council again on the same question after making changes, having the previous transcript lets them (or a future agent) see how the thinking evolved.

### step 7 (optional): register the decision in an outcome ledger

If the user wants their council decisions graded on realized outcomes (recommended
for recurring use), append one `pending` row to `council-ledger.jsonl` in the same
folder as the reports. This is the council's own honest metric: a council you only
ever grade at decision time is grading its own homework. Append a JSON object:

```json
{"id": "<report timestamp stem>", "date": "<YYYY-MM-DD>", "mode": "decision|red-team",
 "question": "<one line>", "recommendation": "<the chairman's rec, one-two lines>",
 "report": "<report filename>", "status": "pending", "resolve_after": "<a nudge date>",
 "resolve_hint": null, "outcome": null, "graded_by": null, "graded_at": null}
```

Do NOT fill `status`/`outcome` now — grading is a separate, later act by the
human, not the model that ran the council. When a `resolve_after` date has passed,
the human revisits the row and sets `outcome` to `good-call` / `bad-call` /
`unclear`; the ledger's hit-rate over graded rows is the council's track record.

## output format

Every council session produces two files:

```
council-report-[timestamp].html    # visual report for scanning
council-transcript-[timestamp].md  # full transcript for reference
```

The user sees the HTML report. The transcript is there if they want to dig deeper or reference specific advisor arguments later.

## important notes

- **Pick the mode first** (see the table at the top). Choosing → decision mode; checking an artifact → red-team mode. When ambiguous, ask one question.
- **Always spawn advisors/lenses in parallel.** Sequential spawning wastes time and lets earlier responses bleed into later ones.

Decision mode:
- **Always anonymize for peer review.** If reviewers know which advisor said what, they'll defer to certain thinking styles instead of evaluating on merit.
- **The chairman can disagree with the majority.** If 4 out of 5 advisors say "do it" but the reasoning of the 1 dissenter is strongest, the chairman should side with the dissenter and explain why.

Red-team mode:
- **Lenses must read the actual files.** A lens that reasons about the artifact without opening it is just a decision-mode persona wearing a lab coat — it will miss every code-level defect, which is the whole reason to use this mode.
- **Refute, don't rank.** The middle step defaults every finding to "refuted"; only concretely-reachable defects survive. This is what keeps a confident-but-wrong finding from surviving on eloquence.
- **Spot-verify live claims yourself.** Before telling the user "your shipped code has hole X," open the file and confirm it — the lenses and refuters are sub-agents, and a live security/correctness claim earns a direct look.
- **An empty lens is a real result.** A lens that returns nothing is a clean bill on that axis — report it, don't pad it. Don't invent findings to fill a quota.

Both modes:
- **Don't council trivial questions.** One right answer, a factual lookup, or a pure creation task → just do it. The council is for genuine uncertainty (decision) or an artifact worth breaking before you commit (red-team).
- **The visual report matters.** Most users will scan the report, not read the full transcript. Make the HTML output clean and scannable.
