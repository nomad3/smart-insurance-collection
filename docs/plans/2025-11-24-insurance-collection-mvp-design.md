# Smart Insurance Collection MVP - Design Document

**Date:** November 24, 2025
**Version:** 1.0
**Status:** Approved for Implementation

## Executive Summary

An AI-first insurance collection platform for US healthcare providers to streamline claim submission, payment tracking, and denial management. The MVP focuses on creating an exceptional user experience with AI-assisted coding and validation, while maintaining HIPAA compliance and preparing for future clearinghouse integrations.

**Target Timeline:** 1-2 months
**Architecture:** Modular monolith with Docker containerization
**Deployment:** GCP VM with docker-compose

## Goals & Vision

### Primary Goals
1. **Speed**: Reduce claim creation time from 10+ minutes to <3 minutes
2. **Accuracy**: AI-assisted coding with >85% suggestion accuracy
3. **Intelligence**: Predict denials before submission (>75% accuracy)
4. **Compliance**: 100% HIPAA compliant from day one
5. **Extensibility**: Architecture ready for real clearinghouse integrations

### Target Users
- Small to medium medical practices
- Urgent care clinics
- Multi-specialty healthcare providers
- Hospital billing departments

### Core Value Proposition
Traditional billing software is complex and error-prone. Our AI-first approach turns natural language descriptions into accurate claims, validates before submission, and predicts issues before they happen.

## System Architecture

### Container Structure (Docker Compose)

```yaml
services:
  - frontend: Next.js 14+ with TypeScript
  - backend: FastAPI with Python 3.11+
  - database: PostgreSQL 15+ with encryption
  - mcp-server: Claude MCP server for AI features
  - redis: Session storage and caching
  - nginx: Reverse proxy with SSL/TLS termination
```

### Backend Module Structure

```
backend/
├── modules/
│   ├── claims/        # Claim creation, submission, tracking
│   ├── ai/            # AI assistance features
│   ├── users/         # Authentication, roles (Biller, Manager, Admin)
│   ├── patients/      # Patient registry
│   ├── insurers/      # Insurance company/plan database
│   ├── integrations/  # Clearinghouse placeholders
│   └── analytics/     # Reporting and dashboards
├── core/              # Shared utilities, database, security
└── mcp/               # MCP protocol handlers
```

### Data Flow

```
User creates claim
  → AI validates/suggests codes
  → Stored in DB
  → (Future: Submit to clearinghouse)
  → Status tracking
  → Payment reconciliation
```

### HIPAA Compliance Architecture

- All data encrypted in transit (TLS 1.3) and at rest (PostgreSQL encryption)
- Audit logging for all PHI access
- Role-based access control (RBAC)
- Session management with secure tokens
- Container network isolation
- Only nginx exposed to internet

## Data Model

### Core Entities

#### Users Table
```
- id, email, password_hash, role (biller/practice_manager/admin)
- organization_id (multi-tenant support)
- created_at, last_login, is_active
```

#### Patients Table
```
- id, organization_id, first_name, last_name, dob, ssn_encrypted
- address, phone, email
- insurance_primary_id, insurance_secondary_id
- ai_risk_score (likelihood of claim issues)
- common_diagnoses_json (AI-learned patterns)
- typical_procedures_json (AI-learned patterns)
- ai_notes (AI-generated patient summary)
- created_at, updated_at, created_by_user_id
```

#### Insurance_Plans Table
```
- id, payer_name, payer_id, plan_name, plan_type
- clearinghouse_id (placeholder: "change_healthcare", "availity", "waystar")
- contact_info, submission_url_placeholder
- integration_status (placeholder/sandbox/production)
- submission_method (edi_837/fhir/api)
- is_active
```

#### Patient_Insurance Table
```
- id, patient_id, insurance_plan_id
- member_id, group_number, policy_holder_name
- effective_date, termination_date, coverage_type (primary/secondary)
```

