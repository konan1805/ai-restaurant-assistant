# AI Restaurant Assistant

**An AI-powered restaurant ordering and pickup experience built with a stateful, multi-agent workflow.**

The AI Restaurant Assistant is an engineering portfolio project that explores how conversational AI can be integrated into a practical restaurant workflow: browsing a menu, managing a cart, verifying inventory, placing an order with explicit customer confirmation, and retrieving an existing order using a four-digit pickup number.

The application combines **LangGraph orchestration**, **LangChain-based agent/tool integration**, **tool calling and graph-based routing**, **stateful memory and checkpoints**, **human-in-the-loop interrupts**, a **Gradio storefront**, **SQLite persistence**, and **LangSmith observability**. The full implementation is maintained separately in a private source-code repository; this public repository documents the design, behavior, and engineering decisions.

> **Project status:** Working local prototype. The application has passed **107 automated tests** and has been exercised through the Gradio interface. This is a portfolio demonstration, not a claim of production deployment.

## At a glance

| Area | Capability |
| --- | --- |
| Customer experience | Conversational ordering alongside a visual, eight-item Gradio menu |
| Cart | Add items, review selections, and check inventory before checkout |
| Checkout | Explicit customer confirmation before the order is committed |
| Persistence | SQLite-backed orders, inventory, and pickup-number tracking |
| Reliability | Atomic order placement and idempotent checkout behavior |
| Pickup | Four-digit order lookup with itemized details and a three-failure session limit |
| Agent behavior | Restaurant-focused responses, including when questions mix ordering with unrelated trivia |
| Observability | LangSmith tracing of LangGraph executions, with latency, token, and cost information where available |
| Validation | 107 passing automated tests at the latest recorded test run |

## The problem

A restaurant assistant needs to do more than answer questions. It must connect natural-language interaction to **correct, auditable business actions**. A useful system should handle ambiguous requests without losing customer intent, avoid placing orders without consent, prevent duplicate inventory deductions on retries, and provide a reliable way to find an existing pickup order.

This project approaches those requirements as an **orchestrated application**, where the LLM helps interpret the customer's intent while deterministic tools enforce critical ordering rules.

## Customer-facing experience

Customers can interact with the assistant in natural language or use the menu displayed in the Gradio interface.

**Example journey**

1. Browse the menu and add items to the cart.
2. Ask the assistant to review the order.
3. Request checkout and explicitly **confirm**, **modify**, or **cancel**.
4. Receive an itemized receipt and a four-digit pickup number.
5. Return later to retrieve order details using that number.

### Application screenshots

#### Visual storefront and AI chat

![AI Restaurant Assistant storefront showing the menu, cart, and conversational interface](assets/storefront.png)

*Eight-item visual menu with cart actions and a conversational ordering assistant.*

#### Order review and customer approval

![Cart review with explicit Confirm, Modify, and Cancel controls](assets/order-review.png)

*The customer can confirm, modify, or cancel an order before inventory is deducted or an order is placed.*

#### Confirmed order and pickup receipt

![Confirmed order receipt with line items, total, and pickup number](assets/confirmed-order.png)

*After confirmation, the customer receives an itemized order receipt and a four-digit pickup number.*

#### Pickup verification and order details

![Pickup lookup showing verified pickup number, order status, items, quantities, and total](assets/pickup-lookup.png)

*Existing orders can be retrieved with a pickup number. Failed lookups are limited to three per conversation.*

## Applied AI engineering concepts

The project was developed alongside an advanced LangChain/LangGraph learning track. The table below distinguishes **features demonstrated in this application** from **related techniques studied separately**, rather than presenting every course topic as a shipped feature.

| Engineering concept | Application in this project |
| --- | --- |
| **Multi-agent orchestration** | A LangGraph supervisor routes customer messages between restaurant conversation, ordering, and pickup responsibilities. |
| **Graph nodes, edges, and conditional routing** | The workflow selects an appropriate path based on intent and current conversation state. |
| **Tool calling and tool boundaries** | Agents invoke application tools for menu/cart operations, inventory checks, order placement, and pickup lookups rather than directly modifying business data. |
| **Typed shared state** | Workflow state carries cart and checkout information, pickup-attempt counts, and conversation context between steps. |
| **Short-term memory and checkpointing** | A LangGraph in-memory checkpointer supports continuation within a running application session; checkpoint history is **not yet durably stored across process restarts**. |
| **Human-in-the-loop interrupts** | Checkout pauses for explicit customer approval; confirm, modify, and cancel lead to different workflow paths. |
| **Multi-step execution and tool results** | Cart mutation, inventory validation, customer approval, and persistence are distinct operations with validation between them. |
| **SQL-integrated agents and tools** | Business tools use SQLite transactions to persist orders and inventory; the LLM is not granted unrestricted database access. |
| **Guardrails and scoped behavior** | Restaurant-focused intent handling redirects unrelated questions, and pickup verification enforces three failed attempts per conversation. |
| **Observability / trace inspection** | LangSmith captures LangGraph executions for debugging tool calls, routing, latency, token usage, and estimated costs where available. |
| **Testing and reliability** | pytest tests exercise graph behavior, confirmation branches, stock invariants, concurrency, idempotency, migration, and pickup limits. |

