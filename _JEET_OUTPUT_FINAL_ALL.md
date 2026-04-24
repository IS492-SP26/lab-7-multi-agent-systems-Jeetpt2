# Lab 7 — Multi-Agent Systems: Exercises Completed

**Student:** Jeet Thakore
**Date:** April 24, 2026  
**Frameworks:** AutoGen (Microsoft) vs CrewAI  
**Model:** Groq `llama-3.3-70b-versatile` (free tier)

---

## Setup & Validation

Before running any exercise, configuration was validated:

```bash
python shared_config.py
```

Output confirmed Groq API key loaded, model `llama-3.3-70b-versatile`, endpoint `https://api.groq.com/openai/v1`.

Both frameworks were verified installed:

```bash
# autogen: 0.2.35
# crewai: 1.14.2
```

---

## Exercise 1 — Run and Understand

### AutoGen Demo

**Command:**
```bash
python autogen/autogen_simple_demo.py
```

**What happened:**
- 5 agents placed in a `GroupChat`: ProductManager (UserProxyAgent) + ResearchAgent, AnalysisAgent, BlueprintAgent, ReviewerAgent (all AssistantAgents).
- ProductManager sent the initial message to the `GroupChatManager`.
- `GroupChatManager` used LLM-based `speaker_selection_method="auto"` to pick the next speaker each round — it reads each agent's `description` field and the conversation history to decide.
- Conversation ran for **5 rounds** in logical order: ProductManager → ResearchAgent → AnalysisAgent → BlueprintAgent → ReviewerAgent.
- ReviewerAgent ended with `TERMINATE`, stopping the chat.
- An LLM-generated executive summary was produced via `summary_method="reflection_with_llm"`.
- Full transcript saved to `autogen/groupchat_output_20260424_110611.txt`.

**Key observation:** Agents saw the full conversation history and each explicitly referenced prior agents' output (e.g., AnalysisAgent said "based on ResearchAgent's findings…"). Speaker order **emerged from LLM reasoning**, not hardcoded rules.

---

### CrewAI Demo

**Command:**
```bash
python crewai/crewai_demo.py
```

**What happened:**
- 4 agents created: FlightAgent, HotelAgent, ItineraryAgent, BudgetAgent — each with a `role`, `goal`, `backstory`, and a dedicated `@tool` function.
- 4 tasks created, one per agent, each with a `description` and `expected_output`.
- `Crew(process="sequential")` executed tasks in fixed order: Flight → Hotel → Itinerary → Budget.
- Each agent used its tool (e.g., `search_flight_prices`) to fetch static data, then the LLM composed a structured response.
- Each task's output automatically became context for the next task.
- Final budget report produced with itemized costs at budget/mid-range/luxury levels.
- Output saved to `crewai/crewai_output_iceland.txt`.

**Key observation:** Agents worked independently on isolated tasks. They did not converse — each only saw the prior task's output as context, not a shared chat.

---

### Comparison Observed

| Aspect | AutoGen | CrewAI |
|--------|---------|--------|
| Agent awareness | Full conversation history | Only previous task output |
| Speaker order | LLM-selected (emergent) | Fixed sequential |
| Interaction style | Conversational back-and-forth | Assembly-line pipeline |
| Termination | Agent says TERMINATE | All tasks complete |

---

## Exercise 2 — Modify Agent Behavior

### AutoGen — Changed ResearchAgent Domain

**File edited:** `autogen/autogen_simple_demo.py`

**Change 1 — ResearchAgent system_message** (lines 59–70):
- **Before:** Focused on AI interview platforms (HireVue, Pymetrics, Codility, Interviewing.io)
- **After:** Focused on **AI-powered employee onboarding tools** (Deel, Rippling, BambooHR, Workday)

```python
# Before
system_message="""You are a market research analyst specializing in AI-powered recruitment technology.
...
- Analyze 3-4 major competitors in AI interview platforms (HireVue, Pymetrics, Codility, Interviewing.io)
"""

# After
system_message="""You are a market research analyst specializing in AI-powered HR technology.
...
- Analyze 3-4 major competitors in AI-powered employee onboarding tools (Deel, Rippling, BambooHR, Workday)
"""
```

**Change 2 — Initial message** updated to match the new domain (onboarding platform).

**Result after re-running:**
- ResearchAgent analyzed Deel, Rippling, BambooHR, Workday — identified gaps: limited remote/hybrid support, no Slack/Teams integration, no sentiment analysis.
- AnalysisAgent (unchanged) automatically pivoted to onboarding-specific opportunities.
- BlueprintAgent (unchanged) designed "OnboardGenie" with features like Virtual Onboarding Assistant and Employee Feedback Mechanism.
- ReviewerAgent (unchanged) gave onboarding-specific strategic recommendations.
- **Speaker order remained identical** (LLM still selected the same logical sequence).