#### Claims Table
```
- id, organization_id, patient_id, insurance_plan_id
- claim_number, submission_date, service_date_from, service_date_to
- provider_npi, billing_provider_id, rendering_provider_id
- total_charge_amount, status, current_substatus
- ai_validation_results_json (detailed validation)
- ai_denial_risk_score (0-100, AI prediction)
- ai_denial_reasons_json (predicted reasons with confidence)
- ai_suggested_codes_json (alternative suggestions)
- ai_optimization_score (reimbursement optimization)
- ai_processing_status (queued/processing/completed/failed)
- user_accepted_ai_suggestions (boolean, for training)
- created_by_user_id, created_at, updated_at
```

#### Claim_Lines Table
```
- id, claim_id, line_number
- cpt_code, icd10_codes (array), quantity, charge_amount
- place_of_service, modifiers
- ai_suggested_cpt_code, ai_cpt_confidence_score
- ai_suggested_icd10_codes (array), ai_icd10_confidence_scores (array)
- user_original_input (natural language description)
- ai_code_rationale (explanation)
```

#### Claim_Status_History Table
```
- id, claim_id, status, substatus, notes
- changed_by_user_id, changed_at, reason
- response_from_payer (JSON for future EDI responses)
```

#### Payments Table
```
- id, claim_id, payment_date, amount_paid
- expected_amount, adjustment_amount, adjustment_reason
- payment_method, check_number, era_file_id (future)
- reconciled (boolean), reconciled_by_user_id, reconciled_at
```

### AI-Specific Tables

#### AI_Context Table
```
- id, organization_id, context_type (patient_history/claim_patterns/denial_history)
- embeddings (vector), metadata_json
- created_at, last_used_at
```
*Purpose: RAG (Retrieval Augmented Generation) for smarter AI suggestions*

#### AI_Suggestions Table
```
- id, claim_id, suggestion_type (code/validation/denial_prevention)
- original_value, suggested_value, confidence_score
- rationale, references (links to coding guidelines)
- user_action (accepted/rejected/modified), user_feedback
- created_at, actioned_at
```
*Purpose: Track AI suggestions and user feedback for continuous learning*

#### Denial_Patterns Table
```
- id, organization_id, insurance_plan_id
- denial_code, denial_reason, frequency_count
- common_fix_json (AI-learned successful resolutions)
- pattern_embeddings (vector for similarity matching)
- last_occurred_at, pattern_confidence
```
*Purpose: AI learns from historical denials to predict future issues*

#### AI_Model_Performance Table
```
- id, model_name, model_version, feature_type
- accuracy_score, precision, recall, f1_score
- sample_size, evaluation_date
- notes (what changed, why retrained)
```
*Purpose: Monitor and improve AI accuracy over time*

#### MCP_Sessions Table
```
- id, user_id, claim_id (nullable)
- session_type (code_suggestion/validation/denial_analysis)
- context_json (full conversation context)
- tokens_used, cost_estimate
- started_at, completed_at, status
```
*Purpose: Track MCP usage for cost monitoring and debugging*

#### Audit_Log Table
```
- id, user_id, action_type, resource_type, resource_id
- phi_accessed (boolean), ip_address, user_agent
- request_data_encrypted, response_data_encrypted
- timestamp
```
*Purpose: HIPAA audit trail (immutable, 6-year retention)*

## AI Features & MCP Integration

### 1. AI-Assisted Code Suggestion

**User Flow:**
1. User types natural language: "Patient came in for annual checkup with diabetes follow-up"
2. AI suggests: CPT 99213 (Office visit), ICD-10 E11.9 (Type 2 diabetes), Z00.00 (General exam)
3. Shows confidence scores and rationale
4. User can accept, modify, or reject

**Technical Implementation:**
- MCP server receives natural language via Claude API
- Uses RAG: Queries AI_Context for similar past claims (vector similarity)
- Retrieves current coding guidelines from embeddings
- Returns structured JSON with codes, confidence, rationale
- Backend stores in AI_Suggestions table for learning

### 2. Real-Time Claim Validation

**Validation Checks (AI-powered):**
- Missing required fields detection
- Code compatibility (CPT + ICD-10 combinations)
- Age/gender mismatches (e.g., pregnancy code for male patient)
- Frequency limits (same procedure too soon)
- Authorization requirements by payer
- Documentation sufficiency prediction

