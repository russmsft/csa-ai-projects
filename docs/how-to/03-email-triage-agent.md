# 03 — Email Triage Agent: Classify, Draft, and Route Incoming Email

Stop drowning in support email. This guide builds a multi-step agent pipeline that reads from a Service Bus queue, classifies urgency and category, drafts a response, and routes each email to the right team.

---

## What You're Building

An Azure Functions-triggered pipeline with three stages: GPT-4.1-mini classifies each incoming email (urgency: high/medium/low; category: billing/technical/general). GPT-5.4-mini drafts a response using a knowledge base. A `FunctionTool` routes the email to the correct downstream Service Bus queue based on classification. The whole flow is orchestrated in Python and runs end-to-end in under 10 seconds per email.

---

## Prerequisites

- Microsoft Foundry project with GPT-4.1-mini and GPT-5.4-mini deployed
- Azure Service Bus namespace with three queues: `incoming-email`, `billing-team`, `technical-team`, `general-team`
- Python 3.11+
- `azure-ai-projects`, `azure-servicebus`, `azure-identity`, `azure-functions`

```bash
pip install azure-ai-projects azure-identity azure-servicebus azure-functions python-dotenv
```

---

## Architecture

```
Service Bus: incoming-email queue
        │
        ▼
Azure Function (trigger)
        │
        ▼
Stage 1: GPT-4.1-mini
  ├── Classify urgency (high/medium/low)
  └── Classify category (billing/technical/general)
        │
        ▼
Stage 2: GPT-5.4-mini
  └── Draft response using knowledge base context
        │
        ▼
Stage 3: FunctionTool → route_email()
  ├── billing-team queue
  ├── technical-team queue
  └── general-team queue
        │
        ▼
Enriched message: original + classification + draft response
```

---

## Step-by-Step Build

### Step 1 — Set up Service Bus queues

```bash
NAMESPACE="my-email-triage-ns"
RG="my-resource-group"
LOCATION="eastus2"

az servicebus namespace create \
  --name $NAMESPACE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard

for queue in incoming-email billing-team technical-team general-team; do
  az servicebus queue create \
    --name $queue \
    --namespace-name $NAMESPACE \
    --resource-group $RG
done

# Get connection string for local dev
az servicebus namespace authorization-rule keys list \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv
```

### Step 2 — Client setup

```python
import os
import json
from dotenv import load_dotenv
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.servicebus import ServiceBusClient, ServiceBusMessage

load_dotenv()

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
SB_CONN_STR = os.environ["SERVICE_BUS_CONNECTION_STRING"]

ai_client = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential()
)
openai = ai_client.get_openai_client()
sb_client = ServiceBusClient.from_connection_string(SB_CONN_STR)
```

### Step 3 — Stage 1: Classification

```python
CLASSIFICATION_SCHEMA = {
    "type": "json_schema",
    "json_schema": {
        "name": "email_classification",
        "strict": True,
        "schema": {
            "type": "object",
            "properties": {
                "urgency": {
                    "type": "string",
                    "enum": ["high", "medium", "low"]
                },
                "category": {
                    "type": "string",
                    "enum": ["billing", "technical", "general"]
                },
                "reasoning": {
                    "type": "string",
                    "description": "One sentence explaining the classification."
                },
                "sentiment": {
                    "type": "string",
                    "enum": ["frustrated", "neutral", "positive"]
                }
            },
            "required": ["urgency", "category", "reasoning", "sentiment"],
            "additionalProperties": False
        }
    }
}

def classify_email(subject: str, body: str) -> dict:
    """Classify email urgency and category using GPT-4.1-mini."""
    response = openai.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are an email triage specialist. Classify incoming customer emails.\n"
                    "Urgency: high=system down/data loss/SLA breach, "
                    "medium=degraded functionality, low=question/general request.\n"
                    "Category: billing=invoices/payments/pricing, "
                    "technical=bugs/errors/performance, general=everything else."
                )
            },
            {
                "role": "user",
                "content": f"Subject: {subject}\n\nBody:\n{body}"
            }
        ],
        response_format=CLASSIFICATION_SCHEMA,
        temperature=0.0,
        max_tokens=300
    )
    return json.loads(response.choices[0].message.content)
```

