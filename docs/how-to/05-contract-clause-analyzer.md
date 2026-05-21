# 05 — Contract Clause Analyzer: Risk-Score a Contract with GPT-5.4

Extract every clause from a contract PDF, run GPT-5.4's 1M-token context window over the whole thing, and get back a risk-scored table with specific clause annotations.

---

## What You're Building

A Python pipeline that uses Azure AI Document Intelligence to extract clean text from a contract PDF, then sends the entire document to a Foundry agent running GPT-5.4 (which fits contracts up to ~700 pages in a single context window). The agent uses Code Interpreter to generate a risk scoring table. Output: a markdown risk summary plus clause-level annotations highlighting problematic language.

---

## Prerequisites

- Microsoft Foundry project with GPT-5.4 deployed
- Azure AI Document Intelligence resource (any tier; F0 free tier works for testing)
- Python 3.11+
- `azure-ai-projects`, `azure-ai-documentintelligence`, `azure-identity`
- A contract PDF (NDA, SaaS agreement, or employment contract)

```bash
pip install azure-ai-projects azure-ai-documentintelligence azure-identity python-dotenv
```

---

## Architecture

```
Contract PDF
        │
        ▼
Azure AI Document Intelligence
  (Layout model — extracts text, tables, page structure)
        │
        ▼
Clean structured text (markdown)
        │
        ▼
Foundry Agent: GPT-5.4 (1M context)
  └── CodeInterpreterTool
        │  ├── Risk scoring logic (Python)
        │  └── Generates risk table as CSV/markdown
        ▼
Outputs:
  ├── risk_summary.md (overall risk assessment)
  ├── risk_table.csv (clause-by-clause scores)
  └── annotations.json (specific risky phrases)
```

---

## Step-by-Step Build

### Step 1 — Create a Document Intelligence resource

```bash
az cognitiveservices account create \
  --name contract-doc-intel \
  --resource-group $RG \
  --kind FormRecognizer \
  --sku S0 \
  --location eastus2 \
  --yes

# Get endpoint and key
DOC_INTEL_ENDPOINT=$(az cognitiveservices account show \
  --name contract-doc-intel --resource-group $RG \
  --query properties.endpoint -o tsv)

DOC_INTEL_KEY=$(az cognitiveservices account keys list \
  --name contract-doc-intel --resource-group $RG \
  --query key1 -o tsv)
```

### Step 2 — Extract text from contract PDF

```python
import os
import pathlib
from dotenv import load_dotenv
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.core.credentials import AzureKeyCredential

load_dotenv()

DOC_INTEL_ENDPOINT = os.environ["DOC_INTEL_ENDPOINT"]
DOC_INTEL_KEY = os.environ["DOC_INTEL_KEY"]

doc_client = DocumentIntelligenceClient(
    endpoint=DOC_INTEL_ENDPOINT,
    credential=AzureKeyCredential(DOC_INTEL_KEY)
)

def extract_contract_text(pdf_path: str) -> str:
    """Extract text from PDF using Document Intelligence Layout model."""
    with open(pdf_path, "rb") as f:
        poller = doc_client.begin_analyze_document(
            model_id="prebuilt-layout",
            body=f,
            content_type="application/pdf",
            output_content_format="markdown"   # Get markdown with table structure
        )
    result = poller.result()

    # Concatenate all page content
    full_text = result.content
    print(f"Extracted {len(full_text)} characters from {result.pages} pages")
    return full_text
```

> **Why Document Intelligence instead of raw PDF parsing?** PyPDF2 and pdfplumber struggle with multi-column layouts, headers/footers, and tables (common in legal documents). Document Intelligence's Layout model correctly handles these and outputs clean markdown — which the model processes much better.

### Step 3 — Create the analysis agent

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.projects.models import (
    CodeInterpreterTool,
    PromptAgentDefinition
)

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
ai_client = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential()
)

