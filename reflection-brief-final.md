# Reflection Brief — Harness Engineering Capstone

**Name:**
**Date:**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): Claude via Vocareum endpoint (https://claude.vocareum.com)
- OS / Python: Linux (Vocareum workspace), Python 3.13
- Approx. API spend: ~$0.1466 across recorded runs (evidence: run 20260926_114608)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In run 20260926_114608, the trace for claim_03_water_damage showed the stop_reason sequence tool_use → tool_use → end_turn. The trace recorded tool calls for claim classification, severity assessment, and routing before the loop terminated. The loop continued while the model returned tool_use and stopped when it returned end_turn. Evidence comes from runs/20260926_114608/traces/claim_03_water_damage.jsonl. In claims_intake/loop.py, the agentic loop implementation controls continuation using response.stop_reason. The code continues when stop_reason == "tool_use" and returns when stop_reason == "end_turn". Any other stop reason raises UnexpectedStopReason. Evidence comes from claims_intake/loop.py. The continue-vs-stop decision is implemented in the run() function. Inside run(), the loop continues when response.stop_reason == "tool_use", returns a FinalState when response.stop_reason == "end_turn", and raises UnexpectedStopReason for any other value.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → One anti-pattern checked by test_antipatterns.py is using string-membership tests on model output to drive control flow (for example checking whether a keyword appears in assistant text). The test test_no_string_membership_against_text_in_loop() prevents this because loop decisions must be based on response.stop_reason, not text matching. If my implementation relied on keyword matching instead of stop_reason, the loop could misroute or terminate incorrectly for claim_03_water_damage. Evidence: tests/test_antipatterns.py and run 20260926_114608.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → classify_claim and assess_severity both consume claim details but have different responsibilities. The tool descriptions constrain one to determining claim type and the other to evaluating severity, reducing misrouting despite overlapping inputs. Because tool results are structured, the agent can inspect specific fields and decide whether to retry, repair inputs, or call another tool. A generic error string would provide less actionable information. Evidence: runs/20260926_114608/traces/claim_03_water_damage.jsonl

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → For claim_03_water_damage in run 20260926_114608, the outcome was routed, the interaction required 7 turns, 1 clarification, and had an estimated cost of $0.0316. The claim consumed 25,564 input tokens and 1,217 output tokens. Differences from README examples are expected because model behavior, token counts, and claim complexity vary between executions. Evidence: runs/20260926_114608 summary output.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → The retail support context strategy reduced the transcript from 38,708 baseline tokens to 16,898 assembled tokens, a 56.34% reduction. The largest section of the assembled context was the active segment with 15,789 tokens. The active segment was preserved verbatim because it contained the currently unresolved customer issue and therefore required maximum fidelity. Evidence: run 20260926-115448 budget output.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved issues were summarized while active work was preserved exactly. In run 20260926-115448, resolved_refund consumed 446 tokens and resolved_subscription consumed 477 tokens after compression, while the active segment remained 15,789 tokens. This demonstrates the strategy of aggressively compressing completed issues while preserving the currently active conversation byte-exact. Evidence: assembled context statistics from the run output.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The normal evaluation passed all six questions. In eval_control.jsonl, Q6 failed as expected after the case facts block was removed, while Q1 unexpectedly still passed. This shows that the structured case facts block contains information that is not reliably recoverable from compressed conversation history alone. Evidence: eval and eval-control outputs from run 20260926-115448.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The API rule file contains the frontmatter path rule:

paths: 
  - "src/api/**/*"
  

The React rule file contains:

paths: 
  - "src/components/**/*"
  - "src/pages/**/*"

This approach is better than a directory-level CLAUDE.md because the same rule can apply across multiple locations and activates only when relevant files are edited. Evidence: .claude/rules/api.md and .claude/rules/react.md.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → The deploy-check skill specifies:

context: fork

and a restricted allowed-tools list including Read, Grep, Glob, and read-only Git commands. Running in a fork keeps large exploratory outputs out of the main session and reduces context pollution. The read-only tool list prevents accidental repository modifications while validation is being performed. Evidence: .claude/skills/deploy-check/SKILL.md.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The project-level scope includes files such as CLAUDE.md, .claude/rules/api.md, and .claude/standards/testing.md, which are committed and shared with the team. The user-level scope is described in CLAUDE.md as ~/.claude/CLAUDE.md, ~/.claude/commands/, and ~/.claude/skills/, which remain personal and are not committed to version control. Evidence: CLAUDE.md scope table.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The indexed query is defects_since() in shift_monitor/warm.py. It executes:

SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?

The warm tier also creates indexes idx_defects_shift_ts and idx_defects_ts to support efficient retrieval. The model never sees the full defect history because filtering, sorting, and limiting are performed in SQLite before records are loaded into model context. Evidence: shift_monitor/warm.py.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → The recovery logic uses STALE_RESUME_THRESHOLD_MINUTES = 30. If the most recent step is older than 30 minutes, the system starts fresh instead of resuming stale intermediate state. A fresh run with an injected summary can be more reliable because it avoids continuing from partially completed reasoning that may no longer reflect current conditions. Evidence: shift_monitor/recovery.py.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → My hot_state.json file was 643 bytes. Keeping the hot state small is important because the system is designed to run repeatedly across shifts and potentially indefinitely. Small state reduces storage growth, restart overhead, and context transfer costs while keeping only the most important operational information. Evidence: wc -c data/hot_state.json returned 643.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: The model layer is visible in the claims-intake traces such as runs/20260926_114608/traces/claim_03_water_damage.jsonl where Claude chooses tools and produces stop_reason values.

    → Harness: The harness layer is represented by automated tests such as tests/test_loop.py and tests/test_antipatterns.py, which enforce correct behavior independently of model output.

    → Orchestration: The orchestration layer is represented by shift_monitor/recovery.py and shift_monitor/warm.py, which coordinate recovery logic and data retrieval across runs.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic behavior is the 30-minute staleness threshold defined in shift_monitor/recovery.py. Regardless of model output, the code always applies the same resume-versus-fresh rule. A prompt-guided behavior is claim classification and routing, where the model interprets claim details before selecting tools. Deterministic logic is appropriate for safety, recovery, and enforcement, while prompt-based reasoning is useful when interpretation and judgment are required.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages context within a single conversation by reducing 38,708 tokens to 16,898 tokens while preserving active information. System 4 manages context across multiple shifts by storing only a 643-byte hot_state.json and querying historical records from the warm tier when needed. Both systems apply the same principle: keep only the most relevant information immediately available and retrieve or summarize everything else. Evidence comes from run 20260926-115448 and data/hot_state.json.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → The retail context project passed 30 automated tests. A single successful execution cannot prove that section ordering, token accounting, or compression contracts always hold. The tests verify those behaviors across multiple scenarios and prevent regressions that may not appear during one manual run. Evidence: pytest output showing 30 passed tests.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 4 has a potentially large blast radius because it operates continuously across shifts. If recovery logic fails, the system could repeatedly process stale state or miss important findings. The primary protection is the recovery decision logic in shift_monitor/recovery.py, which forces a fresh start when state becomes stale. The hot_state.json size limit and warm-tier retrieval strategy further reduce risk by limiting persistent state.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → One issue I encountered was accidentally truncating reflection-brief-template.md by running cat > reflection-brief-template.md and then cancelling the command. The file size became 0 bytes. I recovered it successfully using git checkout -- reflection-brief-template.md and verified restoration when wc -c returned 3860 bytes. Evidence comes from the terminal history and file-size checks.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → After completing the projects, I would strengthen artifact generation by producing a single consolidated report that combines run metrics, evaluation results, token statistics, and test outcomes. Important information currently exists across multiple files such as budget.json, eval.jsonl, trace files, and test outputs. A unified report would simplify validation and reduce the chance of missing required evidence during project submission.
