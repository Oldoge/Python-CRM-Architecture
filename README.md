# Pipedrive CRM Automation Assistant

## Description

An enterprise-grade intelligent automation system for processing orders in Pipedrive CRM. This FastAPI-based application handles the complete order lifecycle from draft generation to shipping notification through webhook-driven workflows. It leverages OpenAI's **gpt-5-mini** for intelligent data extraction and product matching, with optional Perplexity **sonar-pro** integration for AI-powered product descriptions. The system features asynchronous task processing via RabbitMQ, comprehensive error monitoring with Sentry, real-time Telegram notifications, and automated weekly performance reports with cost tracking.

## Table of Contents

- [Quick Start](#quick-start)
- [Key Features](#key-features)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Workflow Stages](#workflow-stages)
- [Technology Stack](#technology-stack)
- [Requirements](#requirements)
- [RabbitMQ Message Queue](#rabbitmq-message-queue)
- [Installation](#installation)
- [Usage](#usage)
- [API Endpoints](#api-endpoints)
- [Detailed Workflow Documentation](#detailed-workflow-documentation)
- [Configuration](#configuration)
- [Monitoring & Statistics](#monitoring--statistics)
- [Error Handling](#error-handling)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Performance Optimization](#performance-optimization)
- [Best Practices](#best-practices)
- [AI Model Configuration](#ai-model-configuration)
- [Contributing](#contributing)
- [License](#license)
- [Support](#support)
- [Roadmap](#roadmap)
- [Acknowledgments](#acknowledgments)

## Quick Start

### Prerequisites
- Docker and Docker Compose installed
- Pipedrive account with API access
- OpenAI API key
- (Optional) Perplexity API key, Sentry account, Telegram bot

### 5-Minute Setup

1. **Clone and configure:**
   ```bash
   git clone <repository-url>
   cd Assistents
   cp .env.example .env
   # Edit .env with your API keys
   ```

2. **Start services:**
   ```bash
   docker-compose up -d
   ```

3. **Start worker:**
   ```bash
   docker exec -it pipedrive_bot python -m src.services.mq_worker
   ```

4. **Configure Pipedrive webhook:**
   - Get ngrok URL from http://localhost:4040
   - Add webhook in Pipedrive: `https://your-ngrok-url.com/webhook/pipedrive/`
   - Select triggers: "Deal updated", "Deal stage updated"

5. **Test:**
   - Move a deal to Stage 18 in Pipedrive
   - Check logs: `docker logs -f pipedrive_bot`
   - DOCX estimate should be generated and uploaded

### What Happens Next?

- **Stage 18:** System generates DOCX estimate with products, images, VAT, and leasing
- **Stage 19:** Sends estimate to customer via email
- **Stage 20:** Sends shipping notification
- **Product Created:** AI generates description automatically
- **Weekly Reports:** Statistics sent to Telegram every Monday

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        PIPEDRIVE CRM                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Stage    │  │ Stage    │  │ Stage    │  │ Products │         │
│  │  Draft   │  │  Quote   │  │ Shipped  │  │ Created  │         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
└───────┼─────────────┼─────────────┼─────────────┼───────────────┘
        │             │             │             │
        │   Webhooks  │             │             │
        ▼             ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FASTAPI APPLICATION                          │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  Webhook Handler → Validate → Publish to RabbitMQ      │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      RABBITMQ QUEUE                             │
│  [Task 1] [Task 2] [Task 3] ... [Task N]                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      WORKER PROCESS                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ AI Service   │  │   Database   │  │  Monitoring  │           │
│  │ gpt-5-mini   │  │  PostgreSQL  │  │    Sentry    │           │
│  │  sonar-pro   │  │  Statistics  │  │   Telegram   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  1. Parse Content      → Extract customer & products     │   │
│  │  2. Match Products     → AI-powered fuzzy matching       │   │
│  │  3. Generate DOCX      → With images, VAT, leasing       │   │
│  │  4. Upload to Pipedrive → Send email notifications       │   │
│  │  5. Log Statistics     → Track costs & performance       │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Key Components:**
- **FastAPI:** Receives webhooks, validates, and queues tasks
- **RabbitMQ:** Reliable message queue for async processing
- **Worker:** Consumes tasks and executes business logic
- **AI Services:** gpt-5-mini for intelligence, sonar-pro for descriptions
- **PostgreSQL:** Tracks all operations, costs, and tokens
- **Monitoring:** Sentry for errors, Telegram for alerts

## Key Features

### Core Functionality
- **Multi-Stage Workflow Automation:** Automated processing for three critical deal stages:
  - **Stage (Generate Draft):** Parse content → AI extracts product info → Match products → DOCX estimate creation
  - **Stage (Send Quote):** Automated email delivery with DOCX attachments via Pipedrive mailbox
  - **Stage (Order Shipped):** Customer shipping notifications
- **Smart Duplicate Prevention:** Ignores webhook triggers when stage hasn't actually changed or only metadata updated
- **Priority-Based Product Discovery:** Three-tier fallback system:
  1. Manual products from Pipedrive "Products" section (highest priority)
  2. Product search from cleaned deal title
  3. Product extraction from last note content

### AI-Powered Intelligence
- **Multi-Model AI Strategy:**
  - **gpt-5-mini** for data extraction, product matching, and general intelligence
  - **sonar-pro** (Perplexity) for enhanced product descriptions (optional)
- **Smart Content Parsing:** AI extracts structured customer and product data from unstructured emails or notes
- **Two-Stage Product Matching:**
  - Python-based fuzzy search through full product catalog (up to 5000+ products)
  - AI-powered selection of best match from top 30 candidates using model numbers and codes
- **Brand/Model Extraction:** Automatically extracts product brands and models from product links
- **Deal Title Cleaning:** Removes prefixes and numeric suffixes for cleaner product search
- **Intelligent Description Merging:** Combines item descriptions from deal products and full product database

### Advanced Document Generation
- **Professional DOCX Estimates:** Template-based document generation with python-docx and docxtpl
- **Product Images:** Automatic download and embedding of product images from Pipedrive
- **Leasing Calculations:** Integrated leasing offers with tiered interest rates (7.5% - 18.61%) based on total amount
- **VAT Calculation:** Automatic 21% VAT calculation and breakdown
- **Custom Branding:** Company logo integration with file:// URI support
- **Precise Calculations:** Uses Decimal type for accurate financial computations

### Message Queue & Asynchronous Processing
- **RabbitMQ Integration:** Reliable task queue for background processing
- **Worker Process:** Dedicated consumer for handling long-running operations
- **Heartbeat Management:** Maintains persistent connections for extended tasks
- **Task Persistence:** Durable messages survive RabbitMQ restarts
- **QoS Control:** Single-threaded processing ensures sequential task completion
- **Auto-Reconnect:** Built-in resilience with automatic connection retry logic

### Monitoring & Error Handling
- **Sentry Integration:** Real-time error tracking with full stack traces
- **Telegram Notifications:** Instant alerts for critical errors with deal links
- **Comprehensive Logging:** Detailed activity logs with timestamp and context
- **Global Exception Handler:** Catches all unhandled errors and notifies administrators
- **Graceful Degradation:** System continues operating when optional services unavailable

### Statistics & Reporting
- **PostgreSQL Statistics:** Tracks all AI operations with token usage and costs
- **Weekly Automated Reports:** Scheduled report generation every Monday at 9:00 AM
- **Cost Tracking:** Precise USD cost calculation for OpenAI and Perplexity API calls
- **Token Monitoring:** Input/output token tracking per operation
- **Action Logging:** Records document generation, descriptions, and email parsing events

### Pipedrive Integration
- **Bidirectional Sync:** Real-time webhook processing and API updates
- **Product Management:**
  - Automatic product creation for unmatched items (price = 0)
  - Full catalog download for AI-powered matching
  - Product image download from Pipedrive
  - AI-generated product descriptions via webhook trigger
- **File Management:** DOCX upload with validation and old file cleanup
- **Email Automation:** Direct email sending through Pipedrive mailbox API with file attachments
- **Activity Creation:** Automated task creation when manual review is needed (price = 0)
- **Notes System:** Comprehensive HTML notes with document preview for managers
- **Dual Webhook Support:** Separate handlers for deal updates and product creation

### Robust Architecture
- **Asynchronous Processing:** FastAPI with async/await patterns throughout
- **Error Recovery:** Comprehensive exception handling with traceback logging
- **Duplicate Prevention:** Smart detection of actual vs. superficial changes
- **Database Support:** PostgreSQL integration with SQLAlchemy async ORM
- **Docker Deployment:** Complete containerization with Docker Compose
- **Ngrok Tunneling:** Built-in development webhook tunnel with alpine image
- **Scheduled Tasks:** APScheduler for weekly reports and maintenance operations

## Project Structure

```
Assistents/
│
├── main.py                      # Main FastAPI application & webhook endpoints
├── requirements.txt             # Python dependencies
├── docker-compose.yml           # Docker Compose configuration (app + PostgreSQL + RabbitMQ + ngrok)
├── Dockerfile                   # Application container configuration
├── README.md                    # Project documentation
├── bot_logs.log                 # Application logs
├── .env                         # Environment variables (not in repo)
├── .gitignore                   # Git ignore rules
│
├── img/                         # Static assets
│   ├── logo_1x.png              # Company logo for documents
│   └── ids.png                  # Additional images
│
└── src/                         # Source code
    ├── config.py                # Configuration & environment variables
    ├── database.py              # PostgreSQL async database setup with BotStat model
    ├── find_keys.py             # Utility for finding custom field keys
    ├── products_audit.csv       # Product audit data
    │
    ├── services/                # Business logic services
    │   ├── ai_assistant.py      # OpenAI GPT & Perplexity integration
    │   ├── email_renderer.py    # Jinja2 email template rendering
    │   ├── email_transport.py   # Email sending functionality
    │   ├── mq_worker.py         # RabbitMQ client and worker process
    │   ├── pipedrive.py         # Pipedrive API client (deals, products, emails)
    │   ├── report_service.py    # Weekly statistics report generation
    │   ├── stats_service.py     # Database statistics logging
    │   └── workflow.py          # Main workflow orchestration (3 stages)
    │
    ├── templates/               # Jinja2 HTML templates
    │   ├── estimate_pdf.html    # PDF estimate template (legacy)
    │   ├── invoice_email.html   # Email notification template
    │   └── template.docx        # Word document template for DOCX generation
    │
    ├── utils/                   # Utility modules
    │   ├── DocxEstimate.py      # DOCX document generation with docxtpl
    │   ├── LeasCalculator.py    # Leasing payment calculations with tiered rates
    │   ├── logger.py            # Centralized logging configuration
    │   ├── PDFEstimate.py       # PDF generation with WeasyPrint (legacy)
    │   ├── telegram.py          # Telegram bot notifications
    │   └── fonts/               # Custom fonts for PDF
    │       ├── dejavu-sans.bold.ttf
    │       ├── dejavu-sans.condensed-bold.ttf
    │       └── DejaVuSansCondensed.ttf
    │
    └── test/                    # Test suite
        ├── init_test.py         # Test initialization
        ├── test_webhook.py      # Webhook endpoint tests
        ├── test_workflow.py     # Workflow logic tests
        ├── Test_pdf.py          # PDF generation tests
        └── test_*.pdf/jpg       # Test fixtures
```

## Architecture Overview

```mermaid
graph TB
    %% External Triggers
    subgraph Pipedrive [Pipedrive CRM]
        direction TB
        WH1[Webhook: Deal Stage Change]
        WH2[Webhook: Product Created]
        Files[DOCX Uploads]
        Notes[Notes & Tasks]
        Mail[Emails Sent]
    end
    
    %% Entry Point
    WH1 -->|HTTP POST| FastAPI[FastAPI Application]
    WH2 -->|HTTP POST| FastAPI

    %% Message Queue
    FastAPI -->|Publish Task| RMQ[RabbitMQ Queue]
    RMQ -->|Consume Task| Worker[MQ Worker Process]

    %% Processing Stages
    subgraph Processing [Workflow Engine]
        direction TB
        Stage1[Stage : Generate Draft]
        Stage2[Stage : Send Quote]
        Stage3[Stage : Order Shipped]
        DescGen[Product Description Generator]
    end

    Worker -->|Route Task| Stage1
    Worker -->|Route Task| Stage2
    Worker -->|Route Task| Stage3
    Worker -->|Route Task| DescGen

    %% AI Services
    subgraph AI [AI Services]
        direction TB
        GPT[gpt-5-mini<br/>Data Extraction & Matching]
        Perplexity[sonar-pro<br/>Product Descriptions]
    end

    Stage18 -->|Parse & Match| GPT
    DescGen -->|Generate Text| Perplexity
    DescGen -->|Fallback| GPT

    %% Data & Storage
    subgraph Data [Data Layer]
        direction TB
        PG[(PostgreSQL<br/>Statistics & Tokens)]
        Catalog[Product Catalog<br/>5000+ Items]
    end

    Stage18 -->|Download| Catalog
    GPT -->|Log Stats| PG
    Perplexity -->|Log Stats| PG

    %% Document Generation
    Stage18 -->|Generate| DOCX[DOCX Generator<br/>docxtpl + python-docx]
    DOCX -->|Upload| Files

    %% Notifications & Monitoring
    subgraph Monitoring [Monitoring & Alerts]
        direction TB
        Sentry[Sentry Error Tracking]
        Telegram[Telegram Bot]
        Scheduler[APScheduler<br/>Weekly Reports]
    end

    Worker -->|Log Errors| Sentry
    Worker -->|Send Alerts| Telegram
    Scheduler -->|Query Stats| PG
    Scheduler -->|Send Report| Telegram

    %% Outputs
    Stage18 -->|Create| Notes
    Stage19 -->|Send Email| Mail
    Stage20 -->|Send Email| Mail

    %% Styling
    style Processing fill:#4A90E2,stroke:#2E5C8A,stroke-width:2px,color:#fff
    style AI fill:#50C878,stroke:#2E7D4E,stroke-width:2px,color:#fff
    style Data fill:#9B59B6,stroke:#6C3483,stroke-width:2px,color:#fff
    style Monitoring fill:#E74C3C,stroke:#A93226,stroke-width:2px,color:#fff
    style Pipedrive fill:#FF9800,stroke:#E65100,stroke-width:2px,color:#fff
    style RMQ fill:#FF6B35,stroke:#C7522A,stroke-width:2px,color:#fff
```

### Data Flow

1. **Webhook Trigger:** Pipedrive sends webhook when deal stage changes or product is created
2. **Task Queuing:** FastAPI validates webhook and publishes task to RabbitMQ
3. **Worker Processing:** Dedicated worker consumes task and routes to appropriate workflow
4. **AI Processing:** gpt-5-mini extracts data and matches products; sonar-pro generates descriptions
5. **Document Generation:** DOCX estimate created with images, VAT, and leasing calculations
6. **Statistics Logging:** All AI operations logged to PostgreSQL with token counts and costs
7. **Error Handling:** Sentry captures errors; Telegram sends instant alerts
8. **Reporting:** Weekly automated reports summarize operations and costs
### Workflow Stages

#### Stage 1: Generate Draft (STAGE_GENERATE_DRAFT = 1)

**Trigger:** Deal moves to Stage 1 or products are updated while in Stage 1

**Priority-Based Product Discovery:**

1. **[Highest Priority] Manual Products from "Products" Section:**
   - Checks Pipedrive deal's "Products" tab for manually added items
   - Retrieves product details including name, price, quantity, code
   - Fetches descriptions from three sources (in order):
     - `item_description` (deal-specific note)
     - `description` (from product list)
     - Full product database lookup if description is empty
   - Downloads product images from Pipedrive API

2. **[Medium Priority] Deal Title Search:**
   - If no manual products found, cleans deal title:
     - Removes "Price request..." or "Deal - " prefix
     - Removes numeric suffixes (4+ digits)
     - Example: "Price request - Product 1234" → "Product"
   - Performs fuzzy product search if title is meaningful (>3 chars, not "darījums")

3. **[Lowest Priority] Last Note Content:**
   - If still no products, extracts content from most recent note
   - Takes first 150 characters as search term
   - Performs product search using this content

**Two-Stage Product Matching:**

- **Stage A (Python Fuzzy Search):**
  - Downloads full product catalog (~5000 products) with id, name, code
  - Cleans search term: removes dots, dashes, spaces
  - Performs fuzzy matching:
    - Checks if search term appears in product name
    - Checks if search term appears in product code
    - Checks if individual words (>2 chars) appear in name
  - Returns top 30 candidates ranked by similarity

- **Stage B (AI Selection):**
  - **gpt-5-mini** analyzes top 30 candidates
  - Considers model numbers, codes, and brand names
  - Handles format variations (spaces, dots, dashes, case)
  - Returns best match ID or null if no good match
  - If match found: downloads full product details
  - If no match: creates new product with price = 0

**DOCX Generation:**

1. **Customer Data Collection:**
   - Retrieves person email from Pipedrive deal
   - Attempts to get phone from deal details
   - If no phone, parses last note with AI to extract it

2. **Document Creation:**
   - Uses docxtpl template engine with template.docx
   - Embeds product images (downloaded to temp files)
   - Calculates VAT (21%) on subtotal
   - Generates 36-month leasing offer with tiered rates
   - Uses Decimal precision for financial calculations

3. **Upload & Validation:**
   - Validates DOCX file exists and size > 0 bytes
   - Uploads to Pipedrive deal files (removes old generated files first)
   - Logs detailed error if upload fails

4. **Quality Control:**
   - Creates HTML note with document preview
   - If any product has price = 0:
     - Creates manual review task
     - Moves deal to Stage 4 (Manual Check)
     - Adds activity for manager review

5. **Cleanup:**
   - Removes temporary DOCX file
   - Deletes downloaded product images (temp_prod_*.jpg)

6. **Statistics Logging:**
   - Records "docx_generated" action in PostgreSQL
   - Logs AI usage (model, tokens, cost) for all operations

#### Stage 2: Send Quote (STAGE_SEND_INVOICE = 2)

**Trigger:** Deal moves to Stage 2

**Process Flow:**

1. **File Retrieval:**
   - Searches deal files for DOCX invoice (filename contains "invoice_")
   - Validates file exists and is accessible

2. **Email Preparation:**
   - Retrieves customer email from deal person
   - Renders HTML email template using Jinja2
   - Includes customer name and total amount
   - Attaches DOCX invoice file

3. **Email Delivery:**
   - Sends via Pipedrive mailbox API
   - Uses configured Pipedrive email template (ID: 11)
   - Logs delivery status

4. **Error Handling:**
   - If file not found: logs error and sends Telegram alert
   - If email fails: captured by Sentry, Telegram notification sent

#### Stage 3: Order Shipped (STAGE_ORDER_SHIPPED = 3)

**Trigger:** Deal moves to Stage 3

**Process Flow:**

1. **Customer Notification:**
   - Retrieves customer email from deal
   - Sends shipping notification email
   - Includes deal ID for reference

2. **Email Template:**
   - Uses simple notification template
   - Confirms order has been shipped
   - Provides tracking information if available

#### Product Description Generation (Webhook Trigger)

**Trigger:** New product created in Pipedrive

**Process Flow:**

1. **Webhook Reception:**
   - Separate webhook endpoint for product events
   - Extracts product ID and name

2. **AI Description Generation:**
   - **Primary:** Uses Perplexity **sonar-pro** for rich descriptions
   - **Fallback:** Uses **gpt-5-mini** if Perplexity unavailable
   - Generates professional product description

3. **Update Product:**
   - Saves generated description back to Pipedrive
   - Logs operation to statistics database

4. **Cost Tracking:**
   - Records action_type: "ai_description"
   - Logs model used, tokens, and USD cost

### Technology Stack

**Backend Framework:**
- FastAPI - Modern async web framework
- Uvicorn - ASGI server
- Pydantic - Data validation

**AI & Machine Learning:**
- OpenAI **gpt-5-mini** - Email parsing & product matching
- Perplexity **sonar-pro** - Product descriptions (optional)

**Document Generation:**
- docxtpl - Template-based DOCX generation
- python-docx - DOCX manipulation
- WeasyPrint - HTML to PDF conversion (legacy)
- Pillow - Image processing

**Message Queue:**
- RabbitMQ - Task queue and message broker
- Pika - Python RabbitMQ client

**Database:**
- PostgreSQL 15 - Production database
- SQLAlchemy - Async ORM
- asyncpg - PostgreSQL async driver

**Monitoring & Alerts:**
- Sentry SDK - Error tracking and performance monitoring
- Telegram Bot API - Real-time notifications

**Task Scheduling:**
- APScheduler 3.10.4 - Automated weekly reports

**Integrations:**
- Pipedrive API - CRM operations
- Requests - HTTP client
- Jinja2 - Template rendering

**DevOps & Infrastructure:**
- Docker & Docker Compose
- Ngrok - Webhook tunneling for development

**Utilities:**
- python-dotenv - Environment management
- Custom leasing calculator with Decimal precision
- Comprehensive logging system

## Requirements

- `fpdf2~=2.8.5` - PDF generation library
- `requests~=2.32.5` - HTTP client for API calls
- `openai~=2.8.1` - OpenAI and Perplexity API client
- `Jinja2~=3.1.6` - Template engine
- `python-dotenv~=1.2.1` - Environment variable management
- `fastapi~=0.122.0` - Web framework
- `uvicorn~=0.24.0` - ASGI server
- `pydantic~=2.11.9` - Data validation
- `python-multipart~=0.0.6` - Form data parsing
- `sqlalchemy` - Database ORM
- `asyncpg` - PostgreSQL async driver
- `weasyprint` - HTML to PDF conversion
- `docxtpl` - DOCX template engine
- `python-docx` - DOCX manipulation
- `pillow` - Image processing
- `sentry-sdk` - Error tracking
- `starlette` - ASGI framework components
- `apscheduler==3.10.4` - Task scheduling
- `pika` - RabbitMQ client

## RabbitMQ Message Queue

### Overview

The system uses RabbitMQ as a message broker to decouple webhook processing from task execution. This architecture ensures:
- Webhooks respond immediately (no timeout)
- Long-running tasks don't block the API
- Task retry capability
- Scalable worker processes

### Architecture

```
FastAPI Webhook → RabbitMQ Queue → Worker Process → Business Logic
     (Publisher)                      (Consumer)
```

### Message Flow

1. **Webhook Reception:**
   - FastAPI receives webhook from Pipedrive
   - Validates payload and determines task type
   - Publishes message to RabbitMQ queue
   - Returns 200 OK immediately

2. **Task Queuing:**
   - Message structure:
     ```json
     {
       "task_type": "generate_draft|send_invoice|order_shipped|describe_product",
       "deal_id": 123,
       "extra_data": {"product_id": 456, "product_name": "..."}
     }
     ```
   - Durable messages (persist to disk)
   - Delivery mode: 2 (persistent)

3. **Worker Consumption:**
   - Dedicated worker process consumes messages
   - QoS prefetch_count = 1 (sequential processing)
   - Routes to appropriate workflow function
   - Acknowledges message after completion

### Worker Process

**Starting the Worker:**
```bash
# Inside Docker container
docker exec -it pipedrive_bot python -m src.services.mq_worker

# Local development
python -m src.services.mq_worker
```

**Worker Features:**
- Automatic reconnection on connection loss
- Graceful shutdown handling
- Comprehensive error logging
- Telegram alerts on failures
- Sentry error capture

**Process Flow:**
```python
def process_message(ch, method, properties, body):
    # 1. Parse message
    data = json.loads(body)
    
    # 2. Route to workflow
    if task_type == "generate_draft":
        workflow.run_generate_draft(deal_id)
    elif task_type == "send_invoice":
        workflow.run_send_invoice(deal_id)
    # ... etc
    
    # 3. Acknowledge
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

### Monitoring

**RabbitMQ Management UI:**
- Access: http://localhost:15672
- View queue depth, message rates, consumers
- Manual message requeue if needed

**Key Metrics:**
- **Ready Messages:** Tasks waiting for processing
- **Unacked Messages:** Tasks currently being processed
- **Consumer Count:** Active worker processes
- **Message Rate:** Throughput per second

### Error Handling

**Connection Failures:**
- Automatic retry every 5 seconds
- Logs connection attempts
- Worker waits for RabbitMQ availability

**Task Failures:**
- Error logged to Sentry
- Telegram notification sent
- Message acknowledged (removed from queue)
- Deal moved to manual review if needed

**Message Persistence:**
- Messages survive RabbitMQ restarts
- Durable queue configuration
- No message loss on server restart

### Best Practices

1. **Single Worker for Sequential Processing:**
   - Prevents race conditions
   - Ensures deal integrity
   - QoS prefetch_count = 1

2. **Idempotent Operations:**
   - Duplicate webhook detection
   - File cleanup before generation
   - Safe retry logic

3. **Long-Running Tasks:**
   - Heartbeat disabled (heartbeat = 0)
   - No timeout on task processing
   - Progress logging

## Usage

### Setting Up Pipedrive Webhooks

1. **Get your webhook URL:**
   - If using Docker with ngrok: Check http://localhost:4040 for your public URL
   - If running locally: Use your server's public URL

2. **Configure Deal Update webhook in Pipedrive:**
   - Go to Settings → Webhooks
   - Create a new webhook for "Deal" events
   - Set URL to: `https://your-domain.com/webhook/pipedrive/`
   - Select triggers: "Deal stage updated", "Deal updated"

3. **Configure Product Creation webhook in Pipedrive:**
   - Create another webhook for "Product" events
   - Set URL to: `https://your-domain.com/webhook/products/`
   - Select trigger: "Product created"

4. **Test the integration:**
   - Move a deal to Stage 1 (Generate Draft) in Pipedrive
   - Check logs: `docker logs -f pipedrive_bot` or `bot_logs.log`
   - Create a new product to test AI description generation

### Docker Commands

```bash
# Start all services
docker-compose up -d

# View logs (API server)
docker logs -f pipedrive_bot

# View logs (RabbitMQ)
docker logs -f pipedrive_rabbitmq

# Start worker process
docker exec -it pipedrive_bot python -m src.services.mq_worker

# Restart the bot
docker-compose restart pipedrive_bot

# Stop all services
docker-compose down

# Rebuild after code changes
docker-compose up -d --build

# View RabbitMQ management interface
# Open http://localhost:15672 (guest/guest)
```

## API Endpoints

### `POST /webhook/pipedrive/`

**Description:** Handles incoming webhooks from Pipedrive for deal updates. Processes deal stage changes and triggers appropriate workflows through RabbitMQ.

**Request Body:** Pipedrive webhook payload
```json
{
  "current": {
    "id": 123,
    "stage_id": 18
  },
  "previous": {
    "stage_id": 17,
    "products_count": 2
  },
  "meta": {
    "action": "updated"
  }
}
```

**Responses:**
- `200 OK`: `{"status": "queued", "stage": 1, "reason": "stage_changed"}` - Task queued in RabbitMQ
- `200 OK`: `{"status": "ignored", "reason": "stage_did_not_change"}` - Stage unchanged (duplicate prevention)
- `200 OK`: `{"status": "ignored", "reason": "other_stage"}` - Not a target stage (1/2/3)
- `200 OK`: `{"status": "ignored", "reason": "no_data"}` - Missing deal ID or stage
- `200 OK`: `{"status": "error"}` - Processing error

**Duplicate Prevention Logic:**
- Processes if action is "added" (new deal)
- Processes if stage_id actually changed (old != new)
- For Stage 1: Processes if products_count or value changed (product updates)
- Ignores superficial updates (only update_time, files, or notes changed)

**Notes:**
- Uses RabbitMQ message queue to prevent webhook timeout
- Supports both `current` and `data` fields for compatibility
- Implements intelligent duplicate detection

### `POST /webhook/products/`

**Description:** Handles incoming webhooks from Pipedrive for product creation. Triggers AI-powered product description generation.

**Request Body:** Pipedrive product webhook payload
```json
{
  "current": {
    "id": 456,
    "name": "Industrial Freeze Dryer L80"
  },
  "meta": {
    "action": "added"
  }
}
```

**Responses:**
- `200 OK`: `{"status": "queued"}` - Description generation task queued
- `200 OK`: `{"status": "ignored"}` - Missing product ID or name
- `200 OK`: `{"status": "error"}` - Processing error

**Process Flow:**
1. Extracts product ID and name
2. Publishes "describe_product" task to RabbitMQ
3. Worker generates description using sonar-pro (primary) or gpt-5-mini (fallback)
4. Updates product in Pipedrive with generated description
5. Logs operation statistics (tokens, cost) to PostgreSQL

### `GET /`

**Description:** Health check endpoint to verify service status.

**Response:**
- `200 OK`: `{"status": "IamAlive"}`

### `GET /sentry-debug`

**Description:** Test endpoint to trigger Sentry error tracking (debugging purposes only).

**Response:**
- `500 Internal Server Error` - Intentional division by zero error

## Detailed Workflow Documentation

### Stage 1: Generate Draft

**Trigger:** Deal moves to Stage 1 or products are updated while in Stage 1

**Process Flow:**

1. **Cleanup:** Removes old generated files from Pipedrive to prevent conflicts

2. **Priority-Based Product Discovery:**

   **[Priority 1] Manual Products from "Products" Section:**
   - Retrieves manually added products from Pipedrive deal
   - For each product:
     - Fetches description from three sources (priority order):
       1. `item_description` - Deal-specific notes
       2. `description` - Product list field
       3. Full product database lookup if description is empty
     - Downloads product image using Pipedrive API
     - Saves image temporarily as `temp_prod_{id}.jpg`
   - If products found, skips to document generation

   **[Priority 2] Deal Title Search:**
   - If no manual products, cleans deal title:
     - Removes "Price request..." or "Deal - " prefix
     - Removes numeric suffixes (4+ digits at end)
     - Example: "Price request - TEFCOLD 1234" → "TEFCOLD"
   - Validates title is meaningful (>3 chars, not generic words like "darījums")
   - If valid, performs product search using cleaned title

   **[Priority 3] Last Note Content:**
   - If still no products, retrieves last note content
   - Takes first 150 characters as search term
   - Performs product search using this excerpt

3. **Two-Stage Product Matching:**

   **Stage A: Python Fuzzy Search**
   ```python
   # Downloads full catalog (~5000 products)
   # For each candidate:
   - Cleans search term (removes dots, dashes, spaces)
   - Checks if search term appears in product name
   - Checks if search term appears in product code
   - Checks if individual words (>2 chars) appear in name
   - Ranks by similarity
   # Returns top 30 candidates
   ```

   **Stage B: AI Selection with gpt-5-mini**
   ```
   Prompt: Analyzes top 30 candidates
   Rules:
   - Prioritizes model numbers and codes over names
   - Handles format variations (spaces, dots, dashes, case)
   - Returns best match ID or null if no good match
   
   Examples:
   - "L80" matches "L.80" or "L-80"
   - "CRPF2130" matches "CRPF2130WD"
   - Partial matches accepted if context is clear
   ```

   **If Match Found:**
   - Downloads full product details
   - Retrieves product image
   - Adds to items list

   **If No Match:**
   - Creates new product in Pipedrive
   - Sets price = 0 (flags for manual review)
   - Records product ID

4. **Customer Data Collection:**
   - Retrieves person email from deal
   - Attempts to get phone from deal details
   - If no phone, parses last note with AI (gpt-5-mini) to extract it

5. **DOCX Document Generation:**
   - Uses docxtpl template engine with `template.docx`
   - Embeds product images (file paths to temp files)
   - Calculates subtotal from all items
   - Calculates VAT (21% of subtotal)
   - Generates leasing offer:
     ```
     Term: 36 months
     Rate Selection (tiered)
     Monthly Payment = P * [r(1+r)^n] / [(1+r)^n - 1]
     Uses Decimal precision
     ```
   - Saves as `invoice_{deal_id}_{timestamp}.docx`

6. **Upload & Validation:**
   - Validates DOCX file exists
   - Checks file size > 0 bytes
   - Uploads to Pipedrive deal files
   - Logs detailed error if upload fails

7. **Quality Control & Notes:**
   - Creates comprehensive HTML note with:
     - Document preview/summary
     - Product list with prices
     - Total amount
   - If any product has price = 0:
     - Creates manual review activity in Pipedrive
     - Moves deal to Stage 4 (Manual Check)
     - Adds task for manager to verify products

8. **Cleanup:**
   - Deletes temporary DOCX file
   - Removes downloaded product images (`temp_prod_*.jpg`)

9. **Statistics Logging:**
   - Records action_type: "docx_generated" in PostgreSQL
   - Logs all AI operations:
     - Model name (gpt-5-mini)
     - Input tokens used
     - Output tokens generated
     - Cost in USD
   - Links statistics to deal ID

**AI Prompt Strategy for Email Parsing:**
```
Input: Email or note text
Output: JSON with:
{
  "first_name": "...",
  "last_name": "...",
  "email": "...",
  "phone": "...",
  "organization": "...",
  "items": [
    {"search_term": "Brand Model", "quantity": 1}
  ]
}

Special handling:
- Extracts brand and model from product URLs
- Removes category words (gazes, cepsanas, virsma)
- Handles multilingual content (EN/LV/RU)
- Parses quantity from various formats (2 gab, 3 шт)
```

### Stage 2: Send Quote

**Trigger:** Deal moves to Stage 2

**Process Flow:**

1. **File Retrieval:**
   - Searches deal files for DOCX invoice
   - Filters by filename pattern: contains "invoice_"
   - Validates file exists and is accessible

2. **Email Preparation:**
   - Retrieves customer email from deal person
   - Fetches deal details for personalization
   - Renders HTML email template using Jinja2:
     ```html
     Subject: Your Quote from [Company]
     Body: Personalized message with customer name
     Attachment: invoice_{deal_id}.docx
     ```

3. **Email Delivery:**
   - Sends via Pipedrive mailbox API
   - Uses configured template (PIPEDRIVE_TEMPLATE_ID = 11)
   - Logs delivery status

4. **Error Handling:**
   - If file not found: Logs error + Telegram alert
   - If email fails: Sentry captures error + Telegram notification
   - All errors include deal link for quick access

### Stage 3: Order Shipped

**Trigger:** Deal moves to Stage 3

**Process Flow:**

1. **Customer Notification:**
   - Retrieves customer email from deal
   - Sends shipping notification email
   - Includes deal ID for customer reference

2. **Email Template:**
   - Simple notification confirming shipment
   - Provides tracking information if available
   - Thanks customer for their business

### Product Description Generation

**Trigger:** New product created in Pipedrive (webhook)

**Process Flow:**

1. **Webhook Reception:**
   - Separate endpoint: `/webhook/products/`
   - Extracts product ID and name from payload

2. **Task Queuing:**
   - Publishes "describe_product" task to RabbitMQ
   - Includes product_id and product_name in extra_data

3. **AI Description Generation:**
   - **Primary Method:** Uses Perplexity **sonar-pro** API
     - Generates rich, detailed product descriptions
     - Optimized for commercial equipment
   - **Fallback Method:** Uses **gpt-5-mini**
     - Activates if Perplexity unavailable
     - Maintains description quality

4. **Update Product:**
   - Saves generated description back to Pipedrive
   - Updates description field via API

5. **Cost Tracking:**
   - Records in PostgreSQL:
     - action_type: "ai_description"
     - model_name: "sonar-pro" or "gpt-5-mini"
     - input_tokens, output_tokens
     - cost_usd (calculated from token counts)
     - related_id: product_id

## Configuration


### Stage Configuration (src/config.py)

```python
STAGE_GENERATE_DRAFT = 1   # Trigger draft generation
STAGE_SEND_INVOICE = 2     # Send quote to customer
STAGE_ORDER_SHIPPED = 3   # Send shipping notification
STAGE_MANUAL_CHECK = 4     # Manual review required
```

### Custom Fields

The system uses Pipedrive custom fields:
- `KEY_BRAND` - Product brand identifier (ID: `a653db32429e0fca5dbc067c9a38383739210084`)

### Leasing Calculator Configuration

The system includes a sophisticated leasing calculator with tiered interest rates based on the total amount. It uses Python's `Decimal` type for precise financial calculations.

**Features:**
- Generates 36-month leasing offers
- Annuity payment formula with precise monthly rate calculation
- Automatic rate selection based on total amount
- Returns structured `LeasingOffer` dataclass with monthly payment and total cost

## Error Handling

### Multi-Layer Error Tracking

**1. Sentry Integration:**
- Automatic error capture with full stack traces
- Performance monitoring and tracing
- Integration with FastAPI and logging
- Configurable via `SENTRY_DSN` environment variable
- Gracefully disabled if DSN not provided

**2. Telegram Notifications:**
- Real-time alerts for critical errors
- Includes deal ID and direct link to Pipedrive
- HTML-formatted messages with error details
- Automatic fallback if Telegram unavailable
- Configure with `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`

**3. Comprehensive Logging:**
- Centralized logger configuration in `src/utils/logger.py`
- Detailed context for all operations
- File-based logging to `bot_logs.log`
- Traceback capture with `exc_info=True`
- Structured log messages with timestamps

**4. Global Exception Handler:**
- FastAPI global exception handler catches all unhandled errors
- Automatically notifies administrators via Telegram
- Returns user-friendly error messages
- Logs full error context to Sentry

**5. Graceful Degradation:**
- System continues operating when optional services unavailable
- Perplexity API failures fallback to gpt-5-mini
- Missing images don't crash document generation
- RabbitMQ auto-reconnect on connection loss
- Database connection retry logic

**6. Validation & Quality Control:**
- File size validation before upload (checks > 0 bytes)
- Product price validation (creates review task if price = 0)
- Email and phone format validation
- Duplicate webhook detection
- Deal stage change verification

**7. Task Queue Resilience:**
- RabbitMQ durable messages survive restarts
- Heartbeat management for long-running tasks
- Automatic reconnection with exponential backoff
- QoS control prevents task overload
- Worker process isolation from API server

## Testing

```bash
# Run all tests
pytest

# Run specific test file
pytest src/test/test_workflow.py

# Run with coverage
pytest --cov=src tests/
```

## Monitoring & Statistics

### Real-Time Monitoring

**Sentry Error Tracking:**
- Automatic capture of all unhandled exceptions
- Full stack traces with context
- Performance monitoring and transaction tracing
- Custom tags for deal IDs and workflow stages
- Sample rate: 100% for traces and profiles

**Telegram Notifications:**
- Instant alerts for critical errors
- Weekly performance reports
- HTML-formatted messages with:
  - Deal ID and direct Pipedrive link
  - Error message and context
  - Timestamp and severity

**Application Logs:**
- Centralized logging to `bot_logs.log`
- Log rotation and retention
- Structured log format:
  ```
  [TIMESTAMP] [LEVEL] Message with context
  ```
- DEBUG level for development
- INFO/WARNING/ERROR for production

### Statistics Database

**PostgreSQL BotStat Table:**
```sql
CREATE TABLE bot_stats (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    action_type VARCHAR,  -- "docx_generated", "ai_description", "email_parsed"
    model_name VARCHAR,   -- "gpt-5-mini", "sonar-pro"
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cost_usd FLOAT DEFAULT 0.0,
    related_id INTEGER    -- Deal ID or Product ID
);
```

**Tracked Operations:**
1. **docx_generated** - Document generation events
2. **ai_description** - Product description generation
3. **email_parsed** - Email/note parsing operations

**Cost Calculation:**
- OpenAI gpt-5-mini pricing applied automatically
- Perplexity sonar-pro pricing tracked
- Input/output tokens recorded separately
- USD cost calculated per operation

### Weekly Automated Reports

**Schedule:** Every Monday at 9:00 AM (APScheduler)

**Report Contents:**
```
📊 Weekly Bot Performance Report
(DD.MM - DD.MM)

Documents Generated: X
AI Descriptions: Y

Cost: $X.XXXX
Total Tokens: XXX,XXX
```

**Implementation:**
- Queries PostgreSQL for last 7 days
- Aggregates counts, costs, and tokens
- Sends via Telegram
- Automatic error handling

### Metrics Access

**RabbitMQ Management UI:**
- URL: http://localhost:15672
- Default credentials: guest/guest (or configured)
- Features:
  - Queue depth monitoring
  - Message rates
  - Consumer status
  - Connection health

**Database Queries:**
```sql
-- Cost per day
SELECT DATE(timestamp), SUM(cost_usd)
FROM bot_stats
GROUP BY DATE(timestamp)
ORDER BY DATE(timestamp) DESC;

-- Most expensive operations
SELECT action_type, model_name, AVG(cost_usd) as avg_cost
FROM bot_stats
GROUP BY action_type, model_name;

-- Token usage
SELECT model_name, 
       SUM(input_tokens) as total_input,
       SUM(output_tokens) as total_output
FROM bot_stats
GROUP BY model_name;
```

**Health Check Endpoints:**
- `GET /` - Basic health check
- `GET /sentry-debug` - Test Sentry integration

## Troubleshooting

### Common Issues

**Webhook not triggering:**
- Check ngrok tunnel is active: `http://localhost:4040`
- Verify webhook URL in Pipedrive settings
- Check deal is moving to correct stage (1, 2, or 3)
- Review webhook logs in Pipedrive dashboard
- Check bot_logs.log for incoming webhook data

**Worker process not consuming tasks:**
- Ensure RabbitMQ is running: `docker ps | grep rabbitmq`
- Check worker process is started: `docker exec -it pipedrive_bot python -m src.services.mq_worker`
- View RabbitMQ queue status: http://localhost:15672
- Check for connection errors in logs
- Verify RABBITMQ_URL environment variable

**Document generation fails:**
- Ensure docxtpl and python-docx installed
- Check template.docx exists in `src/templates/`
- Verify product images download successfully
- Check temp directory permissions
- Review logs for specific error messages

**PDF generation fails (legacy):**
- Ensure WeasyPrint dependencies installed (see Dockerfile)
- Check font files exist in `src/utils/fonts/`
- Verify logo file exists at `img/logo_1x.png`
- Check HTML template syntax in `src/templates/estimate_pdf.html`

**Database connection errors:**
- Verify PostgreSQL container is running: `docker ps`
- Check DATABASE_URL in .env
- Ensure database credentials match docker-compose.yml
- Wait for PostgreSQL to fully initialize (may take 30s)
- Check database logs: `docker logs db`

**RabbitMQ connection errors:**
- Verify RabbitMQ container is running
- Check RABBITMQ_URL format: `amqp://user:pass@host:5672/`
- Ensure credentials match RABBITMQ_DEFAULT_USER and RABBITMQ_DEFAULT_PASS
- Check RabbitMQ logs: `docker logs pipedrive_rabbitmq`
- Verify network connectivity between containers

**AI parsing errors:**
- Verify OPENAI_API_KEY is valid
- Check OpenAI API quota and limits
- Review logs for specific error messages
- Ensure timeout is sufficient (default: 60s)
- Check network connectivity to OpenAI API

**Product matching not working:**
- Verify product catalog is accessible in Pipedrive
- Check fuzzy matching algorithm parameters
- Review AI selection prompt for edge cases
- Ensure product codes and names are consistent
- Check logs for matching candidates and AI selection

**Email sending fails:**
- Verify Pipedrive mailbox API access
- Check PIPEDRIVE_TEMPLATE_ID is correct
- Ensure customer email address is valid
- Check Pipedrive email sending limits
- Review email transport logs

**Statistics not recording:**
- Verify PostgreSQL connection is active
- Check BotStat table exists (created automatically)
- Ensure stats_service.py is imported
- Review database write permissions
- Check for SQL errors in logs

**Weekly reports not sending:**
- Verify APScheduler is running
- Check TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID
- Ensure scheduler is started in main.py
- Review scheduled task configuration
- Check timezone settings

**Sentry errors not appearing:**
- Verify SENTRY_DSN is correct
- Check Sentry project quota
- Ensure sentry-sdk is installed
- Review Sentry integration configuration
- Check network connectivity to Sentry

## Security Considerations

### API Key Management

**Environment Variables:**
- Never commit `.env` file to version control
- Add `.env` to `.gitignore`
- Use strong, unique API keys
- Rotate keys periodically
- Use separate keys for development and production

**Docker Secrets (Production):**
```yaml
# docker-compose.yml
services:
  pipedrive_bot:
    secrets:
      - openai_key
      - pipedrive_token
secrets:
  openai_key:
    file: ./secrets/openai_key.txt
  pipedrive_token:
    file: ./secrets/pipedrive_token.txt
```

### Network Security

**Webhook Endpoints:**
- Use HTTPS in production (not HTTP)
- Implement webhook signature verification (if Pipedrive supports)
- Rate limiting to prevent abuse
- IP whitelisting for Pipedrive webhooks

**Database:**
- Use strong PostgreSQL passwords
- Restrict database access to application containers only
- Enable SSL/TLS for database connections in production
- Regular automated backups

**RabbitMQ:**
- Change default credentials (guest/guest)
- Use strong passwords for RABBITMQ_DEFAULT_USER
- Restrict management UI access
- Enable SSL/TLS for production

### Data Privacy

**Customer Information:**
- Encrypt sensitive data at rest
- Use HTTPS for all external API calls
- Temporary files deleted after processing
- Minimal data retention (only statistics)
- Comply with GDPR/local privacy regulations

**Logging:**
- Avoid logging sensitive data (passwords, full API keys)
- Sanitize logs before external monitoring
- Implement log rotation and retention policies

### Access Control

**Pipedrive API:**
- Use least-privilege API tokens
- Restrict token permissions to required operations
- Monitor API usage for anomalies

**Telegram Bot:**
- Restrict bot to specific chat IDs
- Validate chat_id in notifications
- Don't expose sensitive data in messages

## Performance Optimization

### Caching Strategies

**Product Catalog:**
- Cache product catalog in memory (5-15 minute TTL)
- Reduce API calls to Pipedrive
- Implement background refresh

**Image Caching:**
- Store frequently used images locally
- Implement CDN for production
- Compress images before embedding

### Async Operations

**Database Queries:**
- Use connection pooling (SQLAlchemy async)
- Batch operations where possible
- Index frequently queried columns

**API Calls:**
- Use async HTTP requests (httpx instead of requests)
- Parallel API calls where safe
- Implement retry with exponential backoff

### Resource Management

**Memory:**
- Limit product catalog size in memory
- Stream large files instead of loading fully
- Clean up temporary files promptly

**RabbitMQ:**
- QoS prefetch_count = 1 prevents overload
- Monitor queue depth
- Scale workers horizontally if needed

**Docker:**
- Set memory limits per container
- Use multi-stage builds to reduce image size
- Implement health checks

## Best Practices

### Development Workflow

**Code Quality:**
- Follow PEP 8 style guidelines
- Use type hints for function parameters
- Write docstrings for complex functions
- Keep functions small and focused

**Testing:**
- Unit tests for business logic
- Integration tests for workflows
- Mock external API calls in tests
- Test error handling paths

**Version Control:**
```bash
# Feature branches
git checkout -b feature/new-workflow

# Commit messages
git commit -m "feat: add product description generation"
git commit -m "fix: resolve duplicate webhook processing"
git commit -m "docs: update API endpoint documentation"
```

### Deployment Best Practices

**Pre-Deployment:**
- Test in staging environment
- Review all environment variables
- Backup database before updates
- Check API rate limits and quotas

**Monitoring:**
- Set up Sentry alerts for critical errors
- Configure Telegram notifications
- Monitor RabbitMQ queue depth
- Track API costs and usage

**Rollback Plan:**
- Keep previous Docker images
- Document rollback procedure
- Test rollback in staging
- Maintain database backups

### Operational Guidelines

**Regular Maintenance:**
- Review and clean old logs weekly
- Update dependencies monthly
- Rotate API keys quarterly
- Review and optimize costs monthly

**Incident Response:**
1. Check Sentry for error details
2. Review Telegram notifications
3. Inspect RabbitMQ queue status
4. Check application logs
5. Verify external service status (OpenAI, Pipedrive)

**Scaling Considerations:**
- Horizontal scaling: Run multiple worker processes
- Database: Consider read replicas for analytics
- RabbitMQ: Use clustering for high availability
- Caching: Implement Redis for shared cache

## AI Model Configuration

### OpenAI gpt-5-mini

**Use Cases:**
- Email and note parsing
- Product matching from candidates
- Customer data extraction
- General intelligence tasks

**Configuration:**
```python
model = "gpt-5-mini"
timeout = 60.0
response_format = {"type": "json_object"}  # For structured data
```

**Cost Optimization:**
- Use structured prompts to reduce token usage
- Cache common queries
- Monitor token consumption via statistics
- Set reasonable timeout limits

### Perplexity sonar-pro

**Use Cases:**
- Product description generation
- Rich content creation
- Technical specifications

**Configuration:**
```python
base_url = "https://api.perplexity.ai"
model = "sonar-pro"
```

**Fallback Strategy:**
- Primary: Perplexity sonar-pro
- Fallback: OpenAI gpt-5-mini
- Ensures continuous operation

**Cost Tracking:**
- All API calls logged to PostgreSQL
- Token usage recorded per operation
- Weekly cost reports via Telegram

## Contributing

We welcome contributions! Please follow these guidelines:

### Getting Started

1. **Fork the repository**
   ```bash
   git clone https://github.com/yourusername/Assistents.git
   cd Assistents
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Set up development environment**
   ```bash
   python -m venv .venv
   .venv\Scripts\activate  # Windows
   pip install -r requirements.txt
   ```

### Development Guidelines

**Code Style:**
- Follow PEP 8 Python style guide
- Use meaningful variable names
- Add type hints to function signatures
- Write docstrings for public functions

**Commit Messages:**
- Use conventional commits format:
  - `feat:` New features
  - `fix:` Bug fixes
  - `docs:` Documentation changes
  - `refactor:` Code refactoring
  - `test:` Test additions or updates
  - `chore:` Maintenance tasks

**Testing:**
- Add tests for new features
- Ensure existing tests pass
- Test with actual Pipedrive webhooks when possible
- Mock external API calls in unit tests

**Documentation:**
- Update README.md for new features
- Add docstrings to new functions
- Update configuration examples if needed
- Include code comments for complex logic

### Pull Request Process

1. **Update documentation**
   - README.md with new features
   - Inline code comments
   - Configuration examples

2. **Test thoroughly**
   - Run existing tests
   - Add new tests for your changes
   - Test in Docker environment
   - Verify with actual Pipedrive integration

3. **Submit pull request**
   - Provide clear description of changes
   - Reference related issues
   - Include screenshots for UI changes
   - Wait for review and feedback

4. **Code review**
   - Address reviewer comments
   - Make requested changes
   - Keep commits clean and focused

### Areas for Contribution

**High Priority:**
- Additional workflow stages
- Enhanced error handling
- Performance optimizations
- Additional AI model integrations
- Webhook signature verification

**Features:**
- Multi-language support
- Custom document templates
- Advanced product matching algorithms
- Additional notification channels (Slack, Discord)
- Analytics dashboard

**Documentation:**
- Deployment guides for different platforms
- Troubleshooting scenarios
- API integration examples
- Video tutorials

**Testing:**
- Unit test coverage
- Integration tests
- Load testing
- Security testing

## License

This project is licensed under the MIT License. See the LICENSE file for details.

## Support

### Getting Help

**Documentation:**
- Read this README thoroughly
- Check inline code comments
- Review example configurations

**Debugging:**
1. **Check Application Logs:**
   ```bash
   # Docker
   docker logs -f pipedrive_bot
   
   # Local
   tail -f bot_logs.log
   ```

2. **Review Webhook Logs:**
   - Pipedrive Dashboard → Settings → Webhooks
   - Check delivery status and response codes

3. **Monitor RabbitMQ:**
   - Management UI: http://localhost:15672
   - Check queue depth and consumer status

4. **Check Sentry:**
   - Review error reports with full stack traces
   - Analyze error patterns and frequency

5. **Telegram Notifications:**
   - Real-time error alerts with deal links
   - Weekly performance reports

**Common Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pipedrive API Reference](https://developers.pipedrive.com/)
- [OpenAI API Documentation](https://platform.openai.com/docs/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials/)
- [Docker Compose Reference](https://docs.docker.com/compose/)

### Reporting Issues

**Bug Reports:**
Include the following information:
- Python version and OS
- Docker version (if using Docker)
- Full error message and stack trace
- Steps to reproduce
- Expected vs actual behavior
- Relevant configuration (sanitized)

**Feature Requests:**
- Clear description of desired functionality
- Use case and business value
- Potential implementation approach
- Any relevant examples or references

### Community

**Contact:**
- GitHub Issues: For bug reports and feature requests
- GitHub Discussions: For questions and general discussion
- Pull Requests: For code contributions





## Acknowledgments

**Technologies:**
- FastAPI for the excellent async web framework
- OpenAI for powerful AI capabilities
- Perplexity for enhanced content generation
- RabbitMQ for reliable message queuing
- Pipedrive for the comprehensive CRM API
- Sentry for world-class error tracking

**Community:**
- Contributors and testers
- Open source projects that made this possible
- Documentation and tutorial authors

## Changelog

### Version 2.0.0 (Current)
-  Added RabbitMQ message queue integration
-  Implemented Sentry error tracking
-  Added Telegram notifications
-  PostgreSQL statistics tracking
-  Weekly automated reports
-  Product description generation via webhook
-  DOCX document generation (replaced PDF)
-  Enhanced duplicate prevention logic
-  Multi-priority product discovery
-  Fixed description merging from multiple sources
-  Fixed stage change detection
-  Comprehensive documentation update

### Version 1.0.0
- Initial release with PDF generation
- Basic webhook processing
- AI-powered product matching
- Email automation

---

*Last Updated: February 3, 2026*


