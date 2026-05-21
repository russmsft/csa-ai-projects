# CSA Onboarding AI Projects

A curated portfolio of AI project ideas for new Cloud Solution Architects at Microsoft CSU Cloud & AI. Built on Microsoft Foundry.

## What's in Here

| File | Description |
|------|-------------|
| `docs/ai-brainstorming.md` | 12 AI project ideas with impact/difficulty ratings, Azure service mappings, phased roadmap, and model selection cheat sheet |

## Who This Is For

New CSAs with solid cloud/dev backgrounds who are new to Azure AI and Microsoft Foundry. The ideas serve two purposes:

- **Demo assets** — personal projects you build once and bring into every customer meeting
- **Co-build templates** — POC starters you adapt to a customer's data and deploy alongside them

## Quick Start

Start with these three ideas from the brainstorm (in order):

1. **Ask My Docs** — RAG in a day using Foundry Agent Service + File Search
2. **Meeting Minutes Agent** — Audio transcription + structured extraction
3. **Internal Policy Chatbot** — Foundry IQ (turnkey RAG, no pipeline to build)

These three give you working Foundry experience and customer-ready demos within your first two weeks.

## Prerequisites

- Azure subscription with Microsoft Foundry access
- Azure CLI (`az`) installed and logged in
- Python 3.11+ or .NET 8+ depending on the idea you're building
- VS Code with the Foundry extension (optional but recommended)

## Environment Setup

```bash
# Install Azure CLI
winget install Microsoft.AzureCLI

# Log in
az login

# Create a Foundry resource and project via portal:
# https://ai.azure.com

# Install the Foundry SDK
pip install azure-ai-projects azure-identity
```

## How to Use the Brainstorm

Open `docs/ai-brainstorming.md` and follow the phased roadmap:

- **Phase 1 (0-3 months):** Ideas 1, 2, 4, 3, 6 — build your foundation
- **Phase 2 (3-9 months):** Pick from Ideas 5, 7, 8, 9 based on your customer mix
- **Phase 3 (9-18 months):** Ideas 10, 11, 12 — lead complex engagements

## Key Resources

- [Microsoft Foundry portal](https://ai.azure.com)
- [Foundry Agent Service overview](https://learn.microsoft.com/azure/ai-foundry/agents/overview)
- [Microsoft Agent Framework](https://learn.microsoft.com/agent-framework/overview/agent-framework-overview)
- [Foundry Models catalog](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Baseline reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)

## Notes

- All ideas use **Microsoft Foundry** as the AI platform — not standalone Azure OpenAI resources
- Model recommendations are current as of May 2026; check the Foundry catalog for the latest
- The `.gitignore` excludes this folder from the main repo — generated artifacts stay local
