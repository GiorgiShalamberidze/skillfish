---
name: business-analyst
description: Business analysis: requirements elicitation, user story mapping, process modeling (BPMN), gap analysis, feasibility studies, and stakeholder communication.
---

# Business Analyst

End-to-end business analysis toolkit covering requirements elicitation, user story mapping, process modeling, gap analysis, feasibility studies, data analysis, stakeholder management, and documentation templates. Apply these techniques to bridge the gap between business needs and technical solutions.

---

## Table of Contents

- [Requirements Elicitation](#requirements-elicitation)
- [User Story Mapping](#user-story-mapping)
- [Process Modeling](#process-modeling)
- [Gap Analysis](#gap-analysis)
- [Feasibility Studies](#feasibility-studies)
- [Data Analysis for BA](#data-analysis-for-ba)
- [Stakeholder Management](#stakeholder-management)
- [Documentation Templates](#documentation-templates)

---

## When to Use

- Gathering, analyzing, or documenting business requirements
- Creating user story maps or slicing features into releases
- Modeling business processes with BPMN or swimlane diagrams
- Performing gap analysis between current and desired state
- Evaluating technical, financial, or operational feasibility
- Defining data requirements, flows, or entity relationships
- Managing stakeholder expectations, communication, or conflict
- Producing BRDs, functional specs, use cases, or traceability matrices

---

## Requirements Elicitation

Systematically discover, document, and validate stakeholder needs through multiple techniques.

### Elicitation Techniques

| Technique | Best For | Effort | Fidelity |
|-----------|----------|--------|----------|
| Stakeholder Interviews | Deep understanding of individual needs | Medium | High |
| Workshops (JAD) | Consensus building, complex domains | High | High |
| Observation (Job Shadowing) | Uncovering tacit knowledge, actual vs stated behavior | Medium | Very High |
| Surveys/Questionnaires | Large audience, quantitative data | Low | Medium |
| Document Analysis | Existing systems, regulatory constraints | Low | Medium |
| Prototyping | Validating UI/UX assumptions early | High | High |
| Interface Analysis | System integration points | Medium | High |

### Interview Guide Template

```
INTERVIEW PLAN
==============
Stakeholder:     [Name, Role]
Date/Time:       [Scheduled]
Duration:        [45-60 min recommended]
Objective:       [What you need to learn]

OPENING (5 min)
- Introduce purpose and confidentiality
- Confirm time available

CONTEXT QUESTIONS (10 min)
1. Describe your role and daily responsibilities.
2. What systems/tools do you use regularly?
3. Who do you interact with most in this process?

CORE QUESTIONS (25 min)
4. Walk me through [process] from start to finish.
5. Where do you experience the most friction?
6. What would an ideal outcome look like?
7. What constraints or rules must be followed?
8. How do you handle exceptions or edge cases?

VALIDATION (10 min)
9. Let me summarize what I heard: [restate]. Is that accurate?
10. What did I miss or misunderstand?

WRAP-UP (5 min)
- Next steps and follow-up timeline
- Permission to revisit for clarification
```

### Requirement Types

| Type | Definition | Example |
|------|-----------|---------|
| Business | High-level organizational objective | Reduce order processing time by 40% |
| Functional | Specific system behavior | System shall calculate tax based on shipping address |
| Non-Functional | Quality attribute or constraint | Page load time under 2 seconds at 1000 concurrent users |
| Transition | Temporary need for migration | Migrate 5 years of historical data to new schema |
| Interface | Integration with external systems | Accept payment via Stripe API v3 |
| Regulatory | Legal or compliance mandate | Store PII encrypted at rest per GDPR Article 32 |

### MoSCoW Prioritization

Categorize every requirement into one of four buckets with the full stakeholder group:

| Priority | Meaning | Rule of Thumb |
|----------|---------|---------------|
| **Must Have** | Non-negotiable for launch; system is unusable without it | ~60% of effort |
| **Should Have** | Important but workarounds exist; painful to omit | ~20% of effort |
| **Could Have** | Desirable enhancement; include if time permits | ~20% of effort |
| **Won't Have (this time)** | Explicitly out of scope for this release; may return later | Documented backlog |

**Example MoSCoW Table:**

| ID | Requirement | Priority | Rationale |
|----|------------|----------|-----------|
| R-001 | User login with MFA | Must | Security compliance |
| R-002 | Dashboard with real-time KPIs | Should | Manual report workaround exists |
| R-003 | Dark mode UI theme | Could | User preference, not critical |
| R-004 | AI-powered forecasting | Won't | Deferred to Phase 2 |

### Anti-Patterns

- **Leading questions** -- "Don't you think X would be better?" biases the answer. Ask open-ended questions instead.
- **Skipping observation** -- Relying solely on interviews misses tacit knowledge. Always observe real workflows.
- **Gold-plating requirements** -- Adding "nice to have" items without stakeholder validation inflates scope.
- **Single-source elicitation** -- Using only one technique produces blind spots. Combine at least two methods.

---

## User Story Mapping

Organize user activities into a visual map that drives incremental delivery.

### Story Map Structure

```
                        BACKBONE (User Activities)
    +-----------+    +-----------+    +-----------+    +-----------+
    | Activity 1|    | Activity 2|    | Activity 3|    | Activity 4|
    +-----------+    +-----------+    +-----------+    +-----------+
         |                |                |                |
    USER TASKS        USER TASKS       USER TASKS      USER TASKS
    +---------+      +---------+      +---------+      +---------+
    | Task 1a |      | Task 2a |      | Task 3a |      | Task 4a |
    | Task 1b |      | Task 2b |      | Task 3b |      | Task 4b |
    | Task 1c |      |         |      | Task 3c |      |         |
    +---------+      +---------+      +---------+      +---------+
         |                |                |                |
    DETAILS/STORIES   DETAILS         DETAILS           DETAILS
    +---------+      +---------+      +---------+      +---------+
    | Story A |      | Story D |      | Story F |      | Story H |
    | Story B |      | Story E |      | Story G |      | Story I |
    | Story C |      |         |      |         |      |         |
    +---------+      +---------+      +---------+      +---------+

    ---- Release 1 line ----    (Walking skeleton / MVP)
    ---- Release 2 line ----    (Enhanced functionality)
    ---- Release 3 line ----    (Full feature set)
```

### Building the Backbone

1. **Identify the user persona** performing the journey
2. **List user activities** left to right in chronological order (the "big things" they do)
3. **Break activities into tasks** -- the steps within each activity
4. **Add detail stories** beneath each task -- specific functionality needed
5. **Arrange vertically by priority** -- most critical stories at the top

### Slicing for Releases

| Slice | Goal | Criteria |
|-------|------|----------|
| Release 1 (MVP) | Walking skeleton -- end-to-end flow with minimum functionality | User can complete the core journey, even if rough |
| Release 2 | Usable product -- key pain points addressed | Covers 80% of common scenarios |
| Release 3 | Feature-complete -- edge cases and enhancements | Full scope delivered |

**Slicing rules:**
- Each release must deliver a coherent, testable user journey
- Slice thin across the backbone, not deep in one activity
- Every slice should be independently deployable and valuable

### Acceptance Criteria with Gherkin

Write acceptance criteria in Gherkin syntax for unambiguous, testable specifications:

```gherkin
Feature: Shopping Cart Checkout
  As a returning customer
  I want to check out with my saved payment method
  So that I can complete purchases quickly

  Scenario: Successful checkout with saved card
    Given I have items in my cart totaling $50.00
    And I have a saved Visa ending in 4242
    When I select "Pay with saved card"
    And I confirm the purchase
    Then the order is placed successfully
    And I receive a confirmation email within 60 seconds
    And my card is charged $50.00

  Scenario: Checkout with expired saved card
    Given I have items in my cart
    And my saved card expired last month
    When I select "Pay with saved card"
    Then I see a message "Your card has expired. Please update your payment method."
    And I am redirected to the payment update page

  Scenario Outline: Checkout with various totals
    Given I have items in my cart totaling <amount>
    When I complete checkout
    Then my card is charged <amount>
    And the receipt shows <amount>

    Examples:
      | amount   |
      | $10.00   |
      | $99.99   |
      | $1000.00 |
```

### Anti-Patterns

- **Flat backlog syndrome** -- A prioritized list without a story map loses the narrative flow. Always map before slicing.
- **Thick slices** -- Releasing one fully complete activity instead of thin cross-cutting slices delays feedback.
- **Missing acceptance criteria** -- Stories without Gherkin scenarios become ambiguous during development.
- **Ignoring the backbone** -- Jumping to detail stories without establishing activities and tasks produces fragmented scope.

---

## Process Modeling

Visualize and analyze business processes using standardized notation.

### BPMN 2.0 Core Elements

| Symbol | Name | Use |
|--------|------|-----|
| Circle (thin border) | Start Event | Entry point of the process |
| Circle (thick border) | End Event | Termination point |
| Rounded rectangle | Task/Activity | A unit of work |
| Diamond | Gateway | Decision or parallel split |
| Diamond with X | Exclusive Gateway | XOR -- exactly one path taken |
| Diamond with + | Parallel Gateway | AND -- all paths taken simultaneously |
| Diamond with O | Inclusive Gateway | OR -- one or more paths taken |
| Dashed rectangle | Subprocess | Collapsed group of activities |
| Dotted arrow | Message Flow | Communication between pools |
| Solid arrow | Sequence Flow | Order of execution |

### BPMN Process Template (Text Notation)

```
PROCESS: Order Fulfillment
OWNER: Operations Department

[Start: Order Received]
    |
    v
[Task: Validate Order]
    |
    v
<XOR Gateway: Order Valid?>
    |                    |
    | Yes                | No
    v                    v
[Task: Check          [Task: Notify Customer
 Inventory]            of Invalid Order]
    |                    |
    v                    v
<XOR Gateway:         [End: Order Rejected]
 In Stock?>
    |            |
    | Yes        | No
    v            v
[Task: Pick    [Task: Back-order
 & Pack]        Item]
    |            |
    v            v
[Task: Ship   [Task: Notify Customer
 Order]        of Delay]
    |            |
    v            |
[End: Order    [Timer: Item Arrives]
 Delivered]      |
                 v
               [Task: Pick & Pack]
                 |
                 v
               [Task: Ship Order]
                 |
                 v
               [End: Order Delivered]
```

### Swimlane Diagram Template

```
+------------------+------------------+------------------+
|   Customer       |   Sales Team     |   Warehouse      |
+------------------+------------------+------------------+
|                  |                  |                  |
| [Place Order] -->| [Review Order]   |                  |
|                  |       |          |                  |
|                  |       v          |                  |
|                  | [Approve?]       |                  |
|                  |  Yes  |  No     |                  |
|                  |   |   |-------->|                  |
|                  |   v        Reject notification     |
|                  | [Send to   |    |                  |
|                  |  Warehouse]|--->| [Pick Items]     |
|                  |            |    |       |          |
|                  |            |    |       v          |
|                  |            |    | [Pack & Ship]    |
|                  |            |    |       |          |
| [Receive         |            |    |       v          |
|  Package] <------|------------|----| [Dispatch]       |
|                  |            |    |                  |
+------------------+------------------+------------------+
```

### As-Is vs To-Be Analysis

| Dimension | As-Is (Current State) | To-Be (Future State) |
|-----------|----------------------|---------------------|
| **Purpose** | Document how things work today | Design the improved process |
| **Method** | Observation, interviews, document review | Workshops, benchmarking, best practices |
| **Notation** | BPMN with pain-point annotations | BPMN with improvement annotations |
| **Output** | Current process map + issue register | Target process map + transition plan |
| **Timing** | Always do first | After gap analysis |

**Process improvement indicators to annotate on As-Is maps:**
- Bottlenecks (where work queues)
- Manual handoffs (error-prone transitions)
- Redundant approvals (unnecessary gates)
- Rework loops (repeated steps due to defects)
- Wait states (idle time between activities)

### Value Stream Mapping

```
VALUE STREAM MAP: Customer Onboarding

Process Step     | Lead Time | Process Time | %C&A | FTE
=================|===========|==============|======|====
Application      |   2 days  |    30 min    |  70% | 0.5
  --> queue                                         
Credit Check     |   3 days  |    15 min    |  90% | 0.3
  --> queue                                         
Account Setup    |   1 day   |    45 min    |  85% | 1.0
  --> queue                                         
Welcome Kit      |   2 days  |    10 min    |  95% | 0.2
  --> queue                                         
First Contact    |   1 day   |    20 min    |  80% | 0.5

TOTALS:
  Total Lead Time:    9 days
  Total Process Time: 2 hours
  Process Efficiency: 2.8% (Process Time / Lead Time)
  Target Efficiency:  15%+
```

### Anti-Patterns

- **Modeling the exception first** -- Start with the happy path, then layer in exceptions. Otherwise diagrams become unreadable.
- **Over-detailed BPMN** -- A single diagram with 50+ tasks is unusable. Decompose into subprocesses.
- **Skipping as-is** -- Jumping to to-be without documenting current state leads to unrealistic designs.
- **Pool/lane confusion** -- Pools represent separate organizations; lanes represent roles within one organization. Mixing them produces incorrect models.

---

## Gap Analysis

Identify and prioritize the differences between current capabilities and desired outcomes.

### Gap Analysis Framework

```
CURRENT STATE          GAP               DESIRED STATE
+------------+    +----------+    +----------------+
| Where we   |    | What's   |    | Where we need  |
| are today  |--->| missing  |--->| to be          |
+------------+    +----------+    +----------------+
      |                |                   |
   Assessed via     Quantified as       Defined by
   observation,     measurable          business
   metrics, audit   deltas              objectives
```

### Current State Assessment Template

```
CURRENT STATE ASSESSMENT
========================
Domain:          [Process / System / Capability]
Assessed By:     [Analyst Name]
Date:            [Assessment Date]
Method:          [Observation / Interview / Audit / Metrics Review]

CAPABILITY INVENTORY
| Capability       | Maturity (1-5) | Evidence              | Owner        |
|-----------------|----------------|----------------------|--------------|
| [Capability 1]  | [Score]        | [How you know]       | [Who owns]   |
| [Capability 2]  | [Score]        | [How you know]       | [Who owns]   |
| [Capability 3]  | [Score]        | [How you know]       | [Who owns]   |

PERFORMANCE METRICS (Baseline)
| Metric                | Current Value | Measurement Period |
|-----------------------|---------------|-------------------|
| [Processing Time]     | [Value]       | [Period]          |
| [Error Rate]          | [Value]       | [Period]          |
| [Customer Satisfaction]| [Value]      | [Period]          |
| [Throughput]          | [Value]       | [Period]          |

KEY FINDINGS
1. [Finding with supporting data]
2. [Finding with supporting data]
3. [Finding with supporting data]
```

### Gap Identification Matrix

| ID | Area | Current State | Desired State | Gap Description | Impact (H/M/L) | Effort (H/M/L) | Priority |
|----|------|--------------|---------------|-----------------|-----------------|-----------------|----------|
| G-001 | Process | Manual data entry, 20 errors/week | Automated ingestion, <2 errors/week | No integration between CRM and ERP | High | Medium | 1 |
| G-002 | People | 2 staff trained on new platform | All 15 staff proficient | Training program not yet developed | Medium | Low | 2 |
| G-003 | Technology | On-premise SQL Server 2012 | Cloud-native managed database | Legacy infrastructure, no migration plan | High | High | 3 |
| G-004 | Data | Inconsistent customer records across 3 systems | Single source of truth | No master data management strategy | High | High | 4 |

### Capability Maturity Model

Score each capability on a 1-5 scale:

| Level | Name | Description |
|-------|------|-------------|
| 1 | Initial | Ad hoc, unpredictable, reactive |
| 2 | Managed | Planned and tracked at project level |
| 3 | Defined | Standardized process across the organization |
| 4 | Quantitatively Managed | Measured with statistical process control |
| 5 | Optimizing | Continuous improvement with innovation |

### Action Planning Template

```
GAP CLOSURE ACTION PLAN
========================
Gap ID:          [G-XXX]
Gap Description: [What is missing]
Owner:           [Accountable person]
Target Date:     [Deadline]

ACTIONS
| # | Action                     | Responsible | Deadline   | Status      |
|---|----------------------------|-------------|------------|-------------|
| 1 | [Specific action step]     | [Name]      | [Date]     | Not Started |
| 2 | [Specific action step]     | [Name]      | [Date]     | Not Started |
| 3 | [Specific action step]     | [Name]      | [Date]     | Not Started |

DEPENDENCIES
- [Dependency 1: e.g., Budget approval from CFO]
- [Dependency 2: e.g., Vendor contract signed]

RISKS
- [Risk 1 + mitigation]
- [Risk 2 + mitigation]

SUCCESS CRITERIA
- [Measurable outcome 1]
- [Measurable outcome 2]
```

### Anti-Patterns

- **Vague gap descriptions** -- "We need better reporting" is not a gap. Quantify: "Reports take 4 hours to produce manually; target is <5 minutes automated."
- **Ignoring root cause** -- Treating symptoms (e.g., "hire more staff") instead of analyzing why the gap exists (e.g., broken process).
- **Unpriorized gap list** -- Listing 50 gaps without impact/effort scoring makes action planning impossible.
- **Static analysis** -- Performing gap analysis once and never revisiting. Gaps shift as the organization evolves.

---

## Feasibility Studies

Evaluate whether a proposed solution is viable across technical, financial, and operational dimensions.

### Feasibility Dimensions

| Dimension | Key Questions | Assessment Method |
|-----------|--------------|-------------------|
| **Technical** | Can we build it? Do we have the skills and infrastructure? | Architecture review, PoC, vendor assessment |
| **Economic** | Does the investment pay off? What is the ROI? | Cost-benefit analysis, NPV, payback period |
| **Operational** | Will users adopt it? Can we support it? | Change readiness assessment, training plan |
| **Schedule** | Can we deliver on time? | Resource analysis, critical path estimation |
| **Legal/Regulatory** | Are there compliance or legal barriers? | Regulatory review, legal consultation |

### Technical Feasibility Assessment

```
TECHNICAL FEASIBILITY
=====================
Proposed Solution: [Solution name]
Assessed By:       [Architect / Tech Lead]
Date:              [Date]

TECHNOLOGY STACK EVALUATION
| Component       | Proposed Technology | Team Expertise (1-5) | Market Maturity | Risk |
|----------------|--------------------|-----------------------|-----------------|------|
| Frontend       | [Technology]       | [Score]               | [Mature/Growth] | [H/M/L] |
| Backend        | [Technology]       | [Score]               | [Mature/Growth] | [H/M/L] |
| Database       | [Technology]       | [Score]               | [Mature/Growth] | [H/M/L] |
| Infrastructure | [Technology]       | [Score]               | [Mature/Growth] | [H/M/L] |

INTEGRATION REQUIREMENTS
| External System  | Integration Method | Complexity | API Available? |
|-----------------|-------------------|------------|----------------|
| [System 1]      | [REST/SOAP/File]  | [H/M/L]   | [Yes/No]       |
| [System 2]      | [REST/SOAP/File]  | [H/M/L]   | [Yes/No]       |

PROOF OF CONCEPT RESULTS
- [PoC finding 1]
- [PoC finding 2]

VERDICT: [Feasible / Feasible with Conditions / Not Feasible]
CONDITIONS: [If applicable]
```

### ROI and NPV Calculation

```
FINANCIAL FEASIBILITY
=====================
Project:         [Name]
Time Horizon:    [3-5 years typical]
Discount Rate:   [Organization's WACC or hurdle rate, e.g., 10%]

COST SUMMARY
| Category              | Year 0    | Year 1    | Year 2    | Year 3    |
|-----------------------|-----------|-----------|-----------|-----------|
| Development           | $200,000  | $0        | $0        | $0        |
| Infrastructure        | $50,000   | $30,000   | $30,000   | $30,000   |
| Licensing             | $0        | $24,000   | $24,000   | $24,000   |
| Training & Change Mgmt| $30,000   | $10,000   | $5,000    | $5,000    |
| Maintenance/Support   | $0        | $40,000   | $40,000   | $40,000   |
| Total Costs           | $280,000  | $104,000  | $99,000   | $99,000   |

BENEFIT SUMMARY
| Category              | Year 0    | Year 1    | Year 2    | Year 3    |
|-----------------------|-----------|-----------|-----------|-----------|
| Labor savings         | $0        | $150,000  | $150,000  | $150,000  |
| Error reduction       | $0        | $50,000   | $50,000   | $50,000   |
| Revenue increase      | $0        | $100,000  | $200,000  | $300,000  |
| Total Benefits        | $0        | $300,000  | $400,000  | $500,000  |

NET CASH FLOW
| Year    | Net Flow   | Discount Factor | Present Value |
|---------|-----------|-----------------|---------------|
| Year 0  | -$280,000 | 1.000           | -$280,000     |
| Year 1  | $196,000  | 0.909           | $178,164      |
| Year 2  | $301,000  | 0.826           | $248,626      |
| Year 3  | $401,000  | 0.751           | $301,151      |

NPV = $447,941
ROI = (Total Benefits - Total Costs) / Total Costs = 107%
Payback Period = 1.4 years
```

### Recommendation Framework

| NPV | ROI | Payback | Recommendation |
|-----|-----|---------|----------------|
| > 0, strong | > 50% | < 2 years | Strong Go |
| > 0, moderate | 20-50% | 2-3 years | Conditional Go (mitigate risks) |
| ~ 0 or marginal | < 20% | 3-5 years | Defer or redesign |
| < 0 | Negative | > 5 years | No Go |

### Anti-Patterns

- **Sunk cost bias** -- Continuing a project because of past investment rather than future value. Evaluate feasibility based on forward-looking data only.
- **Optimism bias in estimates** -- Understating costs and overstating benefits. Add contingency (15-25%) and discount benefit projections.
- **Ignoring operational feasibility** -- A technically and financially sound solution that users refuse to adopt is still a failure.
- **Single-option analysis** -- Evaluating only one solution. Always compare at least two alternatives plus "do nothing."

---

## Data Analysis for BA

Define data requirements, model relationships, and document data flows.

### Data Requirements Specification

```
DATA REQUIREMENTS
=================
Project:     [Name]
Domain:      [Business domain]

DATA ENTITIES
| Entity        | Description                    | Source System    | Volume      | Growth Rate |
|--------------|-------------------------------|-----------------|-------------|-------------|
| Customer     | Individual or business client   | CRM (Salesforce) | 500,000     | 5K/month    |
| Order        | Purchase transaction            | ERP (SAP)        | 2,000,000   | 50K/month   |
| Product      | Item available for sale          | PIM              | 15,000      | 100/month   |
| Invoice      | Billing record                  | Finance System   | 1,800,000   | 45K/month   |

DATA QUALITY REQUIREMENTS
| Entity    | Attribute      | Rule                          | Current Compliance |
|-----------|---------------|-------------------------------|-------------------|
| Customer  | Email          | Valid email format, unique      | 85%               |
| Customer  | Phone          | E.164 format                   | 60%               |
| Order     | Total Amount   | > $0, matches line items sum   | 99%               |
| Product   | SKU            | Unique, alphanumeric, 8 chars  | 95%               |
```

### Data Flow Diagram (DFD)

```
CONTEXT DIAGRAM (Level 0)
=========================

  +----------+                              +----------+
  | Customer |---[Order Request]----------->|          |
  |          |<--[Order Confirmation]-------|  Order   |
  +----------+                              | Processing|
                                            |  System  |
  +----------+                              |          |
  | Warehouse|<--[Pick List]---------------|          |
  |          |---[Shipment Confirmation]--->|          |
  +----------+                              +----------+
                                                 |
  +----------+                                   |
  | Finance  |<--[Invoice Data]------------------+
  |  System  |---[Payment Status]--------------->|
  +----------+


LEVEL 1 DFD: Order Processing
===============================

[Customer]--{Order}-->[1.0 Validate Order]
                           |
                       {Valid Order}
                           |
                           v
                      [2.0 Check Inventory]<-->{Product DB}
                           |
                      {Available Items}
                           |
                           v
                      [3.0 Process Payment]<-->[Finance System]
                           |
                       {Payment OK}
                           |
                           v
                      [4.0 Fulfill Order]---->{Order DB}
                           |
                       {Pick List}
                           |
                           v
                       [Warehouse]
```

### Entity-Relationship Model

```
ER MODEL: E-Commerce Domain
============================

  +------------+       +-------------+       +-----------+
  | CUSTOMER   |       | ORDER       |       | PRODUCT   |
  |------------|       |-------------|       |-----------|
  | *customer_id|  1  M| *order_id   |  M  M| *product_id|
  | name        |------| order_date  |------| name       |
  | email       |      | total_amount|      | price      |
  | phone       |      | status      |      | category   |
  | address     |      | customer_id |      | sku        |
  +------------+       +-------------+       +-----------+
                              |
                              | 1
                              |
                              | M
                       +-------------+
                       | ORDER_LINE  |
                       |-------------|
                       | *line_id    |
                       | order_id    |
                       | product_id  |
                       | quantity    |
                       | unit_price  |
                       +-------------+

  LEGEND:  * = Primary Key    1 = One    M = Many
           --- = Relationship line
```

### Data Dictionary Template

| Field Name | Entity | Data Type | Length | Required | Default | Business Rule | Example |
|-----------|--------|-----------|--------|----------|---------|---------------|---------|
| customer_id | Customer | UUID | 36 | Yes | Auto-generated | Unique identifier | 550e8400-e29b... |
| email | Customer | VARCHAR | 255 | Yes | None | Valid format, unique per customer | jane@example.com |
| order_date | Order | TIMESTAMP | - | Yes | Current timestamp | Cannot be future date | 2026-04-05T14:30:00Z |
| total_amount | Order | DECIMAL | 10,2 | Yes | 0.00 | Sum of line items + tax - discount | 149.99 |
| quantity | Order_Line | INTEGER | - | Yes | 1 | Must be > 0, <= available stock | 3 |
| status | Order | ENUM | - | Yes | PENDING | One of: PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED | CONFIRMED |

### Anti-Patterns

- **Undefined data ownership** -- If no one owns the data, no one maintains its quality. Assign a data steward per entity.
- **Skipping data quality assessment** -- Assuming source data is clean leads to failed integrations and bad analytics.
- **Logical-physical confusion** -- Mixing logical ER models (business concepts) with physical database design (indexes, partitions) in the same diagram.
- **Missing data lineage** -- Not documenting where data originates, transforms, and terminates makes troubleshooting impossible.

---

## Stakeholder Management

Identify, analyze, and engage stakeholders to build alignment and manage expectations.

### Stakeholder Mapping (Power/Interest Grid)

```
                        INTEREST
               Low                    High
          +----------------+------------------+
          |                |                  |
    High  |   KEEP         |   MANAGE         |
          |   SATISFIED    |   CLOSELY        |
  POWER   |                |                  |
          | (CFO, Legal)   | (Sponsor, Key    |
          |                |  Users, CTO)     |
          +----------------+------------------+
          |                |                  |
    Low   |   MONITOR      |   KEEP           |
          |   (Minimal     |   INFORMED       |
          |    effort)     |                  |
          | (External      | (End users,      |
          |  vendors)      |  support team)   |
          +----------------+------------------+
```

### Stakeholder Register

| Stakeholder | Role | Power | Interest | Attitude | Strategy | Primary Concern |
|------------|------|-------|----------|----------|----------|-----------------|
| Sarah Chen | VP Operations | High | High | Champion | Manage closely, involve in decisions | Process efficiency |
| Mark Johnson | CFO | High | Low | Neutral | Keep satisfied, provide ROI updates | Budget impact |
| Dev Team | Engineering | Medium | High | Supportive | Keep informed, solicit input | Technical feasibility |
| Call Center Staff | End Users | Low | High | Resistant | Keep informed, address concerns early | Job security |
| Acme Corp | Vendor | Low | Low | Neutral | Monitor | Contract terms |

### RACI Matrix

Define roles for every major deliverable or decision:

| Deliverable / Decision | BA | Sponsor | Dev Lead | QA Lead | PM |
|------------------------|-----|---------|----------|---------|-----|
| Requirements Document | **R** | **A** | C | I | C |
| Solution Design | C | A | **R** | C | I |
| Test Strategy | C | I | C | **R** | **A** |
| Go/No-Go Decision | C | **R,A** | C | C | C |
| Training Material | **R** | I | C | I | **A** |
| Change Request Approval | C | **A** | **R** | I | C |

**Legend:** **R** = Responsible (does the work), **A** = Accountable (final decision), **C** = Consulted (input needed), **I** = Informed (notified of outcome)

**RACI Rules:**
- Every row must have exactly one **A**
- Every row must have at least one **R**
- Minimize the number of **C** entries to avoid decision paralysis
- **A** and **R** can be the same person but ideally are not for governance

### Communication Plan

| Stakeholder Group | Channel | Frequency | Content | Owner | Feedback Loop |
|-------------------|---------|-----------|---------|-------|---------------|
| Steering Committee | In-person meeting | Bi-weekly | Status, risks, decisions needed | PM | Action items tracked in JIRA |
| Sponsor | 1:1 meeting | Weekly | Progress, blockers, budget | BA | Email follow-up within 24h |
| Dev Team | Stand-up + Slack | Daily | Requirements clarification, priorities | BA | Slack thread resolution |
| End Users | Newsletter + demos | Monthly | Feature previews, training schedule | Change Lead | Survey after each demo |
| Executives | Email dashboard | Monthly | KPIs, milestones, ROI tracking | PM | Reply-requested for decisions |

### Conflict Resolution Framework

When stakeholders disagree on requirements or priorities:

1. **Acknowledge** -- Validate each party's concern and restate their position
2. **Analyze** -- Identify the root cause of disagreement (data, values, interests, relationships)
3. **Options** -- Generate at least three resolution options collaboratively
4. **Evaluate** -- Score options against agreed criteria (business value, cost, risk, timeline)
5. **Decide** -- Escalate to the accountable stakeholder (the "A" in RACI) if consensus fails
6. **Document** -- Record the decision, rationale, and dissenting views in the decision log

**Decision Log Entry:**

```
DECISION LOG
=============
Decision ID:    DEC-042
Date:           2026-04-05
Topic:          Authentication method for customer portal
Options:        (A) SSO via SAML  (B) Username/Password  (C) Social login only
Decision:       Option A -- SSO via SAML
Decided By:     Sarah Chen (VP Operations)
Rationale:      Enterprise customers require SSO; security audit mandates it
Dissent:        Dev team preferred Option B for faster delivery
Impact:         +3 weeks to timeline, +$15K for identity provider integration
```

### Anti-Patterns

- **Stakeholder neglect** -- Identifying stakeholders once and never revisiting. New stakeholders emerge; attitudes shift.
- **Over-communication** -- Sending everything to everyone creates noise. Tailor content to each group.
- **Avoiding conflict** -- Unresolved disagreements fester and surface later as scope creep or resistance.
- **Missing RACI** -- Without clear accountability, decisions stall or get made by the wrong person.

---

## Documentation Templates

Standardized templates for core BA deliverables.

### Business Requirements Document (BRD)

```
BUSINESS REQUIREMENTS DOCUMENT
================================
Project Name:    [Name]
Version:         [1.0]
Author:          [BA Name]
Date:            [Date]
Status:          [Draft / In Review / Approved]
Approvers:       [Names and roles]

1. EXECUTIVE SUMMARY
   [2-3 paragraphs: what, why, expected outcome]

2. BUSINESS OBJECTIVES
   | ID    | Objective                              | Success Metric        | Target    |
   |-------|----------------------------------------|-----------------------|-----------|
   | BO-01 | [Objective]                            | [How measured]        | [Value]   |
   | BO-02 | [Objective]                            | [How measured]        | [Value]   |

3. SCOPE
   3.1 In Scope
       - [Item 1]
       - [Item 2]
   3.2 Out of Scope
       - [Item 1]
       - [Item 2]
   3.3 Assumptions
       - [Assumption 1]
       - [Assumption 2]
   3.4 Constraints
       - [Constraint 1]
       - [Constraint 2]

4. STAKEHOLDERS
   [Reference stakeholder register]

5. REQUIREMENTS
   5.1 Business Requirements
       [Reference MoSCoW-prioritized requirements table]
   5.2 Functional Requirements
       [Detailed functional specifications]
   5.3 Non-Functional Requirements
       [Performance, security, availability, etc.]

6. PROCESS FLOWS
   [Reference BPMN diagrams]

7. DATA REQUIREMENTS
   [Reference data dictionary and ER model]

8. ACCEPTANCE CRITERIA
   [Reference Gherkin scenarios]

9. RISKS AND DEPENDENCIES
   | ID    | Risk/Dependency | Impact | Likelihood | Mitigation            |
   |-------|----------------|--------|------------|----------------------|
   | R-01  | [Description]  | [H/M/L]| [H/M/L]   | [Mitigation action]  |

10. APPROVAL
    | Name           | Role              | Signature | Date |
    |----------------|-------------------|-----------|------|
    | [Approver 1]   | [Role]            |           |      |
    | [Approver 2]   | [Role]            |           |      |

APPENDICES
A. Glossary
B. Reference Documents
C. Revision History
```

### Functional Specification Template

```
FUNCTIONAL SPECIFICATION
=========================
Feature:         [Feature Name]
Module:          [System Module]
Author:          [BA Name]
Version:         [1.0]
Status:          [Draft / Approved]

1. PURPOSE
   [Why this feature exists, which business requirement it addresses]

2. USER ROLES
   | Role          | Permissions              | Notes                    |
   |---------------|--------------------------|--------------------------|
   | [Role 1]      | [Read/Write/Admin]       | [Context]                |
   | [Role 2]      | [Read/Write/Admin]       | [Context]                |

3. FUNCTIONAL DESCRIPTION
   3.1 [Sub-feature 1]
       - Input: [What the user provides]
       - Processing: [What the system does]
       - Output: [What the user sees/receives]
       - Business Rules:
         * [Rule 1]
         * [Rule 2]

   3.2 [Sub-feature 2]
       - Input: [...]
       - Processing: [...]
       - Output: [...]
       - Business Rules:
         * [...]

4. UI/UX REQUIREMENTS
   - [Wireframe reference or description]
   - [Interaction patterns]
   - [Accessibility requirements]

5. VALIDATION RULES
   | Field          | Rule                     | Error Message              |
   |----------------|--------------------------|---------------------------|
   | [Field 1]      | [Validation]             | [User-facing message]     |
   | [Field 2]      | [Validation]             | [User-facing message]     |

6. ERROR HANDLING
   | Condition        | System Behavior          | User Message              |
   |------------------|--------------------------|---------------------------|
   | [Error 1]        | [What happens]           | [What user sees]          |
   | [Error 2]        | [What happens]           | [What user sees]          |

7. DEPENDENCIES
   - [System / service dependency 1]
   - [System / service dependency 2]

8. ACCEPTANCE CRITERIA
   [Gherkin scenarios]
```

### Use Case Template

```
USE CASE SPECIFICATION
=======================
Use Case ID:     UC-[XXX]
Use Case Name:   [Descriptive Name]
Actor(s):        [Primary actor, secondary actors]
Priority:        [Must / Should / Could]
Status:          [Draft / Approved]

DESCRIPTION
[1-2 sentence summary of what the actor accomplishes]

PRECONDITIONS
- [What must be true before the use case starts]
- [System state, user state, data state]

TRIGGER
[What initiates this use case]

MAIN SUCCESS SCENARIO (Happy Path)
| Step | Actor Action                    | System Response                  |
|------|---------------------------------|----------------------------------|
| 1    | [Actor does something]          | [System responds]                |
| 2    | [Actor does something]          | [System responds]                |
| 3    | [Actor does something]          | [System responds]                |
| 4    | [Actor confirms]                | [System completes operation]     |

EXTENSIONS (Alternate/Exception Flows)
| Step | Condition                       | Action                           |
|------|---------------------------------|----------------------------------|
| 2a   | [Invalid input]                 | System displays error, return to 2 |
| 3a   | [Timeout]                       | System saves draft, notifies user  |
| 3b   | [Insufficient permissions]      | System denies and logs attempt     |

POSTCONDITIONS
Success:
- [What is true after successful completion]
Failure:
- [What is true after failure -- system state preserved]

BUSINESS RULES
- [BR-01: Rule that governs this use case]
- [BR-02: Rule that governs this use case]

RELATED USE CASES
- [UC-YYY: Included use case]
- [UC-ZZZ: Extending use case]
```

### Requirements Traceability Matrix

```
REQUIREMENTS TRACEABILITY MATRIX
==================================
Project:     [Name]
Last Updated: [Date]

| Req ID | Requirement Description          | Source      | Priority | Design Ref | Test Case | Status     |
|--------|----------------------------------|------------|----------|------------|-----------|------------|
| R-001  | User login with MFA              | BRD 5.1    | Must     | DS-004     | TC-010    | Implemented|
| R-002  | Dashboard KPI display            | BRD 5.2    | Should   | DS-007     | TC-015    | In Dev     |
| R-003  | Export to PDF                    | Interview-3| Must     | DS-012     | TC-022    | Not Started|
| R-004  | Role-based access control        | BRD 5.3    | Must     | DS-005     | TC-011    | In QA      |
| R-005  | Email notification on approval   | Workshop-2 | Should   | DS-018     | TC-030    | Not Started|
| R-006  | Mobile responsive layout         | BRD 5.4    | Must     | DS-009     | TC-025    | In Dev     |

COVERAGE SUMMARY
| Status       | Count | Percentage |
|-------------|-------|------------|
| Implemented | 1     | 17%        |
| In Dev      | 2     | 33%        |
| In QA       | 1     | 17%        |
| Not Started | 2     | 33%        |
| Total       | 6     | 100%       |

TRACEABILITY GAPS
- [R-XXX: Missing test case -- needs QA assignment]
- [R-YYY: Missing design reference -- needs architecture review]
```

### Anti-Patterns

- **Template worship** -- Filling every field mechanically without judgment. Adapt templates to project size: a 2-week project does not need a 40-page BRD.
- **Write-and-forget** -- Documentation that is never updated becomes misleading. Assign a review cadence.
- **Ambiguous language** -- Words like "fast," "user-friendly," and "intuitive" are untestable. Replace with measurable criteria.
- **Missing traceability** -- Requirements without links to tests and design create coverage gaps discovered only in UAT.
