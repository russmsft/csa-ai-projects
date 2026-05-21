# 06 — Multi-Agent Incident Responder: Fan Out Diagnostics, Research, and Communication

When an incident hits, you need three things fast: what broke, how to fix it, and what to tell stakeholders. This guide wires up three specialized agents that run concurrently and hand off to each other.

---

## What You're Building

Three Foundry prompt agents working in concert: a **Diagnostician** that queries Application Insights via function calling, a **Researcher** that searches a runbook index in Azure AI Search, and a **Communicator** that drafts a stakeholder update. A Python orchestrator runs the Diagnostician and Researcher in parallel (fan-out), then feeds their outputs to the Communicator (fan-in). Total time to first stakeholder update: under 60 seconds.

---

## Prerequisites

- Microsoft Foundry project with GPT-4.1 deployed
- Azure Application Insights resource with query access
- Azure AI Search index containing runbook documents
- Python 3.11+
- `azure-ai-projects`, `azure-identity`, `azure-monitor-query`, `azure-search-documents`

```bash
pip install azure-ai-projects azure-identity azure-monitor-query \
  azure-search-documents python-dotenv asyncio
```

---

## Architecture

```
Incident trigger (alert / manual)
        │
        ▼
Python Orchestrator
  ├──── [concurrent] ────┐
  │                       │
  ▼                       ▼
Agent 1: Diagnostician   Agent 2: Researcher
  └── FunctionTool         └── FunctionTool
      (App Insights KQL)       (AI Search runbook query)
  │                       │
  └──── [fan-in] ─────────┘
                │
                ▼
        Agent 3: Communicator
          └── Draft stakeholder update
                │
                ▼
        Slack/Teams/Email output
```

---

## Step-by-Step Build

### Step 1 — Set up the Application Insights function

```python
import os
import json
import asyncio
from dotenv import load_dotenv
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.monitor.query import LogsQueryClient, LogsQueryStatus
from azure.search.documents import SearchClient
from azure.ai.projects.models import FunctionTool, PromptAgentDefinition, MessageRole

load_dotenv()

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
APP_INSIGHTS_WORKSPACE_ID = os.environ["APP_INSIGHTS_WORKSPACE_ID"]
SEARCH_ENDPOINT = os.environ["SEARCH_ENDPOINT"]
SEARCH_INDEX = os.environ.get("SEARCH_INDEX", "runbooks")
SEARCH_KEY = os.environ["SEARCH_KEY"]

credential = DefaultAzureCredential()
ai_client = AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=credential)
logs_client = LogsQueryClient(credential)
search_client = SearchClient(
    endpoint=SEARCH_ENDPOINT,
    index_name=SEARCH_INDEX,
    credential=credential
)
```

### Step 2 — Define the tool functions

```python
# --- Tool: query Application Insights ---

def query_app_insights(kql_query: str, time_range_hours: int = 1) -> str:
    """Execute a KQL query against Application Insights."""
    from datetime import timedelta
    try:
        result = logs_client.query_workspace(
            workspace_id=APP_INSIGHTS_WORKSPACE_ID,
            query=kql_query,
            timespan=timedelta(hours=time_range_hours)
        )
        if result.status == LogsQueryStatus.SUCCESS:
            rows = []
            for table in result.tables:
                for row in table.rows:
                    rows.append(dict(zip(table.columns, row)))
            return json.dumps(rows[:50])  # cap at 50 rows
        else:
            return json.dumps({"error": "Query failed", "details": str(result.partial_error)})
    except Exception as e:
        return json.dumps({"error": str(e)})


APP_INSIGHTS_TOOL_DEF = {
    "name": "query_app_insights",
    "description": (
        "Query Application Insights using KQL to investigate errors, performance, "
        "and availability. Returns up to 50 rows."
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "kql_query": {
                "type": "string",
                "description": "Valid KQL query. Use requests, exceptions, traces, dependencies tables."
            },
            "time_range_hours": {
                "type": "integer",
                "description": "Time range in hours to query. Default 1.",
                "default": 1
            }
        },
        "required": ["kql_query"]
    }
}


# --- Tool: search runbooks ---

def search_runbooks(query: str, top: int = 5) -> str:
    """Search the runbook index in Azure AI Search."""
    results = search_client.search(
        search_text=query,
        top=top,
        include_total_count=True
    )
    docs = []
    for r in results:
        docs.append({
            "title": r.get("title", "Unknown"),
            "content": r.get("content", "")[:1000],  # first 1000 chars
            "score": r["@search.score"]
        })
    return json.dumps(docs)


SEARCH_RUNBOOK_TOOL_DEF = {
    "name": "search_runbooks",
    "description": "Search the runbook knowledge base for troubleshooting procedures.",
    "parameters": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Natural language query describing the issue."
            },
            "top": {
                "type": "integer",
                "description": "Number of results to return (1-10). Default 5.",
                "default": 5
            }
        },
        "required": ["query"]
    }
}
```