**Technical Implementation:**
- On claim save, trigger async AI validation
- MCP server analyzes full claim context
- Compares against organization's historical denials (Denial_Patterns)
- Returns validation_results_json with severity levels (error/warning/info)
- Frontend shows inline validation messages

### 3. Denial Risk Prediction

**Prediction Model:**
- Analyzes: patient history, claim details, payer patterns, similar claims
- Generates risk score (0-100) with breakdown by risk factor
- Suggests preemptive fixes before submission

**Technical Implementation:**
- Runs after validation, before submission
- Queries Denial_Patterns for this payer + procedure combination
- AI model trained on organization's historical claim outcomes
- Stores prediction in claim record
- Tracks accuracy over time in AI_Model_Performance

### 4. MCP Native Architecture

**MCP Server Structure:**
```
mcp-server/
├── tools/
│   ├── code_suggester.py     # CPT/ICD-10 suggestions
│   ├── validator.py           # Claim validation
│   ├── denial_predictor.py    # Risk analysis
│   └── pattern_analyzer.py    # Learn from history
├── resources/
│   ├── coding_guidelines/     # CMS guidelines as resources
│   └── payer_rules/           # Payer-specific rules
└── prompts/
    └── templates/             # Structured prompts for consistency
```

**Communication Flow:**
- Frontend → Backend API → MCP Server (via HTTP/SSE)
- MCP uses Claude API (with optional specialized models for specific tasks)
- Responses cached in Redis for frequently asked patterns
- All interactions logged for HIPAA audit trail

### 5. Continuous Learning

- User feedback (accepted/rejected suggestions) stored in AI_Suggestions
- Weekly batch job retrains models on new data
- A/B testing for model improvements
- Performance metrics tracked in AI_Model_Performance

## Frontend Architecture

### Technology Stack
- Next.js 14+ (App Router)
- TypeScript for type safety
- TailwindCSS + shadcn/ui components
- TanStack Query for data fetching/caching
- Zustand for state management
- Recharts for analytics visualizations

### Page Structure

#### 1. Dashboard (Landing Page)
- **Top Metrics Cards:** Total claims, pending amount, denial rate, AI suggestions accepted
- **Claim Pipeline Visualization:** Kanban-style board (Draft → Submitted → Processing → Paid/Denied)
- **AI Insights Panel:** "High denial risk claims need attention" with quick actions
- **Recent Activity Timeline:** Latest claim updates, payments received
- **Quick Actions:** "New Claim" button (primary CTA), "Import Patients", "Run Reports"

#### 2. Claims Management
- **Table View:** Sortable/filterable claims list with inline status badges
- **Filters:** Status, date range, payer, assigned biller, AI risk score
- **Bulk Actions:** Export, reassign, batch submission (future)
- **AI Risk Indicators:** Color-coded flags (green/yellow/red) on each claim
- **Quick Preview:** Click row for slide-over panel with claim summary

#### 3. Claim Creation Wizard (AI-First)

**Step 1 - Patient Selection:**
- Smart search with autocomplete
- "Quick add patient" inline form
- AI shows: "Patient has 3 previous claims with this payer, 100% success rate"

**Step 2 - Service Details (AI Magic Happens Here):**
- Natural language input box: "Describe the visit in plain English"
- AI suggests codes in real-time as user types
- Side-by-side view: User input | AI suggestions with confidence scores
- Click to accept suggestion or manually search codes
- AI validation runs immediately: ✓ Valid combination or ⚠️ Warning with fix suggestion

**Step 3 - Billing Details:**
- Provider selection, place of service, dates
- AI checks: "This procedure typically takes 2-3 weeks for payment from this payer"
- Expected reimbursement amount (AI-predicted based on history)

**Step 4 - Review & Submit:**
- Full claim preview
- **AI Validation Summary Card:**
  - Denial risk score with gauge visualization
  - List of checks passed/failed
  - Suggested optimizations ("Adding modifier -25 increases approval by 15%")