def create_contract_agent() -> str:
    """Create the contract analysis agent with Code Interpreter."""
    code_interpreter = CodeInterpreterTool()

    agent_def = PromptAgentDefinition(
        model="gpt-5.4",   # 1M context — fits even long contracts
        name="contract-analyzer",
        instructions="""You are an expert contract attorney and risk analyst.
When given a contract, you will:

1. Identify and extract all clauses (give each a short name and clause number)
2. Score each clause for risk on a scale of 1-5:
   - 1 = Standard, no concern
   - 2 = Slightly unusual, monitor
   - 3 = Moderate risk, recommend review
   - 4 = High risk, recommend negotiation
   - 5 = Severe risk, recommend rejection or legal counsel
3. Flag specific risky language with exact quotes
4. Use Code Interpreter to generate a risk scoring table

Risk categories to check:
- Liability caps and indemnification (especially uncapped liability)
- IP ownership (look for "work for hire" and IP assignment clauses)
- Termination rights (especially unilateral termination without cause)
- Data privacy and security obligations
- Dispute resolution (binding arbitration, jurisdiction)
- Non-compete and non-solicitation scope
- Auto-renewal with short cancellation windows

Output format:
1. Executive summary (3-5 sentences)
2. Risk table (generated via Code Interpreter as markdown)
3. High-risk clause details (risk score 4-5 with exact quotes)
4. Recommended actions
""",
        tools=[code_interpreter.definitions],
        tool_resources=code_interpreter.resources,
    )
    agent = ai_client.agents.create_version(definition=agent_def)
    print(f"Contract agent created: {agent.id}")
    return agent.id
```

### Step 4 — Run the analysis

```python
from azure.ai.projects.models import MessageRole
import json

def analyze_contract(agent_id: str, contract_text: str, contract_name: str) -> dict:
    """Analyze a contract and return structured results."""
    thread = ai_client.agents.create_thread()

    # Attach the contract text as the message
    # GPT-5.4 has 1M context — even a 300-page contract is ~150K tokens
    ai_client.agents.create_message(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=(
            f"Please analyze this contract: **{contract_name}**\n\n"
            "Generate a complete risk assessment with a clause-by-clause risk table.\n\n"
            f"--- CONTRACT TEXT ---\n\n{contract_text}"
        )
    )

    print("Running analysis (this takes 30-90 seconds for large contracts)...")
    run = ai_client.agents.create_and_process_run(
        thread_id=thread.id,
        agent_id=agent_id
    )

    if run.status != "completed":
        raise RuntimeError(f"Analysis failed: {run.status}\n{run.last_error}")

    messages = ai_client.agents.list_messages(thread_id=thread.id)
    last = messages.get_last_message_by_role(MessageRole.ASSISTANT)

    # Collect text content and any generated files
    analysis_text = ""
    generated_files = []

    for block in last.content:
        if hasattr(block, "text"):
            analysis_text += block.text.value + "\n"
        elif hasattr(block, "image_file"):
            generated_files.append(block.image_file.file_id)

    # Also check for file attachments from Code Interpreter
    for msg in messages.data:
        for att in getattr(msg, "attachments", []) or []:
            if hasattr(att, "file_id"):
                generated_files.append(att.file_id)

    return {
        "analysis": analysis_text,
        "generated_files": generated_files,
        "thread_id": thread.id
    }
```

### Step 5 — Download Code Interpreter outputs

```python
def download_agent_files(file_ids: list[str], output_dir: str = "outputs") -> list[str]:
    """Download any files generated by Code Interpreter."""
    pathlib.Path(output_dir).mkdir(exist_ok=True)
    paths = []

    for file_id in file_ids:
        # Get file metadata to determine name/extension
        file_info = ai_client.agents.get_file(file_id)
        filename = getattr(file_info, "filename", f"{file_id}.bin")
        out_path = f"{output_dir}/{filename}"

        # Download file content
        content = ai_client.agents.get_file_content(file_id)
        with open(out_path, "wb") as f:
            for chunk in content:
                f.write(chunk)
        paths.append(out_path)
        print(f"Downloaded: {out_path}")

    return paths
