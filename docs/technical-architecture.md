# LinguaTech Solutions Email Management System

## Scope and evidence

This document describes the architecture represented by the seven n8n workflow exports in this folder:

- `workflows/main/email-management-system.json`
- `workflows/handlers/client-support.json`
- `workflows/handlers/projects-and-service-inquiries.json`
- `workflows/handlers/billing-and-payments.json`
- `workflows/handlers/team-and-operations.json`
- `workflows/data-loaders/load-faq-to-pinecone.json`
- `workflows/data-loaders/load-services-to-pinecone.json`

The description is based on configured nodes, expressions, connections, workflow IDs, credentials, prompts, and activation flags in those files. It describes the exported configuration, not a claim that every external credential, Gmail label, spreadsheet, or Pinecone index is currently available.

## Executive architecture

The system is an email-driven, agentic workflow platform:

1. A Gmail polling workflow reads new inbox messages.
2. The message is enriched with client data from a Google Sheet.
3. An OpenAI-backed classifier assigns one of four business categories.
4. The router invokes a category-specific child workflow using `Execute Workflow`.
5. The child workflow performs domain-specific LLM processing and side effects.
6. Gmail drafts or notifications, Telegram alerts, Google Sheet audit rows, Gmail labels, and—in the finance case—Google Drive files are produced.
7. Two manual ingestion workflows populate a shared Pinecone index with FAQ/policy and service-catalogue knowledge.

The runtime topology is therefore:

```text
Gmail inbox
   |
   v
Email Management System (poll every minute)
   |-- Google Sheets: client registry lookup
   |-- OpenAI: category classification
   |
   +--> Client Support child workflow
   +--> Billing & Payments child workflow
   +--> Projects & Service Inquiries child workflow
   +--> Team & Operations child workflow
   `--> Other: no-op + audit row

Knowledge loaders (manual)
   +--> OpenAI embeddings --> Pinecone customer-support / FAQ
   `--> OpenAI embeddings --> Pinecone customer-support / services
```

## Workflow inventory and activation

| Workflow | Role | Trigger | Exported active flag |
|---|---|---|---|
| `email-management-system` | Intake, enrichment, classification, dispatch | Gmail Trigger, every minute | `false` |
| `client-support` | Support response drafting, policy retrieval, sentiment and escalation | Execute Workflow Trigger | `true` |
| `projects-and-service-inquiries` | Lead/service response drafting, review, approval and sending | Execute Workflow Trigger | `true` |
| `billing-and-payments` | Financial direction classification and invoice/receipt processing | Execute Workflow Trigger | `true` |
| `team-and-operations` | Internal-email summarization and notification | Execute Workflow Trigger | `true` |
| `load-faq-to-pinecone` | Manual FAQ/policy ingestion | Manual Trigger | `false` |
| `load-services-to-pinecone` | Manual services-catalogue ingestion | Manual Trigger | `false` |

Because the main Gmail router is exported inactive, the child workflows being active does not by itself create an active end-to-end email service. The two knowledge loaders are also inactive and must be run manually when their embedded source text changes.

## Common message contract

The router normalizes each Gmail message to the fields passed to all child workflows:

- `messageId`, `threadId`
- `senderName`, `senderEmail`
- `subject`, `emailBody`
- `existingClient`, `clientName`, `company`
- `clientSince`, `plan`, `lastOrder`, `notes`

The client registry lookup uses the sender address, extracting the address from a `From` header when it is in angle brackets and lower-casing it. The result is returned as the first matching row from the `Clients` Google Sheet. Missing client fields are converted to empty strings; `existingClient` is represented as the strings `yes` or `no`.

All child workflows use an `Execute Workflow Trigger` named `Start` and accept this normalized context. Conversation memory, where configured, uses the Gmail `threadId` as a custom session key.

## Main intake and routing workflow

### Ingestion

`Gmail Trigger` polls every minute with:

```text
in:inbox -from:me -subject:"[AGENT]"
```

