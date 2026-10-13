# Topics covered in the lab notebooks

Generated 2026-10-09 from the `Task-1.ipynb` … `Task-4.ipynb` notebooks in `Building-Advanced-Agentic-Systems/` (lab: Building and Deploying Intelligent Financial Agents with Strands and AgentCore). Only features actually used in code cells are listed; items the notebook describes but does not run are marked *mentioned only*.

## Task-1: Build a Personal Budget Assistant with Strands Agents

Scenario: a single budget-assistant agent (50/30/20 budgeting, spending analysis, no investment advice) built up step by step and exported to `budget_agent.py`.

### Strands Agents
- Agent construction: `Agent(model=..., system_prompt=..., tools=..., conversation_manager=..., callback_handler=None)`
- Model config: `BedrockModel(model_id="us.amazon.nova-pro-v1:0", region_name, temperature=0.0)`
- Bedrock Guardrails on the model: `guardrail_id`, `guardrail_version="DRAFT"`, `guardrail_trace="enabled"` (guardrail blocks Bitcoin advice); `guardrail_redact_input/_output` *mentioned only*
- System prompt design (Task 1.1): `BUDGET_SYSTEM_PROMPT`
- Conversation managers (Task 1.2): `SummarizingConversationManager(summary_ratio, preserve_recent_messages)`; `SlidingWindowConversationManager` and `NullConversationManager` *mentioned only*
- Streaming / async (Task 1.3): `agent.stream_async()` with `async for event`, `event["data"]`; blocking vs streaming comparison
- Custom tools (Task 1.4): `@tool` decorator on `calculate_budget`, `create_financial_chart`
- strands_tools built-in: `calculator`
- Structured output (Task 1.5): Pydantic `BaseModel`/`Field` models `BudgetCategory`, `FinancialReport`; `agent.structured_output(output_model=..., prompt=...)` with retry loop; note that guardrails break `structured_output` (`stop_reason="guardrail_intervened"`)
- Message history inspection: `agent.messages` via `pretty_print_messages`
- Exporting the agent as a reusable module (`%%writefile budget_agent.py`)

### AgentCore
- none

### Other AWS services / libraries
- Amazon Bedrock Guardrails created via `boto3` `bedrock.create_guardrail` (`utils/guardrail.py`: content filters, word policy, profanity list)
- Amazon Nova Pro model (Bedrock model access / Marketplace permissions)
- `matplotlib` (pie charts), `pydantic`, Python `logging`

## Task-2: Build multi-agent workflows with Strands

Scenario: an orchestrator agent routes between the Task-1 budget agent and a new investment-analysis agent (stock data, portfolios), exported to `main.py`.

### Strands Agents
- Multi-agent pattern "Agents as Tools": specialist agents wrapped with `@tool` (`budget_agent_tool`, `financial_analysis_agent_tool`) and given to an orchestrator `Agent`
- Hierarchical orchestration / request routing via `ORCHESTRATOR_PROMPT`
- Structured output as a tool return type: `budget_agent_tool(query) -> FinancialReport` calling `budget_agent.structured_output(...)`
- Custom tools with external data: `@tool` `get_stock_analysis`, `create_diversified_portfolio`, `compare_stock_performance` (input validation and error handling patterns in the exported file)
- `BedrockModel`: plain for the financial analysis agent, with guardrails (`guardrail_id`, `guardrail_version`, `guardrail_trace`) for the orchestrator
- `SummarizingConversationManager(summary_ratio=0.3, preserve_recent_messages=5)` on the orchestrator
- `callback_handler=None` to silence specialist agent output
- Importing agents from modules (`from budget_agent import ...`, `from financial_analysis_agent import ...`)
- Graph, Swarm, Workflow, A2A: not used

### AgentCore
- none

### Other AWS services / libraries
- Amazon Bedrock Guardrails (`create_guardrail`, `get_guardrail_id`)
- `yfinance` for market data; `boto3`; `logging`

## Task-3: Deploy agents on Amazon Bedrock AgentCore

Scenario: deploy the Task-2 orchestrator to AgentCore Runtime behind Cognito JWT auth, with AgentCore Memory for cross-session personalization.

### Strands Agents
- Same orchestrator stack reused: `Agent`, `BedrockModel` + guardrails, `SummarizingConversationManager`, agents-as-tools
- Streaming inside the entrypoint: `async for event in orchestrator_agent.stream_async(...)` yielding chunks
- Dynamic system prompt: `orchestrator_agent.system_prompt = ...` enriched with retrieved memories

### AgentCore
- Runtime (Task 3.1): `from bedrock_agentcore import BedrockAgentCoreApp`, `app = BedrockAgentCoreApp()`, `@app.entrypoint` async generator `invoke(payload)`, `app.run()`; `/invocations` and `/ping` on port 8080 explained
- Starter toolkit (Task 3.3): `from bedrock_agentcore_starter_toolkit import Runtime`; `Runtime().configure(entrypoint, execution_role, auto_create_ecr=True, requirements_file, agent_name, authorizer_configuration)` and `.launch(env_vars=...)`; Dockerfile generation, ARM64 note
- Inbound auth (Task 3.2): `customJWTAuthorizer` with `allowedClients` and `discoveryUrl` from Cognito
- Memory (Task 3.2.1): `MemoryClient`, `StrategyType.SEMANTIC`, `StrategyType.USER_PREFERENCE`, `StrategyType.SUMMARY`, namespaces `finance/user/{actorId}/...`, `create_memory_and_wait(..., event_expiry_days=90)`, `list_memories`
- Memory at runtime: `memory_client.query_records(memory_id, query, namespace, max_results)` before the call, `memory_client.create_event(memory_id, actor_id, session_id, content)` after; `AGENTCORE_MEMORY_ID` env var
- Invocation (Task 3.4): HTTPS POST to `bedrock-agentcore.<region>.amazonaws.com/runtimes/<arn>/invocations` with `Authorization: Bearer`, `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id`, `qualifier=DEFAULT`, SSE stream parsing
- Resource verification: `bedrock-agentcore-control` `list_agent_runtimes`
- Observability: env var `OTEL_PYTHON_EXCLUDED_URLS`; AgentCore Observability *mentioned only* in intro (no dashboard/ADOT setup)

