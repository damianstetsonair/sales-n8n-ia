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
3. Trigger webhook at `/parallel-kam-agent` with contact data

```bash
curl -X POST https://your-n8n/webhook/parallel-kam-agent \
  -H "Content-Type: application/json" \
  -d @webhook-input-contact-example.json
```

## Architecture: 3-Wave Pipeline

```
WAVE 1: ANALYSIS (Parallel Execution, max 3 iterations)
├── Agent 1: Context & Relationship Analyzer (executeWorkflow)
├── Agent 2: Opportunity Detector (executeWorkflow)
├── Agent 3: State Analyzer (executeWorkflow)
└── Agent 4: Timing Strategist (executeWorkflow)
    ↓ Merge Wave1_results
    ↓ Orchestrator Wave 1 Evaluation (validates accuracy)
    ↓ Parse evaluation JSON
    ↓ Retry Switch (continue or retry, max 3 iterations)

WAVE 2: EXECUTION (Parallel Execution)
├── Get historical recommendations (Supabase)
├── Agent 5: Channel Selector (executeWorkflow)
├── Agent 6: Content Generator (executeWorkflow)
└── Agent 7: Sequence Strategist (executeWorkflow)
    ↓ Merge Wave2_results

WAVE 3: SYNTHESIS
└── Orchestrator Synthesis (final decision maker)
    ↓ Parse JSON
    ↓ Save to Supabase
    ↓ Notify via Slack/Zapier
```

## 🚨 V12.0 Active Customer Detection (CRITICAL FIX)

### Bug Fixed in V12.0

**The Problem:**
The system was not scanning ALL activities correctly. It looked at only the first few activities in the array and calculated `days_since_last_activity` incorrectly, leading to:
- Active customers classified as "dormant" (113 days silence when real silence was 2 days)
- Recent meetings ignored
- Re-engagement messages sent to customers with meetings scheduled for tomorrow

**Real Example of the Bug:**
```
What the system thought:
  - "113 días de silencio"
  - "Deal won pero dormant"
  - Generated: "Ça fait un moment depuis notre démo de fin août..."

Reality (scanning ALL activities):
  - 16/12 → André RESPONDE pidiendo mover reunión a viernes 14h
  - 16/12 → Bertran CONFIRMA viernes 14h
  - 17/12 → MEETING "André x Bertran - Weekly"
  - Last activity was 2 days ago, NOT 113 days ago
```

### The Fix: Mandatory Activity Scan (STEP -1)

All agents now MUST execute this BEFORE any analysis:

```
STEP -1: SCAN ALL ACTIVITIES
  FOR EACH activity in activities[]:
    - Parse activity.recorded_on
    - Track MAX(recorded_on) = true last_activity_date

  Calculate: days_since = (reference_date - last_activity_date)

  ⚠️ CRITICAL: activities[] may NOT be sorted by date!
  ⚠️ activities[0] is NOT necessarily the most recent
```

### Active Customer Detection Rules

| Condition | Classification | Action Type |
|-----------|---------------|-------------|
| Deal closed + activity within 30 days | DEAL_WON_ACTIVE | CUSTOMER_SUCCESS |
| Deal closed + activity 31-90 days | DEAL_WON_ACTIVE | CUSTOMER_SUCCESS |
| Deal closed + activity > 90 days | DEAL_WON_DORMANT | re_engagement |
| Deal closed + no post-close activity | DEAL_WON_DORMANT | re_engagement |

### Critical Validation Checks

```
IF days_since_last_activity <= 14:
  → NEVER classify as "dormant"
  → NEVER use re-engagement templates
  → Check for upcoming meetings FIRST

IF has_upcoming_meeting = true:
  → action = "wait"
  → DO NOT generate outreach
  → wait_type.type = "meeting_scheduled"
```

### Forbidden Patterns for Active Customers

These phrases are **FORBIDDEN** for contacts with activity in last 30 days:
- ❌ "Ça fait un moment depuis..."
- ❌ "Cela fait plusieurs mois..."
- ❌ "Depuis notre démo de..."
- ❌ "Vous êtes toujours sur le sujet?"

### New Output Requirements

All agents must include activity scan verification:
```json
"activity_scan_verification": {
  "total_activities_scanned": 155,
  "scan_complete": true,
  "last_activity_date": "2025-12-16",
  "days_since_last_activity": 2
}
```

Synthesis must include active customer check:
```json
"active_customer_check": {
  "is_active_customer": true,
  "days_since_last_activity": 2,
  "has_upcoming_meeting": true,
  "action_override": "wait"
}
```

---

## 🚨 V13.0 Churn Signal Detection (CRITICAL FIX)

### Bug Fixed in V13.0

**The Problem:**
The system was not detecting formal contract termination emails. It ignored:
- Formal churn emails from decision makers (CDIO, CEO, etc.)
- Meeting cancellations related to churn ("Refusé: André x Bertran")
- Explicit future recontact timing ("automne 2026")
- The fact that our team had already responded to the churn

**Real Example of the Bug:**
```
What the system thought:
  - "wait_type: meeting_scheduled" (meeting was CANCELLED)
  - "deal_status: closed_won" (customer CHURNED)
  - Generated outreach to a churned customer

Reality (reading email body):
  - 13/12 → CDIO sends formal email: "nous avons pris la décision de dénoncer le contrat"
  - 13/12 → CDIO explains: "le produit ne correspond plus exactement à nos besoins"
  - 13/12 → CDIO gives timeline: "automne 2026" for potential future contact
  - 13/12 → Bertran RESPONDS acknowledging the churn
  - 16/12 → André CANCELS the weekly meeting ("Refusé: André x Bertran - Weekly")
```

### The Fix: Email Body Signal Detection (STEP -0.2)

State Analyzer now MUST scan EMAIL metadata.body for critical signals:

```
STEP -0.2: SCAN EMAIL BODY FOR CRITICAL SIGNALS

FOR EACH activity WHERE activity_type = "EMAIL":
  body = activity.metadata.body.lower()

  // Check for CHURN signals
  CHURN_KEYWORDS = [
    "dénoncer le contrat", "résiliation", "résilier",
    "ne souhaite pas poursuivre", "mettre fin", "fin de contrat",
    "terminate the contract", "termination", "cancel subscription"
  ]

  FOR EACH keyword IN CHURN_KEYWORDS:
    IF keyword IN body:
      churn_detected = true
      churn_source = activity

  // Check sender authority
  IF churn_detected AND sender_job contains ("Chief", "Director", "VP", "Head"):
    authority_level = "decision_maker"
    → Email OVERRIDES HubSpot deal status
```

### Meeting Cancellation Detection

Scan activity titles for cancellation patterns:

```
CANCELLATION_PREFIXES = ["Refusé:", "Declined:", "Annulé:", "Cancelled:"]

FOR EACH activity:
  IF activity.metadata.title.startswith(any CANCELLATION_PREFIX):
    meeting_cancelled = true
    → has_upcoming_meeting = false (even if meeting exists in activities)
```

### Response Detection

Before flagging a churn signal, check if we already responded:

```
IF churn_detected:
  churn_date = churn_activity.recorded_on

  FOR EACH activity WHERE recorded_on > churn_date:
    IF direction = "OUTBOUND":
      already_responded = true
      response_date = activity.recorded_on
      → action = "wait" (we already handled this)
```

### Future Timing Extraction

Parse timing keywords from churn emails:

| Keyword | Parsed Date |
|---------|-------------|
| "automne 2026" | 2026-09-01T09:00:00+02:00 |
| "printemps 2026" | 2026-03-21T09:00:00+01:00 |
| "début 2026" | 2026-01-06T09:00:00+01:00 |
| "fin 2026" | 2026-11-15T09:00:00+01:00 |
| "l'année prochaine" | First Monday of January |

### Churn Signal Detection Output

State Analyzer must include:
```json
"critical_signal_detection": {
  "churn_detected": true,
  "churn_signals": [{
    "signal_type": "contract_termination",
    "sender": "Pierre Martin",
    "sender_role": "Chief Digital & Information Officer",
    "authority_level": "decision_maker",
    "keyword_matched": "dénoncer le contrat",
    "verbatim_quote": "nous avons pris la décision de dénoncer le contrat"
  }],
  "meeting_cancelled": true,
  "meeting_cancellation": {
    "cancelled_meeting_title": "Refusé: André x Bertran - Weekly",
    "cancellation_context": "post_churn"
  },
  "future_timing": {
    "exists": true,
    "verbatim": "automne 2026",
    "parsed_date_iso": "2026-09-01T09:00:00+02:00"
  },
  "response_status": {
    "already_responded": true,
    "response_date": "2025-12-13T15:30:00+00:00"
  }
}
```

### Deal Status Override

Email content OVERRIDES HubSpot deal status:

```
IF churn_detected AND authority_level = "decision_maker":
  deal_status_override = "CHURNED"
  original_hubspot_status = deal_status
  is_active_customer = false
  customer_status = "CHURNED"
```

### Synthesis Churn Safety Check

Synthesis must include:
```json
"churn_signal_check": {
  "churn_detected": true,
  "deal_status_override": {
    "original_status": "CLOSED_WON",
    "overridden_to": "CHURNED"
  },
  "already_responded": true,
  "future_timing": {
    "exists": true,
    "parsed_date": "2026-09-01T09:00:00+02:00"
  },
  "action_determination": {
    "action": "wait",
    "reason": "Churn acknowledged, future timing agreed",
    "reevaluate_at": "2026-09-01T09:00:00+02:00"
  }
}
```

### Forbidden Patterns for Churned Contacts

These combinations are **FORBIDDEN**:
- ❌ Churn detected + wait_type = "meeting_scheduled" (meeting was CANCELLED)
- ❌ Churn detected + deal_status = "CLOSED_WON" (should be CHURNED)
- ❌ Churn detected + is_active_customer = true
- ❌ Already responded + action = "send_message"
- ❌ Future timing "automne 2026" + reevaluate_at in January

### Content Generator Churn Protection

Content Generator must block all sales content for churned contacts:

```
IF churn_detected = true:
  should_generate = false
  reason = "Churn detected and acknowledged"

  ONLY exception: If churn NOT acknowledged yet:
    Generate CHURN ACKNOWLEDGMENT (not sales content)
    Template: graceful_churn_response
```

### V13.1 Improvements (Testing Session Fixes)

Based on testing session feedback, the following improvements were made:

1. **STEP -0.9 Moved to Start**: Churn detection now executes at the very beginning of State Analyzer, before any other analysis. This ensures churned contacts are caught immediately.

2. **BLOCK Logic in Content Generator**: Added explicit BLOCK 0, BLOCK 1, BLOCK 2 checks at the very start of Content Generator that return immediately if churn is detected.

3. **Channel Selector Override**: Added OVERRIDE 0 and OVERRIDE 1 at the start of Channel Selector that return `primary_channel: "NONE"` for churned contacts.

4. **Explicit NO FILTER Rules**: Activity scan rules now explicitly state:
   - ❌ DO NOT filter by deals[] array content
   - ❌ DO NOT filter by owner_hubspot_id
   - ❌ DO NOT filter by activity_type
   - ❌ DO NOT filter by direction

5. **Response Detection Before Action**: All agents now check if we already responded to a churn signal before recommending any action.

### Agent Churn Handling Matrix

