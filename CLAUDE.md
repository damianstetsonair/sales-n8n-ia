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

## V10.0 Decision Pipeline Stages

The multi-agent system follows a structured decision pipeline inspired by V10.0 algorithm:

### Stage 1: ANÁLISIS CRONOLÓGICO (State Analyzer)
```
┌─────────────────────────────────────────────────────────────────┐
│ 1️⃣  Sort activities[] by recorded_on DESC                       │
│ 2️⃣  Find last INBOUND (filter direction="INBOUND")              │
│ 3️⃣  Find last OUTBOUND (filter direction="OUTBOUND")            │
│ 4️⃣  Compare timestamps → Determine "most_recent"                │
│ 5️⃣  Calculate days_since_last_contact from reference_date       │
└─────────────────────────────────────────────────────────────────┘
OUTPUT: chronology_verification, last_inbound, last_outbound
```

### Stage 2: CLASIFICACIÓN DE DEALS (Opportunity Detector)
```
┌─────────────────────────────────────────────────────────────────┐
│ Check deal existence → activities[].deals[]                     │
│                                                                 │
│ Classification Logic:                                           │
│ ├─ NO deals found → NO_DEAL → "Lead"                           │
│ ├─ Name contains "nurturing/lost/perdu" → DEAL_NURTURING       │
│ ├─ closed_date > today or NULL → DEAL_OPEN → "Prospect"        │
│ └─ closed_date < today:                                         │
│    ├─ post_close_activity in 90d → DEAL_WON_ACTIVE             │
│    └─ no post_close_activity → DEAL_WON_DORMANT                │
└─────────────────────────────────────────────────────────────────┘
OUTPUT: deal_classification, evidences_type
```

### Stage 3: CLASIFICACIÓN DE INBOUND (State Analyzer)
```
┌─────────────────────────────────────────────────────────────────┐
│ CONVERSATIONAL (triggers ATTENTE):                              │
│ ├─ EMAIL (INBOUND)                                              │
│ ├─ CALL (INBOUND)                                               │
│ └─ LINKEDIN MESSAGE (INBOUND)                                   │
│                                                                 │
│ ENGAGEMENT_SIGNAL (triggers ACTION_RECOMMANDÉE):                │
│ ├─ LINKEDIN LIKE/REACTION                                       │
│ ├─ LINKEDIN COMMENT                                             │
│ ├─ LINKEDIN VISIT PROFILE                                       │
│ ├─ LINKEDIN CONNECT                                             │
│ ├─ DOCUMENT VIEW                                                │
│ └─ CONTENT VIEW                                                 │
└─────────────────────────────────────────────────────────────────┘
OUTPUT: last_inbound_classification, engagement_signals_detected
```

### Stage 4: VERIFICACIONES TOP PRIORITY (State Analyzer + Opportunity Detector)
```
┌─────────────────────────────────────────────────────────────────┐
│ ⚠️  BLOCKING CHECKS (must resolve before proceeding):           │
│                                                                 │
│ ├─ Séquence ouverte? (< 14 days + topic + no closure)          │
│ │   → Stay on topic, don't change subject                      │
│ │                                                               │
│ ├─ Meeting scheduled? (has_upcoming_meeting = true)            │
│ │   → ATTENTE_PROGRAMMÉE, no outreach needed                   │
│ │                                                               │
│ ├─ Explicit timing agreement? (contact said "recontactez en X")│
│ │   → Respect agreed date                                       │
│ │                                                               │
│ ├─ Boucle non fermée? (unfulfilled promise, unanswered question)│
│ │   → Address this FIRST before anything else                  │
│ │                                                               │
│ └─ Date coherence? (no "bonne année" in December, etc.)        │
│     → Validate temporal references in content                   │
└─────────────────────────────────────────────────────────────────┘
OUTPUT: conversation_scan, meeting_scheduled, explicit_timing_agreement
```