```

### Step 6 — Wire it all together

```python
import pathlib
from datetime import datetime

def main(pdf_path: str):
    contract_name = pathlib.Path(pdf_path).stem

    print("Step 1: Extracting contract text...")
    contract_text = extract_contract_text(pdf_path)

    print("Step 2: Creating analysis agent...")
    agent_id = create_contract_agent()

    print("Step 3: Analyzing contract...")
    results = analyze_contract(agent_id, contract_text, contract_name)

    print("Step 4: Saving results...")
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    output_dir = f"outputs/{contract_name}_{ts}"
    pathlib.Path(output_dir).mkdir(parents=True, exist_ok=True)

    # Save main analysis
    analysis_path = f"{output_dir}/risk_analysis.md"
    with open(analysis_path, "w") as f:
        f.write(f"# Contract Risk Analysis: {contract_name}\n\n")
        f.write(results["analysis"])
    print(f"Saved analysis: {analysis_path}")

    # Download Code Interpreter outputs (tables, charts)
    if results["generated_files"]:
        download_agent_files(results["generated_files"], output_dir)

    print(f"\nAnalysis complete. Results in: {output_dir}/")

if __name__ == "__main__":
    import sys
    main(sys.argv[1] if len(sys.argv) > 1 else "contract.pdf")
```

```bash
python contract_analyzer.py my-vendor-agreement.pdf
```

---

## Test It

Use a public sample contract (NDAs and SaaS agreements are widely available):

```bash
# Download a sample NDA from GitHub
curl -L "https://raw.githubusercontent.com/microsoft/Commercial-Marketplace-SaaS-Accelerator/main/docs/saas-agreement-sample.pdf" \
  -o sample-contract.pdf

python contract_analyzer.py sample-contract.pdf
```

Expected output structure:

```markdown
# Contract Risk Analysis: sample-contract

## Executive Summary
This SaaS agreement presents moderate overall risk. Key concerns include an 
uncapped indemnification clause in Section 8.2 and a 90-day auto-renewal window 
with only 15-day cancellation notice required...

## Risk Table
| Clause | Section | Risk Score | Category | Notes |
|--------|---------|-----------|----------|-------|
| Indemnification | 8.2 | 4/5 | Liability | Uncapped mutual indemnification |
| Auto-renewal | 12.1 | 3/5 | Termination | 15-day cancellation window |
| IP Assignment | 6.1 | 2/5 | IP | Standard work-for-hire, expected |
```

---

## Common Mistakes

- **Document Intelligence returns empty text.** This happens with password-protected PDFs. Remove password protection before processing.
- **GPT-5.4 token limit for very large contracts.** 1M tokens ≈ 750K words. Most contracts fit easily, but if you hit limits, split by section headers returned by Document Intelligence.
- **Code Interpreter generates a chart but returns no risk table.** Specify in your prompt: "Generate the risk table as both a markdown table AND a CSV file attachment."

---

## Extend It

1. **Clause comparison:** Upload two versions of a contract and have the agent diff them, highlighting what the other party changed and whether those changes increase risk.
2. **Precedent library:** Store analyzed contracts in Cosmos DB. When a new contract comes in, have the agent compare it to similar contracts from your precedent library to identify unusual clauses.
3. **Automated redlines:** Have the agent generate suggested alternative clause language for all risk-scored 4+ clauses. Export as a Word document with tracked changes using `python-docx`.

---

## Resources

- [Azure AI Document Intelligence Layout model](https://learn.microsoft.com/azure/ai-services/document-intelligence/concept-layout)
- [Code Interpreter in Foundry agents](https://learn.microsoft.com/azure/ai-foundry/agents/how-to/tools/code-interpreter)
- [GPT-5.4 model capabilities](https://learn.microsoft.com/azure/ai-foundry/openai/concepts/models)
- [azure-ai-documentintelligence SDK](https://learn.microsoft.com/python/api/overview/azure/ai-documentintelligence-readme)
