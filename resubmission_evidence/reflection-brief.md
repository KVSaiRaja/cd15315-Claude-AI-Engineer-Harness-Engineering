# Reflection Brief — Harness Engineering Capstone

**Name:** Kuppa Venkata Sai Raja
**Date:** 2026-09-26

## Environment

* **Model(s):** Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) and recorded Claude responses
* **OS / Python:** Linux / Python 3.11.16
* **Approx. API spend:** System 1 run approximately **$0.1017**; System 2 used Claude for context compression and evaluation

---

## Part 1 — Per-system

### System 1 — Agentic loop

**1. Loop control.**
In the trace for `claim_05_auto_collision`, the `stop_reason` sequence was **`tool_use → tool_use → tool_use → end_turn`**, recorded in `03-dynamic-decomposition/solution/runs/20260926_150035/traces/claim_05_auto_collision.jsonl`. The file `claims_intake/loop.py` uses the `run_agentic_loop` function to control the loop. It continues when `stop_reason == "tool_use"` and stops when `stop_reason == "end_turn"`. Other stop reasons raise an error.

**2. Anti-pattern.**
One anti-pattern checked by `tests/test_antipatterns.py` is an **integer-literal iteration cap**, such as `for _ in range(5)`. If this were used, the loop could stop after a fixed number of iterations before the model returned `end_turn`. This could leave a claim incomplete. The System 1 anti-pattern tests passed as part of the **29-test** suite.

**3. Tool design.**
Two tools with related decisions are `route_to_adjuster` and `escalate_to_human` in `claims_intake/tools.py`. Their descriptions separate normal routing when confidence is sufficient from escalation when confidence is low, facts are missing, ambiguity remains, or there is a policy dispute. The structured escalation tool keeps information such as the policy ID, root cause, and candidate claim types, which gives the next human or process useful information instead of only a generic error message. The System 1 full test suite passed **29 tests**.

**4. Your numbers.**
For `claim_05_auto_collision`, the run summary records **4 turns** and an estimated cost of **$0.0215** in `03-dynamic-decomposition/solution/run-summary.md`. The README sample expected approximately **$0.05 total** for the complete run and different fixture outcomes. My live run produced a total estimated cost of **$0.1017** and 3 routed claims, while the README expected 7 routed and 1 escalated. The difference is from live model/API behavior; the implementation itself matched the reference solution and all **29 tests passed**.

---

### System 2 — Context strategy

**5. The reduction.**
The latest System 2 run was `runs/20260926-175948`. The baseline context was **38,708 tokens**, while the assembled context was **16,998 tokens**, giving a **56.09% reduction**. The `active` section was the largest section at **15,789 tokens** because it contains the current unresolved conversation and should remain available in detail. Keeping the active conversation verbatim helps preserve the exact current situation and recent user interactions.

**6. Summarize vs preserve.**
Resolved topics are compressed into summaries, while the active conversation and structured case facts are preserved. In the latest run, `resolved_refund` was **505 tokens**, `resolved_subscription` was **518 tokens**, `case_facts` was **204 tokens**, and `active` was **15,789 tokens**. This keeps important structured facts available while reducing older resolved conversation content.

**7. Facts block.**
The latest evaluation produced **6/6 passes** in `eval.jsonl`. In `eval-control.jsonl`, Q1 unexpectedly passed, but Q6 failed as expected. Q6 required the exact structured status `in_progress`, which existed in the case-facts block. This shows that the case-facts block is load-bearing for structured information that may not be recoverable from the remaining conversation.

---

### System 3 — Claude Code config

**8. Path-scoped rules.**
The frontmatter in `.claude/rules/api.md` contains:

`paths: ["src/api/**/*"]`

The rule therefore applies only to matching API files. This is better than putting the convention in a directory-level `CLAUDE.md` when the convention is cross-cutting but should not affect unrelated files. The System 3 validator returned **`OK`**, and the complete exercise suite passed **35 tests**.

**9. Forked skill.**
The `.claude/skills/deploy-check/SKILL.md` contains `context: fork` and an `allowed-tools` list containing read-only tools such as `Read`, `Grep`, `Glob`, and restricted Git/GitHub commands. Running it in a fork keeps detailed discovery work out of the main session and prevents the skill from making deployment or repository-write changes. Without the read-only allowlist, the skill could have a larger blast radius because it would have access to more powerful operations. The configuration validator returned **`OK`**.

**10. Scope.**
The project-level example is `.claude/skills/deploy-check/SKILL.md`, which belongs to this repository's `.claude` configuration. The user-level example described by the configuration is `~/.claude/skills/deploy-check-strict/`. Project-level configuration travels with the repository, while user-level configuration is personal to the developer. The project configuration passed the validator with **`OK`**.

