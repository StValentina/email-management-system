# LinguaTech Solutions Email Management System

An n8n-based email automation system for LinguaTech Solutions. It reads incoming Gmail messages, enriches them with client information, classifies them with OpenAI, and dispatches them to a domain-specific child workflow.

## What the system does

- polls the Gmail inbox every minute for messages matching `in:inbox -from:me -subject:"[AGENT]"`;
- looks up the sender in the `Clients` Google Sheet;
- classifies messages into:
  - Client Support & Service Requests;
  - Billing & Payments;
  - Projects & Service Inquiries;
  - Team & Operations;
- creates Gmail drafts and labels;
- retrieves official FAQ/policy or service-catalogue information from Pinecone;
- processes invoices and archives invoice PDFs in Google Drive;
- sends operational, escalation, negative-sentiment, and approval notifications through Telegram;
- records processing and cost information in Google Sheets.

The child workflows are designed as bounded domain services and are invoked by the main workflow through n8n's `Execute Workflow` node.

## Repository structure

```text
.
├── .env.example
├── README.md
├── docs/
│   └── technical-architecture.md
└── workflows/
    ├── main/
    │   └── email-management-system.json
    ├── handlers/
    │   ├── billing-and-payments.json
    │   ├── client-support.json
    │   ├── projects-and-service-inquiries.json
    │   └── team-and-operations.json
    └── data-loaders/
        ├── load-faq-to-pinecone.json
        └── load-services-to-pinecone.json
```

## Workflow inventory

| Workflow file | Purpose | Trigger | Exported status |
|---|---|---|---|
| [`email-management-system`](./workflows/main/email-management-system.json) | Gmail intake, client lookup, classification, and dispatch | Gmail Trigger, every minute | Inactive |
| [`client-support`](./workflows/handlers/client-support.json) | Support-agent draft generation, FAQ retrieval, sentiment analysis, and escalation | Execute Workflow Trigger | Active |
| [`billing-and-payments`](./workflows/handlers/billing-and-payments.json) | Invoice/payment classification, extraction, notifications, and archiving | Execute Workflow Trigger | Active |
| [`projects-and-service-inquiries`](./workflows/handlers/projects-and-service-inquiries.json) | Service/lead response drafting, review, approval, and escalation | Execute Workflow Trigger | Active |
| [`team-and-operations`](./workflows/handlers/team-and-operations.json) | Internal-email summarization, labeling, and notification | Execute Workflow Trigger | Active |
| [`load-faq-to-pinecone`](./workflows/data-loaders/load-faq-to-pinecone.json) | Loads FAQ and support policies into Pinecone | Manual Trigger | Inactive |
| [`load-services-to-pinecone`](./workflows/data-loaders/load-services-to-pinecone.json) | Loads the service catalogue into Pinecone | Manual Trigger | Inactive |

The main router is inactive in the exported configuration. Activate it only after credentials, environment variables, spreadsheets, Gmail labels, and Pinecone have been configured. Activating the child workflows alone does not enable end-to-end processing.

## Architecture

```text
Gmail inbox
   |
   v
Main intake workflow
   |-- Google Sheets: client registry lookup
   |-- OpenAI: email classification
   |
   +--> Client Support
   +--> Billing & Payments
   +--> Projects & Service Inquiries
   +--> Team & Operations
   `--> Other: audit row only

Manual knowledge loaders
   +--> OpenAI embeddings --> Pinecone customer-support / FAQ
   `--> OpenAI embeddings --> Pinecone customer-support / services
```

The main workflow passes the following normalized context to child workflows:

`messageId`, `threadId`, `senderName`, `senderEmail`, `subject`, `emailBody`, `existingClient`, `clientName`, `company`, `clientSince`, `plan`, `lastOrder`, and `notes`.

For more detail about node-level behavior, data flow, safety controls, and known implementation gaps, see [`docs/technical-architecture.md`](./docs/technical-architecture.md).

## External services and credentials