| Agent | Check | Action if Churn Detected |
|-------|-------|--------------------------|
| State Analyzer | STEP -0.9 (first step) | Set customer_status = "CHURNED", action_needed = false if responded |
| Channel Selector | OVERRIDE 0 | Return primary_channel = "NONE" |
| Content Generator | BLOCK 0 | Return content_blocked = true |
| Synthesis Orchestrator | CHECK 0 | Force action = "no_action" if responded |
| Evaluation Orchestrator | CHECK 1 | Validate all agents detected churn |

---

## 🚨 V14.0 Multi-Deal, Multi-Owner & Safety Checks (CRITICAL)

### Overview

V14 addresses three critical bugs where the system was not properly handling:
1. Contacts with multiple deals (old lost deal + new open deal)
2. Multiple team members working the same contact
3. Scheduled touchpoints buried in email/note content

### STEP -0.5: Multi-Deal Contact Handling

A contact may have MULTIPLE deals. The system now scans all activities and classifies each deal:

```
Priority Logic:
IF any deal is OPEN → contact_status = ACTIVE_PROSPECT (NOT nurture!)
ELIF any deal is CLOSED_WON → contact_status = ACTIVE_CUSTOMER
ELSE (all CLOSED_LOST) → contact_status = NURTURE_CANDIDATE
```

**Output:**
```json
"multi_deal_analysis": {
  "deals_found": [
    {"deal_hubspot_id": "123", "status": "CLOSED_LOST", "dealstage": "Ghosting to nurture"},
    {"deal_hubspot_id": "456", "status": "OPEN", "dealstage": "Go Meeting Pro2LT"}
  ],
  "total_deals": 2,
  "has_open_deal": true,
  "primary_deal": {"deal_hubspot_id": "456", "reason": "OPEN deal takes priority"},
  "contact_status": "ACTIVE_PROSPECT"
}
```

### STEP -0.4: Multi-Owner Activity Tracking

Multiple team members may work the same contact. The system now tracks all team member activities:

```
IF other_contributor.last_activity < 14 days AND has scheduled next_step:
  → action = "wait" or "no_action"
  → rationale = "Another team member is actively working this contact"
```

**Output:**
```json
"team_involvement": {
  "team_members_active": ["bertran_ruiz", "roland_bouchut"],
  "primary_owner": "bertran_ruiz",
  "other_contributors": [{
    "name": "roland_bouchut",
    "last_activity": "2025-12-17",
    "days_since_last": 3,
    "recent_activities": ["EMAIL: Déjeuner reprogrammé"]
  }],
  "coordination_needed": true
}
```

### STEP -0.3: Scheduled Touchpoint Detection

Scans emails/notes for scheduled meetings that may not be in the calendar yet:

**Detection patterns:**
- Email: "semaine du 13/1", "le 07/01", "OK pour", "c'est noté"
- Note: "Suivi booké", "RDV prévu le", "Next step:"
- Meeting: recorded_on > today

**Output:**
```json
"scheduled_touchpoints": {
  "has_scheduled_touchpoint": true,
  "touchpoints": [{
    "type": "lunch",
    "with": "roland_bouchut",
    "scheduled_for": "2026-01-13 to 2026-01-17",
    "verbatim": "plutot la semaine du 13/1"
  }],
  "implication": "No outreach needed - scheduled touchpoint exists"
}
```

### Synthesis Safety Checks (5 Checks) - ENHANCED V15.0

Before final output, Synthesis runs 5 safety checks:

| Check | What it Validates | Action if Failed |
|-------|-------------------|------------------|
| Unclosed Loops | Unfulfilled promises < 30d must be addressed | BLOCK |
| Multi-Owner Coordination | Other team member active < 14d | BLOCK |
| Temporal Reference | Message matches actual recency | REJECT |
| Open Deal Template | OPEN deal → no re-engagement template | REJECT |
| Churn Signal | Churned customers properly handled | BLOCK |

**Output:**
```json
"synthesis_safety_checks": {
  "multi_deal_consistency": {"passed": true},
  "multi_owner_coordination": {"passed": false, "action": "BLOCK"},
  "scheduled_touchpoint_respect": {"passed": false, "action": "WAIT"},
  "temporal_reference_accuracy": {"passed": true},
  "activity_recency_sanity": {"passed": true},
  "overall_verdict": "BLOCK",
  "blocking_reason": "Another team member has scheduled touchpoint"
}
```

### Temporal Reference Accuracy (Content Generator)

Content Generator now validates time phrases against actual activity recency:

| days_since | Allowed Phrases | Forbidden Phrases |
|------------|-----------------|-------------------|
| 0-7 | "suite à notre échange récent" | "ça fait un moment" |
| 8-30 | "suite à notre échange de [month]" | "plusieurs mois" |
| 31-60 | "depuis quelques semaines" | "ça fait un moment" |
| 61-90 | "ça fait quelques mois" | "un an" |
| 91-180 | "ça fait un moment" | "récemment" |
| 180+ | "depuis notre [event] de [year]" | "il y a peu" |

### Context Analyzer: Deal Context

Context Analyzer now outputs `deal_context` with messaging constraints:

```json
"deal_context": {
  "primary_deal_status": "OPEN",
  "relationship_context": "ACTIVE_SALES_CYCLE",
  "appropriate_action_types": ["sales_followup", "value_share"],
  "prohibited_action_types": ["re_engagement", "nurture"],
  "messaging_constraints": {
    "avoid_phrases": ["ça fait un moment", "depuis notre démo de mars"],
    "reason": "Deal is ACTIVE - these phrases imply dormancy"
  }
}
```

### Forbidden Patterns Summary

**For OPEN deals:**
- ❌ "Ça fait un moment"
- ❌ "Depuis notre démo de [old date]"
- ❌ "Le sujet est-il toujours d'actualité"

**For active customers (activity < 14 days):**
- ❌ Re-engagement templates
- ❌ Any phrase implying dormancy

