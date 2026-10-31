# Blog-writer Agent Optimised

An AI agent that turns an idea into a finished social post. It researches the web, drafts in your brand voice, generates an image, **waits for your approval**, and then publishes through Buffer or LinkedIn. It is the running project for the GDG London **Agentic DevCamp**: you build it in Lab 1, run it for real in Lab 2, lock it down in Lab 3, and optimise it in Lab 4.

> **Start here:** [Setup guide](docs/setup-guide.html) (do this before Saturday) → [Lab 1: Build](docs/lab-1-build.html). All docs are listed in [docs/index.html](docs/index.html).
>
> This repo is the finished reference, not step 1. In Lab 1 you build a smaller version of it yourself. Use this repo when you get stuck or want to see where the labs end up.

## How it works

<img src="docs/img/architecture-build.png" alt="The social_poster orchestrator takes your idea, uses skills, an image tool and a research sub-agent, shows a draft, and only publishes through Buffer or LinkedIn after you approve" width="800">

You give the **orchestrator** (`social_poster`) an idea. It loads **Agent Skills** for your brand voice, calls a **research sub-agent** for facts and an **image tool** for a picture, then shows you the draft. Nothing is published until you press approve. The approval gate is enforced by the framework, not by a polite instruction the model could ignore.

## Tech stack