### Step 4 — Stage 2: Response drafting with FunctionTool

Define the routing function. Foundry's `FunctionTool` calls this when the agent decides to route:

```python
from azure.ai.projects.models import FunctionTool, PromptAgentDefinition

# --- Function definitions for the agent ---

ROUTE_FUNCTION = {
    "name": "route_email",
    "description": (
        "Route the processed email to the appropriate team queue. "
        "Call this after generating the draft response."
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "target_queue": {
                "type": "string",
                "enum": ["billing-team", "technical-team", "general-team"],
                "description": "The Service Bus queue to send this email to."
            },
            "priority": {
                "type": "string",
                "enum": ["high", "medium", "low"],
                "description": "Urgency level for queue prioritization."
            },
            "assignee_hint": {
                "type": "string",
                "description": "Optional: suggested team member based on issue type."
            }
        },
        "required": ["target_queue", "priority"]
    }
}

def route_email(target_queue: str, priority: str, assignee_hint: str = "") -> str:
    """Actually send the message to the Service Bus queue."""
    # This runs locally — the agent calls this via the tool mechanism
    return json.dumps({
        "status": "queued",
        "queue": target_queue,
        "priority": priority,
        "assignee_hint": assignee_hint
    })
```

### Step 5 — Create the response-drafting agent

```python
KNOWLEDGE_BASE = """
Common responses:
- Billing issues: "Our billing team reviews invoices within 2 business days. For urgent 
  billing disputes, please reference your invoice number."
- Technical outages: "Our SRE team is notified immediately for P1 issues. 
  Expected response time is 30 minutes."
- Password resets: "Use the self-service portal at https://aka.ms/resetpw"
- General inquiries: "Our support team responds within 1 business day."
"""

def create_triage_agent() -> str:
    """Create the agent that drafts responses and routes emails."""
    function_tool = FunctionTool(functions=[ROUTE_FUNCTION])

    agent_def = PromptAgentDefinition(
        model="gpt-5.4-mini",      # fast reasoning for response quality
        name="email-triage-agent",
        instructions=(
            "You are a customer support triage agent. Given an email and its classification, "
            "draft a professional, empathetic response (3-4 sentences max). "
            "Then call route_email() to route to the correct team.\n\n"
            f"Knowledge base:\n{KNOWLEDGE_BASE}"
        ),
        tools=[function_tool.definitions],
    )
    agent = ai_client.agents.create_version(definition=agent_def)
    return agent.id
```

### Step 6 — Full processing pipeline

```python
from azure.ai.projects.models import MessageRole, SubmitToolOutputsAction

def process_email(
    agent_id: str,
    email_id: str,
    subject: str,
    body: str,
    sender: str
) -> dict:
    """Full pipeline: classify → draft → route."""

    # Stage 1: classify
    classification = classify_email(subject, body)
    print(f"  Classified: {classification['category']} / {classification['urgency']}")

    # Stage 2 & 3: draft + route via agent
    thread = ai_client.agents.create_thread()
    ai_client.agents.create_message(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=(
            f"Process this customer email:\n\n"
            f"From: {sender}\nSubject: {subject}\nBody:\n{body}\n\n"
            f"Classification: {json.dumps(classification)}\n\n"
            "Draft a response and route this email to the correct team."
        )
    )

    # Run with tool call handling
    run = ai_client.agents.create_run(
        thread_id=thread.id,
        agent_id=agent_id
    )

    # Poll and handle tool calls
    route_result = {}
    while run.status in ("queued", "in_progress", "requires_action"):
        run = ai_client.agents.get_run(thread_id=thread.id, run_id=run.id)

        if run.status == "requires_action":
            action = run.required_action
            if isinstance(action, SubmitToolOutputsAction):
                tool_outputs = []
                for call in action.submit_tool_outputs.tool_calls:
                    if call.function.name == "route_email":
                        args = json.loads(call.function.arguments)
                        result = route_email(**args)
                        route_result = json.loads(result)
                        tool_outputs.append({
                            "tool_call_id": call.id,
                            "output": result
                        })
                run = ai_client.agents.submit_tool_outputs_to_run(
                    thread_id=thread.id,
                    run_id=run.id,
                    tool_outputs=tool_outputs
                )

    if run.status != "completed":
        raise RuntimeError(f"Run failed: {run.status} — {run.last_error}")

    messages = ai_client.agents.list_messages(thread_id=thread.id)
    draft_response = messages.get_last_message_by_role(
        MessageRole.ASSISTANT
    ).content[0].text.value

    return {
        "email_id": email_id,
        "classification": classification,
        "draft_response": draft_response,
        "routing": route_result
    }
```

