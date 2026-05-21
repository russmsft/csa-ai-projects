# 04 — Policy Chatbot: Ground a Chatbot in HR/IT Documents with Foundry IQ

Build an HR or IT policy chatbot that stays on-topic, answers grounded in your actual documents, and refuses to make things up — no custom chunking pipeline required.

---

## What You're Building

A web chatbot deployed on Azure App Service with Entra ID authentication. The backend is a FastAPI app that creates a Foundry prompt agent with Foundry IQ knowledge grounding. You upload policy PDFs through the Foundry portal; the agent retrieves relevant chunks automatically. A minimal HTML/JS frontend handles the chat UI.

---

## Prerequisites

- Microsoft Foundry project with Foundry IQ enabled (check: Project → Knowledge → Foundry IQ)
- Azure App Service (B1 or higher, Python 3.11 runtime)
- Entra ID app registration for auth (or use Easy Auth)
- Python 3.11+, `fastapi`, `uvicorn`, `azure-ai-projects`, `azure-identity`
- 1-5 PDF policy documents to test with

```bash
pip install azure-ai-projects azure-identity fastapi uvicorn python-dotenv
```

---

## Architecture

```
Browser (HTML/JS)
        │  HTTP POST /chat
        ▼
Azure App Service (FastAPI)
  └── Entra ID auth (Easy Auth middleware)
        │
        ▼
AIProjectClient
  └── Prompt Agent (GPT-4.1-mini)
        │
        ▼
Foundry IQ (knowledge grounding)
  ├── HR Policy.pdf
  ├── IT Security Policy.pdf
  └── Employee Handbook.pdf
        │
        ▼
Answer + source citations
```

---

## Step-by-Step Build

### Step 1 — Upload documents via Foundry portal