### Stage 5: DETERMINACIÓN DE STATUT (State Analyzer → Synthesis)
```
┌─────────────────────────────────────────────────────────────────┐
│ STATUT Decision Tree:                                           │
│                                                                 │
│ Has upcoming meeting?                                           │
│ └─ YES → ATTENTE_PROGRAMMÉE                                    │
│                                                                 │
│ Last = INBOUND CONVERSATIONAL?                                  │
│ └─ YES → ATTENTE                                               │
│                                                                 │
│ Last = ENGAGEMENT SIGNAL? ⚡                                    │
│ └─ YES → ACTION_RECOMMANDÉE (capitalize immediately!)          │
│                                                                 │
│ Créneaux proposés sans réponse?                                │
│ ├─ < 48h → ATTENTE                                             │
│ ├─ 48-72h → ACTION_POSSIBLE                                    │
│ └─ > 72h → ACTION_RECOMMANDÉE                                  │
│                                                                 │
│ Last OUTBOUND X days ago?                                       │
│ ├─ < 3 days → ATTENTE                                          │
│ ├─ 3-14 days → ACTION_POSSIBLE                                 │
│ └─ > 14 days → ACTION_RECOMMANDÉE                              │
│                                                                 │
│ Consecutive outbounds?                                          │
│ ├─ 0-2 → Normal flow                                           │
│ ├─ 3-4 → ACTION_POSSIBLE (lighter touch)                       │
│ └─ 5+ → NO_ACTION or BREAK_UP                                  │
└─────────────────────────────────────────────────────────────────┘
OUTPUT: statut.value, statut.primary_reason, timing_verdict
```

### Stage 6: DECISIÓN DE STATUS → ACTION (Synthesis)
```
┌─────────────────────────────────────────────────────────────────┐
│ STATUT → ACTION Mapping:                                        │
│                                                                 │
│ ATTENTE            → action="wait"                              │
│ ATTENTE_PROGRAMMÉE → action="wait" (meeting_scheduled)         │
│ ACTION_POSSIBLE    → action="send_message" (lighter touch)      │
│ ACTION_RECOMMANDÉE → action="send_message" (priority)           │
│ SUIVI_ACTIF        → action="send_message" (customer success)   │
│                                                                 │
│ ⚡ ENGAGEMENT SIGNAL OVERRIDE:                                  │
│ If hot signal detected (< 7 days):                              │
│ → FORCE action="send_message"                                   │
│ → urgency="high" or "critical"                                  │
│ → Use signal as primary hook                                    │
└─────────────────────────────────────────────────────────────────┘
OUTPUT: action, urgency, engagement_signal_processing
```

### Stage 7: GENERACIÓN DE SALIDA (Content Generator + Synthesis)
```
┌─────────────────────────────────────────────────────────────────┐
│ If action="send_message":                                       │
│ ├─ Select channel (Channel Selector recommendation)            │
│ ├─ Get content (Content Generator output EXACTLY)              │
│ ├─ Set execute_at (Timing Strategist window)                   │
│ └─ Build sequence_plan if applicable                           │
│                                                                 │
│ If action="wait":                                               │
│ ├─ Set wait_type (MANDATORY)                                   │
│ ├─ Set reevaluate_at date                                      │
│ └─ Build sequence_plan for follow-up                           │
│                                                                 │
│ ALWAYS include:                                                 │
│ ├─ opportunity_strength_breakdown (with evidence)              │
│ ├─ confidence_factors (from all agents)                        │
│ ├─ evidence_trail (json_path for each claim)                   │
│ └─ risk_factors + success_criteria                             │
└─────────────────────────────────────────────────────────────────┘
OUTPUT: final_decision (complete JSON structure)
```

### Agent Responsibility Matrix (V10.0 Stages)

| V10.0 Stage | Primary Agent | Supporting Agents |
|-------------|---------------|-------------------|
| Análisis Cronológico | State Analyzer | - |
| Clasificación Deals | Opportunity Detector | - |
| Clasificación INBOUND | State Analyzer | - |
| Verificaciones TOP | State Analyzer | Opportunity Detector |
| Determinación STATUT | State Analyzer | Synthesis |
| Decisión Action | Synthesis | All Wave 1 agents |
| Generación Salida | Synthesis | Content Generator, Channel Selector |

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

