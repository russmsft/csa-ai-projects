# AI Project Ideas for New CSA Starters — Microsoft CSU Cloud & AI

**Audience:** New Cloud Solution Architects with solid cloud/dev backgrounds, new to AI  
**Purpose:** Ramp-up projects + customer-ready templates  
**Platform:** Microsoft Foundry (current as of May 2026)  
**Models referenced:** GPT-5.5, GPT-5.4 series, GPT-4.1 series, o-series reasoning models

---

## The Big Picture

You just joined CSU. You need two things fast: hands-on confidence with the Azure AI stack, and something to show customers in your first meetings. These 12 ideas are designed to give you both.

Some are personal demo assets — things you build once and bring into every conversation. Others are co-build templates you adapt to a customer's data and deploy alongside them in a POC sprint. The phased roadmap at the bottom tells you what to tackle first.

Every idea runs on Microsoft Foundry. Not standalone Azure OpenAI resources. Not third-party platforms. Foundry is where model access, agent hosting, and application development live now — and it's what you'll be recommending to customers.

---

## Quick Reference: Ideas at a Glance

| # | Idea | Type | Impact | Difficulty | Time to First Value |
|---|------|------|--------|------------|---------------------|
| 1 | Ask My Docs — RAG Starter | Demo asset | ⭐⭐⭐⭐ | ⭐⭐ | 1-2 days |
| 2 | Meeting Minutes Agent | Demo asset | ⭐⭐⭐⭐ | ⭐⭐ | 2-3 days |
| 3 | Customer Email Triage Agent | Co-build | ⭐⭐⭐⭐⭐ | ⭐⭐ | 1 week |
| 4 | Internal Policy Chatbot | Co-build | ⭐⭐⭐⭐ | ⭐⭐ | 1 week |
| 5 | Contract Clause Analyzer | Co-build | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 2 weeks |
| 6 | Multi-Agent Incident Responder | Demo asset | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 2 weeks |
| 7 | Data Pipeline QA Agent | Co-build | ⭐⭐⭐⭐ | ⭐⭐⭐ | 2-3 weeks |
| 8 | Voice-Enabled Field Assistant | Demo asset | ⭐⭐⭐⭐ | ⭐⭐⭐ | 2-3 weeks |
| 9 | Competitive Intelligence Dashboard | Co-build | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 3-4 weeks |
| 10 | Agentic Workflow for Approvals | Co-build | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 4-6 weeks |
| 11 | Computer-Use Process Automator | Demo asset | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 4-6 weeks |
| 12 | Multi-Modal Quality Inspector | Co-build | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 6-8 weeks |

---

## Idea 1: Ask My Docs — RAG Starter Kit

**Type:** Demo asset (reusable across every customer conversation)  
**Impact:** ⭐⭐⭐⭐ | **Difficulty:** ⭐⭐ | **Time to value:** 1-2 days