### Step 3 — Create the three agents

```python
def create_diagnostician_agent() -> str:
    tool = FunctionTool(functions=[APP_INSIGHTS_TOOL_DEF])
    agent_def = PromptAgentDefinition(
        model="gpt-4.1",
        name="incident-diagnostician",
        instructions=(
            "You are an SRE incident diagnostician. When given an incident description:\n"
            "1. Query Application Insights to understand the scope and impact\n"
            "2. Identify the error type, affected services, and blast radius\n"
            "3. Determine the timeline (when did it start?)\n"
            "4. Estimate affected user count if possible\n\n"
            "Useful KQL queries:\n"
            "- Error rate: requests | where success == false | summarize count() by bin(timestamp, 5m)\n"
            "- Exceptions: exceptions | order by timestamp desc | take 20\n"
            "- Affected dependencies: dependencies | where success == false | summarize count() by target\n\n"
            "Return a structured diagnostic report."
        ),
        tools=[tool.definitions],
    )
    agent = ai_client.agents.create_version(definition=agent_def)
    return agent.id


def create_researcher_agent() -> str:
    tool = FunctionTool(functions=[SEARCH_RUNBOOK_TOOL_DEF])
    agent_def = PromptAgentDefinition(
        model="gpt-4.1",
        name="incident-researcher",
        instructions=(
            "You are an incident response researcher. Given an incident description:\n"
            "1. Search the runbook index for relevant procedures\n"
            "2. Extract the specific remediation steps\n"
            "3. Identify any prerequisites or dependencies for the fix\n"
            "4. Note any known past incidents of the same type\n\n"
            "Return: recommended runbook title, step-by-step remediation, "
            "estimated resolution time, and escalation path."
        ),
        tools=[tool.definitions],
    )
    agent = ai_client.agents.create_version(definition=agent_def)
    return agent.id


def create_communicator_agent() -> str:
    agent_def = PromptAgentDefinition(
        model="gpt-4.1",
        name="incident-communicator",
        instructions=(
            "You are an incident communications specialist. Given diagnostic findings "
            "and runbook recommendations, draft a clear stakeholder update.\n\n"
            "Format:\n"
            "## Incident Update — [Severity] — [Short title]\n"
            "**Status:** [Investigating / Identified / Mitigating / Resolved]\n"
            "**Impact:** [What is affected and how many users]\n"
            "**What we know:** [2-3 sentences of root cause]\n"
            "**What we're doing:** [Numbered remediation steps in progress]\n"
            "**ETA:** [Estimated resolution time]\n"
            "**Next update:** [When to expect next communication]\n\n"
            "Write at a business level — no jargon, no stack traces, no blame."
        ),
    )
    agent = ai_client.agents.create_version(definition=agent_def)
    return agent.id
```

### Step 4 — Agent runner with tool call handling

```python
from azure.ai.projects.models import SubmitToolOutputsAction
import time

TOOL_HANDLERS = {
    "query_app_insights": query_app_insights,
    "search_runbooks": search_runbooks
}

def run_agent(agent_id: str, message: str) -> str:
    """Run an agent to completion, handling tool calls. Returns final text."""
    thread = ai_client.agents.create_thread()
    ai_client.agents.create_message(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=message
    )
    run = ai_client.agents.create_run(
        thread_id=thread.id,
        agent_id=agent_id
    )

    while run.status in ("queued", "in_progress", "requires_action"):
        time.sleep(1)
        run = ai_client.agents.get_run(thread_id=thread.id, run_id=run.id)

        if run.status == "requires_action":
            action = run.required_action
            if isinstance(action, SubmitToolOutputsAction):
                outputs = []
                for call in action.submit_tool_outputs.tool_calls:
                    fn_name = call.function.name
                    fn_args = json.loads(call.function.arguments)
                    handler = TOOL_HANDLERS.get(fn_name)
                    result = handler(**fn_args) if handler else json.dumps({"error": f"Unknown function: {fn_name}"})
                    outputs.append({"tool_call_id": call.id, "output": result})
                run = ai_client.agents.submit_tool_outputs_to_run(
                    thread_id=thread.id,
                    run_id=run.id,
                    tool_outputs=outputs
                )

    if run.status != "completed":
        raise RuntimeError(f"Agent {agent_id} failed: {run.status}")

    messages = ai_client.agents.list_messages(thread_id=thread.id)
    return messages.get_last_message_by_role(MessageRole.ASSISTANT).content[0].text.value
```