## 🚀 Filosofía Comercial Proactiva (v4.0)

### Mentalidad Central
**SOMOS VENDEDORES, NO ROBOTS DE CRM.**

Nuestro trabajo NO es encontrar excusas para no contactar. Es **MANTENER RELACIONES VIVAS** y **GENERAR OPORTUNIDADES DE VENTA**.

### Principios Fundamentales

1. **LA CERCANÍA VENDE**
   - Un "¿cómo estás?" genuino > 10 emails de producto
   - Proponer café/desayuno SOLO si el contacto está en PARIS
   - Preguntar por sus proyectos y problemas REALES

2. **BUSCAR REUNIONES SIEMPRE**
   - Objetivo final = VERSE CARA A CARA (si París) o CALL/VISIO (si no)
   - Cada mensaje debe abrir la puerta a una reunión
   - "¿Tomamos un café?" solo para contactos en París
   - "¿Un call de 15 min?" para contactos fuera de París

3. **RETOMAR PROBLEMAS CONVERSADOS**
   - Si mencionaron un problema → "¿Cómo van con X?"
   - Si tuvieron una demo → "¿Qué tal les fue implementando Y?"
   - Si hubo interés → "Me acordé de ustedes viendo Z"

4. **ENGAGEMENT = ACCIÓN INMEDIATA**
   - Like en LinkedIn → Mensaje EN EL MOMENTO
   - Visita perfil → "Vi que pasaste, ¿hablamos?" (café solo si París)
   - Vio documento → "¿Te quedaron dudas?"

### Ghosting Thresholds (UPDATED)
| Consecutive Outbounds | Action |
|----------------------|--------|
| 0-2 | Normal follow-up OK |
| 3-4 | Lighter touch (check-in, coffee, channel switch) |
| 5+ | Break-up or no_action |

**Engagement signals (like, comment, view) RESET the counter.**

### Tipos de Mensaje Prioritarios
- `coffee_catch_up`: Proponer café/desayuno SOLO SI contacto en París
- `problem_follow_up`: "¿Cómo van con [problema que mencionaron]?"
- `friendly_check_in`: "Comment ça va" para 30-90 días de silencio
- `social_signal_response`: Capitalizar likes/comments/visits INMEDIATAMENTE
- `meeting_request`: Propuesta concreta de reunión con días/horarios (call si no París)

### Tono de Mensajes: Conexión Humana Primero
- Proponer café/desayuno SOLO si contacto está en París
- Para otros: proponer call/visio
- "J'ai pensé à toi en voyant..." para reconexión cálida
- "Comment ça se passe chez [company]?" muestra interés genuino
- "¿Cómo vienen con [problema]?" demuestra que escuchaste

### Regla de Ubicación para Reuniones
| contact_info.location | Tipo de reunión |
|----------------------|-----------------|
| Contiene "Paris" | café, breakfast, en persona |
| Otra ciudad o desconocido | call, visio |

Menos pitch de ventas, más energía de colega amigable.

### Cuándo Generar Contenido (SIEMPRE, excepto...)
Generar contenido para CUALQUIER contacto con historia, A MENOS QUE:
1. Reunión ya agendada (fecha confirmada)
2. Contacto dijo explícitamente "contáctame en [fecha]"
3. 5+ outbounds con CERO respuesta
4. Contacto explícitamente pidió no ser contactado

### Pregunta Clave Antes de Decidir
> "Si este fuera MI mejor cliente y no le he hablado en 30 días, ¿le escribiría?"
> La respuesta es SÍ. Siempre.

## 🎁 Value Density System (v5.0)

### Regla de Valor Obligatorio para Re-engagement
Cuando hook_age_days > 90 Y es re-engagement, el mensaje DEBE incluir al menos UNO de:
1. **RESOURCE**: Recurso de `related_resources_to_offer`
2. **INSIGHT**: Insight de industria relevante a sus pain points
3. **PRODUCT_UPDATE**: Nueva feature que responde a sus necesidades
4. **CONCRETE_OFFER**: Oferta de reunión física (solo si París)

