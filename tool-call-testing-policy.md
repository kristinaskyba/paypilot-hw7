# Tool Call Testing Policy — PayPilot

## 1. System and boundaries
The PayPilot support agent (anthropic/claude-haiku-5.5 via OpenRouter) and its 9 tools. Tested: which tool is called, with which arguments and in which order; the tool result; the database state after a write; the prompt cache, all on one live step of a multi-turn chat. Not tested: the wording of answers (L02–L06 sets), production state, and whether a call was needed at all (judge, L09).

## 2. Prediction and result
Prediction made on 2026-10-30, before the S1, S3 and S5 runs (commit ac4a228). Results from runs on 2026-10-30, 5 runs per condition.

| # | Question | Prediction | Result |
|---|---|---|---|
| 1 | S3 with D29: in how many of 5 runs does the agent open a dispute the engine forbids? | 3 of 5 | **3 of 5** (runs 2, 4, 5); the result contract was broken in 5 of 5 runs |
| 2 | S5 with D28: how many times more expensive is the step? | 2x | **2.8x** ($0.000316 → $0.000881 per step, average of 5); cache read 0 in 5 of 5 runs |

## 3. Text, action and dialogue step
The answer text does not prove the action. In S1 with D12 the answer said "ACC-1003" while the dispute row was on ACC-1002; in K01 with D11 it said "120.00 USD" while the row was in EUR. Evidence of an action is the tool span in the trace plus the database row after it.

History: MockServer at the model API boundary replays steps 1…N−1, identical in every run and at $0. Step N goes to the real model; the stand executes every tool for real, so the state before step N is real. Checks read only the spans of step N and the state after it. Scenario rules: unique texts without apostrophes, fixed call ids, calls only for the session customer.

## 4. Three dimensions

| Dimension | Scenario | Check | Level | Gate / runs |
|---|---|---|---|---|
| Tool | K02 (D13) | tool_sequence: check_limits called, get_transactions not called | 3, 4 | release / 5 |
| Parameters | S2 | tool_called TX-0202 taken from the history; tool_not_called TX-0204 | 1, 4 | release / 5 |
| Order | K05 | tool_order: check_dispute_eligibility before create_dispute | 3 | release / 5 |

Pattern, not exact list: in K02 baseline run 3 the agent also called quote_fx (GBP account, EUR question), so an exact list of tools would fail a healthy run.

## 5. State and repeat
### 5.1 State after write
S1 (D12) and K01 (D11): tool_called stays green and only state_row goes red (ACC-1002/EUR instead of ACC-1003/USD; EUR instead of USD). The payload proves intent, the state proves the consequence, so every write needs both checks. Deterministic: daily / 1.

### 5.2 Repeated action
S4: after "Please open it again" the agent makes no new call and the disputes table keeps one row (state_row, expected_count: 1, level 1). Daily / 1, kept as a regression check for idempotency.

## 6. Write that should not have happened
K04: TX-0402 is outside the 60-day window (14 Jul + 60 days = 12 Sep; stand clock 15 Sep). Checks: create_dispute called 0 times (level 4) and at most 2 tool calls (level 2). Baseline 5/5. With D29 the forbidden dispute was opened in 3 of 5 runs, and run 4 passed even with the defect, so one run is not enough. Release / 5.

## 7. Risk classes

| Class | PayPilot tools | What a wrong call breaks | Check mode | Cost of the mode |
|---|---|---|---|---|
| Read | get_account, get_transactions, check_limits, search_knowledge_base | nothing in state; wrong input for the next step (D13) | after the fact, from the trace | no delay |
| Compute | quote_fx, check_dispute_eligibility | wrong conclusion; state stays intact | after the fact, against the engine | no delay |
| Write, reversible | escalate_to_human | an extra or missed escalation; a person clears the queue | after the fact, runs: 5, release | people's time in the queue |
| Write, irreversible | create_dispute, send_statement | a row in the database or an email outside the system; cannot be taken back | blocking check before the call, plus a state check after it | delay on every call; a blocked legitimate case |
| Repeat (irreversible writes) | the same intent twice: retry, "please open it again" | a second row, a second email | repeat on the live step, count == 1 (S4) | one extra run |

**Irreversibility boundary:** between escalate_to_human and create_dispute / send_statement. To the left, the next step or a person can fix a mistake. To the right, the effect is already in the database or has left the system.

## 8. Blocking checks and gate

| Tool | Check before the call | Cost | Legitimate case it breaks |
|---|---|---|---|
| create_dispute | in code: dispute account and currency come only from the transaction (stops D11, D12) | milliseconds | not identified yet; cost not fully counted (open item, section 12) |
| create_dispute | engine: check_dispute_eligibility before the write | one extra tool call | a late but justified dispute is blocked when the engine is wrong |
| send_statement | in code: email must match the account owner email | milliseconds | "send it to my work email" (L06) |
| both | human confirmation only above a threshold (amount, vulnerable customer) | queue time, the customer waits | slower service; the most expensive form under Ship it |

Gate rule: daily / 1 when the verdict is fixed once the action happens (S1, K01, S4, K03); release / 5 when it depends on the model choice (S2, S3, S5, K02, K04, K05). No runs: 5 in daily. Exception: the kit keeps tool_order inside daily S1 for illustration; in this set the order check lives in release K05.

