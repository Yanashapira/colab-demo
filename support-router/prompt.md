# TaskNest Support Router — Technical Specification

## Objective

Classify each TaskNest support ticket into one of three categories, assign the corresponding support team, and return a short, polite fixed reply. The workflow is a demonstration and does not contact production systems.

## Runtime and credentials

- Use `gpt-4o-mini` through `langchain_openai.ChatOpenAI`, with `temperature=0`.
- `gpt-4o-mini` was selected as the low-cost model for a simple three-way text classification task.
- Install `langgraph`, `langchain-openai`, and `pydantic` in Google Colab.
- Read `OPENAI_API_KEY` from Google Colab Secrets at notebook startup. Never use `getpass` or hardcode an API key.

## State contract

The LangGraph state is a `TypedDict` with exactly four fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `ticket` | `str` | Customer's incoming ticket text |
| `category` | `Literal["billing", "technical", "general"]` | Classification result |
| `team` | `str` | Assigned support team |
| `reply` | `str` | Fixed polite acknowledgement for the customer |

## Classification rules

Use one structured LLM call in the `classify` node. Its Pydantic result restricts `category` to `billing`, `technical`, or `general`, and the node returns only `{"category": ...}` to the graph state.

- `billing`: payments, invoices, refunds, and subscriptions.
- `technical`: bugs, crashes, errors, and login issues.
- `general`: feature requests, general questions, and thank-you notes.
- Prioritize the user's intent over isolated keywords. For example, a crash while attempting payment is primarily a technical failure; route it to `technical`.

No other node or routing function may call the model. The pure-Python `route(state)` function returns `state["category"]` as the conditional-edge key.

## Graph and team handling

The conditional edge maps `billing` to `billing_team`, `technical` to `tech_team`, and `general` to `general_team`. Each terminal team node is pure Python and sets both its team name and a short, fixed, polite reply, then transitions to `END`:

- `billing_team` → `Billing Team`
- `tech_team` → `Tech Support Team`
- `general_team` → `General Support Team`

## Notebook output

Compile and visualize the graph with `draw_mermaid_png()`. Run seven example tickets and print each ticket, category, team, and reply: two billing examples, two technical examples, two general examples, and the ambiguous case “The app crashes every time I try to pay”. The ambiguous case is routed according to intent priority.