**When other team member active:**
- ❌ Sending parallel outreach
- ❌ Ignoring scheduled touchpoints

---

## 🚀 V15.0 AE-Agent Enhancements (December 2025)

V15.0 integrates proven business rules from AE-Agent to enrich prompt quality.

### New Features Added:

#### 1. Unclosed Loops Detection (State Analyzer)
Detects unfulfilled promises and unanswered questions:
- Scans OUTBOUND for promises: "je t'envoie", "je te rappelle", "on se cale"
- Verifies if promises were fulfilled (doc sent, call made, meeting scheduled)
- Outputs `unfulfilled_promises[]` with `days_overdue` and `must_address_first` flag

#### 2. Must Stay On Topic Flag (State Analyzer → Content Generator)
```json
"must_stay_on_current_topic": {
  "flag": true,
  "current_topic": "send_document",
  "topic_reason": "Unfulfilled promise to send document",
  "instruction_for_content_generator": "Message MUST address the unfulfilled promise before introducing new topics"
}
```

#### 3. Hook Generation for Engagement Signals (State Analyzer)
Each engagement signal now includes ready-to-use hooks:
```json
"hook_for_content_generator": {
  "primary_hook": "J'ai vu que tu as consulté notre doc sécurité, as-tu des questions?",
  "alternative_hooks": ["Tu as jeté un œil..."],
  "usage_instruction": "Use primary_hook as opening line"
}
```

#### 4. Holiday Coherence Validation (Content Generator)
Prevents embarrassing date-incoherent messages:
- "Bonne année" only allowed Dec 20 - Jan 15
- "Bonnes vacances" only allowed Jun 15 - Aug 31
- "Bonne rentrée" only allowed Aug 20 - Sep 15

#### 5. Engagement Signal Channel Mapping (Channel Selector)
Maps engagement signals to preferred response channels:
| Signal | Preferred Channel | Boost |
|--------|-------------------|-------|
| LINKEDIN_LIKE | LINKEDIN | +0.30 |
| LINKEDIN_COMMENT | LINKEDIN | +0.35 |
| DOCUMENT_VIEW | EMAIL | +0.30 |

#### 6. Post-Synthesis Validation Scoring (Orchestrator Evaluation)
5-dimension validation system:
- Temporal Accuracy (0.0-1.0)
- Tone Appropriateness (0.0-1.0)
- Value Density (0.0-1.0)
- Hook Usage (0.0-1.0)
- Structure Compliance (0.0-1.0)

Overall score >= 0.80 = PASS

#### 7. Enhanced Synthesis Safety Checks
Added override priority (highest to lowest):
1. CHURN DETECTED + RESPONDED → wait
2. CHURN DETECTED + NOT RESPONDED → send_message (acknowledgment)
3. MULTI-OWNER ACTIVE TOUCHPOINT → wait
4. MEETING SCHEDULED → wait
5. EXPLICIT TIMING AGREEMENT → wait until date
6. UNCLOSED LOOP < 30 DAYS → must address in content
7. OPEN DEAL → must use sales template

#### 8. Chronology Verification Output (State Analyzer)
Structured output for downstream validation:
```json
"chronology_verification": {
  "reference_date_used": "2025-12-20T00:00:00+01:00",
  "last_inbound": {...},
  "last_outbound": {...},
  "most_recent_overall": "OUTBOUND",
  "days_since_last_contact": 3,
  "sequence_analysis": {
    "who_spoke_last": "us",
    "awaiting_response_from": "them"
  }
}
```

---

## 🚀 V15.1 Bug Fixes (December 2025)

V15.1 addresses two critical bugs identified through decision evaluation testing.

### Bug 1: Consecutive Outbound Miscounting (CRITICAL)

**The Problem:**
State Analyzer was incorrectly counting consecutive outbound messages after the last inbound. It would report "2 consecutive outbounds" when the actual count was 7+ messages.

**Real Example of the Bug:**
```
What the system reported:
  - "consecutive_outbound_without_reply": 2
  - Risk assessment: "approaching ghosting threshold"

Reality (scanning ALL activities after last INBOUND):
  2025-09-29 OUTBOUND: "Sous la 🌊?"
  2025-09-19 OUTBOUND: LinkedIn event link
  2025-09-12 OUTBOUND: "Jeudi 18, 15h?"
  2025-09-12 OUTBOUND: "Avec plaisir"
  2025-09-12 OUTBOUND: "Oui je me souvient..."
  2025-09-12 OUTBOUND: "Désolé pour le délai"
  2025-09-12 OUTBOUND: "Semaine prochaine..."
  2025-09-12 OUTBOUND: "Bonjour Anne-claire"
  → Actual count: 8 consecutive outbounds (well past ghosting threshold!)
```

**The Fix:**
1. Added detailed algorithm in State Analyzer for counting ALL outbounds after last inbound
2. Added mandatory `consecutive_outbound_tracking` output with full list of outbound activities
3. Added Check 6 in Orchestrator Evaluation to independently verify the count
4. Common bug patterns now explicitly documented:
   - Bug: Only counting recent messages
   - Bug: Counting unique dates instead of individual messages
   - Bug: Multiple messages on same day counted as one

### Bug 2: Multiple Questions in Message (Best Practice Violation)

**The Problem:**
Content Generator was producing messages with 2+ questions, violating Best Practice #1: "Une seule question par email."

**Real Example of the Bug:**
```
❌ Generated message with 2 questions:
"Je te l'envoie si ça t'intéresse? On se cale un call de 15 min la semaine prochaine?"

✅ Correct (1 combined question):
"Je te l'envoie et on en parle en call la semaine prochaine?"
```

**The Fix:**
1. Added STEP -1.35 in Content Generator: Maximum One Question Rule
2. Added question count validation algorithm with regeneration logic
3. Added question combination patterns (how to merge 2 questions into 1)
4. Added Check 6 in Orchestrator Synthesis to validate question count
5. If Content Generator fails, Synthesis MUST fix before outputting