**What it is:** A prompt agent in Foundry Agent Service with File Search enabled against a small document set. Upload 10-20 PDFs (public Azure docs, a customer's annual report, whatever fits the meeting), and you've got a grounded chatbot that answers questions with citations.

**Why this is your first build:** Every customer asks "can AI search our documents?" This is the answer. Having a working demo running in Foundry — not a slide deck about RAG — changes the energy of the room.

**Business case:** Reduces time-to-answer for knowledge workers from hours of searching to seconds, with source attribution.

**Azure services:**
- Microsoft Foundry (project + prompt agent)
- Foundry Agent Service with File Search tool
- GPT-4.1-mini (cost-efficient, fast) or GPT-5.4-mini for better reasoning
- Azure Blob Storage for document ingestion

**What you'll learn:** Foundry project setup, agent creation, File Search tool, document chunking behavior, grounding quality tuning.

**Stretch:** Swap File Search for Azure AI Search with custom chunking + text-embedding-3-large vectors. Now you understand the full RAG stack.

---

## Idea 2: Meeting Minutes Agent

**Type:** Demo asset  
**Impact:** ⭐⭐⭐⭐ | **Difficulty:** ⭐⭐ | **Time to value:** 2-3 days

**What it is:** Record a Teams meeting (or use a sample audio file). Feed it through Foundry audio models for transcription, then hand the transcript to a prompt agent that extracts action items, decisions, and a summary. Output goes to a structured markdown or JSON doc.

**Why it matters:** This is the "oh, I actually want that" demo. Everyone sits in too many meetings. When a CSA shows this working live — transcribe, summarize, extract — it clicks immediately.

**Business case:** Knowledge workers spend 5+ hours per week on meeting follow-ups. Automating notes and action extraction reclaims that time.

**Azure services:**
- Microsoft Foundry
- GPT-4o audio / Whisper for transcription
- GPT-4.1-mini for summarization agent
- Foundry Agent Service (prompt agent)
- Azure Functions (timer trigger for batch processing, optional)

**What you'll learn:** Audio model APIs, prompt engineering for structured extraction, chaining model outputs.

---

## Idea 3: Customer Email Triage Agent

**Type:** Co-build template  
**Impact:** ⭐⭐⭐⭐⭐ | **Difficulty:** ⭐⭐ | **Time to value:** 1 week

**What it is:** An agent that reads incoming customer emails (or support tickets from a queue), classifies them by urgency and topic, drafts a response, and routes them to the right team. The agent uses function calling to hit the customer's ticketing API and a knowledge base for response templates.

**Why customers care:** Every company with a support inbox is drowning. Classification accuracy above 90% with sub-second latency? That's a staffing conversation.

**Business case:** Cuts first-response time from hours to minutes. Reduces misrouted tickets by 70-80%, based on typical classification accuracy with GPT-4.1-mini on domain-specific categories.

**Azure services:**
- Microsoft Foundry (prompt agent with function calling)
- GPT-4.1-mini (fast classification) + GPT-5.4-mini (draft generation)
- Azure Service Bus (email queue ingestion)
- Azure AI Search + Foundry IQ (knowledge base for response templates)
- Azure Cosmos DB (ticket state and routing history)

**What you'll learn:** Function calling, multi-model routing (cheap model for classification, better model for generation), queue-based architectures.

---

## Idea 4: Internal Policy Chatbot

**Type:** Co-build template  
**Impact:** ⭐⭐⭐⭐ | **Difficulty:** ⭐⭐ | **Time to value:** 1 week

**What it is:** Foundry IQ pointed at a customer's HR policies, IT guidelines, or compliance documentation. Employees ask "how many vacation days do I have left?" or "what's the approval process for a laptop?" and get grounded, cited answers instead of searching a 200-page PDF.

This is different from Idea 1 because it uses Foundry IQ — the turnkey RAG option that requires zero chunking/embedding pipeline work. You bring the documents, Foundry handles the rest.

**Business case:** HR and IT helpdesks handle thousands of "where do I find..." questions monthly. Deflecting 40-60% to a self-service agent saves real headcount.

**Azure services:**
- Microsoft Foundry with Foundry IQ
- GPT-4.1-mini (handles policy Q&A well at low cost)
- Microsoft Entra ID (authentication — employees only)
- Azure App Service (chat UI frontend)

**What you'll learn:** Foundry IQ setup, enterprise auth integration, when to use Foundry IQ vs. building your own RAG pipeline. Spoiler: start with IQ, graduate to custom when you need control over chunking.

---

## Idea 5: Contract Clause Analyzer

**Type:** Co-build template  
**Impact:** ⭐⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐ | **Time to value:** 2 weeks

**What it is:** Upload a contract (PDF or Word). An agent with Code Interpreter extracts clauses, flags risky language against a configurable ruleset, and generates a risk summary with clause-by-clause annotations. Think "redline review assistant" — not replacing lawyers, but cutting their review time in half.

**Why this one punches above its weight:** Legal review is expensive. $500/hour expensive. Showing a demo that correctly flags an indemnification clause with unusual liability caps gets attention from the C-suite, not just IT.

**Business case:** Legal teams spend 60% of contract review time on standard clause identification. Automation cuts review time by 40-50% on routine agreements.

**Azure services:**
- Microsoft Foundry (prompt agent with Code Interpreter)
- GPT-5.4 (needs the larger context window — contracts are long, and you want the full document in context)
- Azure AI Document Intelligence (PDF/image extraction for scanned contracts)
- Azure Blob Storage (contract repository)
- Azure Cosmos DB (clause library and risk rules)

**What you'll learn:** Code Interpreter for document parsing, long-context prompt design, structured output with confidence scoring. Also: how to talk to legal teams about AI without scaring them.

---

## Idea 6: Multi-Agent Incident Responder

**Type:** Demo asset  
**Impact:** ⭐⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐ | **Time to value:** 2 weeks

**What it is:** Three agents working together to handle a production incident. Agent 1 (Diagnostician) reads logs and metrics from Application Insights. Agent 2 (Researcher) searches runbooks and past incident reports. Agent 3 (Communicator) drafts stakeholder updates. Orchestrated as a workflow agent in Foundry.

This is your "wow" demo for platform engineering and DevOps conversations. Most customers have heard about multi-agent systems. Almost none have seen one running.

**Business case:** Mean time to resolution (MTTR) for P1 incidents drops when the initial diagnosis and communication happen in parallel instead of sequentially. We're talking 30-40% reduction for well-instrumented systems.

**Azure services:**
- Microsoft Foundry (workflow agent orchestrating 3 prompt agents)
- Foundry Agent Service (managed hosting + orchestration)
- GPT-5.4-mini (fast reasoning for log analysis)
- Azure Monitor / Application Insights (data source via MCP or function calling)
- Azure AI Search (runbook and incident history index)
- Microsoft Agent Framework (if you want code-level control over the orchestration)

**What you'll learn:** Multi-agent orchestration patterns (specifically the concurrent fan-out pattern), workflow agents in Foundry, connecting agents to live Azure data sources. This is the project where the Agent Framework SDK starts making sense.

---

## Idea 7: Data Pipeline QA Agent

**Type:** Co-build template  
**Impact:** ⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐ | **Time to value:** 2-3 weeks

**What it is:** An agent that monitors data pipelines (Azure Data Factory, Synapse, Fabric) and does automated quality checks. When a pipeline run completes, the agent inspects output data for anomalies — null spikes, schema drift, distribution shifts — and either auto-remediates or alerts the data team with a diagnosis.

**Why data teams love this:** Nobody wants to be the person who discovers bad data in a dashboard two days after the pipeline broke. This agent catches it within minutes.

**Business case:** Data quality issues cost enterprises an average of $12.9M per year (Gartner). Catching anomalies within minutes vs. days prevents downstream decision errors.

**Azure services:**
- Microsoft Foundry (prompt agent with Code Interpreter for statistical analysis)
- GPT-4.1-mini (anomaly description) + codex-mini (code generation for remediation scripts)
- Azure Functions (Cosmos DB change feed trigger or Event Grid trigger from pipeline completion)
- Azure Cosmos DB (pipeline run metadata and quality scores)
- Azure Data Factory / Microsoft Fabric (pipeline integration)

**What you'll learn:** Event-driven agent triggering, Code Interpreter for data analysis, reasoning model selection (codex-mini for code generation tasks).

---

## Idea 8: Voice-Enabled Field Assistant

**Type:** Demo asset  
**Impact:** ⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐ | **Time to value:** 2-3 weeks

**What it is:** A mobile-friendly web app where field technicians speak their question — "what's the torque spec for the Model 7200 compressor valve?" — and get a spoken answer grounded in equipment manuals. Hands-free, eyes-free. Built on GPT-4o Realtime for low-latency voice interaction.

**Why it lands:** Manufacturing and energy customers light up when they see voice interaction with their own technical docs. Their field workers can't scroll through PDFs while holding a wrench.

**Business case:** Field service calls where technicians need remote expert support cost $150-300 per call. A voice assistant that answers 60% of those questions from documentation saves six figures annually for mid-size operations.

**Azure services:**
- Microsoft Foundry
- GPT-4o Realtime (speech-in, speech-out with low latency)
- Azure AI Search (equipment manual index with text-embedding-3-large)
- Azure Container Apps (backend API)
- Azure App Service (mobile-optimized frontend)
- Azure Blob Storage (manual PDFs + images)

**What you'll learn:** Realtime audio APIs, voice UX design, mobile-first architecture. Good conversation starter for manufacturing, energy, and logistics customers.

---

## Idea 9: Competitive Intelligence Dashboard

**Type:** Co-build template  
**Impact:** ⭐⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐⭐ | **Time to value:** 3-4 weeks

**What it is:** An agent that continuously monitors competitor websites, news feeds, SEC filings, and social media. It summarizes changes, identifies trends, and updates a dashboard. Marketing and strategy teams get a daily digest instead of manually scanning 15 sources.

The agent uses Bing Web Search grounding for real-time data and Code Interpreter to generate charts and trend analysis.

**Business case:** Strategy teams spend 15-20 hours per week on manual competitive monitoring. Automating collection and initial analysis frees them to focus on interpretation and response.

**Azure services:**
- Microsoft Foundry (prompt agent with Web Search + Code Interpreter)
- GPT-5.4 (strong reasoning for trend synthesis across disparate sources)
- Foundry Agent Service (scheduled runs via API)
- Azure Cosmos DB (competitor intelligence history)
- Azure Container Apps (dashboard API)
- Azure Static Web Apps (dashboard frontend)
- Azure Functions (scheduled trigger for daily/weekly runs)

**What you'll learn:** Web grounding, scheduled agent execution, building reporting layers on top of agent outputs.

---

## Idea 10: Agentic Workflow for Approvals

**Type:** Co-build template  
**Impact:** ⭐⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐⭐ | **Time to value:** 4-6 weeks

**What it is:** A workflow agent that handles multi-step business approvals — purchase requests, access provisioning, change management tickets. The agent reads the request, checks it against policies (using Foundry IQ), routes to the right approver, follows up on stale approvals, and logs everything. Human-in-the-loop at the approval step — the agent prepares, the human decides.

This is the project that teaches you why workflow agents exist. Some things shouldn't be fully autonomous.

**Business case:** Approval bottlenecks slow down procurement by an average of 3-5 days. Automated preparation, routing, and follow-up cuts cycle time by 50-70%.

**Azure services:**
- Microsoft Foundry (workflow agent with branching logic and human-in-the-loop)
- Foundry Agent Service (hosting + state management)
- GPT-4.1-mini (fast policy checking)
- Foundry IQ (policy knowledge base)
- Azure Service Bus (request ingestion)
- Azure Cosmos DB (approval state, audit trail)
- Microsoft Graph API (Teams notifications for approvers)
- Microsoft Entra ID (identity and RBAC)

**What you'll learn:** Workflow agent design, human-in-the-loop patterns, state management across long-running processes, Microsoft Graph integration. This is enterprise AI — messy, stateful, and high-value.

---

## Idea 11: Computer-Use Process Automator

**Type:** Demo asset  
**Impact:** ⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐⭐ | **Time to value:** 4-6 weeks

**What it is:** GPT-5.4 supports computer use — the model can see a screen, reason about UI elements, and take actions (click, type, scroll). Build an agent that automates a repetitive process in a legacy web application that has no API. Think: filling out forms in an old ERP system, exporting reports from a vendor portal, or data entry across two systems that don't talk to each other.

**Why this turns heads:** Every enterprise has at least one terrible legacy app that someone spends hours clicking through daily. Showing a model that can see the screen and operate it? That's an "I didn't know AI could do that" moment.

**Business case:** Manual data entry between disconnected systems costs 2-4 FTEs for mid-size companies. Even 50% automation of the simplest workflows saves $100K+ annually.

**Azure services:**
- Microsoft Foundry
- GPT-5.4 with computer-use capability (via Responses API)
- Azure Container Apps (hosted agent runtime with browser automation)
- Microsoft Agent Framework (code-level control over the computer-use loop)
- Azure Key Vault (credentials for legacy system access)

**What you'll learn:** Computer-use API, Responses API (the new agent API replacing Assistants), building safety guardrails around autonomous UI interaction. Important: always demo with human-in-the-loop confirmation before actions.

---

## Idea 12: Multi-Modal Quality Inspector

**Type:** Co-build template  
**Impact:** ⭐⭐⭐⭐⭐ | **Difficulty:** ⭐⭐⭐⭐⭐ | **Time to value:** 6-8 weeks

**What it is:** A production line quality inspection system. Cameras capture product images, the agent analyzes them for defects using GPT-5.4 vision capabilities, cross-references against a defect catalog in Azure AI Search, and triggers alerts or line stops based on severity. Add Code Interpreter for statistical process control charts.

This is the most complex build on the list — but it's also the highest-value demo for manufacturing customers.

**Business case:** Defective products that reach customers cost 5-10x more to address than catching them on the line. Automated visual inspection reduces defect escape rates by 30-50%.

**Azure services:**
- Microsoft Foundry (hosted agent with Microsoft Agent Framework)
- GPT-5.4 (vision + reasoning for defect classification)
- Azure AI Search (defect catalog with image embeddings)
- Azure IoT Hub (camera feed ingestion)
- Azure Container Apps (agent hosting)
- Azure Cosmos DB (inspection history and SPC data)
- Azure Event Hubs (real-time event streaming from production line)
- Azure Monitor + Application Insights (agent observability)

**What you'll learn:** Multi-modal input handling, real-time processing architectures, IoT integration, hosted agents in Foundry. This is a portfolio piece.

---

## Phased Roadmap

### Phase 1: First 90 Days — Build Your Foundation

**Goal:** Get hands-on with Foundry, deploy your first agents, have demos ready for customer meetings.

| Week | Build | Why |
|------|-------|-----|
| 1 | **Idea 1: Ask My Docs** | Foundry basics + RAG in a day. You'll reference this in every customer conversation. |
| 2 | **Idea 2: Meeting Minutes Agent** | Audio models + prompt agents. Universally relatable demo. |
| 3-4 | **Idea 4: Internal Policy Chatbot** | Foundry IQ (turnkey RAG). First co-build template you can adapt to any customer with internal docs. |
| 5-6 | **Idea 3: Customer Email Triage** | Function calling + multi-model patterns. Strong cross-industry co-build. |
| 7-10 | **Idea 6: Multi-Agent Incident Responder** | Your "wow" demo. Multi-agent orchestration, workflow agents. The project where everything clicks. |
| 11-12 | Polish + customize | Take your best demos, swap in customer-relevant data, rehearse the storytelling. |

**You should have by day 90:** 4-5 working demos on your laptop, comfort with Foundry Agent Service, and the ability to spin up a customer POC in a week.

### Phase 2: Months 3-9 — Go Deeper, Co-Build with Customers

**Goal:** Run your first customer POC engagements. Tackle harder problems.

Pick 2-3 from this list based on the customers you're working with:

- **Idea 5: Contract Clause Analyzer** — if you have legal/procurement customers
- **Idea 7: Data Pipeline QA Agent** — if you have data platform customers
- **Idea 8: Voice-Enabled Field Assistant** — if you have manufacturing/field service customers
- **Idea 9: Competitive Intelligence Dashboard** — if you have marketing/strategy conversations

Each teaches you a new skill: long-context reasoning, event-driven agents, voice APIs, or web grounding. Pick based on where your customer conversations are going.

### Phase 3: Months 9-18 — Strategic Builds

**Goal:** Lead complex engagements. Build portfolio pieces that differentiate you.

- **Idea 10: Agentic Workflow for Approvals** — enterprise workflow automation with human-in-the-loop
- **Idea 11: Computer-Use Process Automator** — bleeding edge, high wow factor
- **Idea 12: Multi-Modal Quality Inspector** — deep industry play, complex architecture

These are the projects that turn you from "the new CSA" into "the person I call when we're doing something hard with AI."

---

## Common Architecture Patterns

All 12 ideas share a few patterns worth internalizing:

| Pattern | When to Use | Reference Ideas |
|---------|-------------|-----------------|
| **Single prompt agent** | Simple Q&A, classification, summarization | 1, 2, 4 |
| **Agent + Function Calling** | Agent needs to read/write external systems | 3, 5, 7 |
| **Workflow agent** | Multi-step processes with branching or human approval | 6, 10 |
| **Hosted agent** | Custom orchestration logic, framework control | 11, 12 |
| **Foundry IQ** | Quick RAG without building a pipeline | 4, 10 |
| **Azure AI Search + custom embeddings** | When you need control over chunking, hybrid search | 1 (stretch), 8, 9, 12 |
| **Multi-model routing** | Cheap model for simple tasks, expensive model for hard ones | 3, 7 |

---

## Model Selection Cheat Sheet

| Task | Model | Why |
|------|-------|-----|
| Classification, routing, simple extraction | GPT-4.1-mini | Fast, cheap, accurate enough |
| General Q&A and summarization | GPT-4.1-mini or GPT-5.4-mini | Balance of quality and cost |
| Long documents (contracts, reports) | GPT-5.4 (1M token context) | Fits entire documents without chunking |
| Complex reasoning and synthesis | GPT-5.4 or GPT-5.5 | When quality matters more than latency |
| Code generation and debugging | codex-mini or GPT-5.3-codex | Optimized for code tasks |
| Voice interaction | GPT-4o Realtime | Low-latency speech-in/speech-out |
| Vision / image analysis | GPT-5.4 | Strong multimodal reasoning |
| Embeddings | text-embedding-3-large | Best retrieval quality; use text-embedding-3-small if budget is tight |

---

## One More Thing

The best demo is the one you've actually built and broken and fixed. Slides about RAG don't teach you what happens when chunking goes wrong. A contract analyzer that hallucinates clause numbers teaches you more about grounding than any training course.

Build the first three ideas this month. Not next month. This month.

---

*Generated for Microsoft CSU Cloud & AI new CSA onboarding — May 2026*  
*All services and models verified against current Microsoft Foundry documentation*