| Layer | What we use | Why |
|---|---|---|
| Agent framework | [Google ADK](https://google.github.io/adk-docs/) (Python) | Agents, tools, sub-agents, sessions, evals |
| Models | Gemini (`gemini-2.5-flash`, Gemini image model) | Reasoning, drafting, image generation |
| Agent knowledge | Agent Skills (`skills/`) | Brand voice as a reviewable markdown folder |
| Publishing | MCP servers: Buffer (remote), LinkedIn (optional, stdio) | Real posts behind a human approval gate |
| Agent-to-agent | A2A (`memory_agent`, optional) | A separate memory service the orchestrator can ask |
| Backend | FastAPI + [`ag-ui-adk`](https://github.com/ag-ui-protocol/ag-ui) | Serves the agent to the UI over AG-UI at `/api/adk` |
| Frontend | Next.js 16, React 19, [CopilotKit](https://copilotkit.ai) | Chat, draft preview and the approve button |
| Data | SQLite (local), Cloud Storage (optional images) | Post history and hosted images |
| Cloud | Google Cloud: Agent Runtime, Cloud Run, Cloud Build, Secret Manager | Lab 2 deploy and CI/CD |
| Governance | Model Armor, Cloud DLP, Agent Gateway, Agent Registry | Lab 3 |
| Tooling | `uv`, `agents-cli`, `adk eval`, `gcloud` | Install, deploy, quality gate |

## What you build, lab by lab

| Session | Lab | You add | Tag to check out if you fall behind |
|---|---|---|---|
| Setup | [Setup guide](docs/setup-guide.html) | Google Cloud project, tools, coding agent | `starter` |
| 1. Build | [Lab 1](docs/lab-1-build.html) | The agent, skills, tools, approval gate, web UI | `pillar-build` |
| 2. Scale | [Lab 2](docs/lab-2-scale.html) | Deploy to Agent Runtime, managed sessions and Memory Bank | `pillar-scale` |
| 3. Govern | [Lab 3](docs/lab-3-govern.html) | Model Armor, Cloud DLP, gateway, registry, guardrail evals | `pillar-govern` |
| 4. Optimise | [Lab 4](docs/lab-4-optimize.html) | Golden evals, simulated users, BigQuery analytics, eval-gated deploys | `pillar-optimize` |

```bash
git checkout pillar-build   # for example: the finished result of Lab 1
```

<img src="docs/img/architecture-scale.png" alt="Scale: the web UI calls the agent on Agent Runtime, which keeps conversation state in Sessions and long-term facts in Memory Bank" width="800">

## Production Harness & Optimisation Notes

> **Key takeaway:** *Building an agent is easy; harnessing it for production is the hard part.*
>
> Notes from the **Optimise workshop with Renuka Kelkar** and **programme wrap notes from Sumith Damodaran**.
>
> 📅 **Follow-up session:** **Thursday 22 October (18:00–19:00)** — Optimisation and Q&A.

### The Production Harness

The production harness sits between the user and the agent/tools, handling four essential responsibilities:

1. **Validate In**: Input validation, prompt-injection defense, and ingress safety checks before the model ever sees the prompt.
2. **Track**: Count tokens, log tool calls, monitor latency, and bound retry loops.
3. **Validate Out**: Output validation, DLP masking, schema enforcement, and egress guards before results return to the user.
4. **Report**: Emit structured telemetry, audit trails, and evaluations to BigQuery and Cloud Trace.

### Architecture Demo: The Blog-Writer Agent

Alongside Social Spark, the **blog-writer agent** (root orchestrator, search sub-agent, image sub-agent) illustrates enterprise governance and boundary enforcement:
- **Model Armor templates in `us-central1`**: Screening user input and model completions against hate speech, harassment, and prompt injection.
- **Ingress / egress guards**: Validating prompts on entry and generated drafts on exit.
- **Cloud DLP for sensitive data**: Detecting and redacting PII (phone numbers, credentials, sensitive business data).
- **Agent Identity before Registry**: Registering authenticated service identities before publishing to Agent Registry.
- **Access vs. business policies**: Separating network/IAM access controls from high-level agent business rules.
- **Agent Gateway as the check-post**: Centralized inspection proxy enforcing policy rules across traffic.

### Preferred Cohort Agent Stack: Google ADK + Gemini

For production-grade cohort agents, prefer **Google ADK + Gemini** across all core primitives:
- **Agents**: Specialized, single-purpose roles.
- **Tools**: Deterministic interfaces for actions and integrations.
- **Sessions**: Ephemeral per-turn and conversation tracking.
- **State & Memory**: Durable context and memory persistence across conversations.
- **Artifacts**: Reviewable outputs (drafts, reports, code) stored outside prompt context.
- **Callbacks**: Hooks for lifecycle events, telemetry, and security interventions.

### Key Production & Optimisation Patterns

- **No giant agents**: Never build monolithic agents. Decompose systems into a root orchestrator coordinating focused sub-agents (e.g. search sub-agent, image sub-agent).
- **Don’t ask AI what code already knows**: Offload deterministic rules, brand guidelines, and constraints to **Skills** and configuration files rather than spending model tokens repeatedly asking the model to discover them.
- **AI decides / tools execute**: The model handles reasoning and orchestration; deterministic tools perform mutations, database writes, and external API calls.
- **Dry-run before expensive APIs**: Run dry-runs and validation gates prior to calling paid, rate-limited, or destructive third-party APIs.
- **Measure every run**: Instrument every turn with telemetry (BigQuery Agent Analytics, Cloud Trace) and guard code changes with automated golden eval sets.
- **Right-size models**: Use lightweight models (like Gemini Flash) for routing, intent detection, and validation checks; reserve heavier reasoning models only when complex generation is needed.
- **Shrink context**: Keep context windows lean by stripping conversational bloat and delegating retrieval to tools/sub-agents.
- **Structured JSON outputs**: Enforce strict schema constraints for tool arguments and structured data responses.

> 💡 **Case study: Cost reduction in practice**  
> In Renuka’s video-agent experiment, costs reached **~£100 in a single day** before applying these architectural principles. Introducing tool-based stitching, right-sized Flash routing, and context shrinking cut usage sharply down to **~492 tokens** per operation.

## Run it locally

You need [uv](https://docs.astral.sh/uv/), Node.js 20+, and a Google Cloud project (see the [setup guide](docs/setup-guide.html)). From the repo root:

```bash
cp .env.example backend/social_poster/.env   # set GOOGLE_CLOUD_PROJECT
uv sync
./run-backend.sh                             # terminal 1: http://localhost:8000
./run-frontend.sh                            # terminal 2: http://localhost:3000
```

Publishing is a dry run (`DRY_RUN=true`) until you switch it off, so nothing goes to a real account by accident. More detail: [backend/README.md](backend/README.md) and [frontend/README.md](frontend/README.md).

## What's where

```
backend/social_poster/   the agents, tools and guardrails (agent.py is the place to start)
backend/main.py          FastAPI app: AG-UI route, sessions and memory when deployed
skills/                  Agent Skills (brand voice)
frontend/                Next.js + CopilotKit UI
evals/                   adk eval sets (quality and governance)
docs/                    labs, setup guide, deploy and reference docs
gcp-setup.sh             one-time Google Cloud setup: APIs, templates, grants
```

## Deploy and reference docs

| Doc | Covers |
|---|---|
| [agent-runtime-deploy.md](docs/agent-runtime-deploy.md) | Deploy to Agent Runtime, sessions and memory, the frontend on Cloud Run |
| [cloud-run-deploy.md](docs/cloud-run-deploy.md) | Optional reference (not used in the labs): Dockerfile, `cloudbuild.yaml`, IAM, eval-gated CI/CD |
| [model-armor-setup.md](docs/model-armor-setup.md) | Guardrail templates, grants and switches |
| [observability.md](docs/observability.md) | Cloud Trace and the BigQuery analytics agent |
| [LEARNINGS.md](docs/LEARNINGS.md) | What broke and why, by pillar |

**What's deployed right now?** An Agent Runtime engine scales to zero when idle, and the console only shows one region at a time, so a deployed agent can look missing. Ask the CLI:

```bash
agents-cli deploy --list --project YOUR_PROJECT --no-confirm-project
```

Target and region come from `agents-cli-manifest.yaml` at the repo root.
