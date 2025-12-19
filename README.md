# Sales-IA

AI-powered sales orchestration system using multi-agent architecture on n8n workflow platform.

## Overview

Sales-IA analyzes CRM contacts and generates personalized outreach recommendations using Claude AI through a 3-wave multi-agent pipeline.

## Architecture

### High-Level Flow

```
INPUT (Webhook POST)
    → Validation + Constants
    → WAVE 1 (4 Analysis Agents)
    → Historical Data Fetch
    → WAVE 2 (3 Action Agents + Quality Check)
    → Retry Loop (max 2 iterations)
    → WAVE 3 (Synthesis)
    → Save to Supabase
    → Notify via Slack/Zapier
```

### 3-Wave Pipeline

```mermaid
flowchart TB
    subgraph WAVE1["WAVE 1: ANALYSIS (Parallel)"]
        direction TB
        OD["Orchestrator Distribution<br/>Claude Sonnet 4"]
        T1["Context Analyzer"]
        T2["Opportunity Detector"]
        T3["State Analyzer"]
        T4["Timing Strategist"]
        T1 & T2 & T3 & T4 -->|"ai_tool"| OD
    end

    subgraph WAVE2["WAVE 2: EXECUTION + QC"]
        direction TB
        OE["Orchestrator Evaluation<br/>Claude Opus 4.5"]
        T5["Channel Selector"]
        T6["Content Generator"]
        T7["Sequence Strategist"]
        T5 & T6 & T7 -->|"ai_tool"| OE
    end

    subgraph RETRY["RETRY LOGIC"]
        QC{"Quality Check<br/>Scores ≥ 0.6?"}
        PRC["Prepare Retry<br/>iteration++"]
    end

    subgraph WAVE3["WAVE 3: SYNTHESIS"]
        OS["Orchestrator Synthesis<br/>Claude Opus 4.5"]
    end

    WAVE1 --> WAVE2
    OE --> QC
    QC -->|"No + iteration < 2"| PRC
    PRC --> OD
    QC -->|"Yes OR iteration ≥ 2"| WAVE3
```

### Decision Flow

```mermaid
flowchart TB
    START((Start))

    CHECK_CONTENT{"Content Generated?"}
    CHECK_BLOCKERS{"Hard Blockers?<br/>• Meeting scheduled<br/>• Explicit wait<br/>• 5+ ghosting"}
    CHECK_GHOST{"Ghosting Level?"}
    CHECK_ENGAGE{"Engagement Signal?"}

    SEND["send_message"]
    SEND_LIGHT["send_message<br/>(lighter touch)"]
    SEND_NOW["send_message<br/>(IMMEDIATE)"]
    WAIT["wait"]
    BREAKUP["break_up"]

    START --> CHECK_CONTENT
    CHECK_CONTENT -->|"Yes"| CHECK_BLOCKERS
    CHECK_CONTENT -->|"No"| CHECK_GHOST

    CHECK_BLOCKERS -->|"Meeting"| WAIT
    CHECK_BLOCKERS -->|"Explicit wait"| WAIT
    CHECK_BLOCKERS -->|"5+ ghost"| BREAKUP
    CHECK_BLOCKERS -->|"None"| CHECK_GHOST

    CHECK_GHOST -->|"0-2"| CHECK_ENGAGE
    CHECK_GHOST -->|"3-4"| SEND_LIGHT
    CHECK_GHOST -->|"5+"| BREAKUP

    CHECK_ENGAGE -->|"Yes"| SEND_NOW
    CHECK_ENGAGE -->|"No"| SEND

    style SEND fill:#c8e6c9
    style SEND_LIGHT fill:#dcedc8
    style SEND_NOW fill:#a5d6a7
    style WAIT fill:#fff9c4
    style BREAKUP fill:#ffcdd2
```

## Tech Stack

| Component | Technology |
|-----------|------------|
| Platform | n8n (workflow automation) |
| AI Models | Claude Sonnet 4, Claude Opus 4.5 |
| Database | Supabase (PostgreSQL) |
| Notifications | Slack via Zapier |
| CRM | HubSpot-compatible structure |

## Project Structure

```
sales-ia/
├── n8n-workflow.json              # Main workflow (import into n8n)
├── webhook-input-contact-example.json  # Test payload template
└── prompts/                       # AI agent system prompts
    ├── Orchestrator Distribution/  # Wave 1 consolidation
    ├── Orchestrator Evaluation/    # Wave 2 quality control
    ├── Orchestrator Synthesis/     # Final decision maker
    └── Tools Agents/               # 7 specialized agents
```

## Agent Responsibilities

| Agent | Wave | Purpose |
|-------|------|---------|
| Context Analyzer | 1 | Relationship depth & trajectory |
| Opportunity Detector | 1 | Sales hooks & opportunities |
| State Analyzer | 1 | Conversation state & urgency |
| Timing Strategist | 1 | Optimal contact timing |
| Channel Selector | 2 | Best communication channel |
| Content Generator | 2 | Message creation |
| Sequence Strategist | 2 | Multi-touch planning |

## Quick Start

1. Import `n8n-workflow.json` into n8n
2. Configure Claude/Anthropic API credentials
3. Configure Supabase connection
4. Trigger webhook:

```bash
curl -X POST https://your-n8n/webhook/multi-agent-orchestrator \
  -H "Content-Type: application/json" \
  -d @webhook-input-contact-example.json
```

## Output Actions

| Action | When Used |
|--------|-----------|
| `send_message` | Content exists, no blockers (DEFAULT) |
| `wait` | Meeting scheduled or explicit request |
| `break_up` | 5+ outbounds without response |
| `no_action` | Contact opted out or invalid |

## Documentation

- **[CLAUDE.md](CLAUDE.md)** - Complete technical documentation and agent instructions
- **[WORKFLOW-DIAGRAM.md](WORKFLOW-DIAGRAM.md)** - Detailed workflow architecture with node references

## Execution Time

~7 minutes per contact (async processing with immediate webhook response)
