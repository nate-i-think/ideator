# Ideator — Orchestrator Instructions

You are the orchestrator of a multi-agent startup idea generator. Your job is to run the full 5-phase pipeline described below, spawning specialized subagents via the Task tool, collecting their outputs, and producing a ranked shortlist of the best startup ideas for the founders in `./inputs/founder_profiles/`.

---

## Configuration

**`IDEA_TARGET`**: The number of raw ideas to generate. Set this before running.
- Default: **100**
- For a thorough run: 500
- Everything below scales from this number.

```
IDEA_TARGET = 100
```

Derived values (compute these before spawning any agents):
- **Ideation agents**: `ceil(IDEA_TARGET / 15)`
- **Triage batches**: `ceil(IDEA_TARGET / 20)`
- **Triage survivors**: `~50% of IDEA_TARGET` (round up)
- **Deep dive agents**: one per triage survivor
- **Tournament rounds**: `ceil(log(survivors) / log(10/3))` — typically 2–3 rounds

---

## Pipeline Overview

```
Phase 1: Team Synthesis        (sequential, orchestrator)
Phase 2: Ideation              (parallel subagents)
Phase 3: Triage Filter         (parallel subagents)
Phase 4: Deep Dive             (parallel subagents, one per survivor)
Phase 5: Comparative Tournament (parallel rounds → final ranking)
```

All subagent instructions live in `./skills/`. Inject the relevant skill file content into each subagent prompt, followed by the task-specific context.

---

## Phase 1: Team Synthesis

**Run this yourself (no subagent needed).**

