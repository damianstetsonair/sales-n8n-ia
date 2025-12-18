# CLAUDE.md - Sales-IA Project

## Project Overview

AI-powered sales orchestration system using multi-agent architecture on n8n workflow platform. Analyzes CRM contacts and generates personalized outreach recommendations using Claude AI.

## Tech Stack

- **Platform**: n8n (workflow automation)
- **AI**: Claude API (Anthropic)
- **Language**: JSON (config) + JavaScript (validation) + Markdown (prompts)
- **CRM Integration**: HubSpot-compatible data structure

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
        ├── Context & Relationship Analyzer/  # Agent 1
        ├── Opportunity Detector/             # Agent 2
        ├── State Analyzer/                   # Agent 3
        ├── Timing Strategist/                # Agent 4
        ├── Channel Selector/                 # Agent 5
        ├── Content Generator/                # Agent 6 (most complex)
        └── Sequence Strategist/              # Agent 7
```

## How to Run

1. Import `n8n-workflow.json` into n8n
2. Configure Claude/Anthropic API credentials
3. Trigger webhook at `/multi-agent-orchestrator` with contact data

```bash
curl -X POST https://your-n8n/webhook/multi-agent-orchestrator \
  -H "Content-Type: application/json" \
  -d @webhook-input-contact-example.json
```

## Architecture: 3-Wave Pipeline

```
WAVE 1: ANALYSIS (Parallel)
├── Agent 1: Context & Relationship Analyzer
├── Agent 2: Opportunity Detector
├── Agent 3: State Analyzer
└── Agent 4: Timing Strategist
    ↓ Distribution Orchestrator

WAVE 2: EXECUTION (Sequential)
├── Agent 5: Channel Selector
├── Agent 6: Content Generator
└── Agent 7: Sequence Strategist
    ↓ Evaluation Orchestrator

WAVE 3: SYNTHESIS
└── Final recommendation with channel, content, timing
```

## Key Concepts

### Deal Classification
- `DEAL_WON_ACTIVE` → Client actif
- `DEAL_WON_DORMANT` → Client dormant
- `DEAL_NURTURING` → À réactiver (lost deals)
- `DEAL_OPEN` → Prospect
- `NO_DEAL` → Lead

### Channel States
`awaiting_response`, `response_due`, `active_conversation`, `stale`, `dormant`, `closed`, `never_contacted`, `unresponsive`

### Hook Age Categories
- fresh (0-14 days): Direct reference OK
- recent (15-60 days): Direct reference OK
- old (61-120 days): Acknowledge time gap
- stale (120+ days): New angle needed

### Ghosting Thresholds
Count outbound messages after last inbound:
- 0-2: OK to continue
- 3: Caution
- 4+: Break-up message only

## Anti-Hallucination Protocol (Mandatory)

Every agent follows these rules:
1. Source of truth = JSON input only (no invented facts)
2. Evidence trail required (reference exact JSON paths)
3. When in doubt, leave it out

## Input/Output Format

### Input (Webhook) - Detailed Schema

```json
{
  "contact_info": {
    "hubspot_id": "22218731286",
    "full_name": "marie anne_castro",
    "name": "marie anne_castro",
    "job": "Chief Transformation Officer",
    "job_strategic_role": null,
    "phone": "+33 6 60 71 08 10",
    "location": "Lesquin, Hauts-de-France, France",
    "company": "KIABI",
    "company_hubspot_id": 30103695002,
    "linkedin_url": "https://www.linkedin.com/in/...",
    "linkedin_urn": "ACoAAA...",
    "is_ia_agent_activated": true,
    "owner": {
      "owner_hubspot_id": "48901672",
      "owner_linkedin_profile_url": "https://www.linkedin.com/in/bertranruiz",
      "owner_linkedin_urn": "ACoAABKCK8QB...",
      "owner_name": "bertran_ruiz"
    }
  },
  "activities": [
    {
      "activity_id": "87602226719",
      "activity_type": "EMAIL|CALL|MEETING|NOTE|LINKEDIN MESSAGE|LINKEDIN REACTION|LINKEDIN CONNECT|LINKEDIN FOLLOW PAGE",
      "recorded_on": "2025-08-29T17:04:15.109+00:00",
      "direction": "INBOUND|OUTBOUND|null",
      "metadata": {
        "body": "Message content...",
        "thread": "c821429cf99af141167a933b41cd6041",
        "title": "Re: Subject line",
        "meeting_internal_note": null
      },
      "owner": {
        "owner_hubspot_id": "48901672",
        "owner_linkedin_profile_url": "...",
        "owner_linkedin_urn": "...",
        "owner_name": "bertran_ruiz"
      },
      "sender": {
        "company_name": "airsaas|Kiabi",
        "company_hubspot_id": 30103695002,
        "sender_hubspot_id": "48901672",
        "sender_linkedin_url": "...",
        "sender_linkedin_urn": "...",
        "sender_name": "bertran_ruiz",
        "sender_job": "Chief Transformation Officer",
        "sender_job_strategic_role": "PMO"
      },
      "deals": [
        {
          "deal_hubspot_id": "43098977429",
          "deal_name": "Kiabi Upsell 30",
          "dealstage": null,
          "deal_closed_date": "2025-09-19 08:45:52.593 Z",
          "deal_created_date": "2025-09-01 08:09:32.681 Z",
          "deal_owner_name": "bertran_ruiz",
          "deal_owner_hubspot_id": "48901672",
          "is_deal_ia_agent_activated": "false"
        }
      ]
    }
  ],
  "stats": {
    "total_activities": 155,
    "by_type": {
      "EMAIL": 121,
      "LINKEDIN REACTION": 1,
      "NOTE": 6,
      "MEETING": 17,
      "CALL": 5,
      "LINKEDIN CONNECT": 1,
      "LINKEDIN MESSAGE": 3,
      "LINKEDIN FOLLOW PAGE": 1
    }
  }
}
```

### Activity Types
- `EMAIL` - Email exchanges (INBOUND/OUTBOUND)
- `CALL` - Phone calls
- `MEETING` - Scheduled meetings
- `NOTE` - Internal CRM notes (conversation summaries, etc.)
- `LINKEDIN MESSAGE` - Direct LinkedIn messages
- `LINKEDIN REACTION` - Reactions to LinkedIn posts
- `LINKEDIN CONNECT` - Connection requests
- `LINKEDIN FOLLOW PAGE` - Company page follows

### Output (Recommendation)
```json
{
  "action": "send_message|wait|no_action|break_up",
  "selected_channel": "EMAIL|LINKEDIN|PHONE|WHATSAPP",
  "selected_content": { "message", "tone" },
  "execute_at": "ISO8601 datetime",
  "sequence_plan": [...],
  "confidence_factors": {...}
}
```

## Development Guidelines

### Editing Prompts
- System prompts define agent behavior (500-800 lines each)
- User messages define input templates
- Agent 6 (Content Generator) is most complex - edit carefully
- Maintain anti-hallucination rules in all prompts

### Testing
Use `webhook-input-contact-example.json` as test template with example contact data.

### Language Support
System is bilingual (French/English) - agents detect and adapt to contact's language.

## Critical Rules

- Signatures MUST match deal owner
- Content from Agent 6 used exactly (no rewrites)
- All timestamps in ISO8601 with timezone
- All confidence scores required
- Evidence trail for every claim