### Value Density Score
El Content Generator calcula un score de densidad de valor:

| Factor | Puntos |
|--------|--------|
| has_resource_or_insight | +0.30 |
| has_concrete_next_step | +0.20 |
| references_specific_pain_point | +0.20 |
| has_proximity_offer (Paris) | +0.15 |
| uses_value_bridge_pattern | +0.15 |
| only_asks_question_no_offer | -0.40 |

**Veredictos:**
- Score >= 0.50: PASS
- Score < 0.50 AND hook_age > 90d: FAIL_REGENERATE
- Score < 0.50 AND hook_age <= 90d: PASS_WITH_WARNING

### Template RE-ENGAGEMENT WITH VALUE (hooks > 60 días)
```
ESTRUCTURA OBLIGATORIA:
1. ACKNOWLEDGE_GAP - "Cela fait plusieurs mois..."
2. REFERENCE_SPECIFIC_CONTEXT - Personas/temas específicos
3. VALUE_BRIDGE - "Je pensais à vous en voyant..."
4. VALUE_OFFER - Recurso/insight/update concreto
5. SOFT_ASK - Pregunta sobre su situación
6. CONCRETE_NEXT_STEP - Café (si París) o call
7. SIGNATURE
```

### Anti-patrones (NO VALOR)
❌ "Je me permets de vous recontacter"
❌ "Je voulais prendre de vos nouvelles"
❌ "Si vous avez des questions"
❌ Mensajes que solo preguntan sin ofrecer nada

### Patrones de Valor (SÍ)
✅ "On vient de sortir [resource] qui répond à [pain point]"
✅ "Je pensais à vous en voyant [trend]"
✅ "On a sorti [feature] depuis notre échange"
✅ "Je suis souvent sur Paris, un café serait plus simple"

## 📚 Resource Matching System (v11)

### Resource Catalog
Opportunity Detector mantiene un catálogo de recursos con:
- Videos (quarter_plan_video, project_brief_ai_video, security_overview_video)
- Ebooks & Guides (ebook_why_quarter_plan, guide_ppm_pilotage, checklist_pmo_maturity)
- Case Studies (case_study_retail, case_study_banking, case_study_industry)
- Webinars (webinar_quarterly_planning, webinar_ai_pmo)
- Articles (article_10_authors_qp, article_excel_vs_tool)

### Match Score Calculation
Cada recurso recibe un match_score basado en 4 factores:

| Factor | Weight | Description |
|--------|--------|-------------|
| `keyword_overlap` | 0.35 | Keywords del hook vs tags del recurso |
| `pain_point_alignment` | 0.30 | ¿El recurso resuelve el pain point? |
| `role_fit` | 0.20 | ¿Job del contacto en ideal_for? |
| `format_preference` | 0.15 | Preferencia inferida (video/doc/case study) |

### Thresholds
| match_score | Action |
|-------------|--------|
| >= 0.50 | Strong match → Include |
| 0.35 - 0.49 | Medium match → Include if no better |
| < 0.35 | Weak match → Do NOT include |

### Output Structure (V11)
```json
"related_resources_to_offer": [{
  "resource_id": "quarter_plan_video",
  "match_score": 0.72,
  "match_score_breakdown": {...},
  "why_relevant": "...",
  "hook_connection": {"connected_hook_id": "hook_001"},
  "suggested_intro_phrase": "...",
  "value_proposition": "..."
}],
"primary_resource_recommendation": {
  "resource_id": "quarter_plan_video",
  "selection_reason": "...",
  "alternative_if_rejected": "ebook_why_quarter_plan",
  "usage_instruction_for_content_generator": "..."
},
"resource_gap_detected": {
  "gap_exists": false,
  "content_generator_instruction": null
}
```

### Evaluation Orchestrator Validation
- Verify match_score >= 0.35 for all resources
- Verify primary_resource is in related_resources_to_offer
- Check resource_gap_detected has alternative approach if gap_exists
