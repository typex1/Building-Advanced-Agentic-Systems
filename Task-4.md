# Task 4: Run Agents with Amazon Bedrock AgentCore Harness

This document explains how the pieces in `Task-4.ipynb` relate to each other, using
Mermaid diagrams — the same companion-doc style as `Task-3.md`.

Task 4 does **not** add new agents or new agent code. Where Task 3 deployed a
multi-agent system to **AgentCore Runtime** by writing an orchestration loop in
`main.py`, packaging it into a container, and hosting it, Task 4 reaches the same
outcome — a running, invokable agent — from the opposite direction:

- **No loop code** — the reasoning → tool-call → result → response loop is managed by
  AgentCore (powered by Strands Agents).
- **No container** — nothing is built or pushed to ECR.
- **Configuration, not deployment** — the agent is *declared* (model, system prompt,
  limits) and changed with a single API field, not a code change and redeploy.

So the "relationships" here are two layers: the **control-plane resource lifecycle**
(create → wait → delete) and the **data-plane invocation** (streamed events). They live
on two different boto3 clients.

## 1. Harness vs. Runtime (the Task 3 contrast)

Both run on the same underlying AgentCore Runtime infrastructure, but divide the work
differently. Task 3 is the right column; Task 4 is the left.

```mermaid
graph TD
    subgraph Harness["AgentCore Harness (Task 4)"]
        H1["You provide:<br/>configuration only"]
        H2["Agent loop:<br/>managed (Strands)"]
        H3["Change model / tool:<br/>config field, no redeploy"]
    end

    subgraph Runtime["AgentCore Runtime (Task 3)"]
        R1["You provide:<br/>agent code + ARM64 container"]
        R2["Agent loop:<br/>you write it (main.py)"]
        R3["Change model / tool:<br/>code change + redeploy"]
    end

    Pick{"Need to own the loop,<br/>a specific framework, or<br/>non-loop patterns?"}
    Pick -->|no| Harness
    Pick -->|yes| Runtime
```

Rule of thumb: **start with Harness; drop down to Runtime only when configuration is
not enough.**

## 2. The Two Clients (Control Plane vs. Data Plane)

Every operation in the notebook goes through one of two boto3 clients. Keeping them
straight is the key to reading the code.

```mermaid
graph TD
    subgraph ControlPlane["bedrock-agentcore-control (manage the resource)"]
        Create["create_harness"]
        Get["get_harness (poll status)"]
        Delete["delete_harness"]
    end

    subgraph DataPlane["bedrock-agentcore (talk to the agent)"]
        Invoke["invoke_harness (streamed)"]
    end

    IAM["IAM execution role<br/>(assumed by<br/>bedrock-agentcore.amazonaws.com)"]
    IAM -->|executionRoleArn| Create
    Create --> Get
    Get -->|status READY| Invoke
    Invoke -. same resource .-> Delete
```

Note the response shape: `create_harness` and `get_harness` both nest everything under
a top-level `harness` key — the notebook reads `response["harness"]["harnessId"]`,
`["arn"]`, and `["status"]`, **not** top-level fields.

## 3. The Three-Stage Lifecycle (Create → Wait → Invoke)

This is the defining pattern of the task, and the most common place to go wrong:
**creation alone does not yield a callable agent.** You must wait for `READY` before
invoking.

```mermaid
flowchart TD
    Role["Task 4.1: execution role<br/>(create, or reuse EXISTING_ROLE_ARN)"]
    Role --> Create["Task 4.2: create_harness<br/>(name + executionRoleArn required;<br/>maxIterations / maxTokens / timeoutSeconds)"]

    Create --> Poll{"Task 4.3: get_harness<br/>status?"}
    Poll -->|CREATING| Wait["sleep, poll again"]
    Wait --> Poll
    Poll -->|CREATE_FAILED| Fail["raise RuntimeError<br/>(read failureReason)"]
    Poll -->|READY| Invoke["Task 4.4: invoke_harness<br/>(runtimeSessionId >= 33 chars,<br/>messages=[...])"]

    Invoke --> Override["Task 4.5: invoke_harness<br/>with per-call systemPrompt override"]
    Override --> Cleanup["Task 4.6: delete_harness<br/>(+ delete role only if we created it)"]
```

## 4. The Execution Role and Its Fallback (Task 4.1)

A harness assumes an IAM role. The trust policy is scoped with confused-deputy
conditions so only *this account's* harnesses can assume it. Because restricted lab
accounts often cannot create roles, the cell has a three-way branch.

