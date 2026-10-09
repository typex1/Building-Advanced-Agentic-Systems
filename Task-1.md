# Task 1: Build a Personal Budget Assistant with Strands

This document explains how the agents built in `Task-1.ipynb` relate to each other,
using Mermaid diagrams.

Unlike Task 2 (which coordinates multiple agents at runtime), Task 1 is a **progressive
build-up of a single agent**. Each step starts from the same `BedrockModel` and adds
one capability — a system prompt, then conversation management, then tools, then
structured output — until it reaches the final `budget_agent` that is exported to
`budget_agent.py` for Tasks 2 and 3 to import.

So the "relationship" here is not a call graph between peers; it is a chain of
increasingly capable agent configurations that all share the same underlying model.

## 1. Progressive Agent Build-Up

Each agent in the notebook is a new `Agent(...)` configuration layering one more
feature on top of the previous idea. They are separate variables, not callers of one
another.

```mermaid
graph TD
    Model["BedrockModel<br/>(Nova Pro, temp=0.0)"]

    Model --> A1["agent<br/>(basic, model only)"]
    Model --> A2["budget_agent_sys<br/>+ system prompt"]
    Model --> A3["budget_agent_manager<br/>+ conversation manager"]
    Model --> A4["budget_agent<br/>+ tools"]
    Model --> A5["report_agent / budget_agent.py<br/>+ structured output"]

    A1 -.->|adds persona| A2
    A2 -.->|adds context mgmt| A3
    A3 -.->|adds financial tools| A4
    A4 -.->|adds Pydantic schema| A5

    style A5 fill:#e8f5e9
```

The dashed arrows show *conceptual progression* (each step inherits the ideas of the
previous one), not function calls.

## 2. Feature Layering

Each capability added across the tasks, and which agent variable introduces it.

```mermaid
graph LR
    subgraph "Task 1.0 — Basics"
        B1["Guardrail<br/>(create_guardrail)"]
        B2["BedrockModel<br/>(+ guardrail_id)"]
        B3["agent"]
    end

    subgraph "Task 1.1 — Persona"
        P1["BUDGET_SYSTEM_PROMPT"]
        P2["budget_agent_sys"]
    end

    subgraph "Task 1.2 — Memory"
        M1["SummarizingConversationManager<br/>(ratio 0.5, keep 3)"]
        M2["budget_agent_manager"]
    end

    subgraph "Task 1.4 — Tools"
        T1["calculate_budget"]
        T2["create_financial_chart"]
        T3["calculator"]
        T4["budget_agent"]
    end

    subgraph "Task 1.5 — Structured Output"
        S1["FinancialReport / BudgetCategory<br/>(Pydantic)"]
        S2["report_agent → budget_agent.py"]
    end

    B1 --> B2 --> B3
    B3 --> P2
    P1 --> P2
    P2 --> M2
    M1 --> M2
    M2 --> T4
    T1 --> T4
    T2 --> T4
    T3 --> T4
    T4 --> S2
    S1 --> S2
```

## 3. The Final Budget Agent and Its Tools

The complete `budget_agent` (the one exported to `budget_agent.py`) wraps the Bedrock
model with a system prompt and three tools. Note it does **not** attach the guardrail —
the structured-output call uses a forced tool call that a guardrail intervention would
break, so the exported agent omits it.

```mermaid
graph TD
    BudgetAgent["budget_agent<br/>(system prompt + tools)"]

    BudgetAgent --> Tool1["calculate_budget<br/>(50/30/20 breakdown)"]
    BudgetAgent --> Tool2["create_financial_chart<br/>(matplotlib pie chart)"]
    BudgetAgent --> Tool3["calculator<br/>(strands_tools)"]

    BudgetAgent --> SO["structured_output(FinancialReport)"]
    SO --> Report["FinancialReport<br/>• monthly_income<br/>• budget_categories[]<br/>• recommendations[]<br/>• financial_health_score"]
    Report --> Cat["BudgetCategory<br/>• name<br/>• amount<br/>• percentage"]
```

## 4. Guardrail Interaction

The guardrail (`guardrail-no-bitcoin-advice`) is attached to the model used by the
early agents. When it intervenes, the Strands SDK redacts the offending message in the
conversation history so follow-up questions aren't also blocked.

```mermaid
flowchart TD
    Start([User query]) --> Agent["agent / budget_agent_sys<br/>(guardrail attached)"]
    Agent --> Guard{Guardrail evaluates}
    Guard -->|allowed| Normal[Model responds normally]
    Guard -->|intervened<br/>e.g. 'Bitcoin investment advice'| Redact["Input redacted in history<br/>+ safe refusal returned"]
    Normal --> End([Response])
    Redact --> End
```

The notebook demonstrates this directly: `agent("Bitcoin investment advice")` is
blocked, and `pretty_print_messages` then shows the user turn stored as
`[User input redacted.]`.

## 5. Export and Reuse

The final cell uses `%%writefile budget_agent.py` to serialize the finished agent to a
standalone module. This is why `budget_agent.py` appears inside a code cell: the cell
isn't executed as Python, it writes its body to disk so later notebooks can import it.

```mermaid
graph TD
    Notebook["Task-1.ipynb<br/>(builds budget_agent step by step)"]
    Notebook -->|%%writefile| File["budget_agent.py<br/>(exports budget_agent, FinancialReport)"]

    File -->|from budget_agent import ...| Task2["Task-2.ipynb / main.py"]
    File -->|from budget_agent import ...| Task3["Task-3.ipynb"]
    File -->|python budget_agent.py| Standalone["Standalone run<br/>(__main__ block)"]
```

## Summary

| Step | Agent variable | Capability added | Key objects |
|------|----------------|------------------|-------------|
| 1.0 | `agent` | Basic model + guardrail | `BedrockModel`, `create_guardrail` |
| 1.1 | `budget_agent_sys` | System prompt (persona) | `BUDGET_SYSTEM_PROMPT` |
| 1.2 | `budget_agent_manager` | Conversation memory | `SummarizingConversationManager` |
| 1.3 | *(same agent)* | Streaming responses | `stream_async` |
| 1.4 | `budget_agent` | Financial tools | `calculate_budget`, `create_financial_chart`, `calculator` |
| 1.5 | `report_agent` → `budget_agent.py` | Structured output | `FinancialReport`, `BudgetCategory` |

The defining pattern is **incremental enhancement of one agent**: each task keeps the
same Bedrock model and adds a single new capability, culminating in a reusable,
tool-equipped, structured-output budget agent that becomes the foundation for the
multi-agent system in Task 2.