### New Output Requirements

**State Analyzer must include:**
```json
"consecutive_outbound_tracking": {
  "last_inbound_date": "2025-09-12T07:00:00+00:00",
  "outbound_count_after_last_inbound": 8,
  "outbound_activities_list": [...],
  "ghosting_threshold_exceeded": true
}
```

**Content Generator must include:**
```json
"question_count_check": {
  "questions_found": ["On se cale 15 min?"],
  "question_count": 1,
  "validation_passed": true
}
```

**Orchestrator Synthesis now has 6 safety checks (was 5):**
- Check 1: Unclosed loops
- Check 2: Multi-owner coordination
- Check 3: Temporal reference accuracy
- Check 4: Open deal template consistency
- Check 5: Churn signal detection
- Check 6: Question count validation (NEW)

---

## 🚀 V15.2 Bug Fixes (December 2025)

V15.2 addresses two accuracy issues identified through decision evaluation testing.

### Bug 1: Off-by-One Consecutive Outbound Counting

**The Problem:**
State Analyzer was sometimes including messages with the same timestamp as the last INBOUND in the outbound count, leading to off-by-one errors (e.g., reporting 8 when actual was 7).

**The Fix:**
1. Updated counting algorithm to use strict `>` comparison (not `>=`)
2. Added explicit note: "Count ONLY messages with recorded_on > last_inbound.recorded_on"
3. Updated examples to show correct count (7, not 8)
4. Added validation point: "The last_inbound itself does NOT count"

### Bug 2: Meeting Responsibility Misattribution

**The Problem:**
When a meeting was proposed but never materialized, the system was incorrectly attributing responsibility. Example: If WE asked the contact to send the invite ("Si ok envoie moi une invit bertran@airsaas.io"), the risk_factors incorrectly said "We dropped the ball" when it should have said "Contact was asked to send invite but never followed through."

**The Fix:**
1. Added MEETING RESPONSIBILITY ATTRIBUTION section in State Analyzer
2. Detects invite delegation patterns:
   - "envoie moi une invit" → CONTACT should send
   - "je t'envoie une invit" → WE should send
3. Outputs `meeting_responsibility` object with `who_dropped_ball`
4. Added Check 7 in Orchestrator Synthesis to validate risk_factors attribution

**New Output Structure (State Analyzer):**
```json
"meeting_responsibility": {
  "meeting_proposed": true,
  "meeting_materialized": false,
  "invite_responsibility": "CONTACT",
  "responsibility_quote": "Si ok envoie moi une invit bertran@airsaas.io",
  "who_dropped_ball": "CONTACT",
  "implication_for_risk_factors": "Contact did not follow through - this is THEIR non-response, not our failure"
}
```

**Forbidden Patterns:**
- ❌ When CONTACT was asked: "We dropped the ball on meeting invite"
- ✅ Correct: "Contact was asked to send calendar invite but never followed through"

---

## 🚀 V15.3 Semantic Outbound Analysis (December 2025)

V15.3 introduces semantic distinction between conversation bursts and follow-up attempts for accurate ghosting detection.

### The Problem

The system was counting raw outbound messages without considering context. Example:
- 7 outbound messages after last inbound → reported as "7 consecutive outbounds"
- But 5 of those messages were sent the SAME DAY as the inbound (active conversation)
- Only 2 were actual follow-up attempts on different days

This led to overstated ghosting patterns like "7 consecutive outbounds over 88 days" when the reality was "2 follow-up attempts over 17 days."

### The Fix: Conversation Burst vs Follow-up Attempts

**Definitions:**
- **Conversation Burst**: Multiple messages sent on the SAME DAY as the last inbound. These are part of active conversation, not separate outreach attempts.
- **Follow-up Attempts**: Messages sent on DIFFERENT days after the last inbound. These are distinct attempts to re-engage.

**Example Analysis:**
```
Raw data after last inbound (Sept 12):
  - Sept 12: 5 messages → CONVERSATION BURST (same day = active chat)
  - Sept 19: 1 message → FOLLOW-UP ATTEMPT #1
  - Sept 29: 1 message → FOLLOW-UP ATTEMPT #2

❌ WRONG: "7 consecutive outbounds" (overstates intensity)
✅ CORRECT: "2 follow-up attempts over 17 days" (accurate pattern)
```

### Updated Ghosting Thresholds

Thresholds now apply to DISTINCT FOLLOW-UP ATTEMPTS, not raw message count:

| Distinct Follow-up Attempts | State | Strategy |
|-----------------------------|-------|----------|
| 0-1 | Normal | Standard follow-up OK |
| 2-3 | Caution | Lighter touch recommended |
| 4+ | Ghosting | Break-up or no_action |

### New Output Structure (State Analyzer)

```json
"consecutive_outbound_tracking": {
  "raw_outbound_count": 7,
  "conversation_burst_analysis": {
    "burst_date": "2025-09-12",
    "burst_message_count": 5,
    "burst_context": "Active conversation same day as inbound"
  },
  "follow_up_attempts": {
    "distinct_dates": ["2025-09-19", "2025-09-29"],
    "distinct_attempt_count": 2,
    "days_between_attempts": 10
  },
  "ghosting_assessment": {
    "metric_used": "distinct_follow_up_attempts",
    "count": 2,
    "threshold": 4,
    "threshold_exceeded": false,
    "assessment": "CAUTION - 2 follow-up attempts without reply"
  }
}
```

### Key Insight

**Ghosting is about PERSISTENCE, not VERBOSITY.**

A salesperson who sent 5 messages during an active conversation is NOT the same as someone who sent 5 follow-up attempts over several weeks. The ghosting assessment must reflect the PATTERN of outreach, not just the volume.

---