- "Save as Draft" or "Submit Claim" (future: actual submission)

#### 4. Claim Detail Page
- **Header:** Claim number, patient name, status badge, AI risk score
- **Tabs:**
  - Overview: All claim details, editable fields
  - AI Insights: Detailed validation results, suggestions, confidence scores
  - Status History: Timeline of all status changes with audit trail
  - Payments: Expected vs actual, adjustments, reconciliation
  - Documents: Attachments, EOBs (future)

#### 5. AI Assistant Panel (Global)
- Floating chat widget (bottom-right)
- Context-aware: Knows which page/claim user is viewing
- Can ask: "Why was this claim denied?", "What codes should I use for X?"
- MCP-powered conversations with claim context

#### 6. Analytics & Reports
- **Revenue Dashboard:** Collections over time, outstanding AR aging
- **Payer Performance:** Success rates by insurance company
- **AI Performance:** Suggestion acceptance rate, prediction accuracy
- **Denial Analysis:** Top denial reasons, trends over time
- **Custom Reports:** Filterable, exportable to CSV/PDF

#### 7. Settings & Configuration
- **User Management:** Invite users, assign roles, permissions matrix
- **Organization Profile:** NPI, tax ID, billing address
- **Insurance Plans Library:** Add/edit payers, placeholder for future integrations
- **AI Configuration:** Model selection, confidence thresholds, auto-suggestions on/off

### Mobile Responsive
- All views optimized for tablet/mobile
- Priority: Dashboard, claim list, claim creation
- AI suggestions work seamlessly on mobile

## Security & HIPAA Compliance

### Technical Safeguards

#### 1. Access Control
- Role-Based Access Control (RBAC): Biller, Practice Manager, Admin
- Multi-factor authentication (TOTP via authenticator app)
- Automatic session timeout (15 minutes idle)
- Password requirements: 12+ chars, complexity rules, rotation every 90 days
- All PHI access logged with user_id, timestamp, action, IP address

#### 2. Audit Controls
- Immutable audit logs (append-only)
- Retention: 6 years minimum (HIPAA requirement)
- Real-time alerting for suspicious activity (multiple failed logins, bulk exports)
- All actions on PHI recorded in Audit_Log table

#### 3. Data Encryption
- **At Rest:** PostgreSQL transparent data encryption (TDE)
- **In Transit:** TLS 1.3 for all connections (Frontend ↔ Backend ↔ Database ↔ MCP)
- **Field-Level:** SSN, credit cards encrypted with AES-256 before storage
- **Encryption Keys:** Managed via environment variables, rotated quarterly

#### 4. Data Integrity
- Database backups: Daily full, hourly incremental
- Backup encryption and off-site storage to GCP Cloud Storage
- Disaster recovery plan: RTO 4 hours, RPO 1 hour
- Data validation on all inputs (Pydantic models in FastAPI)

### Network Security

**Docker Network Architecture:**
- All services on private Docker network
- Only nginx exposed to internet
- Service-to-service communication via internal DNS
- Firewall rules on GCP VM (only 80/443 inbound)

**Rate Limiting:**
- nginx: 100 requests/minute per IP
- API endpoints: 1000 requests/hour per user
- MCP server: Token-based throttling

## Deployment

### GCP VM Configuration
- **Compute Engine:** e2-standard-4 (4 vCPU, 16GB RAM) minimum
- **OS:** Ubuntu 22.04 LTS with regular security patches
- **Storage:** Persistent SSD for database (100GB+, auto-expand)
- **Security:** Cloud Armor for DDoS protection
- **Firewall:** Only ports 80/443 inbound, SSH key-based only

### Deployment Process
1. SSH into VM (key-based auth only)
2. Pull code from git repository
3. Environment variables via `.env.production` (never committed)
4. `docker-compose up -d --build`
5. Run database migrations
6. Health checks on all services
7. Zero-downtime updates via rolling restart

### Monitoring & Alerting
- GCP Cloud Monitoring for VM metrics (CPU, memory, disk)
- Application logs to GCP Cloud Logging
- Custom alerts: High error rate, service down, disk space low
- Weekly automated security scans

