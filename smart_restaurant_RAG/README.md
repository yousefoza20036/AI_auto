# 🍽️ Smart Restaurant AI — RAG & Order Automation

An end-to-end **AI-powered restaurant assistant** built with **n8n, OpenAI, Pinecone, Telegram, Google Sheets, Google Drive, and Gmail**.

The system combines **Retrieval-Augmented Generation (RAG)** with an AI orchestration layer to handle restaurant questions, menu retrieval, customer orders, order tracking, cancellations, and multimodal customer input through Telegram.

> **Project:** `smart_restaurant_RAG`
> **Workflow:** n8n
> **Interface:** Telegram
> **Architecture:** AI Orchestrator + Specialist Agents + RAG

---

## 🚀 Overview

The Smart Restaurant AI acts as a virtual restaurant assistant capable of understanding customer requests and routing them to the appropriate specialist.

The workflow supports:

* 💬 Text messages
* 🎙️ Voice messages
* 🖼️ Image messages
* 📚 RAG-based menu and restaurant information retrieval
* 🛒 New order creation
* 🔎 Order tracking
* ✏️ Order status management
* ❌ Order cancellation
* 📧 Cancellation email notifications
* 🧠 Conversation memory
* 🆔 Automatic order ID generation
* 🧮 Tool-based order total calculation
* ☁️ Automatic knowledge-base ingestion from Google Drive
* 🗃️ Order storage using Google Sheets

The complete workflow is exported as an n8n JSON file.

---

## 🏗️ Architecture

```text
                         ┌─────────────────────┐
                         │      Telegram       │
                         │ Text / Voice/Image  │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   Input Processing  │
                         │                     │
                         │ Text → Direct      │
                         │ Voice → Transcribe │
                         │ Image → Analyze     │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │     AI ORCH         │
                         │   Main Assistant    │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
          ┌──────────────────┐            ┌──────────────────┐
          │ Question Agent   │            │   Order Agent    │
          │                  │            │                  │
          │ Menu             │            │ Create Orders    │
          │ Prices           │            │ Track Orders     │
          │ Ingredients      │            │ Cancel Orders    │
          │ Allergens        │            │ Update Status    │
          │ Restaurant Info  │            │ Calculate Total  │
          └────────┬─────────┘            └────────┬─────────┘
                   │                               │
                   └──────────────┬────────────────┘
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │ Pinecone Vector DB  │
                       │        RAG          │
                       └─────────────────────┘

                                  │
                  ┌───────────────┼────────────────┐
                  │               │                │
                  ▼               ▼                ▼
          ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
          │Google Sheets │ │    Gmail     │ │    Telegram  │
          │ Orders/Data  │ │Cancellation  │ │    Response  │
          └──────────────┘ └──────────────┘ └──────────────┘
```

---

## ✨ Key Features

### 1. Multimodal Customer Input

Customers can interact with the restaurant using:

* Text
* Voice notes
* Images

Voice messages are automatically transcribed using OpenAI before being passed to the main AI assistant.

Images are analyzed using a vision-capable OpenAI model and converted into a description that can be processed by the orchestration layer.

---

### 2. AI Orchestration

The `AI ORCH` node acts as the main customer-facing assistant.

Its responsibilities are to:

1. Understand the customer's intent.
2. Determine whether the request is related to:

   * Ordering
   * Menu / restaurant information
   * Order management
3. Route the request to the appropriate specialist.
4. Relay the result naturally to the customer.
5. Keep the customer experience unified instead of exposing internal agents or tools.

---

### 3. RAG Knowledge Base

The project uses **Retrieval-Augmented Generation** to provide menu and restaurant information.

The RAG pipeline uses:

* Google Drive
* Document loading
* Recursive text splitting
* OpenAI embeddings
* Pinecone vector database
* Vector-store retrieval
* OpenAI language models

Restaurant documents can be stored in Google Drive and automatically processed into the Pinecone knowledge base.

The workflow checks which files have already been processed and avoids re-ingesting the same files.

---

### 4. Menu & Restaurant Information Agent

The `questions` specialist handles questions such as:

* Menu items
* Prices
* Ingredients
* Allergens
* Recommendations
* Restaurant locations
* Opening hours
* Reservation policies
* Payment methods
* Restaurant policies
* Contact information

The specialist uses the Pinecone vector store as its primary knowledge source instead of relying only on model memory.

---

### 5. Intelligent Order Agent

The `order` specialist manages the complete delivery/takeaway ordering lifecycle.

It handles:

```text
Customer Request
      ↓
Identify Items
      ↓
Retrieve Prices
      ↓
Collect Delivery Details
      ↓
Check Previous Orders
      ↓
Check Duplicate Orders
      ↓
Calculate Total
      ↓
Generate Order ID
      ↓
Customer Confirmation
      ↓
Save Order
      ↓
Return Order ID
```

Required delivery information:

* Customer name
* Phone number
* Address
* Items and quantities

---

### 6. Price Accuracy Protection

Price accuracy is treated as a critical requirement.

For each ordered item, the order specialist retrieves the item's price from the knowledge base separately.

The workflow is designed to prevent the AI from:

* Guessing prices
* Recalling old prices
* Rounding prices
* Mixing prices between items
* Inventing unavailable products

The order total is calculated through a dedicated `Calculate Order Total` tool rather than relying on the language model to perform the calculation.

---

### 7. Order ID Generation

Every new order receives a sequential 3-digit Order ID:

```text
001
002
003
...
998
999
001
```

The workflow uses n8n workflow static data to maintain the order sequence.

The AI is instructed to use the generated value exactly instead of calculating or guessing the ID itself.

---

### 8. Google Sheets Order Management

Google Sheets is used as the operational order database.

The order schema contains:

| Field        | Purpose                            |
| ------------ | ---------------------------------- |
| Time         | Last order/status update timestamp |
| Item         | Ordered item(s)                    |
| Price        | Grand total                        |
| Order ID     | Unique tracking number             |
| Name         | Customer name                      |
| Phone Number | Customer phone                     |
| Address      | Delivery address                   |
| Status       | Current order status               |

Supported statuses:

```text
In Kitchen
Shipped
Delivered
Cancelled
```

---

### 9. Duplicate Order Protection

Before creating a new order, the workflow checks the customer's previous orders using their phone number.

It looks for an identical order created within a short time window to reduce accidental duplicate submissions.

The customer can still explicitly confirm that they want to place the second order.

---

### 10. Order Cancellation

Customers can request cancellation while an order is still:

```text
In Kitchen
```

Orders that are already:

```text
Shipped
Delivered
```

cannot be cancelled automatically and require staff intervention.

When an eligible order is cancelled:

1. The existing Order ID is retrieved.
2. The existing row is updated.
3. The status becomes `Cancelled`.
4. A cancellation email is sent.
5. The original order timestamp is preserved for the cancellation notification.

---

## 🔄 Automatic Knowledge-Base Ingestion

The workflow includes a separate ingestion pipeline.

```text
Google Drive
     ↓
Find Restaurant Files
     ↓
Compare Against Processed Files
     ↓
Download New Files
     ↓
Document Loader
     ↓
Text Splitter
     ↓
OpenAI Embeddings
     ↓
Pinecone
```

The workflow also uses Google Sheets to keep track of processed Google Drive files.

A scheduled trigger runs the ingestion process periodically.

---

## 🧠 Technologies

| Technology        | Purpose                                |
| ----------------- | -------------------------------------- |
| **n8n**           | Workflow automation and orchestration  |
| **OpenAI**        | LLM, transcription, vision, embeddings |
| **Pinecone**      | Vector database for RAG                |
| **Telegram**      | Customer-facing interface              |
| **Google Sheets** | Order database and ingestion tracking  |
| **Google Drive**  | Restaurant knowledge-base documents    |
| **Gmail**         | Cancellation notifications             |
| **JavaScript**    | Custom order ID and calculation logic  |

---

## 📁 Repository Structure

Recommended repository structure:

```text
smart-restaurant-ai/
│
├── smart_restaurant_RAG.json
├── README.md
├── .env.example
├── .gitignore
│
└── assets/
    └── screenshots/
```

---

## ⚙️ Requirements

Before importing the workflow, you need:

* n8n
* Telegram Bot
* OpenAI API access
* Pinecone account
* Google account
* Google Drive folder
* Google Sheets spreadsheet
* Gmail account

---

## 🔐 Environment Variables

Create a local `.env` file based on `.env.example`.

```env
TELEGRAM_BOT_TOKEN=

OPENAI_API_KEY=

PINECONE_API_KEY=
PINECONE_INDEX_NAME=restaurant
PINECONE_NAMESPACE=

GOOGLE_SHEETS_SPREADSHEET_ID=
GOOGLE_SHEETS_SHEET_NAME=
GOOGLE_DRIVE_FOLDER_ID=

GMAIL_RECIPIENT_EMAIL=

N8N_HOST=
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=
GENERIC_TIMEZONE=Africa/Cairo
```

> **Never commit your real `.env` file or API keys to GitHub.**

---

## 📥 Importing the Workflow

1. Install and start n8n.
2. Open your n8n instance.
3. Create a new workflow.
4. Select **Import from File**.
5. Select:

```text
smart_restaurant_RAG.json
```

6. Reconnect your credentials.
7. Configure your Telegram credentials.
8. Configure OpenAI.
9. Configure Pinecone.
10. Configure Google Drive and Google Sheets.
11. Configure Gmail.
12. Upload your restaurant knowledge-base documents.
13. Verify the Pinecone index and namespace.
14. Activate the workflow.

---

## 🔑 Credentials

The exported workflow references n8n credentials, but credentials should be recreated in the destination n8n instance.

Required integrations include:

```text
Telegram API
OpenAI API
Pinecone API
Google Sheets OAuth2
Google Drive OAuth2
Gmail OAuth2
```

Do not commit credential IDs, OAuth tokens, API keys, or private account information.

---

## 🗃️ Pinecone Configuration

The workflow uses a Pinecone index named:

```text
restaurant
```

The knowledge base uses OpenAI embeddings before storing documents inside Pinecone.

A namespace can be used to isolate restaurant documents.