## Wave 1 Retry Configuration

The workflow includes a retry mechanism for Wave 1 with dynamic configuration from the `Save data call` node:

```
Reference: {{ $('Save data call').item.json.max_wave1_iteration }}
Default value: 3
```

### Retry Logic

| Iteration | Behavior |
|-----------|----------|
| 1 | Normal execution |
| 2 | First retry (if Evaluation returns "retry") |
| N (max) | Final attempt (if still failing, force "continue_with_warnings") |

### Infinite Loop Prevention

The Orchestrator Evaluation must respect `max_wave1_iteration`:
- Value is dynamically referenced from `Save data call` node
- If same errors persist after clear retry instructions → switch to `continue_with_warnings`
- Let Synthesis apply its 7 safety checks to handle edge cases
- Never trigger infinite retry loops

---

## 🚀 V15.4 Evaluation Feedback Improvements (December 2025)

V15.4 addresses feedback from decision evaluation testing to improve accuracy and clarity.

### Fix 1: Evidence Trail Uses activity_id (Not Array Indices)

**The Problem:**
Evidence trail was using array indices like `activities[9]` which are unreliable because activity ordering may vary.

**The Fix:**
Use `activity_id` for stable references:

```json
// ❌ WRONG
"json_path": "activities[9].metadata.body"

// ✅ CORRECT
"activity_id": "urn:li:msg_message:MTc1OTEyOTcwOTM4...",
"recorded_on": "2025-09-29T07:00:00+00:00"
```

### Fix 2: Follow-up Type Classification

**The Problem:**
"2 consecutive outbounds" was misleading - one was a value-add (event link), one was a direct follow-up. Different intensities.

**The Fix:**
State Analyzer now classifies each follow-up attempt:

| Type | Description |
|------|-------------|
| `direct_follow_up` | Explicit nudge ("Sous la 🌊?") |
| `value_add` | Sharing content (event link) |
| `check_in` | Soft touch |
| `meeting_request` | Proposing meeting |

```json
"attempt_details": [
  {"date": "2025-09-19", "type": "value_add", "is_direct_follow_up": false},
  {"date": "2025-09-29", "type": "check_in", "is_direct_follow_up": true}
]
```

### Fix 3: Meeting Responsibility Phrasing

**The Problem:**
Message said "ça n'a pas pu se faire" (mutual-sounding) when CONTACT was supposed to send the invite.

**The Fix:**
Content Generator now checks `meeting_responsibility.invite_responsibility`:

```
IF invite_responsibility = "CONTACT":
  ❌ "On avait prévu de se parler mais ça n'a pas pu se faire"
  ✅ "Tu devais m'envoyer l'invite mais on s'est perdus de vue"

IF invite_responsibility = "US":
  ❌ "Tu devais envoyer l'invite"
  ✅ "Je devais t'envoyer l'invite et ça m'a échappé"
```

### Fix 4: Name Parsing Logic

**The Problem:**
Contact `full_name: "anne-claire_thery"` needs proper parsing for salutation.

**The Fix:**
Content Generator now documents name parsing:
- Replace underscores with hyphens for compound names
- Capitalize properly
- Extract first name for salutation

```json
"name_parsing": {
  "raw_full_name": "anne-claire_thery",
  "parsed_first_name": "Anne-Claire"
}
```

### Fix 5: Base Score Explanation

**The Problem:**
`base_score: 0.5` in opportunity_strength_breakdown was not explained.

**The Fix:**
Synthesis must include `base_score_reasoning`:

| Scenario | base_score | Reasoning |
|----------|------------|-----------|
| No deal, new lead | 0.50 | Neutral starting point |
| Open deal | 0.60 | Existing sales engagement |
| Closed-won active | 0.65 | Proven relationship |
| Churned | 0.30 | Negative exit history |

---

## 🚀 V15.6 Long Timeline Nurturing (December 2025)

V15.6 addresses a critical commercial strategy issue: when explicit timing agreements are long (>6 weeks), the system was incorrectly doing nothing while waiting.

### The Problem

**Feedback received:**
> "When the door is open but in lot of month, the goal is to wait one month or two and start give him some news about airsaas etc. When the deadlines are long, 'several months' for example > 6 weeks: we can't afford to do nothing while we wait, we have to provide quality news."

**Example of the Bug:**
```
Scenario: Customer churned, said "contact me in autumn 2026" (9 months away)

OLD (WRONG) behavior:
  - action = "wait"
  - reevaluate_at = "2026-09-01"
  - overall_strategy = "Any outreach before September 2026 would be disrespectful"
  - Result: 9 months of silence = relationship death
```

### The Fix: Long Timeline Nurturing

When explicit_timing_agreement exists AND timeline > 6 weeks away:
- Create a VALUE NURTURE sequence with periodic touches
- First touch after 30-45 day cooling period
- Continue value touches at 45-60 day intervals
- Final touch at agreed date can request meeting

### New Output Structure

**State Analyzer now outputs:**
```json
"long_timeline_nurturing": {
  "applies": true,
  "reason": "Agreed timeline > 6 weeks - intermediate value touches needed",
  "total_wait_days": 260,
  "agreed_date": "2026-09-01T09:00:00+02:00",
  "recommended_intermediate_touches": 3,
  "touch_interval_days": 60,
  "first_touch_recommended_at": "2026-02-15T09:00:00+01:00"
}
```

### Value Touch Cadence

| days_until_agreed | Touches | Interval |
|-------------------|---------|----------|
| > 180 days (6+ months) | 3-4 | 45-60 days |
| 90-180 days (3-6 months) | 2-3 | 30-45 days |
| 42-90 days (6-12 weeks) | 1-2 | 21-30 days |
| < 42 days (< 6 weeks) | 0 | Wait (OK for short periods) |

### Value Touch Content Guidelines