### Backup Strategy
- **Database:** Daily automated backups to GCP Cloud Storage
- **Audit logs:** Synced to Cloud Storage in real-time
- **Retention:** 6 years for HIPAA compliance
- **Testing:** Monthly restore drills

### Business Associate Agreement (BAA)
Required for:
- GCP (Cloud Storage, Cloud Logging)
- Anthropic (Claude API for MCP)
- Any future third-party services that process PHI

## Integration Architecture

### Clearinghouse Placeholders

**Top US Clearinghouses (for MVP):**

1. **Change Healthcare** (largest, ~50% market share)
   - API documentation URL stored
   - Sandbox credentials placeholder
   - Status: "Integration coming soon"

2. **Availity** (major southeast US presence)
   - Portal URL, contact info
   - Status: "Integration planned"

3. **Waystar** (formerly Navicure/ZirMed)
   - Known for cloud-based solutions
   - Status: "Integration planned"

4. **Trizetto** (Cognizant)
   - Enterprise-focused
   - Status: "Integration planned"

5. **Relay Health** (McKesson)
   - Large hospital systems
   - Status: "Integration planned"

**Top US Payers (Placeholders):**
- United Healthcare
- Blue Cross Blue Shield (by state)
- Aetna (CVS Health)
- Cigna
- Humana
- Kaiser Permanente
- Centene (Medicaid)
- Medicare/Medicaid (CMS)

### Integration Module Structure

```python
backend/modules/integrations/
├── base/
│   ├── clearinghouse_interface.py    # Abstract base class
│   ├── edi_parser.py                  # X12 837/835 format handling
│   └── response_mapper.py             # Map responses to our data model
├── clearinghouses/
│   ├── change_healthcare.py           # Change Healthcare placeholder
│   ├── availity.py                    # Availity placeholder
│   ├── waystar.py                     # Waystar placeholder
│   ├── trizetto.py                    # Trizetto placeholder
│   └── relay_health.py                # Relay Health placeholder
└── payers/
    ├── united_healthcare.py           # Direct payer placeholder
    ├── blue_cross_blue_shield.py
    ├── aetna.py
    ├── cigna.py
    └── humana.py
```

### Mock Submission for MVP

When user clicks "Submit Claim":
1. Validate claim is complete
2. AI final check runs
3. Generate X12 837 format (stored, not sent)
4. Create simulated submission:
   - claim.status = "submitted"
   - claim.submitted_at = now()
   - claim_status_history: "Submitted to [Payer] via [Clearinghouse]"
5. Background job (configurable delay):
   - Simulate response: 80% approved, 15% denied, 5% pending
   - For denials: Pick realistic denial codes (CO-45, CO-16, etc.)
   - Create payment record or denial record
6. User sees realistic workflow without real integration

### Extensibility Design

**ClearinghouseInterface (Abstract Base):**
```python
class ClearinghouseInterface(ABC):
    @abstractmethod
    async def submit_claim(self, claim: Claim) -> SubmissionResponse

    @abstractmethod
    async def check_status(self, claim_id: str) -> StatusResponse

    @abstractmethod
    async def get_era(self, era_id: str) -> ERAResponse  # Electronic Remittance

    @abstractmethod
    async def validate_eligibility(self, patient: Patient) -> EligibilityResponse
```

**When Adding Real Integration (Phase 2):**
1. Implement clearinghouse-specific class
2. Add credentials to environment variables
3. Update integration_status in database
4. Toggle feature flag: `USE_REAL_SUBMISSIONS`
5. No changes to frontend or core business logic

## Implementation Phases

### Phase 1: MVP (1-2 months) - Current Focus

**Week 1-2: Foundation**
- Docker compose setup with all containers
- Database schema implementation with migrations
- Authentication & RBAC
- Basic frontend shell with routing

**Week 3-4: Core Workflow**
- Patient management (CRUD)
- Insurance plan library with US payer placeholders
- Claim creation wizard (manual entry)
- Claim list and detail views

**Week 5-6: AI Integration**
- MCP server setup with Claude API
- AI code suggestion (natural language → CPT/ICD-10)
- Real-time validation engine
- Denial risk prediction

