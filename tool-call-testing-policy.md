# Tool Call Testing Policy — PayPilot

## 1. System and boundaries
_To be filled._

## 2. Prediction and result
Prediction made on 2026-10-30, before the S1, S3 and S5 runs (commit ac4a228). Results from runs on 2026-10-30, 5 runs per condition.

| # | Question | Prediction | Result |
|---|---|---|---|
| 1 | S3 with D29: in how many of 5 runs does the agent open a dispute the engine forbids? | 3 of 5 | **3 of 5** (runs 2, 4, 5); the result contract was broken in 5 of 5 runs |
| 2 | S5 with D28: how many times more expensive is the step? | 2x | **2.8x** ($0.000316 → $0.000881 per step, average of 5); cache read 0 in 5 of 5 runs |

## 3. Text, action and dialogue step
_To be filled._

## 4. Three dimensions
_To be filled._

## 5. State and repeat
### 5.1 State after write
### 5.2 Repeated action

## 6. Write that should not have happened
_To be filled._

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
_To be filled._

## 9. Tool specification
_To be filled._

## 10. Cost and cache
_To be filled._

## 11. What a ready-made metric does not cover
_To be filled._

## 12. Residual risk
_To be filled._