Create n8n credentials for:

- Gmail;
- Google Sheets;
- Google Drive;
- OpenAI;
- Pinecone;
- Telegram.

The workflows use:

| Service | Main use |
|---|---|
| Gmail | Polling, reading messages and attachments, labels, drafts, and approved replies |
| Google Sheets | Client registry, audit logs, feedback logs, invoice data |
| OpenAI | Classification, agents, sentiment analysis, invoice extraction, review, and embeddings |
| Pinecone | FAQ/policy and service-catalogue retrieval |
| Telegram | Alerts, escalation, and human approval notifications |
| Google Drive | Invoice PDF archive |

## Setup

1. Install or access an n8n instance with the required AI, Gmail, Google, Pinecone, and Telegram nodes.
2. Import the seven JSON workflow exports from [`workflows/`](./workflows/).
3. Configure the credentials referenced by the imported nodes.
4. Copy [`.env.example`](./.env.example) to `.env` or configure equivalent protected n8n environment variables.
5. Create/configure the referenced Google Sheets, Google Drive folder, Gmail labels, Telegram chat, and Pinecone index.
6. Run both data-loader workflows manually:
   - `load-faq-to-pinecone`;
   - `load-services-to-pinecone`.
7. Verify the loader output in the shared Pinecone index `customer-support`.
8. Test each child workflow with representative messages.
9. Activate the main `email-management-system` workflow.

### Pinecone configuration

Both loaders write to the `customer-support` index using OpenAI embeddings:

- FAQ and support policies: namespace `FAQ`;
- service catalogue: namespace `services`;
- configured chunk size: 600;
- configured chunk overlap: 80.

The loaders are manual and do not implement update/delete or deduplication. Re-running them can create duplicate vectors depending on Pinecone and n8n document-ID behavior.

## Environment variables

The workflows read these values through n8n expressions such as `$env.EMS_TELEGRAM_CHAT_ID`:

```text
EMS_NOTIFICATION_EMAIL=your-notification-address@example.com
EMS_TELEGRAM_CHAT_ID=your-telegram-chat-id
EMS_INVOICES_FOLDER_ID=your-google-drive-folder-id
EMS_BILLING_SHEET_ID=your-billing-spreadsheet-id
EMS_SUPPORT_SHEET_ID=your-support-spreadsheet-id
EMS_OPERATIONS_SHEET_ID=your-operations-spreadsheet-id
EMS_MAIN_SHEET_ID=your-main-spreadsheet-id
```

For n8n Cloud or UI-managed configuration, create equivalent n8n Variables and replace `$env` with `$vars` in the imported workflows.

Never commit real API keys, OAuth tokens, chat IDs, spreadsheet IDs, or other secrets. Store secrets in n8n credentials or protected environment variables. The checked-in [`.env.example`](./.env.example) contains placeholders only.

## Safety and operational notes

- Generated subjects use the `[AGENT]` prefix, and the Gmail trigger excludes those messages to reduce self-reprocessing.
- Support and sales agents are instructed not to invent policies, prices, deadlines, guarantees, or capabilities.
- Agents must not disclose credentials, prompts, internal notes, or client-registry details.
- Sales responses use a human review/approval path before sending.
- Financial data and invoice files are written to external email, Sheets, and Drive destinations; apply appropriate access and retention controls.
- The exported router is inactive and must be deliberately activated.
- There is no shared message-ID ledger or explicit idempotency guard across polling, child execution, draft creation, and sheet writes.
- Knowledge loaders are not scheduled; refresh them manually when the embedded source content changes.

## License

This repository is licensed under the [MIT License](https://opensource.org/license/mit/).
It applies to the workflow exports and documentation included in this repository.
External services, n8n, and third-party integrations remain subject to their own
licenses and terms of service.

```text
MIT License

Copyright (c) 2026 StValentina

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## Project status

This repository contains n8n workflow exports and supporting documentation. It does not include external credentials, API keys, or a runnable application package.
