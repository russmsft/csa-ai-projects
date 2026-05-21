# 01 — Ask My Docs: Build a Foundry File Search Agent

Give your documents a voice. This guide wires up a Foundry prompt agent with File Search so you can upload PDFs and get cited, accurate answers back in seconds.

---

## What You're Building

A Python script that uploads a PDF to Microsoft Foundry, creates a vector store over it, and attaches a `FileSearchTool` to a GPT-4.1-mini agent. You ask questions in plain English; the agent returns answers with document citations so you can verify every claim.

This is the fastest path to production RAG — no custom chunking pipeline, no vector DB to manage, no embeddings code to write yourself.

---

## Prerequisites

- Azure subscription with Microsoft Foundry (AI Foundry hub + project created at [https://ai.azure.com](https://ai.azure.com))
- Python 3.11+
- `azure-ai-projects >= 1.0.0` and `azure-identity` installed
- Azure CLI logged in (`az login`) with Contributor on the Foundry project
- A PDF you want to query (a product spec, runbook, or HR policy works fine)

```bash
pip install azure-ai-projects azure-identity
```

---

## Architecture

```
You (Python script)
        │
        ▼
AIProjectClient ──► Foundry Agent Service
        │                    │
        │            FileSearchTool
        │                    │
        │            Vector Store (Foundry-managed)
        │                    │
        │            Uploaded PDF(s)
        │
        ▼
  Agent Thread ──► GPT-4.1-mini ──► Response + Citations
```

**Data flow:** Your PDF is uploaded to Foundry storage → chunked and embedded automatically → stored in a managed vector store → retrieved at query time → passed as context to GPT-4.1-mini → response includes `[source: filename, page N]` citations.

---

## Step-by-Step Build

### Step 1 — Set your project endpoint

```bash
# Find your endpoint in AI Foundry portal: Project → Overview → "API endpoint"
# It looks like: https://<hub-name>.ai.azure.com/api/projects/<project-name>
export FOUNDRY_PROJECT_ENDPOINT="https://<hub>.ai.azure.com/api/projects/<project>"
```

Store it in a `.env` file — never hardcode it.

### Step 2 — Install dependencies

```bash
pip install azure-ai-projects azure-identity python-dotenv
```

### Step 3 — Create the client

```python
import os
from dotenv import load_dotenv
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

load_dotenv()

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
client = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential()
)
```

`DefaultAzureCredential` picks up your `az login` token automatically. No API keys needed.

### Step 4 — Upload your document and create a vector store

```python
import pathlib

def upload_pdf_and_create_vector_store(pdf_path: str, store_name: str) -> str:
    """Upload a PDF and return the vector store ID."""
    pdf_bytes = pathlib.Path(pdf_path).read_bytes()
    filename = pathlib.Path(pdf_path).name

    # Upload the file to Foundry
    uploaded = client.agents.upload_file_and_poll(
        file=pdf_bytes,
        filename=filename,
        purpose="assistants"
    )
    print(f"Uploaded file: {uploaded.id}")

    # Create a vector store with the file attached
    vector_store = client.agents.create_vector_store_and_poll(
        file_ids=[uploaded.id],
        name=store_name
    )
    print(f"Vector store ready: {vector_store.id}  status={vector_store.status}")
    return vector_store.id
```

> **Gotcha:** `create_vector_store_and_poll` blocks until the indexing job finishes. For large PDFs (>50 MB) this can take a minute. If you hit a timeout, increase the `polling_interval` kwarg or switch to `create_vector_store` and poll manually.

### Step 5 — Create the agent

```python
from azure.ai.projects.models import FileSearchTool, PromptAgentDefinition

def create_ask_my_docs_agent(vector_store_id: str) -> str:
    """Create the agent and return its agent_id."""
    file_search = FileSearchTool(vector_store_ids=[vector_store_id])

    agent_def = PromptAgentDefinition(
        model="gpt-4.1-mini",
        name="ask-my-docs",
        description="Answers questions using uploaded documents with citations.",
        instructions=(
            "You are a helpful assistant that answers questions based solely on "
            "the provided documents. Always cite the source file and, where possible, "
            "the page number. If the answer isn't in the documents, say so — "
            "do not guess."
        ),
        tools=[file_search.definitions],
        tool_resources=file_search.resources,
    )

    agent = client.agents.create_version(definition=agent_def)
    print(f"Agent created: {agent.id}")
    return agent.id
```

> **Model note:** GPT-4.1-mini is the right pick here — it's fast and cheap for Q&A. Switch to GPT-4.1 if you need deeper reasoning over complex technical docs.

### Step 6 — Run a query

```python
from azure.ai.projects.models import MessageRole

def ask(agent_id: str, question: str) -> str:
    """Create a thread, post a message, run the agent, return the answer."""
    thread = client.agents.create_thread()

    client.agents.create_message(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=question
    )

    run = client.agents.create_and_process_run(
        thread_id=thread.id,
        agent_id=agent_id
    )

    if run.status != "completed":
        raise RuntimeError(f"Run ended with status: {run.status}\n{run.last_error}")

    messages = client.agents.list_messages(thread_id=thread.id)
    last = messages.get_last_message_by_role(MessageRole.ASSISTANT)

    # Print answer with citations
    for block in last.content:
        if hasattr(block, "text"):
            print(block.text.value)
            for ann in block.text.annotations:
                if hasattr(ann, "file_citation"):
                    print(f"  ↳ Source: {ann.file_citation.file_id}")
    return last.content[0].text.value
```

### Step 7 — Wire it all together

```python
def main():
    PDF_PATH = "your-document.pdf"       # change this
    STORE_NAME = "ask-my-docs-store"

    vector_store_id = upload_pdf_and_create_vector_store(PDF_PATH, STORE_NAME)
    agent_id = create_ask_my_docs_agent(vector_store_id)

    questions = [
        "What are the key SLA commitments in this document?",
        "List any security requirements mentioned.",
        "What are the escalation procedures?",
    ]

    for q in questions:
        print(f"\n❓ {q}")
        ask(agent_id, q)

if __name__ == "__main__":
    main()
```

---

## Test It

Run the script against a PDF you know well so you can verify accuracy:

```bash
python ask_my_docs.py
```

Expected output:

```
Uploaded file: file-abc123
Vector store ready: vs-xyz789  status=completed
Agent created: agent-def456

❓ What are the key SLA commitments?
The document specifies a 99.9% uptime SLA for the production environment [1], with
a 4-hour RTO and 1-hour RPO for critical systems [2].
  ↳ Source: file-abc123
```

**Verify citations are accurate:** Open the PDF and confirm the cited content actually says what the agent claims. If citations are wrong, check that the PDF isn't a scanned image — Document Intelligence preprocessing (Guide 05) may be needed.

**Add a second document:**

```python
uploaded2 = client.agents.upload_file_and_poll(
    file=pathlib.Path("second-doc.pdf").read_bytes(),
    filename="second-doc.pdf",
    purpose="assistants"
)
# Add to existing vector store
client.agents.create_vector_store_file(
    vector_store_id=vector_store_id,
    file_id=uploaded2.id
)
```

---

## Common Mistakes

- **Scanned PDFs return empty text.** Foundry's file upload does basic text extraction. If your PDF is a scanned image, pre-process it with Azure AI Document Intelligence first (see Guide 05).
- **Agent answers questions not in the docs.** Check your system prompt — add `"Only answer using the provided documents."` if hallucination is a problem.
- **Thread reuse leaks context.** Create a new thread for each user session. Reusing threads across users exposes previous conversation history.

---

## Extend It

1. **Multi-tenant isolation:** Create a separate vector store per user/team. Pass the correct `vector_store_id` based on the authenticated user's group membership from Entra ID.
2. **Streaming responses:** Replace `create_and_process_run` with `create_stream` and yield tokens to a frontend for real-time display.
3. **Slack / Teams bot:** Wrap the `ask()` function in an Azure Function with an HTTP trigger, then register it as a Teams bot via Bot Framework. Now your whole org can query the docs from Teams.

---

## Resources

- [Foundry Agent Service — File Search](https://learn.microsoft.com/azure/ai-foundry/agents/how-to/tools/file-search)
- [azure-ai-projects SDK reference](https://learn.microsoft.com/python/api/overview/azure/ai-projects-readme)
- [Vector stores in Foundry](https://learn.microsoft.com/azure/ai-foundry/agents/concepts/vector-stores)
- [DefaultAzureCredential auth chain](https://learn.microsoft.com/python/api/azure-identity/azure.identity.defaultazurecredential)