```mermaid
flowchart TD
    Start([Task 4.1 runs]) --> Check{EXISTING_ROLE_ARN set?}
    Check -->|yes| Reuse["use that ARN<br/>created_role = False"]
    Check -->|no| Try["iam.create_role(...)"]
    Try -->|success| Made["created_role = True<br/>attach bedrock:InvokeModel policy"]
    Try -->|AccessDenied| Err["raise PermissionError:<br/>'set EXISTING_ROLE_ARN and re-run'"]

    Reuse --> Done([execution_role_arn ready])
    Made --> Done
```

`created_role` is the important flag: the cleanup cell (Task 4.6) deletes the role
**only when the notebook created it**, so a user-supplied `EXISTING_ROLE_ARN` is never
removed.

## 5. Invocation and the Streaming Event Format (Task 4.4)

`invoke_harness` returns an **event stream**, not a single string. The notebook's
`invoke_and_stream` helper iterates `response["stream"]` and prints text deltas as they
arrive — the same incremental experience as the console playground.

```mermaid
sequenceDiagram
    participant NB as Notebook (client)
    participant DP as bedrock-agentcore<br/>(data plane)
    participant HM as Harness (managed loop)
    participant BR as Bedrock model

    NB->>DP: invoke_harness(harnessArn,<br/>runtimeSessionId, messages)
    DP->>HM: start session (isolated microVM)
    HM->>BR: run agent loop
    BR-->>HM: tokens
    HM-->>DP: event stream
    DP-->>NB: messageStart
    DP-->>NB: contentBlockDelta (text) ...
    DP-->>NB: messageStop (stopReason: end_turn)
    DP-->>NB: metadata (token usage, latency)
```

Key points the helper relies on:

- **`runtimeSessionId` ≥ 33 chars** — a `uuid4()` (36 chars with hyphens) satisfies it.
  Reusing the same id continues the conversation in the same environment.
- **`contentBlockDelta`** carries incremental `text` (also tool-use / tool-result /
  reasoning in richer configs).
- **`messageStop.stopReason`** reports *why* a message ended: `end_turn`, `tool_use`,
  `max_iterations_exceeded`, `timeout_exceeded`, `max_tokens`, etc.

## 6. Per-Invocation Overrides (Task 4.5)

The feature that makes a harness convenient for experimentation: a single call can
override the stored configuration **without redeploying**. The notebook overrides
`systemPrompt` for one call; the harness resource is unchanged.

```mermaid
graph TD
    Stored["Stored harness config<br/>systemPrompt = 'careful finance assistant'"]

    Stored --> Call1["invoke_harness(prompt)<br/>→ uses stored prompt"]
    Stored --> Call2["invoke_harness(prompt,<br/>systemPrompt=[terse bullets])<br/>→ override for THIS call only"]
    Stored --> Call3["invoke_harness(prompt)<br/>→ stored prompt again"]

    Note["Overridable per call: model, systemPrompt,<br/>tools, allowedTools, skills, limits, actorId"]
    Note -.-> Call2
```

> **Trust boundary:** overrides are caller-supplied. If a harness is exposed to
> untrusted users, strip or allowlist override fields (especially `model` and `skills`)
> in the application layer before calling `invoke_harness`.

## Summary

| Layer | Component | Role |
|-------|-----------|------|
| Auth / IAM | Execution role (`AgentCoreHarnessLabRole` or `EXISTING_ROLE_ARN`) | Role the harness assumes; confused-deputy-scoped trust |
| Control plane | `bedrock-agentcore-control`: `create_harness` / `get_harness` / `delete_harness` | Manage the harness resource and poll for `READY` |
| Agent loop | Managed Strands loop (no `main.py`, no container) | Reasoning → tool-call → result → response, run by AgentCore |
| Data plane | `bedrock-agentcore`: `invoke_harness` | Invoke the agent; returns a stream of typed events |
| Config | `systemPrompt`, `maxIterations`, `maxTokens`, `timeoutSeconds` | Declared behavior and execution-limit guardrails |
| Flexibility | Per-invocation overrides | Change model/prompt/limits for one call, no redeploy |

The defining pattern of Task 4 is **config-over-code**: where Task 3 productionized an
agent graph you wrote, Task 4 produces a running agent from declaration alone. The three
stages — **create → wait for `READY` → invoke (streamed)** — and the clean split between
the control-plane and data-plane clients are the whole mental model.

> **Note:** AgentCore Harness became generally available in June 2026, after the earlier
> tasks in this lab were authored, so Task 4 is a standalone addition rather than a
> continuation of the Task 1–3 finance-agent build.