**Answer to lab question:** One agent's changed persona fully steered all downstream content without touching other agents. The GroupChatManager selected speakers in the same order because the logical flow (research → analysis → blueprint → review) was still the most coherent path.

---

### CrewAI — Added Budget Constraint to FlightAgent

**File edited:** `crewai/crewai_demo.py`

**Change — `create_flight_agent()` backstory** (line ~249):

```python
# Added to backstory:
"You focus on budget airlines and cost savings above all — you always "
"recommend the cheapest viable option and explicitly warn travelers "
"about any hidden fees (baggage, seat selection) so the budget agent "
"can produce the most accurate total cost estimate."
```

**Result after re-running:**
- FlightAgent recommended PLAY Airlines ($349) as primary pick over Icelandair ($485), explicitly flagged "no checked bags included" and seat selection fees.
- BudgetAgent's final output used $349 as the budget flight baseline and included a line item for hidden fees.

**Answer to lab question:** The constraint propagated through the sequential pipeline via context. The BudgetAgent received FlightAgent's output as input and incorporated the budget-first framing into its estimates.

---

## Exercise 3 — Add a Fifth Agent

### AutoGen — Added CostAnalyst

**File edited:** `autogen/autogen_simple_demo.py`

**Step 1 — Created CostAnalyst agent** in `_create_agents()` (inserted before ReviewerAgent):

```python
self.cost_agent = autogen.AssistantAgent(
    name="CostAnalyst",
    system_message="""You are a financial analyst specializing in SaaS product development costs.
Your role is to ESTIMATE costs after the BlueprintAgent presents features.
- Estimate development costs and timeline for each feature
- Provide a rough cost breakdown: engineering hours, infrastructure, third-party tools
- Rank features by cost-benefit ratio
- Estimate time-to-market for MVP
After your analysis, invite the ReviewerAgent...""",
    llm_config=self.llm_config,
    description="Financial analyst who estimates development costs and prioritizes features by cost-benefit ratio.",
)
```

**Step 2 — Updated BlueprintAgent's system_message** to invite CostAnalyst (not ReviewerAgent) after presenting features. This was critical — the first run skipped CostAnalyst because BlueprintAgent still said "invite the ReviewerAgent", so the LLM's speaker selection followed that cue.

**Step 3 — Added CostAnalyst to GroupChat agents list** and increased `max_round` from 8 to 10.

**Result:**
- 6 rounds, all 6 agents spoke in correct order.
- CostAnalyst estimated total dev cost of **$630,000** across 5 features, ranked Virtual Mentorship Program and AI-Driven Feedback Mechanism as highest cost-benefit.
- ReviewerAgent explicitly cited the cost data: "prioritize the Virtual Mentorship Program and AI-Driven Feedback Mechanism based on the CostAnalyst's estimates."

**Key learning:** When adding a new agent to AutoGen, you must also update the *preceding* agent's system_message to invite the new agent — otherwise the LLM's speaker selection will follow the existing invitation cue and skip the new agent.

---

### CrewAI — Added LocalExpert Agent

**File edited:** `crewai/crewai_demo.py`

**Step 1 — Created `create_local_expert_agent()` function:**

```python
def create_local_expert_agent(destination: str):
    return Agent(
        role="Local Expert",
        goal=f"Provide insider knowledge about {destination}: local customs, safety tips, "
             f"cultural etiquette, hidden gems, and money-saving local secrets...",
        backstory="You are a seasoned traveler and cultural expert who has lived in "
                  f"{destination} for years. You know the unwritten rules, the safest "
                  "neighbourhoods, the local scams to avoid...",
        tools=[search_travel_costs],
        verbose=True,
        allow_delegation=False
    )
```

**Step 2 — Created `create_local_expert_task()` function:**
- Placed between itinerary and budget tasks.
- `expected_output` explicitly stated the budget agent should incorporate the tips.

**Step 3 — Updated `main()` function:**
- Added `[4/5]` agent creation step.
- Added `local_expert_task` to tasks list between `itinerary_task` and `budget_task`.
- Updated `Crew(agents=[...], tasks=[...])` to include both.
- Updated print statement: `FlightAgent → HotelAgent → ItineraryAgent → LocalExpert → BudgetAgent`.

**Result:**
- 5-agent pipeline ran successfully.
- LocalExpert produced tips on: Icelandic tipping customs, Flybus vs taxi, free tap water, Hallgrimskirkja free entry, common tourist scams.
- BudgetAgent's final output referenced public transport savings and free attractions in its cost-saving tips section.

---

## Exercise 4 — Custom Domain

### New File Created: `autogen/exercise4_custom_domain.py`

**Domain:** Cloud-native B2B SaaS architecture planning (project management tool, Linear/Jira competitor).

**Problem statement given to agents:**
> Design the architecture for a multi-tenant project management tool targeting engineering teams. Scale: 500 customers, 50,000 users, 1,000 req/sec peak. Must be GDPR-compliant, ship MVP in 6 months, $15k/month infra budget.

