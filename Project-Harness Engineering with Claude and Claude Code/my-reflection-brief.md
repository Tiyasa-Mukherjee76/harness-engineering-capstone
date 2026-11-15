
# Reflection Brief — Harness Engineering Capstone

**Name:** Tiyasa Mukherjee
**Date:** 2026-11-15

## Part 1 — Per-system

### System 1 — Agentic loop

1. Loop control.
In run 20261114_233455, the trace file runs/20261114_233455/traces/claim_07_tree_falls_on_car.jsonl showed the sequence tool_use → tool_use → tool_use → end_turn. The control logic is implemented in claims_intake/loop.py, which explicitly states the loop continues when stop_reason == "tool_use" and stops when stop_reason == "end_turn". This design ensures the loop performs exactly as many tool interactions as needed before termination.

2. Anti-pattern.
One anti-pattern checked by test_antipatterns.py is relying on hard-coded loop behavior instead of stop_reason-driven execution. In run 20261114_233455, different claims required different numbers of turns. For example, claim_01_kitchen_fire completed in 2 turns while claim_07_tree_falls_on_car required 4 turns. A fixed-turn loop could terminate early or waste unnecessary model calls.

3. Tool design.
The tools classify_claim and assess_severity operate on the same claim facts but perform different functions. classify_claim determines the claim category while assess_severity determines urgency and impact. Their descriptions separate classification from prioritization, reducing misrouting. Structured outputs also allow downstream automation to react to specific fields rather than parsing generic error strings.

4. Your numbers.
According to runs/20261114_233455/summary.md, claim_07_tree_falls_on_car required 4 turns, consumed 14,730 input tokens and 1,000 output tokens, and cost approximately $0.0197. Simpler claims such as claim_01_kitchen_fire completed in only 2 turns and cost $0.0089.

### System 2 — Context strategy

5. The reduction.
According to runs/20261114-234852/budget.json, the baseline transcript contained 38,708 tokens and the assembled context contained 16,824 tokens, producing a 56.54% reduction. The largest section was the active portion at 15,789 tokens.

6. Summarize vs preserve.
The strategy summarizes resolved issues and preserves the active issue byte-for-byte. The refund discussion was compressed from 12,334 tokens to 381 and the subscription discussion from 11,475 to 442. Meanwhile, the active section remained 15,789 tokens.

7. Facts block.
Comparing eval.jsonl and eval_control.jsonl, Q6 regressed in the control run. The full context correctly answered "in_progress" while the stripped version returned "unknown". This proves the Case Facts block preserves structured information that is difficult to reconstruct from conversation history alone.

### System 3 — Claude Code config

8. Path-scoped rules.
The root CLAUDE.md imports shared standards including frontend.md, api.md, database.md, and testing.md. The validator returned OK and the project passed all 35 tests. Path-scoped rules allow standards to be reused consistently across the monorepo.

9. Forked skill.
The file .claude/skills/deploy-check/SKILL.md contains context: fork and a read-only allowed-tools section. Running in a forked context prevents exploratory output from polluting the main session while the read-only restrictions prevent accidental modifications.

10. Scope.
The command python -m ecommerce_team_config . returned OK and all 35 tests passed. Project-level examples include CLAUDE.md and .claude/commands/review.md. User-level preferences remain outside source control.

### System 4 — Orchestration

11. Push work down.
The system uses the indexed defects_since() query defined in shift_monitor/warm.py. The test test_defects_since_uses_index_and_does_not_load_full_table verifies this behavior. Only relevant rows are returned to the model rather than the full history.

12. Crash recovery.
The recovery design is validated by test_threshold_constant_is_30_minutes. The comments in shift_monitor/recovery.py describe a 30-minute threshold that determines resume-versus-fresh execution. Fresh execution avoids inheriting stale or partially corrupted state.

13. Small state.
The architecture enforces a bounded hot-state budget. Evidence includes test_hotstate_roundtrip_and_size_budget and test_hotstate_rejects_more_than_20_hashes. This prevents long-term state growth from becoming an operational problem.

## Part 2 — Synthesis

14. Three layers.

Model: runs/20261114_233455/traces/*.jsonl and runs/20261114-234852/eval.jsonl demonstrate model reasoning and answer generation.

Harness: claims_intake/loop.py and CLAUDE.md define how the model is guided and controlled.

Orchestration: shift_monitor/recovery.py, shift_monitor/warm.py, and shift_monitor/state.py coordinate retrieval, persistence, and recovery.

15. Deterministic vs prompt.
A deterministic behavior is the stop_reason loop in claims_intake/loop.py. A prompt-guided behavior is the summarization used in run 20261114-234852. Deterministic controls are appropriate for safety-critical flows, while prompts are effective for summarization and interpretation.

16. Context, two faces.
System 2 reduced context from 38,708 tokens to 16,824 tokens, a 56.54% reduction. System 4 uses hot, warm, and cold storage together with defects_since() retrieval. Both systems reduce unnecessary context but operate at different timescales.

17. Reliability you can't see in one run.
A single run cannot prove crash recovery works correctly. Tests such as test_recovery_decide_truth_table and test_threshold_constant_is_30_minutes validate edge cases that may never appear during a demonstration.

18. Blast radius.
System 1 has a significant blast radius because it routes insurance claims. The stop_reason-based loop in claims_intake/loop.py and escalation handling demonstrated by claim_06_low_confidence_escalation help limit the consequences of mistakes.

## Part 3 — Honest assessment

19. What broke.
Several commands initially failed because they were executed from the wrong directory. Examples included attempting to access .claude content from the System 2 folder and running commands against non-existent paths. Navigating to the correct solution directories resolved the issue.

20. What you'd change.
I would expose more operational metrics through generated runtime artifacts instead of relying primarily on tests. Metrics such as recovery mode selection, state utilization, and retrieval statistics would improve observability while preserving the existing architecture.