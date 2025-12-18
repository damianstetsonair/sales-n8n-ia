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

### Output (Recommendation) - Updated Schema

```json
{
  "final_decision": {
    "action": "send_message|wait|no_action|break_up",
    "selected_channel": "EMAIL|LINKEDIN|PHONE|WHATSAPP|null",
    "selected_content": { "message", "tone", "content_source" },
    "execution_timing": {
      "execute_at": "ISO8601 datetime (MANDATORY)",
      "reevaluate_at": "ISO8601 datetime (for wait decisions)",
      "timezone": "Europe/Paris"
    },
    "wait_type": {
      "type": "prospect_timing|meeting_scheduled|pilot_review|explicit_request|long_cycle|ghosting",
      "risk_level": "low|medium|high|critical",
      "trigger_event": "...",
      "reevaluation_trigger": "..."
    },
    "sequence_instructions": {
      "is_part_of_sequence": true|false,
      "sequence_plan": [
        { "step": 1, "trigger": "...", "channel": "...", "planned_date": "ISO8601" }
      ]
    },
    "rationale": {
      "opportunity_strength": 0.55,
      "opportunity_strength_breakdown": {
        "base_score": 0.50,
        "factors": [{"factor": "...", "impact": "+0.10", "evidence": "..."}],
        "calculated_total": 0.55
      },
      "confidence_factors": {...}
    }
  },
  "conditional_context": {
    "pilot_health": null,
    "stakeholder_map": null,
    "explicit_timing_agreement": null,
    "known_objections": null
  }
}
```

## Output Validation Rules (v2.0)

### INVALID Output If:
1. `action="wait"` AND `execute_at="Not specified"` with timing in reasoning
2. `action="wait"` AND `wait_type` absent
3. `opportunity_strength` without `opportunity_strength_breakdown`
4. Sale cycle >30 days AND `sequence_plan` empty
5. POC/Pilot deal AND `pilot_health` absent
6. `opportunity_strength` ≠ `calculated_total`

### Conditional Structures (include when triggered):
- `pilot_health`: When deal is POC/Pilot/Trial
- `stakeholder_map`: When >2 stakeholders mentioned
- `explicit_timing_agreement`: When recontact date was agreed
- `buying_committee`: When champion ≠ decision maker
- `known_objections`: When objection detected in data
- `competitive_context`: When competition mentioned
- `buying_process`: When cycle >3 months

## Agent Data Flow (v2.0)

### Wave 1 → Synthesis Data Mapping

| Agent | New Outputs | Used By Synthesis For |
|-------|-------------|----------------------|
| Agent 1 (Context) | `stakeholder_map` | `stakeholder_map`, `buying_committee` |
| Agent 2 (Opportunity) | `known_objections`, `competitive_context` | Direct passthrough |
| Agent 3 (State) | `explicit_timing_agreement`, `meeting_scheduled` | `explicit_timing_agreement`, `wait_type` |
| Agent 7 (Sequence) | Enhanced `touches[]` with `step`, `trigger`, `status` | `sequence_plan` |

### Wave 2 → Synthesis Data Mapping

| Agent | Outputs | Used By Synthesis For |
|-------|---------|----------------------|
| Agent 5 (Channel) | `primary_channel`, `per_channel_analysis` | `selected_channel`, `channel_reasoning` |
| Agent 6 (Content) | `generated_content` | `selected_content` (used exactly) |
| Agent 7 (Sequence) | `touches[]`, `exit_conditions` | `sequence_plan` |

## Development Guidelines

### Editing Prompts
- System prompts define agent behavior (500-800 lines each)
- User messages define input templates
- Agent 6 (Content Generator) is most complex - edit carefully
- Maintain anti-hallucination rules in all prompts
- When adding new outputs, update BOTH system-message.txt AND the output schema

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

## Proactive Outreach Philosophy (v3.0)

### Core Mindset
**Default is ACTION, not WAITING.** Silence is worse than a friendly message.

### Ghosting Thresholds (UPDATED)
| Consecutive Outbounds | Action |
|----------------------|--------|
| 0-2 | Normal follow-up OK |
| 3-4 | Lighter touch (check-in, coffee, channel switch) |
| 5+ | Break-up or no_action |

**Engagement signals (like, comment, view) RESET the counter.**

### New Opportunity Types
- `coffee_catch_up`: Propose café/desayuno for contacts with history
- `friendly_check_in`: "Comment ça va" approach for 30-90 days silence
- `social_signal_response`: Capitalize on likes/comments/profile visits immediately

### Message Tone: Human Connection First
- Propose coffee/breakfast when in same city
- "J'ai pensé à toi en voyant..." for warm reconnection
- "Comment ça se passe chez [company]?" shows genuine interest
- Less sales pitch, more friendly colleague energy

### When to Generate (ALWAYS, except...)
Generate content for ANY contact with history, UNLESS:
1. Meeting already scheduled
2. Contact explicitly said "contact me on [date]"
3. 5+ outbounds with ZERO response
4. Contact explicitly opted out