1. Open [https://ai.azure.com](https://ai.azure.com) → your project
2. Navigate to **Knowledge** → **Foundry IQ**
3. Click **Add data source** → **Upload files**
4. Upload your PDFs (HR policy, IT security policy, employee handbook, etc.)
5. Wait for indexing to complete (status turns green)
6. Copy the **Knowledge ID** — you'll need it in the agent definition

> **Gotcha:** Foundry IQ processes PDFs natively, but for PDFs with complex tables or multi-column layouts, accuracy degrades. Use Azure AI Document Intelligence preprocessing for those.

### Step 2 — Create the FastAPI backend

```python
# app.py
import os
import json
from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from dotenv import load_dotenv
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.projects.models import (
    FileSearchTool,
    MessageRole,
    PromptAgentDefinition
)

load_dotenv()

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["POST"],
    allow_headers=["*"]
)

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
KNOWLEDGE_ID = os.environ["FOUNDRY_IQ_KNOWLEDGE_ID"]

ai_client = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential()
)

# Create agent once at startup
_agent_id: str | None = None

def get_agent_id() -> str:
    global _agent_id
    if _agent_id:
        return _agent_id

    # Attach the Foundry IQ knowledge source via FileSearchTool
    file_search = FileSearchTool(vector_store_ids=[KNOWLEDGE_ID])

    agent_def = PromptAgentDefinition(
        model="gpt-4.1-mini",
        name="policy-chatbot",
        instructions=(
            "You are an HR/IT policy assistant for our company. "
            "Answer questions using only the provided policy documents. "
            "Always cite the specific policy document and section. "
            "If the answer isn't in the documents, say: "
            "'I don't have that information in our current policies — "
            "please contact HR directly.' "
            "Never guess at policy details. Never provide legal advice."
        ),
        tools=[file_search.definitions],
        tool_resources=file_search.resources,
    )
    agent = ai_client.agents.create_version(definition=agent_def)
    _agent_id = agent.id
    return _agent_id


class ChatRequest(BaseModel):
    message: str
    thread_id: str | None = None   # None = new conversation


class ChatResponse(BaseModel):
    reply: str
    thread_id: str
    citations: list[dict]


@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    agent_id = get_agent_id()

    # Reuse thread for ongoing conversations
    if req.thread_id:
        thread_id = req.thread_id
    else:
        thread = ai_client.agents.create_thread()
        thread_id = thread.id

    ai_client.agents.create_message(
        thread_id=thread_id,
        role=MessageRole.USER,
        content=req.message
    )

    run = ai_client.agents.create_and_process_run(
        thread_id=thread_id,
        agent_id=agent_id
    )

    if run.status != "completed":
        raise HTTPException(
            status_code=500,
            detail=f"Agent run failed: {run.status}"
        )

    messages = ai_client.agents.list_messages(thread_id=thread_id)
    last = messages.get_last_message_by_role(MessageRole.ASSISTANT)

    reply_text = last.content[0].text.value
    citations = []
    for block in last.content:
        if hasattr(block, "text"):
            for ann in block.text.annotations:
                if hasattr(ann, "file_citation"):
                    citations.append({
                        "file_id": ann.file_citation.file_id,
                        "quote": getattr(ann.file_citation, "quote", "")
                    })

    return ChatResponse(
        reply=reply_text,
        thread_id=thread_id,
        citations=citations
    )


@app.get("/health")
def health():
    return {"status": "ok"}


# Serve the static chat UI
app.mount("/", StaticFiles(directory="static", html=True), name="static")
```

### Step 3 — Build the chat UI

```html
<!-- static/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Policy Assistant</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: "Segoe UI", sans-serif; background: #f5f5f5; }
    #app { max-width: 800px; margin: 40px auto; background: white; 
           border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); 
           display: flex; flex-direction: column; height: 80vh; }
    header { background: #0078d4; color: white; padding: 16px 20px; 
             border-radius: 8px 8px 0 0; }
    header h1 { font-size: 1.2rem; }
    header p { font-size: 0.85rem; opacity: 0.8; margin-top: 4px; }
    #messages { flex: 1; overflow-y: auto; padding: 20px; display: flex; 
                flex-direction: column; gap: 12px; }
    .msg { max-width: 75%; padding: 10px 14px; border-radius: 8px; 
           line-height: 1.5; font-size: 0.9rem; }
    .msg.user { background: #0078d4; color: white; align-self: flex-end; 
                border-bottom-right-radius: 2px; }
    .msg.bot { background: #f0f0f0; color: #333; align-self: flex-start; 
               border-bottom-left-radius: 2px; }
    .citation { font-size: 0.75rem; color: #666; margin-top: 6px; 
                font-style: italic; }
    #input-row { display: flex; padding: 16px; border-top: 1px solid #e0e0e0; gap: 8px; }
    #user-input { flex: 1; padding: 10px 14px; border: 1px solid #ccc; 
                  border-radius: 6px; font-size: 0.9rem; outline: none; }
    #user-input:focus { border-color: #0078d4; }
    #send-btn { background: #0078d4; color: white; border: none; 
                padding: 10px 20px; border-radius: 6px; cursor: pointer; 
                font-size: 0.9rem; }
    #send-btn:disabled { opacity: 0.5; cursor: not-allowed; }
  </style>
</head>
<body>
<div id="app">
  <header>
    <h1>Policy Assistant</h1>
    <p>Ask questions about HR and IT policies</p>
  </header>
  <div id="messages">
    <div class="msg bot">
      Hello! I can answer questions about our HR and IT policies. 
      What would you like to know?
    </div>
  </div>
  <div id="input-row">
    <input id="user-input" type="text" placeholder="Ask a policy question..." 
           autocomplete="off" />
    <button id="send-btn">Send</button>
  </div>
</div>

<script>
  let threadId = null;

  async function sendMessage() {
    const input = document.getElementById('user-input');
    const btn = document.getElementById('send-btn');
    const text = input.value.trim();
    if (!text) return;

    appendMessage(text, 'user');
    input.value = '';
    btn.disabled = true;

    try {
      const res = await fetch('/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: text, thread_id: threadId })
      });
      const data = await res.json();
      threadId = data.thread_id;

      const msgDiv = appendMessage(data.reply, 'bot');
      if (data.citations && data.citations.length > 0) {
        const citeDiv = document.createElement('div');
        citeDiv.className = 'citation';
        citeDiv.textContent = `Sources: ${data.citations.length} policy document(s)`;
        msgDiv.appendChild(citeDiv);
      }
    } catch (err) {
      appendMessage('Sorry, something went wrong. Please try again.', 'bot');
    } finally {
      btn.disabled = false;
      input.focus();
    }
  }

  function appendMessage(text, role) {
    const div = document.createElement('div');
    div.className = `msg ${role}`;
    div.textContent = text;
    const messages = document.getElementById('messages');
    messages.appendChild(div);
    messages.scrollTop = messages.scrollHeight;
    return div;
  }

  document.getElementById('send-btn').addEventListener('click', sendMessage);
  document.getElementById('user-input').addEventListener('keydown', e => {
    if (e.key === 'Enter') sendMessage();
  });
</script>
</body>
</html>
```

### Step 4 — Deploy to Azure App Service

```bash
# Create App Service plan and web app
az appservice plan create \
  --name policy-chatbot-plan \
  --resource-group $RG \
  --sku B1 \
  --is-linux

az webapp create \
  --name policy-chatbot-app \
  --resource-group $RG \
  --plan policy-chatbot-plan \
  --runtime "PYTHON:3.11"

# Configure environment variables
az webapp config appsettings set \
  --name policy-chatbot-app \
  --resource-group $RG \
  --settings \
    FOUNDRY_PROJECT_ENDPOINT="https://<hub>.ai.azure.com/api/projects/<project>" \
    FOUNDRY_IQ_KNOWLEDGE_ID="<your-knowledge-id>" \
    SCM_DO_BUILD_DURING_DEPLOYMENT=true

# Enable managed identity so the app can authenticate to Foundry
az webapp identity assign \
  --name policy-chatbot-app \
  --resource-group $RG

# Grant the managed identity access to the Foundry project
# (use the principal ID from the output above)
az role assignment create \
  --assignee <principal-id> \
  --role "Azure AI Developer" \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.MachineLearningServices/workspaces/<hub>
```

### Step 5 — Enable Entra ID auth (Easy Auth)

```bash
az webapp auth microsoft update \
  --name policy-chatbot-app \
  --resource-group $RG \
  --client-id <entra-app-client-id> \
  --client-secret <entra-app-client-secret> \
  --tenant-id <tenant-id> \
  --issuer https://login.microsoftonline.com/<tenant-id>/v2.0

az webapp auth update \
  --name policy-chatbot-app \
  --resource-group $RG \
  --enabled true \
  --action RedirectToLoginPage
```

Now only users in your tenant can access the chatbot.

---

## Test It

```bash
# Run locally first
uvicorn app:app --reload --port 8000
```

Test queries:
- "How many vacation days do I get in my first year?"
- "What is our password rotation policy?"
- "Can I expense a home office monitor?"

Watch for:
- Answers grounded in documents (check `citations` in the response)
- Refusals when the question is outside policy scope
- No hallucinated policy details

---

## Common Mistakes

- **Agent created on every request.** The code above caches `_agent_id` at startup. Without this, you'd create a new agent on every chat message — this is slow and wastes quota.
- **Thread ID not persisted across browser sessions.** Store `thread_id` in `localStorage` or a server-side session. Without it, every page refresh starts a new conversation.
- **Foundry IQ knowledge ID vs vector store ID.** They're different identifiers. Foundry IQ knowledge IDs start with `ki-`; vector store IDs start with `vs-`. Don't confuse them.

---

## Extend It

1. **Scope by department:** Create separate agents per department (HR, IT, Finance) each with their own Foundry IQ knowledge source. Route requests based on the authenticated user's group from Entra ID claims.
2. **Audit logging:** Log every question + answer pair to Azure Cosmos DB with the user's UPN (from the Easy Auth headers). Build a Power BI report on top for compliance review.
3. **Feedback buttons:** Add thumbs up/down to each bot message. Store feedback in Cosmos DB and review low-rated answers weekly to improve the document corpus.

---

## Resources

- [Foundry IQ (knowledge grounding)](https://learn.microsoft.com/azure/ai-foundry/agents/concepts/knowledge-grounding)
- [Azure App Service Python deployment](https://learn.microsoft.com/azure/app-service/quickstart-python)
- [App Service Easy Auth (Entra ID)](https://learn.microsoft.com/azure/app-service/configure-authentication-provider-aad)
- [Managed identity for App Service](https://learn.microsoft.com/azure/app-service/overview-managed-identity)