**Week 7-8: Polish & Launch**
- Dashboard with analytics
- Claim status tracking and timeline
- Payment reconciliation
- AI insights panel
- HIPAA compliance audit
- Deployment to GCP VM

### Phase 2: Real Integrations (Post-MVP)
- Implement Change Healthcare integration first
- Real claim submission and status tracking
- ERA (Electronic Remittance Advice) parsing
- Add 2-3 more clearinghouses based on customer need

### Phase 3: Advanced Features (Future)
- EHR integrations (Epic, Cerner, Athenahealth)
- Automated denial appeals with AI-generated letters
- Predictive analytics for revenue optimization
- Multi-organization/white-label support

## Success Metrics

### MVP Success Criteria
- User can create claim in <3 minutes (vs 10+ minutes traditional)
- AI code suggestion accuracy >85%
- AI denial prediction accuracy >75% (validated over time)
- Zero security vulnerabilities on launch
- 100% HIPAA compliance checklist
- Mobile-responsive on all core workflows

### Business Metrics (Post-Launch)
- User adoption rate (active users / total signups)
- Claims processed per day per user
- AI suggestion acceptance rate
- Actual denial rate vs predicted
- Time to payment (days)
- Customer satisfaction (NPS score)

### Technical Metrics
- API response time <200ms (p95)
- MCP response time <2s for code suggestions
- System uptime >99.5%
- Database backup success rate 100%
- Zero PHI breaches

## Risks & Mitigations

### Risk 1: AI Accuracy Lower Than Expected
- **Mitigation:** Start with conservative confidence thresholds, improve with user feedback
- **Fallback:** Manual code search always available

### Risk 2: HIPAA Audit Failure
- **Mitigation:** Security review before launch, penetration testing, compliance checklist
- **Insurance:** Cyber liability insurance with BAA coverage

### Risk 3: Integration Complexity Underestimated
- **Mitigation:** Phase 2 buffer, start with sandbox environments, hire integration specialist
- **Fallback:** Focus on one clearinghouse at a time

### Risk 4: MCP/Claude API Costs Higher Than Projected
- **Mitigation:** Response caching, token usage monitoring, tiered user plans
- **Fallback:** Hybrid approach with smaller models for simple tasks

### Risk 5: Regulatory Changes
- **Mitigation:** Subscribe to CMS updates, quarterly compliance reviews
- **Buffer:** Flexible schema design for easy updates

## Appendix

### Key Technologies
- **Backend:** FastAPI, Python 3.11+, SQLAlchemy, Alembic, Pydantic
- **Frontend:** Next.js 14+, TypeScript, TailwindCSS, shadcn/ui, TanStack Query, Zustand
- **Database:** PostgreSQL 15+, Redis 7+
- **AI:** Claude API (via MCP), OpenAI (optional for specialized tasks)
- **Infrastructure:** Docker, docker-compose, nginx, GCP Compute Engine
- **Security:** TLS 1.3, AES-256, TOTP MFA

### Useful Resources
- CMS 1500 Claim Form Specifications
- X12 837 EDI Standards
- ICD-10-CM Code Set
- CPT Code Set (AMA)
- HIPAA Security Rule Technical Safeguards
- NIST Cybersecurity Framework

### Glossary
- **CPT:** Current Procedural Terminology (procedure codes)
- **ICD-10:** International Classification of Diseases (diagnosis codes)
- **EDI:** Electronic Data Interchange (claim submission format)
- **ERA:** Electronic Remittance Advice (payment explanation)
- **EOB:** Explanation of Benefits (what insurance paid/denied)
- **NPI:** National Provider Identifier
- **Clearinghouse:** Intermediary between providers and payers
- **Payer:** Insurance company that pays claims
- **RAG:** Retrieval Augmented Generation (AI technique)
- **MCP:** Model Context Protocol (Anthropic's AI integration standard)

---

**Document Status:** Ready for implementation
**Next Steps:** Create detailed implementation plan, set up git worktree, begin Phase 1 Week 1-2