Example:

```text
restaurant_test_menu.pdf
```

The exact namespace should match the documents being ingested by the workflow.

---

## 🧪 Testing

After configuring the workflow, test the following scenarios.

### Menu Questions

```text
What is the price of the Chicken Caesar Wrap?
```

```text
What ingredients are in the Mojito?
```

```text
What do you recommend if I want something refreshing?
```

### Text Ordering

```text
I want 2 Chicken Caesar Wraps.
```

Verify that the assistant requests any missing delivery information.

---

### Voice Ordering

Send a Telegram voice message containing an order request.

Verify:

```text
Telegram
   ↓
Audio Download
   ↓
Transcription
   ↓
AI ORCH
   ↓
Order Agent
```

---

### Image Input

Send a restaurant/menu image through Telegram.

Verify that the image is analyzed and its description is passed to the AI orchestration layer.

---

### Order Tracking

Test:

```text
Where is my order?
```

with a valid Order ID.

Expected statuses:

```text
In Kitchen → Being prepared
Shipped → On the way
Delivered → Already delivered
Cancelled → Order was cancelled
```

---

### Cancellation

Test cancelling an order that is still:

```text
In Kitchen
```

Verify:

* Existing Order ID is reused.
* The row is updated instead of duplicated.
* Status becomes `Cancelled`.
* Cancellation email is sent.

---

## 🛡️ Reliability & Guardrails

The workflow contains several AI safety and data-integrity rules.

### Order Isolation

Previous orders should not automatically become the contents of a new order.

Every new order is treated as a fresh transaction unless the customer explicitly requests something such as:

```text
Re-order my last order.
```

---

### Tool Separation

New orders use:

```text
Save New Order
```

Status changes and cancellations use:

```text
Update Order Status
```

This separation prevents accidental duplicate order rows.

---

### Order ID Integrity

The AI must not manually generate an Order ID.

New orders use:

```text
Generate Order ID
```

Existing orders reuse the exact ID retrieved from the database.

---

### Price Integrity

The AI must not:

```text
Guess
Round
Modify
Recall
Estimate
```

menu prices.

Prices should come from the restaurant knowledge base.

---

## 📊 Data Flow

### Customer Interaction

```text
Telegram
   ↓
Input Detection
   ↓
Text / Voice / Image Processing
   ↓
AI ORCH
   ↓
Specialist Agent
   ↓
RAG / Order Tools
   ↓
Response
   ↓
Telegram
```

### RAG Pipeline

```text
Google Drive
   ↓
New File Detection
   ↓
Download
   ↓
Document Loader
   ↓
Chunking
   ↓
Embeddings
   ↓
Pinecone
   ↓
Retriever
   ↓
AI Specialist
```

### Order Pipeline

```text
Customer
   ↓
AI ORCH
   ↓
Order Specialist
   ↓
Menu Retrieval
   ↓
Price Validation
   ↓
Total Calculation
   ↓
Order ID
   ↓
Google Sheets
   ↓
Confirmation
```

---

## 🔒 Security

This repository is intended to contain the workflow architecture, not private credentials.

Before publishing:

* Remove API keys.
* Remove OAuth tokens.
* Remove private webhook URLs.
* Remove private Google Drive links where appropriate.
* Remove private spreadsheet IDs where appropriate.
* Add `.env` to `.gitignore`.
* Recreate credentials inside the target n8n instance.

Example `.gitignore`:

```gitignore
.env
.env.*
!.env.example

credentials/
secrets/
*.key
*.pem

node_modules/
.DS_Store
```

---

## ⚠️ Production Considerations

This project is designed as an automation/portfolio implementation.

Before deploying it for a real restaurant, consider adding:

* Authentication and authorization
* Rate limiting
* Error handling and retry logic
* Structured logging
* Monitoring and alerting
* Database transactions
* Persistent order ID generation
* Concurrent-order protection
* Webhook security
* Backup strategy
* Customer data retention policies
* Human escalation workflows
* Production-grade secrets management

For high-volume restaurants, Google Sheets may also be replaced with a dedicated database.

---

## 📌 Project Highlights

This project demonstrates practical implementation of:

* AI Agents
* Agentic workflow orchestration
* RAG
* Vector databases
* Semantic search
* LLM tool calling
* Multimodal AI
* Speech-to-text
* AI vision
* Workflow automation
* API integrations
* Google Workspace automation
* Order management
* Data validation
* AI guardrails
* State management
* Customer-service automation

---

## 🎯 Use Case

The system can be adapted for restaurants that want to automate:

```text
Customer Support
       +
Menu Questions
       +
Order Processing
       +
Order Tracking
       +
Order Cancellation
       +
Knowledge Management
```

while maintaining a centralized restaurant knowledge base.

---

## 👨‍💻 Author

**Youssef Mohamed**

Communications & Electronics Engineering
Cairo University

Interests:

* AI Automation
* AI Agents
* RAG Systems
* Machine Learning
* Digital Design
* Verification
* Telecom

---

## 📄 License

This project is provided for educational, portfolio, and demonstration purposes.

Add an appropriate open-source license before distributing the workflow for commercial use.
