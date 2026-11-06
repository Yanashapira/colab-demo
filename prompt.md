# Dana's Bakery Order Processing Graph — Technical Specification

## Objective

Implement a deterministic Python order-processing workflow with LangGraph. The workflow must not use an LLM, external API, or API key. It accepts an order, validates it, applies stock substitutions and delivery/discount rules, then returns a customer-facing result.

## Input

The graph input contains exactly these fields:

| Key | Type | Meaning |
|---|---|---|
| `item` | `str` | Requested menu item; matching is case-insensitive after trimming whitespace. |
| `quantity` | `int` | Number of items requested. |
| `day` | `str` | Order day; Friday matching is case-insensitive after trimming whitespace. |
| `city` | `str` | Delivery destination; an empty string means no city was provided. |

## Internal State

Track the following internal state fields while processing:

| Key | Type | Meaning |
|---|---|---|
| `status` | `str` | `"accepted"` or `"rejected"`. |
| `item` | `str` | Product ultimately sold; initially the normalized request and updated if substituted. |
| `delivery` | `str` | `"pickup"` or `"delivery"`. |
| `total` | `float` | Final amount in NIS. |
| `message` | `str` | Friendly customer-facing result, including applicable changes. |
| `steps` | `list[str]` | Ordered audit trail of processing decisions. |
| `attempt_count` | `int` | Number of unavailable substitution candidates tried; maximum two. |

The input fields `quantity`, `day`, and `city` remain available internally through processing. `graph.invoke()` must expose only the output fields listed below.

## Menu and Inventory

| Item | Unit price (NIS) | Stock |
|---|---:|---:|
| `challah` | 18 | 10 |
| `sourdough` | 25 | 4 |
| `rugelach` | 30 | 3 |
| `babka` | 45 | 0 |

## Validation and Substitution

1. Normalize the requested item by trimming whitespace and converting it to lowercase.
2. If the item is not on the menu, reject the order with a friendly error message.
3. If `quantity <= 0`, reject the order with a friendly error message.
4. If the requested item's inventory covers the full quantity, sell the requested item.
5. Otherwise, consider products only later in this chain: `babka → rugelach → sourdough → challah`.
6. A candidate is suitable only when its inventory is at least the full requested quantity. On finding a suitable candidate, substitute it and accept the order.
7. Increment `attempt_count` for each unavailable candidate examined. Stop after two unavailable candidates. If the second attempt fails, reject with an apology; do not examine a third candidate.
8. If the chain ends before a suitable candidate is found, reject with an apology. Never sell more units than available stock.

## Delivery and Pickup

- If `city` is empty or is not `Jerusalem` (case-insensitive after trimming), set `delivery="pickup"`, charge 0 NIS, and tell the customer in `message` that the order is pickup because delivery is unavailable for that city.
- If `city` is `Jerusalem`, set `delivery="delivery"` and add 20 NIS to the order.

## Discount (Avi's Rules)

- Calculate the baked-goods subtotal from the final sold product's unit price multiplied by quantity.
- If the baked-goods subtotal is strictly greater than 150 NIS and `day` is not Friday (case-insensitive after trimming), apply a 15% discount to the baked-goods subtotal only.
- Never discount the delivery fee. Add any 20 NIS delivery fee after calculating the discounted baked-goods amount.
- Otherwise, charge the full baked-goods subtotal.
- Store the final total as a `float`.

## Graph Behavior

- Build a compiled LangGraph `StateGraph` with explicit nodes and conditional routing for validation, stock/substitution, fulfillment pricing, and termination.
- Keep all business logic deterministic Python. Do not invoke an LLM or external API.
- Maintain an ordered `steps` audit trail for validation, substitution attempts or outcome, fulfillment choice, and pricing/discount decisions.
- Provide a rendered graph visualization with `graph.get_graph().draw_mermaid_png()`.

## Output Contract

`graph.invoke(input_dict)` returns a dictionary containing exactly these six keys, with no internal or input-only fields:

1. `status`
2. `item`
3. `delivery`
4. `total`
5. `message`
6. `steps`

Rejected orders have `status="rejected"` and `total=0.0`. Their `item` is the normalized requested item when available, `delivery` is an empty string, and the `steps` list explains the rejection.

## Required Notebook Examples

The Google Colab notebook `danas_bakery.ipynb` includes at least six deterministic example checks covering:

1. A valid pickup order.
2. A Jerusalem delivery order and delivery fee.
3. A successful stock substitution.
4. An invalid menu item.
5. Rejection after the second unavailable substitution candidate.
6. A Friday discount exception.

Each example checks the output keys and relevant expected values. The notebook displays the Mermaid PNG visualization and runs without API credentials.