### Step 7 — Azure Function trigger

```python
# function_app.py
import azure.functions as func
import json
import logging

app = func.FunctionApp()

@app.service_bus_queue_trigger(
    arg_name="msg",
    queue_name="incoming-email",
    connection="SERVICE_BUS_CONNECTION_STRING"
)
def triage_email_trigger(msg: func.ServiceBusMessage):
    payload = json.loads(msg.get_body().decode())
    logging.info(f"Processing email: {payload.get('subject', 'no subject')}")

    # agent_id should be stored in app config or environment
    AGENT_ID = os.environ["TRIAGE_AGENT_ID"]

    result = process_email(
        agent_id=AGENT_ID,
        email_id=payload.get("id", "unknown"),
        subject=payload.get("subject", ""),
        body=payload.get("body", ""),
        sender=payload.get("from", "unknown")
    )
    logging.info(f"Processed: {json.dumps(result, indent=2)}")
```

---

## Test It

Send a test message to the queue:

```python
def send_test_email():
    with sb_client:
        sender = sb_client.get_queue_sender("incoming-email")
        with sender:
            test_email = {
                "id": "test-001",
                "from": "customer@example.com",
                "subject": "URGENT: Production system down - cannot process payments",
                "body": (
                    "Our entire payment processing system has been down for 2 hours. "
                    "We are losing thousands of dollars per minute. "
                    "This is completely unacceptable. We need help NOW."
                )
            }
            sender.send_messages(ServiceBusMessage(json.dumps(test_email)))
            print("Test email sent to queue")

send_test_email()
```

Expected classification output:
```json
{
  "urgency": "high",
  "category": "billing",
  "reasoning": "Payment system outage causing financial loss — P1 incident.",
  "sentiment": "frustrated"
}
```

---

## Common Mistakes

- **Tool call loop never resolves.** If `route_email` isn't registered correctly, the run stays in `requires_action` forever. Always validate function name matches exactly between `ROUTE_FUNCTION["name"]` and your handler.
- **Service Bus message ordering.** Standard tier doesn't guarantee order. If you need FIFO for high-urgency emails, use Premium tier with sessions.
- **Draft responses leaking PII.** Add a PII scan step between drafting and routing using Azure AI Language's `RecognizePiiEntities` before sending to downstream teams.

---

## Extend It

1. **Escalation timer:** If a `high` urgency email isn't claimed from `technical-team` queue within 15 minutes, trigger a second Azure Function that pages the on-call engineer via PagerDuty API.
2. **Feedback loop:** After an agent responds, capture customer satisfaction (CSAT) score. Use it to fine-tune or adjust prompts monthly.
3. **Multi-language support:** Detect language with Azure AI Language, translate to English for processing, translate draft response back to original language before routing.

---

## Resources

- [FunctionTool in Foundry agents](https://learn.microsoft.com/azure/ai-foundry/agents/how-to/tools/function-calling)
- [Azure Service Bus Python SDK](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-python-how-to-use-queues)
- [Azure Functions Service Bus trigger](https://learn.microsoft.com/azure/azure-functions/functions-bindings-service-bus-trigger)
- [Structured outputs reference](https://learn.microsoft.com/azure/ai-foundry/openai/how-to/structured-outputs)