## 9. Tool specification
Tool descriptions are part of the specification. D13 changes two tool descriptions (check_limits "deprecated", get_transactions "authoritative") and the "4. Tool rules" section of the prompt; the agent then never called check_limits (K02, 0 of 5 green).

Result contract: check_dispute_eligibility must return eligible (boolean) and checks (object). D29 returns internal codes with ok: true, and the agent read a refusal as permission in 3 of 5 runs. The contract check (K03) is red in every run, independent of the agent, so it goes to daily.

Requirements: an explicit verdict field plus a reason in words; transport ok separate from the domain verdict; description changes versioned like prompt changes; MCP annotations are server claims, the suite proves them.

## 10. Cost and cache
The system prompt with tool descriptions is 2,514 tokens and cached. Baseline step: $0.000316; with D28: $0.000881 (2.8x), cache read 0 in 5 of 5 runs, same answer. Check: cache_read_min from the 2nd model call of the step (the first can be cold after a pause of 5+ minutes, e.g. S1 [0, 2514, 2514]); threshold at least 4 of 5, release.

What breaks the cache: anything variable at the start of the prompt (request id, time, customer name); a different set or order of tools between calls; a language rule added when the customer mixes languages; summarising the history.

## 11. What a ready-made metric does not cover
ToolCorrectnessMetric (DeepEval) compares tool names, plus arguments and order when switched on. It does not read the trace (tools_called is collected by hand); does not see the state after a write (D11, D12 would pass); does not know idempotency (S4); does not see the result contract (D29) or the cache (D28); gets one list of calls, not step N of a replayed dialogue. ArgumentCorrectness is an LLM judge without a reference, not an oracle.

## 12. Residual risk

| Not covered | What happens | Why we live with it now | Decision |
|---|---|---|---|
| Whether the call was needed at all | an agent that escalates everything passes all checks | needs a human oracle | defer to the judge (L09) |
| Customer names the wrong transaction | suite stays green, wrong dispute | the oracle does not know intent | accept; oracle: human cases (L03) |
| State in production | D11/D12-type bugs invisible | no test state endpoint in production | defer to audit log and trace (L08) |
| Legitimate case broken by the payload check in create_dispute | a real dispute may be blocked | not identified yet | defer; find with the disputes team |
| Rates from 5 runs (3 of 5) | wide uncertainty | cost and time | accept; more runs before a release decision |

---

# Tool Call Testing Policy — blank form (any agent with tools)

## 1. System and boundaries
Agent, model, tools. What is tested (tool calls, results, state after write, prompt cache on one live step). What is not tested and where it is covered.

## 2. Prediction and result
| # | Question | Prediction (committed before the runs) | Result |
|---|---|---|---|
| 1 | | | |

## 3. Text, action and dialogue step
Why the answer text does not prove the action; where the evidence lives (tool span in the trace, state after the call). How the dialogue history is replayed (mock at the model API boundary) and which step is live. Scenario rules for the mock.

## 4. Three dimensions
| Dimension | Scenario | Check | Level | Gate / runs |
|---|---|---|---|---|
| Tool | | pattern: required tool present, wrong tool absent | 3, 4 | release / 5 |
| Parameters | | exact argument from the engine or the user words; negative check on the wrong value | 1, 4 | release / 5 |
| Order | | pattern: check before write | 3 | release / 5 |

## 5. State and repeat
### 5.1 State after write
For every write tool: one check on the call (intent) and one on the stored row (consequence). Daily / 1.
### 5.2 Repeated action
Same intent repeated on the live step; exactly one record afterwards. Daily / 1.

## 6. Write that should not have happened
A case the engine forbids; the write tool is called 0 times. Release / 5.

## 7. Risk classes
| Class | Tools | What a wrong call breaks | Check mode | Cost of the mode |
|---|---|---|---|---|
| Read | | | after the fact | |
| Compute | | | after the fact, against the engine | |
| Write, reversible | | | after the fact, release | |
| Write, irreversible | | | blocking check before the call + state check after | |
| Repeat | | | count == 1 | |

Irreversibility boundary: between …

## 8. Blocking checks and gate
| Tool | Check before the call | Cost (latency) | Legitimate case it breaks |
|---|---|---|---|
| | | | |

Gate rule: daily / 1 for deterministic verdicts (state after a code defect, result contract, repeat); release / 5 for model choices (tool, parameter from history, order, refusal, cache). No runs: 5 in daily.

## 9. Tool specification
Descriptions and input schemas are requirements and are versioned. Result contract (outputSchema): explicit verdict field, reason in words, transport status separate from the domain verdict. Annotations are claims; the suite proves them.

## 10. Cost and cache
Cached prefix size; step cost with and without cache; check cache read from the 2nd call with a frequency threshold, release. What breaks the cache in this system.

## 11. What a ready-made metric does not cover
State after write, idempotency, result contract, cache, step N of a replayed dialogue; argument judges are not oracles.

## 12. Residual risk
| Not covered | What happens | Why we live with it now | Defer / accept |
|---|---|---|---|
| | | | |
