# Task 2: Multi-Agent Workflows with Strands

This document explains how the agents built in `Task-2.ipynb` relate to each other,
using Mermaid diagrams.

Task 2 moves from a single agent to a **hierarchical multi-agent system** built with
the Strands ["Agents as Tools"](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/multi-agent/agents-as-tools/)
pattern: specialized agents are wrapped as `@tool` functions and handed to a top-level
**Orchestrator Agent** that routes requests and synthesizes answers.

## 1. High-Level Architecture

The Orchestrator sits on top and delegates to two specialist agents. Each specialist
owns its own domain tools. The user only ever talks to the Orchestrator.

```mermaid
graph TD
    User([User])

    subgraph Orchestration Layer
        Orchestrator["Orchestrator Agent<br/>(Nova Pro + Guardrail)<br/>SummarizingConversationManager"]
    end

    subgraph Specialist Agents
        Budget["Budget Agent<br/>(from Task 1 / budget_agent.py)"]
        FinAnalysis["Financial Analysis Agent<br/>(financial_analysis_agent.py)"]
    end

    User -->|query| Orchestrator
    Orchestrator -->|budget_agent_tool| Budget
    Orchestrator -->|financial_analysis_agent_tool| FinAnalysis
    Budget -.->|FinancialReport| Orchestrator
    FinAnalysis -.->|analysis text| Orchestrator
    Orchestrator -.->|synthesized answer| User
```

## 2. "Agents as Tools" Wrapping

The key mechanism: each specialist agent is wrapped inside an `@tool`-decorated
function. From the Orchestrator's perspective these are just tools it can call — but
each tool call actually invokes a full downstream agent.

```mermaid
graph LR
    subgraph Orchestrator Agent
        Tool1["@tool budget_agent_tool(query)"]
        Tool2["@tool financial_analysis_agent_tool(query)"]
    end

    Tool1 -->|structured_output FinancialReport| BudgetAgent["budget_agent"]
    Tool2 -->|agent call| FinAgent["financial_analysis_agent"]

    BudgetAgent --> B_out["Returns FinancialReport<br/>(Pydantic model)"]
    FinAgent --> F_out["Returns str<br/>(analysis text)"]
```

Note the difference in return contracts:

- `budget_agent_tool` calls `budget_agent.structured_output(...)` and returns a typed
  `FinancialReport` Pydantic object (income, budget categories, recommendations,
  health score). On error it returns a default `FinancialReport`.
- `financial_analysis_agent_tool` calls the agent directly and returns a plain string.
  On error it returns an error string.

## 3. Tools Owned by Each Agent

Each specialist agent carries its own set of domain tools. The Orchestrator does not
call these leaf tools directly — it calls the specialist, which then uses its tools.

```mermaid
graph TD
    Orchestrator["Orchestrator Agent"]

    Orchestrator --> BudgetAgent["Budget Agent"]
    Orchestrator --> FinAgent["Financial Analysis Agent"]

    BudgetAgent --> BT1["calculate_budget<br/>(50/30/20 breakdown)"]
    BudgetAgent --> BT2["create_financial_chart<br/>(pie chart)"]
    BudgetAgent --> BT3["calculator<br/>(strands_tools)"]

    FinAgent --> FT1["get_stock_analysis<br/>(yfinance)"]
    FinAgent --> FT2["create_diversified_portfolio<br/>(risk-based allocation)"]
    FinAgent --> FT3["compare_stock_performance<br/>(multi-stock comparison)"]
```

## 4. Request Routing Flow

How a single user query flows through the system. The Orchestrator decides which
specialist(s) to consult, may call one or both, then combines the results.

```mermaid
flowchart TD
    Start([User query]) --> Guard{Guardrail check}
    Guard -->|blocked| Blocked[Return safe/redacted response]
    Guard -->|allowed| Route{Which domain?}

    Route -->|budgeting / spending / savings| CallBudget[Call budget_agent_tool]
    Route -->|stocks / investing / portfolio| CallFin[Call financial_analysis_agent_tool]
    Route -->|both needed| CallBoth[Call both agents]

    CallBudget --> Synth[Synthesize response]
    CallFin --> Synth
    CallBoth --> Synth

    Synth --> Answer([Coherent answer + next steps])
```

Example of a combined query from the notebook:
*"Compare Tesla and Apple stocks, and tell me if I can afford to invest $2000 with my
$4000 monthly income."* — this triggers **both** the Financial Analysis Agent (stock
comparison) and the Budget Agent (affordability against income), with the Orchestrator
merging the two into one answer.

## 5. Module / Import Relationships

The notebook builds each agent and exports it with `%%writefile` so later stages can
import it. This is how the pieces connect across files.

```mermaid
graph TD
    BudgetPy["budget_agent.py<br/>(exported in Task 1)"]
    FinPy["financial_analysis_agent.py<br/>(exported in Task 2.2)"]
    MainPy["main.py<br/>(exported in Task 2.4)"]
    Utils["utils<br/>(create_guardrail, get_guardrail_id)"]

    BudgetPy -->|from budget_agent import<br/>FinancialReport, budget_agent| MainPy
    FinPy -->|from financial_analysis_agent import<br/>financial_analysis_agent| MainPy
    Utils -->|get_guardrail_id| MainPy

    MainPy --> OrchestratorRuntime["Orchestrator Agent<br/>(runtime system)"]
```

- `budget_agent.py` (produced at the end of Task 1) exports `budget_agent` and the
  `FinancialReport` schema.
- `financial_analysis_agent.py` (produced in Task 2.2) exports
  `financial_analysis_agent`.
- `main.py` (produced in Task 2.4) imports both, wraps each as a tool, attaches a
  Bedrock guardrail via `get_guardrail_id()`, and assembles the Orchestrator.

## Summary

| Layer | Component | Role | Returns |
|-------|-----------|------|---------|
| Top | Orchestrator Agent | Routes queries, coordinates specialists, synthesizes answers, applies guardrail + conversation memory | Combined natural-language answer |
| Specialist | Budget Agent | Budgeting, spending analysis, savings | `FinancialReport` (structured) |
| Specialist | Financial Analysis Agent | Stock analysis, portfolios, comparisons | Analysis text |
| Leaf tools | `calculate_budget`, `create_financial_chart`, `calculator`, `get_stock_analysis`, `create_diversified_portfolio`, `compare_stock_performance` | Domain-specific computation | Strings / charts |

The defining pattern is **hierarchy through tool-wrapping**: specialist agents are
exposed to the Orchestrator as callable tools, letting a single coordinating agent
compose multiple domains of expertise into one coherent response.