### How a customer request travels through the system

1. **Receive input:** Gradio accepts a chat message or a cart action.
2. **Read state:** LangGraph resumes the conversation thread and evaluates the current cart, checkout phase, and pickup status.
3. **Route intent:** The supervisor sends the request to conversation assistance, order handling, or pickup lookup.
4. **Invoke tools:** The selected handler uses narrow, validated tools to perform relevant actions.
5. **Pause when necessary:** Checkout requests enter a human-approval step before placing an order.
6. **Commit deterministically:** An approved checkout uses an idempotency key and an atomic SQLite transaction.
7. **Respond and trace:** The user sees a result while LangSmith traces the supported execution steps.

### State, checkpoints, and persistence are different layers

A key architectural decision is to avoid treating conversational memory and restaurant data as the same thing:

- **LangGraph state:** Tracks per-conversation workflow details, such as cart progress, checkout readiness, and failed pickup attempts.
- **In-memory checkpointing:** Supports stateful continuation and human-in-the-loop resume while the current application process is running.
- **SQLite business persistence:** Retains orders, inventory, and pickup-number allocation independently of chat history, including across application restarts.
- **LangSmith traces:** Record observability data for inspection and debugging; they are not the application's order database.

This separation prevents a restarted chat session from becoming the source of truth for whether an order was actually placed.

### Agent design: reasoning where useful, deterministic rules where necessary

The LLM interprets varied customer language and decides which restaurant capability to use. But core business invariants are enforced in Python tools and database operations: valid items and quantities, sufficient inventory, explicit checkout approval, and duplicate-request handling. This reduces the risk of a plausible-sounding assistant response being mistaken for a completed transaction.

### Additional learning topics informing the design

The associated learning roadmap also includes **custom agents, middleware and hooks, agent skills, subagents, long-running/deep-agent patterns, streaming, structured outputs, retry/error-handling techniques, memory backends, evaluators, and SQL-agent development**. These topics informed the design exploration and future roadmap, but the public documentation does **not** claim that all of them are implemented in this restaurant prototype. For example, durable LangGraph checkpointing, a formal automated LangSmith evaluation dataset, and autonomous deep-agent research workflows are not presented as existing restaurant features.

## System architecture

The application separates **conversation orchestration**, **domain actions**, and **persistent storage**. LangGraph coordinates the interaction; agents interpret intents and requests; tools execute validated business operations; and SQLite stores the resulting order and inventory state.

```mermaid
flowchart TB
    Customer[Customer] --> UI[Gradio storefront and chat]
    UI --> Graph[LangGraph application workflow]

    Graph --> Supervisor[Supervisor / intent routing]
    Supervisor --> Conversation[Conversation / restaurant assistance]
    Supervisor --> Order[Ordering agent]
    Supervisor --> Pickup[Pickup handler]

    Order --> Cart[Cart and menu tools]
    Order --> Inventory[Inventory verification]
    Order --> Review[Customer confirmation gate]
    Review --> Placement[Order placement tool]

    Pickup --> Lookup[Pickup lookup tool]

    Cart --> Catalog[(Menu catalog and images)]
    Inventory --> DB[(SQLite: inventory and orders)]
    Placement --> DB
    Lookup --> DB

    Graph -. execution traces .-> LangSmith[LangSmith observability]
    Order -. agent/tool traces .-> LangSmith
    UI --> Response[Customer-visible reply and receipt]
    Graph --> Response
```

### Architectural responsibilities

| Component | Responsibility |
| --- | --- |
| **Gradio UI** | Presents the visual menu, cart, and chat interaction |
| **LangGraph workflow** | Maintains conversation state and coordinates route transitions |
| **Supervisor** | Classifies restaurant-relevant intent and routes requests |
| **Ordering agent** | Helps customers build a cart and prepare checkout actions |
| **Business tools** | Validate menu items and quantities, check inventory, and persist confirmed orders |
| **Pickup handler** | Looks up existing orders and applies a per-conversation failed-attempt limit |
| **SQLite** | Stores order history, order line items, inventory, and pickup-number sequence |
| **LangSmith** | Captures supported execution traces for debugging and operational analysis |

**Design boundary:** The language model does not get to bypass business checks. Inventory and persistence operations are handled in application tools, and checkout requires an explicit confirmation step.

