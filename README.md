# LLM Council — a Claude Code skill

Run any question, decision, design, or piece of code through a council of AI advisors who independently analyze it, critique each other's work, and synthesize a final verdict. Adapted from [Andrej Karpathy's LLM Council](https://github.com/karpathy/llm-council) — instead of dispatching to multiple models, it uses parallel Claude sub-agents with deliberately different thinking lenses.

## Two modes

**Decision mode** — for choices with stakes ("should I X or Y?"). Five fixed advisors (Contrarian, First Principles, Expansionist, Outsider, Executor) answer independently, peer-rank each other's responses anonymously, and a chairman synthesizes a verdict with one concrete first step.

**Red-team mode** — for artifacts that could be wrong (a design doc, an architecture, a plan, code, a PR). Adversarial lenses chosen for the artifact **read the actual files**, every finding is attacked by a skeptic that defaults to "refuted," and only findings that survive cross-examination reach the synthesis. Choosing → decision mode; checking → red-team mode.

Every session produces a self-contained HTML report plus a full markdown transcript, and can optionally log the decision to an outcome ledger so you can grade the council on realized outcomes later.

## Install

```bash
git clone https://github.com/nfuller-shipped/llm-council ~/.claude/skills/llm-council
```

That's it. In your next Claude Code session, trigger it with phrases like:

- `council this: <a decision with stakes>`
- `red-team this design/plan/PR`
- `pressure-test this`, `find the holes`, `what breaks?`

Or invoke it directly as a skill. No dependencies, no configuration — it's a single `SKILL.md`.

## When to use it (and not)

Use it when being wrong is expensive: pricing decisions, pivots, architecture choices, a design you're about to commit weeks to. Don't use it for factual lookups, trivial yes/no questions, or pure creation tasks — a single answer is faster and just as good there.

## License

MIT