Value touches should:
- ✅ Share product updates relevant to their objection
- ✅ Share industry insights
- ✅ Share case studies from similar companies
- ✅ Be soft and non-pushy

Value touches should NOT:
- ❌ Request meetings (until final touch)
- ❌ Apply sales pressure
- ❌ Mention the churn explicitly
- ❌ Ignore the agreed timeline for decision

### Example Correct Output

```json
{
  "action": "send_message",
  "action_type": "value_nurture",
  "execute_at": "2026-02-15T09:00:00+01:00",
  "sequence_plan": [
    {
      "step": 1,
      "trigger": "scheduled_date",
      "planned_date": "2026-02-15T09:00:00+01:00",
      "action_type": "value_share",
      "content_theme": "product_update",
      "sales_pressure": "none"
    },
    {
      "step": 2,
      "trigger": "scheduled_date",
      "planned_date": "2026-04-15T09:00:00+02:00",
      "action_type": "value_share",
      "content_theme": "case_study",
      "sales_pressure": "none"
    },
    {
      "step": 3,
      "trigger": "scheduled_date",
      "planned_date": "2026-06-15T09:00:00+02:00",
      "action_type": "value_share",
      "content_theme": "industry_insight",
      "sales_pressure": "none"
    },
    {
      "step": 4,
      "trigger": "reevaluation_date",
      "planned_date": "2026-09-01T09:00:00+02:00",
      "action_type": "check_in",
      "content_theme": "direct_ask",
      "sales_pressure": "soft"
    }
  ],
  "overall_strategy": "Churned customer with 9-month timeline. Silence would kill the relationship. Plan: 3 value touches (product updates, case studies, insights) over 6 months, then decision follow-up in September 2026."
}
```

### Agents Updated

| Agent | Changes |
|-------|---------|
| State Analyzer | Added `long_timeline_nurturing` detection and output |
| Timing Strategist | Added `long_timeline_nurturing_timing` with touch dates |
| Sequence Strategist | Added `long_timeline_nurture` sequence type |
| Orchestrator Synthesis | Added PRIORITY 0.55 long timeline override logic |

---

## 🚀 V15.5 Meeting & Pricing Accuracy Improvements (December 2025)

V15.5 addresses accuracy issues identified through decision evaluation testing, focusing on meeting characterization and pricing distinction.

### Fix 1: Meeting/Call Substantive Analysis

**The Problem:**
The system was characterizing all MEETING activities as business-substantive without analyzing the actual content. A 4-minute informal call about room availability was being reported as a "catchup meeting maintaining relationship" when it had no business substance.

**Example of the Bug:**
```
Raw data:
  - Duration: 4 minutes
  - Content: "La conversation a été informelle, sans aborder de sujets commerciaux spécifiques"

❌ WRONG: "recent_meeting_held with relationship maintained"
✅ CORRECT: "brief informal call about logistics, no business substance"
```

**The Fix (State Analyzer):**
Added `meeting_characterization` output with nature classification:

| meeting_nature | Description | Evidence Weight |
|----------------|-------------|-----------------|
| BUSINESS_SUBSTANTIVE | 20+ min, business topics | Full weight |
| BUSINESS_BRIEF | 10-20 min, some business | 0.7x weight |
| INFORMAL_LOGISTICAL | <10 min, no business | 0.2x weight |

New output:
```json
"meeting_characterization": [{
  "activity_id": "98079708032",
  "duration_minutes": 4,
  "meeting_nature": "INFORMAL_LOGISTICAL",
  "business_substance": "low",
  "NOT_a_business_meeting": true
}]
```

### Fix 2: Duration Source Attribution

**The Problem:**
Duration was being cited from NOTE activities instead of the actual CALL activity.

**The Fix:**
Duration must be sourced from CALL activity, not NOTE:
```
❌ WRONG: Citing "32-min demo" from NOTE (activity_id: 96302085438)
✅ CORRECT: Citing "32-min demo" from CALL (activity_id: 96299200111)
```

### Fix 3: Pricing Type Distinction (Opportunity Detector)

**The Problem:**
"14,000€ pricing discussed" implies product pricing, but it was actually "accompagnement" (consulting/implementation) cost.

**The Fix:**
Added `pricing_sub_type` classification:

| Type | Meaning | Examples |
|------|---------|----------|
| PRODUCT_PRICING | SaaS license/subscription | "28,800€ licence annuelle" |
| SERVICE_PRICING | Consulting/implementation | "14,000€ accompagnement" |

New output:
```json
{
  "hook_type": "budget_signal",
  "pricing_sub_type": "SERVICE_PRICING",
  "pricing_clarification": "14,000€ is for accompagnement (workshops), NOT product licensing"
}
```

### Fix 4: synthesis_confidence Calculation (Orchestrator Synthesis)

**The Problem:**
`synthesis_confidence: 0` was being output as a default value, which is invalid.

**The Fix:**
Added mandatory calculation formula:
```
synthesis_confidence = weighted_average(
  relationship_clarity * 0.20,
  channel_certainty * 0.15,
  content_quality * 0.20,
  timing_accuracy * 0.15,
  opportunity_strength * 0.20,
  safety_checks_ratio * 0.10
)
```

Minimum valid value is ~0.30 (all defaults). Value of 0 triggers recalculation.

### Fix 5: Contact Name Parsing (Content Generator)

**The Problem:**
Raw CRM names like "marc_gillot" need parsing for professional salutations.

**The Fix:**
Added name parsing algorithm:
```
"marc_gillot" → "Marc Gillot" (full) / "Marc" (salutation)
"anne-claire_thery" → "Anne-Claire Thery" (full) / "Anne-Claire" (salutation)
```

---

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

## 👥 Stakeholder Mention System (v11)