## Workflow 1 — Conversation and intent routing

A supervisor interprets each message and delegates it to the correct business path. Restaurant-related requests take priority when a message mixes restaurant actions with unrelated questions. Off-topic questions receive a polite redirect rather than a general-knowledge answer.

```mermaid
flowchart TD
    A[Incoming customer message] --> B[Supervisor classifies intent]
    B --> C{Primary intent}
    C -->|Greeting or menu guidance| D[Restaurant-focused conversation reply]
    C -->|Order or cart request| E[Ordering workflow]
    C -->|Pickup request or number| F[Pickup workflow]
    C -->|Unrelated question| G[Redirect to ordering, menu, or pickup]
    C -->|Mixed request| H[Prioritize actionable restaurant request]
    H --> E
    H --> F
    D --> Z[Return response to customer]
    E --> Z
    F --> Z
    G --> Z
```

**Examples:**

- “What is the capital of France?” → redirect to restaurant services.
- “I want a burger, but first what's the capital of France?” → focus on helping select a burger.
- “I'm here for pickup.” → request the four-digit pickup number.

## Workflow 2 — Ordering, inventory, and confirmed checkout

Checkout is a multi-step business transaction, **not** an unrestricted agent action. Cart updates are distinct from inventory deduction; stock changes only when a confirmed order is placed. Repeated requests using the same checkout idempotency key are designed to return the original result rather than create another order.

```mermaid
flowchart TD
    A[Customer selects menu items] --> B[Resolve items and quantities]
    B --> C{Valid menu items and quantities?}
    C -->|No| D[Explain issue and request correction]
    D --> A
    C -->|Yes| E[Update cart]
    E --> F[Check whole-cart inventory]
    F --> G{Enough stock?}
    G -->|No| H[Offer adjustment]
    H --> A
    G -->|Yes| I[Customer requests checkout]
    I --> J[Present review / confirmation request]
    J --> K{Customer decision}
    K -->|Modify| A
    K -->|Cancel| L[No order is placed]
    K -->|Confirm| M[Submit idempotent order placement]
    M --> N[SQLite transaction: verify stock, save order, deduct inventory, allocate pickup number]
    N --> O{Placement result}
    O -->|Failure| P[Explain failure without claiming success]
    O -->|Success or safe retry| Q[Display itemized receipt and four-digit pickup number]
```

### Checkout safeguards

- **Human approval:** A request to checkout initiates review; it does not independently authorize order placement.
- **Atomic persistence:** The order and associated inventory updates are committed together in SQLite.
- **Idempotency:** Retrying an already completed checkout with the same request key should not place the order or deduct stock again.
- **Pickup identifiers:** Customers receive a simple four-digit number; internal order identifiers stay internal.
- **Clear receipts:** The interface displays purchased quantities, line totals, and the **final total before taxes**.

## Workflow 3 — Pickup verification and order retrieval

A customer can retrieve an existing order using its four-digit pickup number. A successful lookup returns **itemized order details**, rather than just “confirmed.” Unsuccessful lookups are counted within the conversation; the pickup session closes after **three failed attempts**.

```mermaid
flowchart TD
    A[Customer requests pickup] --> B[Ask for four-digit pickup number]
    B --> C[Receive number]
    C --> D{Pickup session already closed?}
    D -->|Yes| E[Direct customer to restaurant staff]
    D -->|No| F[Read-only lookup in saved orders]
    F --> G{Matching order found?}
    G -->|Yes| H[Reset failed-attempt counter]
    H --> I[Show status, items, quantities, prices, and total]
    G -->|No| J[Increment failed-attempt counter]
    J --> K{Three failures reached?}
    K -->|No| L[Show remaining attempts]
    L --> B
    K -->|Yes| M[Close pickup session and refer to restaurant staff]
```

**Scope of protection:** The three-failure limit is a **conversation-level usability safeguard**, not an account-wide security rate limiter. A new session can start a new counter. Pickup-number lookup is not a substitute for staff verification or stronger identity checks.

## Observability with LangSmith

LangSmith tracing is enabled and has been verified in the project dashboard. Captured LangGraph runs include ordering actions, checkout requests, pickup interactions, and intent-routing behavior. The tracing view exposes execution status and latency, plus token and estimated cost information for applicable runs.

```mermaid
flowchart LR
    A[Customer request] --> B[LangGraph execution]
    B --> C[Agent and tool steps]
    B -. trace data .-> D[LangSmith project]
    C -. trace data .-> D
    D --> E[Inspect execution paths]
    D --> F[Investigate latency and failures]
    D --> G[Review tokens and estimated cost]
```

**Observed during development:** A LangSmith dashboard screenshot showed **209 traces** at the time it was captured. This is a point-in-time observation, not a performance benchmark or service-level objective.

