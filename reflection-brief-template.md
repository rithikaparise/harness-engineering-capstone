# Reflection Brief — Harness Engineering Capstone

**Name:** Rithika Parise
**Date:** 2026-10-06

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): claude-haiku-4-5-20251001 (default; S1 attempts + all S2 calls) and claude-sonnet-4-6 (S1 runs 20261002_035647 at $0.4909, 20261003_054441 at $0.5212, probe 20261002_040414), all via the Vocareum proxy https://claude.vocareum.com
- OS / Python: code-server Linux, Python 3.13, one venv per system (S1 anthropic 0.39.0 plus an httpx 0.27.2 pin; S2 anthropic 0.69.0)
- Approx. API spend: ~$6.50 of the $20 program budget ($13.48 remaining at last check); S1 Sonnet passes ~$0.50, Haiku passes ~$0.11, the S2 full run ~$1

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   My trace.jsonl (claim_01_kitchen_fire, run 20261002_035647) is two turns: turn 1 stop_reason tool_use (lookup_policy POL-1001 plus five record_claim_fact calls, 2810 in / 365 out), turn 2 stop_reason end_turn with zero tool calls (3484 in / 148 out). The loop therefore terminated the claim as incomplete. The decision lives in claims_intake/loop.py, function run(): it appends tool results and continues only when response.stop_reason == "tool_use", returns the FinalState on "end_turn", and raises UnexpectedStopReason on anything else. No counter or text match participates. Artifact: system_1_claims_loop/trace.jsonl + summary.md (20261002_035647).

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   tests/test_antipatterns.py (29 passed, system_1_claims_loop/tests_TASK1.txt) forbids driving control flow by string-membership on assistant text and forbids a turn-count cap as the stop condition. If the loop branched on text, my claim_06 probe (run 20261002_040414: turn 3 end_turn carrying a plain-text reply, zero tool calls) could be misread as progress or spin forever awaiting a magic token; with stop_reason control it deterministically yields incomplete. The suite likewise pins the Budget object as the only safety net: it raises BudgetExceeded and never silently truncates.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   classify_claim and request_clarification overlap: both judge claim type. classify_claim's description says commit exactly once with claim_type, confidence in [0,1], and a one-sentence rationale; request_clarification says use it only when genuinely ambiguous between named candidate types via ambiguity_between, one focused question per missing fact. That wording is why claim_04 asked one focused question and then routed liability/high in run 20261002_035647 instead of committing blind. Structured tool errors (is_error, error_category, is_retryable, message — e.g. routing before classify) let the model repair ordering on the next tool_use turn; a bare-string error would leave it guessing and likely end the turn, which is exactly how claims go incomplete. Artifacts: summary.md outcomes + run_TASK2 logs.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   In summary.md (run 20261002_035647, claude-sonnet-4-6) claim_05_auto_collision routed auto/high in 5 turns, 0 clarifications, 18551 in / 998 out, $0.0706. The README sample shows the same shape routing in 4 turns near $0.02 on Haiku; mine cost ~3x more because Sonnet inserts an extra record-then-classify round trip (5 turns vs 4) and charges a higher per-token price — the whole run totals $0.4909 vs ~$0.11-0.13 for a Haiku pass. Turn counts legitimately span 2-7 across my attempts: stalled claims die at 2-3 turns while claim_03 once needed 7 with a clarification. Artifact: system_1_claims_loop/summary.md.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   My budget.json (run 20261002-043538): baseline 38708, assembled 16825, reduction 56.53%, clearing the 50% bar. The active segment dominates at 15789 tokens against case_facts 204, resolved_refund 371, and resolved_subscription 479. Active stays verbatim because it holds the live payment-method troubleshooting turns the eval questions quote; compressing it would destroy the exact tokens answers are graded against. Artifact: system_2_context_strategy/budget.json.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   Rule: summarize resolved history, preserve the active segment and case facts byte-exact. In-to-out per budget.json: refund compresses from ~12k to 371 tokens, subscription from ~11k to 479, while active stays 15789 and case_facts 204 verbatim. The suite enforces this with test_active_segment_byte_exact (30 passed in tests_TASK1.txt): byte-exact copying is what keeps Q1-Q5 answerable after 56.53% of the context is gone. Artifacts: budget.json + tests_TASK1.txt.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   eval.jsonl (run 20261002-043538) is 6/6 PASS, including Q6 answering with the exact in_progress status token from the case facts. eval_control.jsonl (same run, facts block stripped) flips Q6 to FAIL: "there is no structured status token recorded for the payment-method update issue" instead of in_progress. That single regression proves the persistent facts block carries the answerable token. Q1 unexpectedly still passed in the control, so the proof rests on Q6, not Q1. Artifacts: system_2_context_strategy/eval.jsonl + eval_control.jsonl.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   My .claude/rules/react.md frontmatter reads: description "Conventions for React components and pages", paths ["src/components/**/*", "src/pages/**/*"]. A path-scoped rule beats a directory-level CLAUDE.md for cross-cutting conventions because one file follows matching paths wherever they live (components, pages, and test files scattered repo-wide), while a directory CLAUDE.md only covers its own subtree and would have to be duplicated per folder. The validator (OK, exit 0 in validator_TASK4.txt) plus tests test_ac_02_04 through test_ac_02_06 pin the glob matching. Artifacts: .claude/rules/*.md + validator_TASK4.txt.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   My .claude/skills/deploy-check/SKILL.md frontmatter sets context: fork with a read-only allowed-tools list (Read, Grep, Glob, git status/diff/log, gh pr view — zero write-capable entries, enforced by test_ac_04_03 and the validator). Forked plus read-only buys isolation in both directions: the check cannot mutate code mid-review, and its long output cannot flood the main session's context — only a pass/fail summary returns. Without it, a pre-deployment check could edit the tree it is judging or drown the session that must act on the verdict. Artifacts: SKILL.md + validator_TASK4.txt.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    The validator documents both scopes (validator_TASK4.txt prints OK only when the scope block is complete). Project-level, versioned in git: ./CLAUDE.md plus .claude/standards/*.md pulled in via @-imports (e.g. line 21 @.claude/standards/frontend.md) — conventions the whole team shares. User-level, never versioned: ~/.claude/ personal preferences such as my own commit-message style, which per the CLAUDE.md scope table stay on my laptop and never reach teammates.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    My shift processed 30 new defects from the SQL slice (since 2026-03-11T00:00:00Z) against a warm-tier total of 40 rows — the model saw 30/40, never the full history. The indexed query is defects_since in shift_monitor/warm.py (test_defects_since_uses_index_and_does_not_load_full_table plus respects_limit pin it: filtering happens in SQL, no Python-side scan). The model never sees full history because the prompt is built from the slice only: bounded tokens per shift, no cross-shift leakage, and constant cost no matter how many shifts accumulate. Artifacts: system_4_orchestration/run_TASK5.txt + hot_state.json + shift_scratchpad.jsonl.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    The resume-vs-fresh call lives in shift_monitor/recovery.py (recovery_decide): resume only when the manifest shows a complete prior shift younger than the staleness threshold; otherwise start fresh. The threshold constant is 30 minutes (test_threshold_constant_is_30_minutes plus the truth-table cases: 1 min + incomplete manifest → resume; 31 min → fresh; completed-then-crashed → fresh). A fresh start with an injected summary is more reliable than resuming because a partial manifest plus a half-written scratchpad is untrusted context — resuming risks reasoning from corrupt state, while the compact summary is a verified checkpoint. Artifacts: tests_TASK1.txt (33 passed) + run_TASK5.txt.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    My data/hot_state.json is 1022 bytes (hot_state_size.txt), against the ~5 KB budget — with recent_defect_hashes capped at 20 entries. The budget matters precisely because the system runs once per shift indefinitely: without the cap the hash list grows forever, and so does every future prompt, cost, and crash-recovery read. test_hotstate_rejects_more_than_20_hashes enforces the ceiling before shipping, which no single good run would ever trip. Artifacts: hot_state.json + hot_state_size.txt + tests_TASK1.txt.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: claims_intake/client.py make_client() and retail_context/client.py complete() — the only places that call the Anthropic API.
    → Harness: claims_intake/loop.py run() (the stop_reason loop) and retail_context/assemble.py build() (section ordering + verbatim rules) — engineering around a single model call.
    → Orchestration: shift_monitor/pipeline.py run_shift() with shift_monitor/warm.py WarmStore — fresh session per shift, SQL pre-filtering, tiered files across invocations. Each layer lives in exactly one named file, which is why a bug in each has a different blast radius.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    Deterministic: loop termination on stop_reason (claims_intake/loop.py — UnexpectedStopReason on anything but tool_use/end_turn), read-only allowed-tools (the System 3 validator rejects write-capable entries), atomic hot-state writes (state.py write_atomic), the 20-hash cap and the 30-minute recovery threshold. Prompt-guided: the 7-step claims process in system_prompt.py, ambiguity_between wording in the clarification tool, the compression prompt in retail_context/prompts/. Code is right for safety and correctness invariants (a missed termination or a stray write is unacceptable at any model quality); prompt is right for judgment calls — classification, severity, summary wording — where strict code would be brittle. One of each, both from my builds.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    Same principle — keep what the next decision needs, push the rest down — different mechanism. System 2 manages context within one conversation: 38708 baseline tokens compressed to 16825 (−56.53%) by summarizing resolved issues to 371 + 479 tokens while the active 15789 stays verbatim (budget.json, run 20261002-043538). System 4 manages context across sessions: a 1022-byte hot file (20 hashes + one summary) plus a 40-row warm SQLite the model queries in 30-row slices, with monthly cold summaries for the rest. Intra-session squeezes tokens out of a prompt; cross-session squeezes state out of files — both keep the model's working set constant-size as history grows unbounded.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    test_hotstate_rejects_more_than_20_hashes guarantees the 20-entry ceiling: my submitted hot_state.json holds exactly 20 hashes at 1022 bytes, and no single successful run would ever attempt the 21st write that the test forces. The same holds for the recovery truth table (31 minutes + stale manifest → fresh) and the pruner contract (exactly the contracted field set survives). These matter before shipping because the orchestrator runs once per shift indefinitely — boundary behavior underasseresses years of growth, and one clean demo run exercises none of it. Artifact: system_4_orchestration/tests_TASK1.txt (33 passed).

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    System 1. Blast radius if it misbehaves: file appends under runs/<ts>/ only — queues/*.jsonl routings and escalations.jsonl; its tools have no network access and touch nothing outside the ClaimSession directory, so the worst case is a misfiled claim record, never an external action. Kill switches: the Budget (500k input tokens / 180s wall clock → BudgetExceeded aborts the run) and UnexpectedStopReason (any stop_reason outside tool_use/end_turn crashes loudly instead of continuing blind). Both are deterministic code, not prompt hopes — verified by the 29-test suite, not by any single run.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    On a fresh code-server container, pip install -e ".[dev]" in System 1 pulled httpx 0.28.1, which removed the proxies keyword that anthropic==0.39.0 passes at client construction — every live call died with TypeError: Client.__init__() got an unexpected keyword argument 'proxies' before spending a cent. I pinned httpx==0.27.2 (matching the last known-good resolve), verified with pip show httpx, re-ran pytest to 29 passed, and the next --all run reached the model fine. Nothing in the reference code was touched; the fix is environment-only and documented here so it reproduces.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    I would add a deterministic completeness nudge to the System 1 loop: when a turn ends with end_turn but no terminal tool was ever called, spend at most one automatic re-prompt ("you stopped without classify/route/escalate — continue the chain or escalate") before accepting incomplete. Grounded in what I observed: claim_06 stalled 0-for-9 across Haiku and Sonnet, seven full runs and a solo probe, always dying right after fact-recording (probe 20261002_040414: 3 turns, 0 clarifications) while the harness could only watch. It keeps the stop_reason contract intact — the model still decides — but converts the most common failure into one more paid turn instead of a dead claim. Costs a little; wastes far less than my retry spiral did.