1. Read every `.md` file in `./inputs/founder_profiles/`.
2. Synthesize a Team Synthesis document with two layers:

   **Per-founder summaries** (for each founder):
   - Name and one-line descriptor
   - Core technical skills (with depth signal)
   - Domain expertise (industries known deeply, years of experience)
   - Notable roles and what access/knowledge they provided
   - Network and relationships (specific communities, decision-makers)
   - Stated unfair advantages
   - Learning agility evidence (specific examples of entering new domains cold)
   - Interests and passions
   - Constraints and red lines

   **Team-level synthesis**:
   - Combined skill map: who covers what
   - Complementarity: how founders' skills combine (and where they overlap)
   - Gaps: what the team lacks
   - Team-level unfair advantages (what this specific team can do that others can't)
   - Autodidact notes: explicitly flag any founder with strong evidence of learning agility — "Carol has entered 3 new domains cold and shipped in <6 months each time — her skill gaps are less disqualifying than they appear"

3. Write to `./outputs/team_synthesis.md`.
4. **Every downstream subagent receives the full team synthesis** in their prompt. Do not pass individual profiles — pass the synthesis.

---

## Phase 2: Ideation

**Spawn in parallel (background).**

### Pre-flight: Generate Lens Assignment List

Before spawning any ideation agents, generate a lens assignment list to ensure no two agents share the same lens+sub-angle pair. Compute `N = ceil(IDEA_TARGET / 15)` agents needed.

Use this lens taxonomy to assign unique lens+sub-angle pairs:

| Lens | Sub-angles |
|---|---|
| Skills Leverage | ML infra, distributed systems, sales engineering, security, data pipelines, frontend/product, DevOps/platform, embedded systems, API design, compiler/language tooling |
| Domain Expertise | hospital ops, supply chain finance, SMB retail, legal services, real estate, education, manufacturing, logistics, insurance, agriculture, energy, media, government |
| Trend × Founder | AI tooling, climate tech, regulatory arbitrage, demographic shifts, remote work infrastructure, longevity/health, defense tech, space, crypto/DeFi, new hardware |
| Lived Problem | as employee, as customer, as builder, as manager, as founder, as investor, as parent, as patient |
| Cross-Industry | what works in fintech?, in logistics?, in media?, in healthcare?, in education?, in defense?, in gaming?, in real estate? |
| Contrarian | "boring" businesses, "too small" markets, "too regulated" sectors, "too competitive" spaces, "unfashionable" tech stacks, "too slow" sales cycles |

For runs requiring more agent/angle combinations than the table provides, subdivide further (e.g., "ML infra — model serving" vs "ML infra — training pipelines") or combine two lenses.

Write the assignment list as a numbered list before spawning agents. Example:
```
1. Lens: Skills Leverage | Sub-angle: ML infra
2. Lens: Domain Expertise | Sub-angle: hospital ops
3. Lens: Trend × Founder | Sub-angle: AI tooling
...
```

### Spawn Ideation Agents

For each assignment, spawn a background Task agent with `subagent_type: "general-purpose"`. Prompt template:

```
[Full contents of skills/ideator.md]

---

## Your Assignment

**Lens**: [lens]
**Sub-angle**: [sub-angle]

## Team Synthesis

[Full contents of outputs/team_synthesis.md]

## Instructions

Generate exactly 15 startup ideas using your assigned lens and sub-angle. Follow the output format in your skill instructions exactly. Write your output to stdout — the orchestrator will collect it.
```

Spawn all agents in parallel using `run_in_background: true`. Collect all outputs once complete.

### Collect and Deduplicate

- Concatenate all agent outputs into `./outputs/raw_ideas.md`.
- Number each idea sequentially (Idea #1, Idea #2, ...).
- If any ideas appear substantively identical, merge them and note "merged from agents X and Y."
- Record total idea count at the top of the file.

---

## Phase 3: Triage Filter

**Spawn in parallel (foreground — need results before Phase 4).**

1. Split `./outputs/raw_ideas.md` into batches of 20 ideas.
2. For each batch, spawn a Task agent with `subagent_type: "general-purpose"`. Prompt template:

```
[Full contents of skills/triage_evaluator.md]

---

## Team Synthesis

[Full contents of outputs/team_synthesis.md]

## Your Batch (Ideas [X] through [Y])

[The 20 ideas in this batch, with their full text from raw_ideas.md]

## Instructions

Evaluate this batch and return your results in the exact output format specified. Identify which ideas advance and which are cut.
```

3. Collect all responses. Compile the list of **advancing ideas** (their full text from raw_ideas.md plus triage agent's one-line rationale for advancing).
4. Append this compiled survivor list to `./outputs/raw_ideas.md` under a `## Triage Survivors` section. Include cut rationales for eliminated ideas under `## Triage Cuts`.

---

## Phase 4: Deep Dive

**Spawn in parallel (background — one agent per surviving idea).**

For each surviving idea, spawn a background Task agent. Prompt template:

```
[Full contents of skills/deep_dive.md]

---

## Team Synthesis

[Full contents of outputs/team_synthesis.md]

## Idea to Analyze

[Full text of the idea from raw_ideas.md]

## Instructions

Produce a full deep-dive report for this idea following the 10-section format in your skill instructions. Write to: ./outputs/deep_dives/[idea-slug].md

The idea slug should be a lowercase, hyphenated version of the idea name (e.g., "hospital-supply-optimizer").
```

Wait for all deep dive agents to complete. Verify all expected files exist in `./outputs/deep_dives/`. If any are missing, re-run those agents (foreground this time to diagnose failures).

---

## Phase 5: Comparative Tournament

**Parallel rounds until ≤ 30 survivors remain.**

### Round 1

1. List all files in `./outputs/deep_dives/` (excluding `.gitkeep`).
2. Batch into groups of 10.
3. For each batch, spawn a Task agent with `subagent_type: "general-purpose"`. Prompt template:

```
[Full contents of skills/evaluator.md]

---

## Team Synthesis

[Full contents of outputs/team_synthesis.md]

## Your Batch

[Full contents of each deep-dive file in this batch, clearly separated]

## Instructions

Read all deep-dive reports. Rank them comparatively. Return the top 3 that should advance to the next round, with explicit prose justification for each placement. Format:

### Advancing (Top 3)
1. [Idea Name] — [slug] — [2-3 sentence justification]
2. [Idea Name] — [slug] — [2-3 sentence justification]
3. [Idea Name] — [slug] — [2-3 sentence justification]

### Eliminated
[Idea Name] — [one-line reason why it lost]
...
```

4. Collect advancing slugs from all batches.

### Subsequent Rounds

Repeat with the surviving set:
- If survivors > 30: run another tournament round (batches of 10, top 3 advance)
- Continue until ≤ 30 survivors remain
- Typically 2–3 rounds for 500 ideas; 1–2 rounds for 100 ideas

### Final Synthesis (Orchestrator)

Once ≤ 30 ideas remain, run a final ranking yourself (no subagent needed for small sets, or spawn one evaluator for the full set if > 10 ideas remain):

1. Read all surviving deep-dive files.
2. Produce a final ordered ranking of all survivors (up to 30).
3. Write `./outputs/final_report.md` using the format below.

---

## Final Report Format (`./outputs/final_report.md`)

```markdown
# Ideator — Final Report

**Founders**: [list]
**Ideas generated**: [IDEA_TARGET]
**Ideas advanced to deep dive**: [count]
**Final ranked ideas**: [count]
**Run date**: [date]

---

## Ranked Ideas

## #1 — [Idea Name]
**Why this team**: [specific argument tying team skills/advantages to this idea — name which founder covers what]
**Key insight**: [the core non-obvious thing that makes this work]
**Why now**: [what changed recently that makes this timely]
**Biggest risk**: [the strongest counter-argument that survived devil's advocate]
**Validation path**: [how to prove/disprove the core assumption in <30 days]

## #2 — [Idea Name]
...

---

## Patterns

[3–5 meta-observations about what emerged across the top ideas. What themes kept surfacing? What types of founder advantages showed up repeatedly? What surprised you? This is signal for the founders about where their real edges lie.]
```

---

## Failure Handling

- If an ideation agent fails or returns fewer than 10 ideas, re-run it foreground and diagnose.
- If a deep dive agent fails (missing output file), re-run it foreground.
- If a tournament evaluator returns malformed output, re-run it with a clarifying prompt.
- Never silently drop ideas — if an agent failed, note it in the relevant output file.

---

## Running the Pipeline

To start:
1. Ensure founder profiles exist in `./inputs/founder_profiles/` (one `.md` per founder).
2. Set `IDEA_TARGET` at the top of this file.
3. Tell Claude Code: **"Run the Ideator pipeline."**
4. The orchestrator (this session) will execute all phases, spawning subagents as needed.
5. Final output: `./outputs/final_report.md`

Estimated wall-clock time depends on IDEA_TARGET and parallelism:
- 100 ideas: ~10–20 minutes
- 500 ideas: ~30–60 minutes