### Other AWS services / libraries
- Amazon Cognito user pool, app client, `initiate_auth` for JWT bearer tokens (`utils/agentcore_utils.py`)
- AWS Secrets Manager (stores Cognito test-user credentials)
- Amazon ECR (`describe_repositories`, `describe_images`), IAM execution role `AmazonBedrockAgentCoreSDKRuntime-<region>`, STS `get_caller_identity`
- Amazon Bedrock Guardrails (reused), `requests`, `urllib.parse`

## Task-4: Run agents with Amazon Bedrock AgentCore Harness

Scenario: the same finance assistant declared as configuration on AgentCore Harness (managed Strands loop), invoked with streaming and per-call overrides, then deleted.

### Strands Agents
- No Strands code written; Harness runs a managed Strands-powered agent loop (*mentioned only*)

### AgentCore
- Harness vs Runtime comparison (config-only vs own code/container)
- Control plane `boto3.client("bedrock-agentcore-control")`: `create_harness(harnessName, executionRoleArn, systemPrompt, maxIterations, maxTokens, timeoutSeconds)`, `get_harness`, `delete_harness`
- Lifecycle: poll `get_harness` until `status == "READY"`; terminal states `CREATE_FAILED` / `UPDATE_FAILED` / `DELETE_FAILED`
- Data plane `boto3.client("bedrock-agentcore")`: `invoke_harness(harnessArn, runtimeSessionId, messages)`; `runtimeSessionId` ≥ 33 chars; session reuse
- Streaming event format: `messageStart`, `contentBlockDelta`, `contentBlockStop`, `messageStop` (`stopReason`), `metadata` (usage)
- Per-invocation overrides: `systemPrompt` override demonstrated; `model`, `tools`, `allowedTools`, `skills`, limits *mentioned only*; override trust-boundary security note
- Execution limits as cost/abuse guardrails: `maxIterations`, `maxTokens`, `timeoutSeconds`
- Next steps *mentioned only*: built-in `shell`/`file_operations` tools, Browser, Code Interpreter, Gateway, remote MCP, Memory (`actorId`), Agent Skills, versions and endpoints

### Other AWS services / libraries
- IAM: `create_role` with confused-deputy trust policy (`aws:SourceAccount`, `aws:SourceArn`, principal `bedrock-agentcore.amazonaws.com`), `put_role_policy` least-privilege `bedrock:InvokeModel*`, cleanup `delete_role_policy` / `delete_role`
- STS `get_caller_identity`; `botocore.exceptions.ClientError` handling

## Cross-notebook summary

| Topic | Task-1 | Task-2 | Task-3 | Task-4 |
|---|---|---|---|---|
| Strands `Agent` + `system_prompt` | ✓ | ✓ | ✓ | |
| `BedrockModel` config (Nova Pro, temperature) | ✓ | ✓ | ✓ | |
| Bedrock Guardrails on `BedrockModel` | ✓ | ✓ | ✓ | |
| `@tool` custom tools | ✓ | ✓ | ✓ | |
| `strands_tools` built-ins (`calculator`) | ✓ | | | |
| `SummarizingConversationManager` | ✓ | ✓ | ✓ | |
| Streaming (`stream_async`) | ✓ | | ✓ | |
| Structured output (Pydantic, `structured_output`) | ✓ | ✓ | ✓ | |
| Multi-agent: agents-as-tools / orchestrator | | ✓ | ✓ | |
| `callback_handler=None` | ✓ | ✓ | | |
| AgentCore Runtime (`BedrockAgentCoreApp`, `@app.entrypoint`) | | | ✓ | |
| AgentCore starter toolkit (`Runtime.configure/launch`) | | | ✓ | |
| AgentCore Memory (`MemoryClient`, strategies, `create_event`, `query_records`) | | | ✓ | |
| Inbound auth: Cognito JWT (`customJWTAuthorizer`) | | | ✓ | |
| Invoke Runtime over HTTPS with bearer token | | | ✓ | |
| AgentCore Observability (env var only / mentioned) | | | ✓ | |
| AgentCore Harness (`create_harness`, `invoke_harness`, overrides) | | | | ✓ |
| IAM execution role / confused-deputy trust policy | | | | ✓ |
| Amazon ECR | | | ✓ | |
| Amazon Cognito / Secrets Manager | | | ✓ | |
| `yfinance` market data | | ✓ | | |
| Not used in any notebook: hooks, session managers, MCP tools, Graph/Swarm/Workflow, A2A, Knowledge Bases, Gateway, Identity, Code Interpreter, Browser, Policy, Evaluations | | | | |