### Step 5 — Orchestrate with concurrent fan-out

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def respond_to_incident(
    incident_description: str,
    diagnostician_id: str,
    researcher_id: str,
    communicator_id: str
) -> dict:
    """Fan-out to diagnostician and researcher, fan-in to communicator."""

    print("Running diagnostician and researcher in parallel...")
    with ThreadPoolExecutor(max_workers=2) as executor:
        diag_future = executor.submit(
            run_agent, diagnostician_id, incident_description
        )
        research_future = executor.submit(
            run_agent, researcher_id, incident_description
        )

        diagnostic_report = diag_future.result()
        runbook_recommendation = research_future.result()

    print("Drafting stakeholder update...")
    communicator_input = (
        f"Incident description:\n{incident_description}\n\n"
        f"Diagnostic findings:\n{diagnostic_report}\n\n"
        f"Runbook recommendations:\n{runbook_recommendation}"
    )
    stakeholder_update = run_agent(communicator_id, communicator_input)

    return {
        "diagnostic_report": diagnostic_report,
        "runbook_recommendation": runbook_recommendation,
        "stakeholder_update": stakeholder_update
    }
```

### Step 6 — Main

```python
def main():
    print("Creating agents...")
    diag_id = create_diagnostician_agent()
    research_id = create_researcher_agent()
    comm_id = create_communicator_agent()

    # Sample incident
    incident = """
    INCIDENT P1 — 2024-03-15 14:32 UTC
    Service: payment-api (production)
    Alert: Error rate exceeded 15% (threshold: 2%)
    Symptom: Customers reporting "Payment failed" errors on checkout
    First seen: ~14:25 UTC
    Region: East US 2
    """

    print("Responding to incident...")
    results = respond_to_incident(incident, diag_id, research_id, comm_id)

    print("\n" + "="*60)
    print("STAKEHOLDER UPDATE:")
    print("="*60)
    print(results["stakeholder_update"])

if __name__ == "__main__":
    main()
```

---

## Test It

```bash
python incident_responder.py
```

Time the execution:

```bash
time python incident_responder.py
```

With parallel fan-out, diagnostician + researcher run simultaneously. Expect total time ~30-45 seconds vs ~60-80 seconds sequential.

Verify:
- Diagnostician made at least one `query_app_insights` call (check run steps)
- Researcher found and cited relevant runbooks
- Communicator output has no technical jargon

---

## Common Mistakes

- **Thread pool size too small.** With 3+ agents, use `max_workers` equal to the number of parallel agents.
- **Tool function returns unserializable objects.** All tool returns must be strings (JSON-encoded). Never return raw Python objects.
- **Agent IDs hardcoded.** Create agents once and store IDs in environment variables or Key Vault. Re-creating agents on every incident wastes time and quota.

---

## Extend It

1. **PagerDuty integration:** After the communicator draft is approved, use a FunctionTool to POST the update to a PagerDuty incident timeline via their API.
2. **Incident post-mortem generation:** After resolution, pass the full diagnostic + remediation timeline to a fourth agent that drafts a 5 Whys post-mortem document.
3. **Severity auto-triage:** Add a fourth agent before the fan-out that reads the alert payload and assigns severity (P1/P2/P3), then skip the researcher for P3s to save cost.

---

## Resources

- [azure-monitor-query KQL Python SDK](https://learn.microsoft.com/python/api/overview/azure/monitor-query-readme)
- [Azure AI Search Python SDK](https://learn.microsoft.com/python/api/overview/azure/search-documents-readme)
- [Foundry agent function calling](https://learn.microsoft.com/azure/ai-foundry/agents/how-to/tools/function-calling)
- [Application Insights KQL reference](https://learn.microsoft.com/azure/azure-monitor/logs/log-analytics-tutorial)