---

### System 4 — Orchestration

**11. Push work down.**
The warm tier contained **40 defects**, while a manual SQL `defects_since()` query using `ts > 2026-04-01T00:00:00Z` returned **17 relevant defects**. The indexed query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, implemented by `WarmStore.defects_since()` in `shift_monitor/warm.py`. The official Shift C run used `--since 2026-04-30T00:00:00Z` and returned **0 new defects**. The model does not need to see all 40 defects because filtering is performed by SQLite before the results reach the model. This reduces context size and moves deterministic data selection into the database.

**12. Crash recovery.**
`shift_monitor/recovery.py` uses a **30-minute** staleness threshold. An incomplete run whose last step is within 30 minutes is resumed; an older incomplete run starts fresh with the findings captured in the manifest injected as a summary. A fresh start can be more reliable because it avoids depending on stale partial conversational state while still preserving the important findings already recorded. The decision is implemented by `decide()` using `STALE_RESUME_THRESHOLD_MINUTES = 30`.

**13. Small state.**
My generated `/tmp/hot_state.json` was **643 bytes**, while the project test enforces a maximum hot-state budget of **5,120 bytes** in `tests/test_us01_tiered_state.py`. Keeping the state small matters because the system runs repeatedly, once per shift, and could continue indefinitely. A bounded hot state prevents historical information from growing without limit and keeps each new model invocation predictable and manageable.

---

## Part 2 — Synthesis

**14. Three layers.**

→ **Model:** The Claude model performs reasoning and tool selection, demonstrated by the System 1 traces in `03-dynamic-decomposition/solution/runs/20260926_150035/`.

→ **Harness:** The harness controls behavior around the model, such as `claims_intake/loop.py`, the Claude Code `allowed-tools` configuration, and the System 2 context assembly code.

→ **Orchestration:** System 4 provides cross-shift state and recovery through `shift_monitor/state.py`, `shift_monitor/warm.py`, and `shift_monitor/recovery.py`.

Together, these separate model reasoning, deterministic controls, and long-running system state.

**15. Deterministic vs prompt.**
A deterministic behavior is the System 3 read-only `allowed-tools` list in `.claude/skills/deploy-check/SKILL.md`; the configuration directly limits what the skill can do. A prompt-guided behavior is the System 1 instruction telling the model to look up policy information early and collect facts before classification. Deterministic controls are appropriate when safety or correctness must be enforced, while prompts are useful for guiding flexible reasoning and task behavior.

**16. Context, two faces.**
System 2 manages context **inside a session** by reducing the baseline from **38,708 tokens to 16,998 tokens**, a **56.09% reduction** in run `20260926-175948`. System 4 manages context **across shifts** by keeping a small 643-byte hot state and moving larger history into the warm SQLite tier. Both systems follow the same principle: keep the information needed for the current task close to the model and move unnecessary historical information out of the active context.

**17. Reliability you can't see in one run.**
The System 4 tests verify that the hot state stays within the **5,120-byte budget** and that the `defects_since()` operation uses the database index. A single successful run would not necessarily reveal whether the state would grow indefinitely or whether the application was loading and filtering the entire defect table. **Automated tests can repeatedly enforce these architectural properties across controlled cases, while a manual run only shows that one particular execution succeeded.**

**18. Blast radius.**
System 4 has a relatively contained blast radius because its persistent state is separated into hot, warm, and scratchpad layers. The `allowed-tools` and forked execution in System 3 provide another safety boundary for operational checks. In System 4, the recovery logic and state files provide control points for restarting a problematic run, while the read-only deployment skill prevents operational checks from directly changing or deploying the repository.

---

## Part 3 — Honest assessment

**19. What broke.**
One environment problem occurred in System 1 because the project required `anthropic==0.39.0`, while the environment initially had a newer Anthropic package that caused a `Client.__init__` compatibility error. I fixed this by installing the required Anthropic version and using a compatible `httpx` version. System 2 had the opposite dependency requirement and needed `anthropic==0.69.0`; after using the correct environment, its full suite passed **28 tests with 2 skipped**.

**20. What you'd change.**
I would improve the project setup by isolating each system's Python dependencies from the beginning. The different Anthropic SDK requirements caused avoidable environment problems while moving between systems. Separate virtual environments from the first step would make the workflow faster and reduce dependency conflicts. The final implementations were still verified successfully: System 1 had **29 passed**, System 2 had **28 passed and 2 skipped**, System 3 had **35 passed**, and System 4 had **33 passed**.