![LangSmith tracing dashboard with observed LangGraph runs, latency, token counts, and cost information](assets/langsmith-tracing.png)

*LangSmith tracing dashboard. Account sidebar removed for privacy; the screenshot reflects a development snapshot.*

> **Privacy note:** Any publicly shared trace screenshot will be reviewed to remove personal information, API details, and potentially sensitive customer or order content.

## Reliability and testing

The project has a pytest suite covering business rules and agent integration. At the latest recorded run, **107 tests passed in 7.56 seconds** on the developer's Windows environment.

Areas exercised include:

- Inventory validation and prevention of premature stock deduction
- Cart updates and invalid quantity handling
- Checkout confirmation, cancellation, and modification paths
- Idempotent order creation and duplicate-request handling
- Concurrent order scenarios and pickup-number allocation
- SQLite migration and compatibility checks
- Itemized pickup retrieval and three-attempt session closure
- Restaurant-focused intent routing and workflow integration

The test count documents the state of the project at that point; it is not a substitute for production load, security, or deployment testing.

## Technical patterns demonstrated

- **Supervisor pattern:** Route intents to specialized capabilities instead of asking one open-ended chatbot to handle everything.
- **State machine / graph workflow:** Track progression through customer requests, review, interruption, and finalization.
- **Human-in-the-loop approval:** Require user intervention before side-effecting checkout operations.
- **Idempotent commands:** Tie retries to stable request identifiers and return an existing order when appropriate.
- **Atomic transactional updates:** Keep inventory deduction and order insertion consistent.
- **Read-only queries:** Pickup retrieval displays a saved order without silently changing its status.
- **Graceful failure paths:** Report invalid products, insufficient stock, missing pickup numbers, and closed sessions without claiming success.
- **Tracing and troubleshooting:** Inspect LangGraph runs in LangSmith to understand routing and latency.

## Technology stack

| Technology | Role |
| --- | --- |
| **Python 3.13** | Application and business logic |
| **LangGraph** | Stateful workflow orchestration |
| **LangChain** | Agent and tool integration |
| **LangSmith** | Trace-based observability and debugging |
| **Gradio** | Interactive customer-facing storefront |
| **SQLite** | Local persistent order and inventory data |
| **pytest** | Automated regression and integration checks |
| **uv** | Python environment and dependency management |

## Engineering decisions and trade-offs

**Deterministic tools for critical actions.** The LLM supports interpretation and dialogue; tools enforce quantity checks, inventory availability, persistent writes, and order lookup.

**Explicit checkout approval.** Order placement is gated by the customer's confirmation, with modify and cancel branches available.

**Transaction safety and retries.** SQLite transactions coordinate updates, while idempotency keys prevent duplicate placement from the same checkout request.

**Session-level pickup restrictions.** Three failed attempts close the current pickup flow. This improves conversational handling but does not replace authentication, authorization, or platform-wide rate limiting.

**Local-first persistence.** The prototype stores order and inventory records in SQLite. Moving to multi-instance or hosted production would require an appropriate persistent database and additional operational controls.

**Historical prices.** Pickup details currently use menu pricing for display; storing an immutable unit-price snapshot at checkout is a planned improvement so historical receipts remain correct after price changes.

## Roadmap

- [ ] Save the original unit price and line total with each order
- [ ] Capture a customer name at checkout and design privacy-aware order recovery
- [ ] Refine the checkout-review wording and customer experience
- [ ] Evaluate deployment requirements, persistent hosting, and stronger operational security
- [ ] Expand trace-based evaluation and performance testing
- [ ] Explore durable LangGraph checkpoint storage for cross-restart conversation history
- [ ] Assess structured outputs, streaming UX, middleware/guardrails, and formal agent evaluations for a future release

## Source code access

The complete implementation is maintained in a **private GitHub repository**. This public repository is intended for project documentation, architecture reviews, and portfolio demonstrations.

**To request a code review:** [Open a source-code access request](../../issues/new?title=Source%20Code%20Access%20Request) and include your GitHub username, professional context, and reason for requesting access. Requests are reviewed individually; access is not guaranteed.

> GitHub does not automatically grant access when an issue is opened. Repository access must be managed separately by the owner. A GitHub organization with the appropriate **Read** role can be used for view-only collaborator permissions.

## Security and repository scope

This public repository is **documentation-only**. It intentionally does not contain application source files, local databases, `.env` files, credentials, customer records, or production secrets. Demonstration material should use fictional or sanitized data.

## Project focus

This project demonstrates how a conversational AI experience can be engineered into a structured workflow with clear boundaries, human approval, state management, persistence, testing, and observability—capabilities relevant to practical AI engineering and forward-deployed application development.