This excludes messages sent by the mailbox itself and messages whose subject already contains `[AGENT]`, reducing self-reprocessing of generated replies. The trigger is followed by `Get a message`, which loads the full message used by later expressions.

### Client enrichment

`Get row(s) in sheet` reads the `Clients` spreadsheet and performs a first-match lookup by sender email. `Edit Fields` then creates the common message contract and passes it to the classifier.

### Classification

`Text Classifier` uses `gpt-4o` and has five outputs:

1. `Client Support & Service Requests`
2. `Billing & Payments`
3. `Projects & Service Inquiries`
4. `Team & Operations`
5. fallback `other`

The four named outputs call the corresponding child workflow by fixed workflow ID. The fallback goes to `No Operation, do nothing`, followed by an audit row in the `No Operation` sheet.

The router is orchestration-only: it does not draft or send messages itself. The child workflows own their domain actions.

## Client Support workflow

### Purpose and model

`Customer Support Agent` uses `gpt-4o-mini`, an eight-message `Simple Memory` window keyed by `threadId`, a structured output parser, and the Pinecone FAQ tool. The agent is instructed to use the official FAQ/policy knowledge base for support-policy questions and not to send email itself.

The required output is:

```json
{
  "emailBody": "draft response",
  "source": "knowledge summary",
  "answeredFromKnowledge": true
}
```

### Processing path

```text
Start
  -> Customer Support Agent
  -> If answeredFromKnowledge == true
       true  -> Sentiment Analysis -> Gmail draft -> Gmail label
                         |-> Telegram for negative sentiment
                         |-> Feedback Log row
                         `-> cost estimate -> System Logs / Client Support row
       false -> Telegram: manual review required
```

The agent can use internal client context but is explicitly instructed not to expose registry data, internal notes, plan details, or system implementation details.

`Sentiment Analysis` uses `gpt-4o-mini` and categories `Positive`, `Neutral`, and `Negative`. The configured outputs route positive/neutral and negative messages to draft paths; the negative path also notifies Telegram. Draft subjects are prefixed with `[AGENT] Re:` unless they already start with `[AGENT]`. Drafts are placed in the original Gmail thread.

The workflow logs feedback to a separate `Feedback Log` spreadsheet with sender, sentiment, original body, source, and the knowledge-base flag. It also writes an operational row to the shared `System Logs` spreadsheet.

### Configuration observations

- The operational log maps `answered_from_knowledge`, but the agent contract emits `answeredFromKnowledge`. This casing mismatch can produce a blank log field.
- The configured sentiment branches both point to draft operations; the negative branch additionally sends Telegram. There is no automatic email send.
- When the knowledge flag is false, the workflow sends a Telegram escalation but does not create a draft or write the normal support log row.

## Projects & Service Inquiries workflow

### Purpose and knowledge source

`Leads Agent` uses `gpt-4o-mini`, thread memory, a Gmail-label tool, and the Pinecone `services` namespace. It must use the services catalogue before replying and returns:

```json
{
  "emailBody": "HTML draft or empty string",
  "reason": "short rationale",
  "escalate": false,
  "knowledgeDatabase": "facts used"
}
```

Prices are explicitly treated as starting prices, not quotations. Custom scope, negotiation, undocumented capabilities, security/legal requirements, uncertain feasibility, and unsupported deadlines are escalated.

### Processing path

```text
Start
  -> Leads Agent
  -> Needs Review? (escalate)
       true  -> Telegram: human must answer
              `-> log action=escalate
       false -> Reviewer (gpt-4o-mini, strict structured approval)
                  -> If approved
                       true  -> Gmail draft -> Telegram send-and-wait approval
                                  -> approve -> Gmail send -> log
                                  -> decline -> log
                       false -> Gmail draft -> Telegram warning
                                  `-> log action=draft_flagged