### Regla: preserve_stakeholder_mentions
Cuando hay múltiples stakeholders en stakeholder_map, los mensajes de re-engagement DEBEN mencionarlos:
- "depuis notre démo avec Kasim sur vos enjeux PPM"
- "Depuis notre échange avec Kasim et toi sur la gestion de vos 40 projets"

### Multi-threading Opportunity
Si stakeholder_map.multi_threading_opportunity.exists = true:
- En step 2-3 de secuencia: "N'hésite pas à transférer à [name] si c'est plus pertinent de son côté"

### Excepciones (NO mencionar stakeholders):
- Stakeholder mencionado en contexto negativo
- Stakeholder dejó la empresa
- Rol subordinado/administrativo (asistente)

### Output tracking:
```json
"stakeholder_handling": {
  "other_stakeholders_available": true,
  "stakeholders_mentioned_in_message": ["Kasim"],
  "multi_threading_enabled": true
}
```

## 🎯 Resource Specificity System (v11)

### Specificity Bonus
Los recursos que abordan pain points ESPECÍFICOS reciben bonus en el scoring:

| Nivel | Bonus | Ejemplos |
|-------|-------|----------|
| HIGH | +0.15 | "Power BI, Notion, PowerPoint", "intégration Jira" |
| MEDIUM | +0.08 | "40 projets" con contexto, "Scrum + standard hybride" |
| LOW | +0.00 | "gestion de capacité", "visibilité portefeuille" |

### Tiebreaker Rules (cuando scores similares ±0.05)
1. **Specificity wins** - Recurso con mayor specificity_bonus
2. **Recency** - Pain point más reciente
3. **Type preference** - case_study > video > ebook
4. **Proof points** - Recursos con datos concretos
5. **Consumption effort** - Menor esfuerzo primero

## 📡 Channel Availability Verification (v11)

### Regla Crítica
El Channel Selector DEBE verificar disponibilidad de canal ANTES de evaluar suitability.

### Verification Matrix

| Channel | Campo Requerido | Regla |
|---------|-----------------|-------|
| EMAIL | `contact_info.email` | Campo DEBE existir Y tener valor |
| LINKEDIN | `contact_info.linkedin_url` | Campo DEBE existir Y tener valor |
| PHONE | `contact_info.phone` | Campo DEBE existir Y tener valor |
| WHATSAPP | `contact_info.whatsapp_number` | Campo DEBE existir Y tener valor |

### availability_check Object (Mandatory)
Cada canal en `per_channel_analysis` DEBE incluir:

```json
"availability_check": {
  "field_checked": "contact_info.email",
  "field_exists": true,
  "field_value": "marie@company.com",
  "available": true
}
```

Si no disponible:
```json
"availability_check": {
  "field_checked": "contact_info.email",
  "field_exists": false,
  "field_value": null,
  "available": false,
  "reason": "email field not present in contact_info"
}
```

### Reglas Críticas
1. **NUNCA asumir disponibilidad** - siempre verificar contra contact_info
2. **Existencia de campo ≠ existencia de dato** - verificar ambos
3. **Activities NO otorgan disponibilidad** - historial de EMAIL no significa que existe contact_info.email
4. **Si no disponible, suitability_score = 0.0**
5. **Primary channel DEBE estar disponible**

### Email en Notes (Informativo)
Si email aparece en activities[].metadata.body pero NO en contact_info.email:
- `available = false` (sigue siendo false)
- Agregar a `negative_signals`: "email_found_in_notes_but_not_contact_info"
- NO hace el canal disponible - solo informativo para CRM hygiene

### Bug Fix Context (v11.1)
Este sistema se agregó para corregir un bug donde el Channel Selector seleccionaba EMAIL como primary channel cuando el campo `contact_info.email` no existía, solo porque había actividades de tipo EMAIL en el historial.

## 🔗 LinkedIn URL Strategy (v11)

### Regla Principal
En mensajes de re-engagement en LinkedIn con relationship_depth < 7.0:
- **NO incluir URLs directas** (filtrado de LinkedIn + se siente transaccional)
- **USAR patrón "offer to send"**: "Je te l'envoie si ça t'intéresse"

### URL Escalation en Secuencias LinkedIn

| Step | Strategy | Pattern |
|------|----------|---------|
| 1 | offer_to_send | "Je te l'envoie si ça t'intéresse" |
| 2 | include_url | "Voici le lien direct : [URL]" |
| 3 | reference_without_url | "Ressources dispo sur notre site" |

### Decisión Logic
```
IF channel = "LINKEDIN":
  IF action_type IN ["re_engagement", "check_in"] AND relationship_depth < 7.0:
    USE offer_to_send
  ELIF relationship_depth >= 7.0:
    MAY include URL
```

### Output tracking:
```json
"linkedin_url_handling": {
  "strategy_used": "offer_to_send",
  "url_included_in_message": false,
  "url_to_send_on_response": "https://...",
  "fallback_if_no_response": "Include URL in step 2"
}
```

## 📝 Ejemplo de Mensaje Final Esperado (V11)

```
Bonjour André,

Cela fait plusieurs mois depuis notre démo avec Kasim sur la gestion de vos 40 projets.

Je pensais à vous en voyant un case study qu'on a publié sur la consolidation d'outils - exactement le sujet Power BI/Notion/PowerPoint dont on avait parlé.

Comment ça a évolué de votre côté ? Vous avez pu avancer sur la visualisation capacitaire ?

Je te l'envoie si ça t'intéresse - sinon pas de souci, dis-le moi.

Bertran
```

**Checklist V11:**
- ✅ Kasim mencionado (stakeholder preservado)
- ✅ Case study de consolidación (recurso específico vs genérico)
- ✅ Pain point específico (Power BI/Notion/PowerPoint)
- ✅ No hay URL directa (offer to send para LinkedIn)
- ✅ Easy exit incluido
- ✅ Value bridge ("Je pensais à vous en voyant...")
