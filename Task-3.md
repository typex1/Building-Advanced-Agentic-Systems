# Task 3: Deploy Agents on Amazon Bedrock AgentCore

This document explains how the agents and infrastructure in `Task-3.ipynb` relate to
each other, using Mermaid diagrams.

Task 3 does **not** add new agents. It takes the exact multi-agent system from Task 2
(Orchestrator + Budget Agent + Financial Analysis Agent) and wraps it in a production
**AgentCore Runtime** deployment, adding four infrastructure concerns around it:

- **Runtime hosting** — the agents run as a serverless container, not in a notebook.
- **Authentication** — Amazon Cognito issues JWT tokens for calling the endpoint.
- **Memory** — AgentCore Memory persists facts, preferences, and summaries across sessions.
- **Guardrail + Observability** — safety and tracing wired into the deployed agent.

So the "relationships" here are two layers: the **agent graph** (unchanged from Task 2)
and the **deployment stack** that hosts and secures it.

## 1. Deployment Architecture (Build → Deploy → Invoke)

The notebook packages `main.py` into a container, pushes it to ECR, creates the
AgentCore Runtime, and then invokes it over HTTPS with a Cognito token.

```mermaid
graph TD
    subgraph "Local / Notebook"
        MainPy["main.py<br/>(@app.entrypoint)"]
        ReqTxt["requirements.txt"]
    end

    subgraph "Deployment (agentcore_runtime)"
        Configure["configure()<br/>generates Dockerfile"]
        Launch["launch()"]
    end

    subgraph "AWS"
        ECR["Amazon ECR<br/>(container image)"]
        Runtime["AgentCore Runtime<br/>(serverless, ARM64)"]
        Cognito["Amazon Cognito<br/>(JWT authorizer)"]
        Memory["AgentCore Memory<br/>(FinancialAdvisorMemory)"]
    end

    MainPy --> Configure
    ReqTxt --> Configure
    Configure --> Launch
    Launch -->|build + push| ECR
    Launch -->|create runtime| Runtime
    ECR --> Runtime
    Cognito -->|auth_config| Runtime
    Memory -->|AGENTCORE_MEMORY_ID env var| Runtime
```

## 2. The Hosted Agent Graph (unchanged from Task 2)

Inside the running container, the agent structure is identical to Task 2: the
Orchestrator wraps both specialists as tools. AgentCore wraps the Orchestrator in an
HTTP app.

```mermaid
graph TD
    App["BedrockAgentCoreApp<br/>(HTTP server :8080)"]
    Entry["@app.entrypoint<br/>async def invoke(payload)"]

    App --> Entry
    Entry --> Orchestrator["orchestrator_agent<br/>(Nova Pro + guardrail +<br/>SummarizingConversationManager)"]

    Orchestrator -->|budget_agent_tool| Budget["budget_agent<br/>(budget_agent.py)"]
    Orchestrator -->|financial_analysis_agent_tool| FinAnalysis["financial_analysis_agent<br/>(financial_analysis_agent.py)"]

    Budget -.->|FinancialReport| Orchestrator
    FinAnalysis -.->|analysis text| Orchestrator
```

The `@app.entrypoint` function `invoke` is the bridge: AgentCore delivers each request
to it, and it streams the Orchestrator's response back chunk by chunk.

## 3. Request Lifecycle with Memory

Each invocation retrieves relevant memories *before* running the Orchestrator and
stores the conversation *after*. This is what gives the deployed agent cross-session
personalization that the local Task 2 version lacked.

```mermaid
flowchart TD
    Client([Client request<br/>prompt + actor_id + session_id]) --> Auth{JWT valid?<br/>Cognito}
    Auth -->|no| Reject[401 / rejected]
    Auth -->|yes| Entry["invoke(payload)"]

    Entry --> MemCheck{AGENTCORE_MEMORY_ID set?}
    MemCheck -->|yes| Query["query_records<br/>namespace finance/user/{actor_id}/facts"]
    Query --> Enhance["Append memories to<br/>ORCHESTRATOR_PROMPT"]
    MemCheck -->|no| Run
    Enhance --> Run["orchestrator_agent.stream_async(prompt)"]

    Run --> Stream["yield chunks to client"]
    Stream --> Store{Memory enabled?}
    Store -->|yes| CreateEvent["create_event<br/>(User + Assistant turn)"]
    Store -->|no| Done([Response complete])
    CreateEvent --> Done
```