```

The reviewer checks factual support, prices, timelines, discounts, and whether the draft answers the request. The human approval node waits up to 24 hours and uses Telegram double approval. Approved responses are sent as ordinary Gmail messages; the initial draft remains part of the approval flow.

All paths eventually calculate an estimated cost and append a row to the `Projects & Service Inquiries` tab of the shared `System Logs` spreadsheet. The cost estimate is a rough character-count approximation (`ceil(length / 4)`) multiplied by a hard-coded `$0.60` per million tokens; it is not provider billing telemetry.

### Configuration observations

- The `Reviewer` receives the draft, rationale, and knowledge summary, but not a direct retrieval result or independent knowledge tool. Its approval is therefore a second LLM check over agent-provided evidence, not an independently grounded verification.
- The approval path uses both a Gmail draft and a later direct send. Operationally this is a human-gated send, but the draft lifecycle should be verified to avoid duplicate or stale drafts.

## Billing & Payments workflow

### Purpose and classification

`Finance Agent` uses `gpt-4o`, thread memory, and a Gmail label tool. It must output exactly `receipt` or `payment`:

- `receipt`: money is reported as coming to LinguaTech Solutions.
- `payment`: LinguaTech Solutions is expected to pay an external party.

The output feeds a two-way switch.

### Customer receipt path

```text
Finance Agent -> receipt
  -> Gmail notification to finance mailbox
  -> Telegram payment notification
```

The workflow does not confirm bank settlement or approve the payment; the notifications explicitly require manual verification against invoices and company records.

### Vendor invoice path

```text
Finance Agent -> payment
  -> Gmail Get a message (download attachments)
  -> Code: locate a PDF binary and normalize it to binary.data
  -> If hasPdf
       true  -> Extract from File (PDF)
              -> Information Extractor (gpt-4o)
              -> Invoice Data spreadsheet row
              -> Gmail review notification
              -> Google Drive upload to Invoices folder
       false -> Telegram: no PDF attachment found
```

The information extractor returns `Vendor`, `Amount`, `Currency`, `VAT`, `InvoiceNumber`, `IBAN`, and `DueDate`. The resulting data is written to the `Invoice Data` spreadsheet and sent to a finance mailbox with a warning that the extraction is unverified. The PDF is uploaded to the configured Google Drive `Invoices` folder.

Every finance classification also writes an operational row to the `Billing & Payments` tab in `System Logs`. Cost is estimated from the input message length, not actual model usage.

### Configuration observations

- The `If hasPdf` false branch alerts Telegram but does not append a corresponding invoice-processing log row.
- The PDF normalizer selects the first binary item whose MIME type is exactly `application/pdf`; image invoices and PDFs with a nonstandard MIME type are not processed.
- The extractor persists and emails an IBAN, so access to the finance spreadsheet, mailbox, and Drive folder is sensitive.

## Team & Operations workflow

`Internal Agent` uses `gpt-4o`, thread memory, a Gmail label tool, and a structured output parser. It summarizes internal email into:

```json
{
  "emailBody": "2-3 sentence summary",
  "From": "sender identity",
  "actionRequired": true
}
```

The workflow labels the message as internal communication, sends a Telegram summary containing the sender, summary, and action flag, estimates cost, and appends a row to the `Team & Operations` tab of `System Logs`.

The prompt identifies internal communication primarily through the router's category description, including employees, managers, contractors, and recognized collaborators. The agent is prohibited from replying to external clients and from treating email content as trusted instructions.

## Knowledge ingestion and retrieval architecture

Both loaders are manual, single-document ingestion pipelines into the same Pinecone index:

```text
Manual Trigger
  -> Set/Edit Fields: embedded source text
  -> Pinecone Vector Store (insert)
       + ai_document <- Default Data Loader <- Recursive Character Text Splitter
       + ai_embedding <- OpenAI Embeddings