**6 agents defined (new roles, different from original demo):**

| Agent | Type | Role |
|-------|------|------|
| CTOProxy | UserProxyAgent | Initiates discussion, sets constraints |
| RequirementsAgent | AssistantAgent | Defines functional + non-functional requirements |
| ArchitectAgent | AssistantAgent | Proposes cloud-native architecture and tech stack |
| SecurityAgent | AssistantAgent | Reviews architecture for security/compliance risks |
| CostAnalyst | AssistantAgent | Estimates AWS cloud spend at launch and 10× scale |
| ReviewerAgent | AssistantAgent | Makes final architecture decision + implementation roadmap |

**GroupChat settings:** `max_round=10`, `speaker_selection_method="auto"`, `allow_repeat_speaker=False`

**Run command:**
```bash
python autogen/exercise4_custom_domain.py
```

**Results — 6 rounds, all agents spoke:**

**RequirementsAgent** defined:
- Functional: multi-tenancy, REST API, real-time events, GitHub/Jira integration, customizable dashboards
- Non-functional: 99.9% uptime, <200ms p95 latency, GDPR compliance, horizontal scalability
- Scale: 50k users → 200k within 12 months; integrations: Auth0, Stripe, SendGrid

**ArchitectAgent** proposed:
- **Microservices + serverless** on AWS (Lambda + EKS)
- Data tier: Aurora PostgreSQL (primary), Redis (cache), OpenSearch (search), SQS (message queue)
- **Schema-per-tenant** for multi-tenancy (GDPR-friendly data isolation)
- End-to-end request flow: API Gateway → Lambda auth → Aurora → SQS → real-time update → response

**SecurityAgent** identified 4 risks:
1. Insufficient auth/authorization — mitigate with strict IAM policies and token validation
2. Insecure API endpoints — mitigate with TLS, input validation, WAF
3. Unencrypted data storage — mitigate with Aurora encryption at rest + KMS
4. Inadequate logging — mitigate with CloudWatch + ELK Stack
- Recommended zero-trust model: micro-segmentation + identity-based access control

**CostAnalyst** estimated:
- **Launch cost: ~$1,447/month** (well within $15k budget)
- **10× scale cost: ~$6,408/month**
- Top cost drivers: Kubernetes/compute (40%), Aurora DB (25%), Redis+OpenSearch (15%)
- Optimisations: reserved instances (−75%), spot instances (−90%), tiered S3 storage (−50%)

**ReviewerAgent** decided:
- **Architecture accepted** with cost optimisations applied
- Roadmap:
  - Phase 1 (Weeks 1–4): MVP infrastructure — Docker, EKS, API Gateway, Aurora
  - Phase 2 (Weeks 5–12): Security hardening — encryption, CloudWatch, pen testing
  - Phase 3 (Post-launch): Horizontal scaling, CDN, DB optimisation
- Pre-launch blockers: auth security + API data exposure

**Output saved to:** `autogen/exercise4_output_20260424_113551.txt`

---

## Notes on Rate Limits

The Groq free tier has a **100,000 tokens/day** limit per model. After completing Exercises 1–3, the `llama-3.3-70b-versatile` daily limit was exhausted. Exercise 4 was run on `meta-llama/llama-4-scout-17b-16e-instruct` (Llama 4 Scout), which has a separate daily limit. The `.env` was restored to `llama-3.3-70b-versatile` after Exercise 4 completed.

Models tried during Exercise 4 troubleshooting:
- `llama-3.1-8b-instant` — available but 6,000 TPM limit caused mid-run failures
- `gemma2-9b-it` — decommissioned by Groq
- `meta-llama/llama-4-scout-17b-16e-instruct` — **worked successfully**

---

## Files Modified / Created

| File | Change |
|------|--------|
| `autogen/autogen_simple_demo.py` | Exercise 2: new ResearchAgent domain; Exercise 3: added CostAnalyst agent |
| `crewai/crewai_demo.py` | Exercise 2: budget constraint on FlightAgent; Exercise 3: added LocalExpert agent + task |
| `autogen/exercise4_custom_domain.py` | **New file** — Exercise 4 custom domain demo |
| `.env` | Temporarily changed `GROQ_MODEL` during Exercise 4 (restored after) |

## Output Files Generated

| File | Contents |
|------|----------|
| `autogen/groupchat_output_20260424_110611.txt` | Exercise 1 AutoGen run |
| `autogen/groupchat_output_20260424_110812.txt` | Exercise 2 AutoGen run (onboarding domain) |
| `autogen/groupchat_output_20260424_111146.txt` | Exercise 3 first attempt (CostAnalyst skipped) |
| `autogen/groupchat_output_20260424_111254.txt` | Exercise 3 final run (CostAnalyst participated) |
| `autogen/exercise4_output_20260424_113551.txt` | Exercise 4 custom domain run |
| `crewai/crewai_output_iceland.txt` | CrewAI runs (overwritten each run — final = Exercise 3) |