## 4. AgentCore Memory Strategies

The memory resource is configured with three strategies, each writing to its own
namespace. These feed the context-retrieval step in diagram 3.

```mermaid
graph TD
    Memory["FinancialAdvisorMemory<br/>(event_expiry_days=90)"]

    Memory --> Semantic["SEMANTIC<br/>'financial_facts'<br/>income, expenses, goals<br/>ns: finance/user/{actorId}/facts"]
    Memory --> Pref["USER_PREFERENCE<br/>'user_preferences'<br/>risk tolerance, priorities<br/>ns: finance/user/{actorId}/preferences"]
    Memory --> Summary["SUMMARY<br/>'conversation_summary'<br/>session summaries<br/>ns: finance/user/{actorId}/sessions/{sessionId}"]
```

## 5. Invocation Path (Client → Deployed Agent)

How the notebook actually calls the deployed agent in Task 3.4: authenticate with
Cognito, then POST a streaming request to the AgentCore invocations endpoint.

```mermaid
sequenceDiagram
    participant NB as Notebook (client)
    participant Cog as Cognito
    participant RT as AgentCore Runtime
    participant Orc as Orchestrator Agent
    participant Mem as AgentCore Memory

    NB->>Cog: reauthenticate_user()
    Cog-->>NB: bearer_token (JWT)
    NB->>RT: POST /runtimes/{arn}/invocations<br/>(Bearer token, prompt, session-id)
    RT->>Mem: query_records (facts)
    Mem-->>RT: relevant memories
    RT->>Orc: stream_async(prompt + memory context)
    Orc-->>RT: response chunks
    RT-->>NB: streamed data: chunks
    RT->>Mem: create_event (store turn)
```

## 6. Module / Import Relationships

`main.py` is rewritten in Task 3 (via `%%writefile`) to be AgentCore-compatible, but it
still imports the same specialist agents produced in Tasks 1 and 2.

```mermaid
graph TD
    BudgetPy["budget_agent.py<br/>(Task 1)"]
    FinPy["financial_analysis_agent.py<br/>(Task 2)"]
    Utils["utils<br/>(get_guardrail_id,<br/>setup_cognito_user_pool,<br/>reauthenticate_user)"]
    AgentCore["bedrock_agentcore<br/>(BedrockAgentCoreApp, MemoryClient)"]

    MainPy["main.py<br/>(Task 3 — AgentCore entrypoint)"]

    BudgetPy -->|budget_agent, FinancialReport| MainPy
    FinPy -->|financial_analysis_agent| MainPy
    Utils -->|get_guardrail_id| MainPy
    AgentCore -->|app, memory| MainPy

    MainPy --> Deployed["Deployed personal_finance_agent<br/>(ECR image → AgentCore Runtime)"]
```

## Summary

| Layer | Component | Role |
|-------|-----------|------|
| Entry | `BedrockAgentCoreApp` + `@app.entrypoint invoke` | HTTP server (:8080), `/invocations` + `/ping`, streams responses |
| Agents | Orchestrator → Budget Agent + Financial Analysis Agent | Same Task 2 "agents as tools" graph |
| Memory | AgentCore Memory (semantic / preference / summary) | Cross-session personalization via query + store |
| Auth | Amazon Cognito (JWT) | Authorizes callers to the runtime endpoint |
| Infra | ECR + AgentCore Runtime | Container registry + serverless hosting (ARM64/Graviton) |
| Safety | Bedrock guardrail + Observability | Content filtering and execution tracing |

The defining pattern of Task 3 is **productionizing an existing agent graph**: the
agents themselves are unchanged from Task 2, but they are now hosted in a managed
runtime, authenticated via Cognito, made stateful with AgentCore Memory, and exposed
through a streaming HTTP endpoint.