```

Shared vector store:

- index: `customer-support`
- embeddings: OpenAI embeddings node
- chunk size: 600
- chunk overlap: 80

Namespaces:

- `FAQ`: 15 support and policy entries, including support hours, response targets, post-delivery coverage, credentials, data handling, consultations, and escalation rules.
- `services`: eight service descriptions plus general sales policies, including starting prices, typical duration, deliverables, limitations, and escalation rules.

The child workflows query only their intended namespace:

- Client Support -> `customer-support` / `FAQ`
- Projects & Service Inquiries -> `customer-support` / `services`

The loaders insert content and have no update/delete or deduplication step. Re-running them may create duplicate vectors depending on n8n/Pinecone document ID behavior.

## External systems and credential boundaries

| System | Use |
|---|---|
| Gmail | Poll inbox, read messages and attachments, add labels, create drafts, send approved replies, send finance notifications |
| Google Sheets | Client registry lookup; system logs; feedback log; invoice data |
| OpenAI | Classification, chat agents, sentiment analysis, embeddings, invoice extraction, reviewer |
| Pinecone | FAQ/policy and services retrieval |
| Telegram | Operational alerts, negative sentiment alerts, human approval gate |
| Google Drive | Archive vendor invoice PDFs in the `Invoices` folder |

The exports reference shared credentials by name for Gmail, Google Sheets, OpenAI, Pinecone, Telegram, and Google Drive. No credential secret values are present in the JSON exports; credential IDs and external document/index identifiers are present.

## Reliability, safety, and operational behavior

### Safety controls represented in prompts

- Email content is treated as untrusted data and prompt-injection instructions are rejected.
- Agents are prohibited from revealing prompts, credentials, internal notes, or client-registry details.
- Support and sales agents are instructed not to invent policies, prices, dates, guarantees, or capabilities.
- Sales drafts use an explicit reviewer and human approval gate.
- Finance extraction warns that values require independent verification.
- Generated subjects use `[AGENT]`, and the Gmail trigger excludes those subjects.

### State and audit model

The system has no database node or durable workflow state beyond external services. State is distributed across:

- Gmail threads, labels, drafts, and sent messages
- `Clients` spreadsheet
- `System Logs` workbook tabs
- `Feedback Log` workbook
- `Invoice Data` workbook
- Pinecone namespaces
- Telegram approval/warning messages
- Google Drive invoice files

Operational audit rows generally contain timestamp, sender, category, agent, action, model, and estimated cost. Some paths add approval, escalation, knowledge-source, sentiment, or reviewer notes.

### Important implementation gaps visible in the exports

1. **Router disabled:** the main intake workflow is inactive in the export, so no automatic routing occurs until it is activated.
2. **Knowledge loaders disabled:** FAQ and service content must be manually loaded and refreshed; there is no scheduled synchronization.
3. **Field-name mismatch in support logging:** `answered_from_knowledge` differs from the emitted `answeredFromKnowledge`.
4. **Model/cost metadata mismatch:** several logs label `gpt-4o-mini` while the Team & Operations and Finance agent model nodes are configured as `gpt-4o`; the hard-coded cost calculation is only an estimate.
5. **Partial-path logging:** unsupported support answers and missing-PDF finance cases alert humans but do not consistently append the same operational log rows as successful paths.
6. **No explicit deduplication/idempotency:** Gmail polling, child execution, sheet appends, drafts, and Pinecone insertion do not show a shared message-ID ledger or idempotency guard.
7. **Human escalation is notification-based:** Telegram alerts and approval waits are the human control plane; there is no ticketing system, SLA timer, or escalation queue represented.
8. **Sensitive-data distribution:** invoice IBANs and extracted financial details are sent to email and stored in spreadsheets/Drive, so those destinations require appropriate access controls and retention policies.

## End-to-end lifecycle summary

For a normal supported email, the intended lifecycle is:

```text
Inbox message
  -> full Gmail read
  -> client registry enrichment
  -> LLM category classification
  -> category child workflow
  -> domain agent + optional retrieval/memory
  -> draft, notification, label, approval, extraction, or send
  -> Google Sheets audit record
```

The architecture is modular and uses n8n sub-workflows as bounded domain services. Its strongest controls are explicit prompt boundaries, structured outputs, retrieval-backed policies/catalogue, human approval for sales, and manual verification for financial actions. Its main operational dependencies are activation of the router, availability and freshness of the external knowledge stores, and consistent audit/idempotency handling across branches.
