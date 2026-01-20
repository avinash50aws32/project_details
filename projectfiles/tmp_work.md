## PROPOSED SYSTEM: Functional Flow, Architecture, Technology Used

### SYSTEM 1: CORPORATE PAYMENT DOCUMENT PROCESSING & REMITTANCE SYSTEM [System Enhancements]

#### Project Overview

Integrated payment processing system handling corporate remittance with automated routing, AI-powered document processing for client onboarding, and ServiceNow workflow integration to enable straight-through processing for qualified transactions.

#### Business Context & Problem Statement

**Business Challenge:**

- Standard Chartered Bank's corporate payment processing faced manual routing decisions across multiple payment systems causing bottlenecks
- Traditional manual processing created delays (4-6 hours per transaction) and routing errors (12% error rate)
- Client onboarding required 30-45 minutes of manual document review for 2D/non-2D forms
- Inefficient approval workflows and limited payment status visibility
- Incomplete audit trails created compliance risks

**Pain Points:**

- `Manual routing` decisions across SWIFT, ACH, Wire, and RTP systems causing 4-6 hour delays per transaction
- 12% `routing error` rate due to complex business rules across transaction types, amounts, countries, and regulatory requirements (OFAC, EU sanctions)
- `Document onboarding time`: 30-45 minutes per client for manual review of company details, Tax IDs, bank info, and signatories
- `Multi-tier approval` workflows (<$50K auto, $50K-$500K manager, >$500K VP) lacked SLA tracking and timeout handling
- `Limited payment status visibility` for customers and operations teams
- Compliance `audit trails` incomplete and manual

**Stakeholder Requirements:**

- Operations team needed automated routing engine with 50+ business rules reducing processing time from 4-6 hours to <2 minutes
- Compliance team required AI-powered document processing with confidence-based routing (High >95%, Medium 85-95%, Low <85%)
- Customer service needed real-time payment status visibility and tracking
- Finance team needed ServiceNow workflow integration with SLA tracking and automated escalation
- IT operations needed routing error reduction from 12% to <1% and 98% SLA compliance target

#### Functional Flow & Process Design

##### FLOW 1: Client Onboarding (Document Processing)

```
Stage 1: Form Submission & Document Upload
#=> Functional: Client fills form → System generates PDF with barcode → Client prints/signs/scans → Uploads to system → Documents stored in Blob → Message queued

[Payment Web Portal - React TypeScript Frontend + ASP.NET Core 8 Backend]
**Hosting:** Frontend on Azure Static Web Apps, Backend API on Azure App Service (Premium P1V2), All microservices on Azure Kubernetes Service (AKS) with auto-scaling

**Authentication & Authorization Flow:**
• **User Authentication:** Azure AD B2C for external clients, Azure AD for internal employees
• **User Authorization:** RBAC via Azure AD groups/roles (ClientOnboardingAgent, BranchOfficer)
• **Frontend → Backend API:** React calls Backend API with user JWT token (from Azure AD authentication)
• **Backend API → Microservices:** ASP.NET Core (App Service) acts as API Gateway, routes to AKS microservices with service JWT
• **Microservices ↔ Microservices:** Call each other via REST APIs with JWT bearer tokens for authentication
• **Microservices → Azure Resources:** Managed Identity for AKS pods accessing Blob Storage, SQL Database, Service Bus, Key Vault

→ Input: Client registration request from web browser (corporate client self-service or branch employee assisted)
→ Actions:
   **Primary Path - Corporate Client Self-Registration (Self-Service Onboarding):**
   • Client accesses bank's corporate onboarding portal via public website
   • Client creates account with Azure AD B2C:
     - Provides corporate email (e.g., cfo@company.com)
     - Email verification via OTP (6-digit code sent to email)
     - Sets password meeting complexity requirements
     - No MFA required for initial registration (added post-approval)
   • Azure AD B2C creates guest user account with basic permissions (can only access onboarding forms)
   • Client fills onboarding form via React UI:
     - Company details (legal name, registration number, tax ID, incorporation country)
     - Business type and industry
     - Authorized signatories (names, titles, email addresses)
     - Bank account details (currency, account type, expected transaction volumes)
     - Contact information (registered address, phone, website)

   **Alternative Path - Branch Employee Assisted (In-Person or Phone Onboarding):**
   • Branch employee authenticates with OAuth 2.0 + MFA (Azure AD integration)
   • RBAC validates employee permissions (role: ClientOnboardingAgent, BranchOfficer)
   • Employee fills form on behalf of client during branch visit or phone call
   • Employee ID tracked in submission record for audit trail

   • ASP.NET Core backend validates input, generates pre-filled PDF with embedded barcode (ZXing.Net)
   • EF Core 8 inserts client submission record (Set A) into Azure SQL Database (ClientSubmissions table)
   • Redis Cache stores session data (30-min TTL) for multi-step form progress
→ Output: Returns unique Submission ID and barcode-embedded PDF to client

[Client Action - Offline]
→ Client prints form, obtains authority signatures, scans documents (2D/non-2D forms)

[Payment Web Portal - React TypeScript + ASP.NET Core 8]
→ Input: Scanned document upload request (PDF/images) from client or branch employee
→ Actions:
   • Client uploads signed forms and supporting documents (2D/non-2D) into system (through branch or online)
   • OAuth 2.0 authenticates user, RBAC validates document upload permissions
   • Frontend performs client-side validation (file size <10MB, allowed formats: PDF/JPG/PNG)
   • Backend pre-upload validation:
     - ASP.NET Core validates file content type (prevents malicious uploads)
     - **Document completeness check BEFORE Azure upload:**
       * Fetches required document rules from Redis cache (sourced from ServiceNow, 15-min TTL)
       * Compares selected documents vs required documents by onboarding type (2D/Non-2D)
       * Validates minimum document count (e.g., 2D form requires: signed form + ID proof + bank statement)
       * If validation fails:
         - Returns error to frontend with missing document list
         - Submission status remains: Draft (not PendingProcessing)
         - Client can add/replace documents and resubmit
         - No Azure storage costs incurred for incomplete submissions

   • Backend processes upload (only after completeness validation passes):
     - Uploads documents to Azure Blob Storage container (onboarding-docs) with unique tracking ID
     - Publishes message to Azure Service Bus queue (onboarding-processing-queue) with JSON payload:
       {submissionId, blobUrls[], documentTypes[], uploadTimestamp}
→ Output: Returns upload confirmation with tracking ID to client, message queued for processing

---

Stage 2: AI Document Processing & Data Extraction
#=> Functional: Blob upload triggers function → Extract barcode (Set B) → OCR extraction (Set C) → Route to validation queue

[Azure Blob Storage]
→ Event: New blob uploaded triggers Azure Function (BlobTrigger)

[Azure Function - Document Processor (.NET Core 8)]
**Hosting:** Azure Functions Consumption Plan (auto-scales, pay-per-execution)
→ Input: Blob upload event from Azure Blob Storage (onboarding-docs container) with document metadata
→ Actions:
   • Retrieves document from Azure Blob Storage to Azure Function memory (BlobClient.DownloadAsync())
   • Barcode extraction using ZXing.Net BarcodeReader:
     - Decodes QR code from scanned form
     - Extracts embedded JSON (Company Name, Tax ID, Bank Account - AES-256 encrypted)
     - Saves barcode data (Set B) to Azure SQL Database (BarcodeExtractions table)
   • Posts document to Azure AI Document Intelligence REST API (POST /formrecognizer/documentModels/prebuilt-document:analyze)
   • Receives OCR results with confidence scores for each field
   • Saves OCR extracted data (Set C) to Azure SQL (OCRExtractions table)
   • Publishes completion message to Azure Service Bus (validation-queue)
→ Output: Sets B and C stored in Azure SQL, message in validation-queue

---

Stage 3: Data Comparison & Business Rule Validation
#=> Functional: Compare Sets A/B/C → Create Set D → Execute business rules → Route to approval/review/rejection queue → Send notifications

[Document Processing Service - .NET Core 8 Microservice]
**Hosting:** Azure Kubernetes Service (AKS) with Horizontal Pod Autoscaler (HPA) based on CPU/memory
**Service Bus Consumption:** .NET SDK with IMessageReceiver interface
  - Event-driven processing: ServiceBusTrigger attribute binds to queue
  - Message handling pattern: CompleteMessageAsync() for success, AbandonMessageAsync() for transient failures
  - Dead-letter queue: Messages failing after 10 delivery attempts moved to validation-queue-deadletter
  - Consumer group: validation-consumer with max concurrent calls: 32
→ Input: Consumes message from Azure Service Bus (validation-queue) published by Azure Function Document Processor
→ Actions:
   • Retrieves Set A (original submission), Set B (barcode), Set C (OCR) from Azure SQL using EF Core (GET query with joins)
   • Data reconciliation algorithm:
     - Company name fuzzy matching (Levenshtein distance <3 chars)
     - Tax ID exact match validation
     - Bank account number comparison
     - Confidence scoring: High >95% match, Medium 85-95%, Low <85%
   • Creates finalized entity set (Set D) with merged/corrected data using rule-based conflict resolution:
     - If Set A = Set B = Set C → Use common value (confidence: 100%)
     - If Set B = Set C ≠ Set A → Trust scanned documents (barcode + OCR agree, confidence: 95%)
     - If Set A = Set B ≠ Set C → Trust original + barcode (OCR error likely, confidence: 90%)
     - If all 3 differ → Flag for manual review (confidence: <85%), use Set B as default
     - Priority order: Set B (barcode) > Set C (OCR) > Set A (manual entry) when conflicts exist
   • Saves Set D to Azure SQL (FinalizedEntities table) with reconciliation metadata:
     - Field-level confidence scores per attribute
     - Source selection flags (which set chosen: A/B/C)
     - Mismatch indicators for manual review
     - Reconciliation timestamp and algorithm version
→ Output: Set D created with metadata, ready for rule validation

[Business Rule Validation Engine - .NET Core 8 Microservice]
**Hosting:** Azure Kubernetes Service (AKS) with Horizontal Pod Autoscaler (HPA)
**Microservice Communication:**
  - Receives Set D data via Azure Service Bus message or synchronous REST API call from Document Processing Service
  - Calls Compliance Service synchronously via REST (POST /api/compliance/screen)
  - Calls ServiceNow REST API for workflow creation (POST /api/now/workflow/review)
→ Input: Set D entity data from Document Processing Service (from Azure SQL FinalizedEntities table)
→ Actions:
   • Fetches business rules from ServiceNow REST API (GET /api/now/table/business_rules?type=onboarding&active=true)
   • Rules cached in Redis (15-min TTL) to reduce ServiceNow API calls
   • Executes validation rules using .NET Rules Engine library:
     - Tax ID format verification (regex patterns)
     - Duplicate client check (SQL query: SELECT COUNT(*) FROM ClientProfiles WHERE TaxID = @taxId)
     - Sanction list screening via Compliance Service REST API (POST /api/compliance/screen with client details)
   • Aggregates validation results (pass/fail per rule, overall confidence score)
   • Decision routing logic:
     - High Confidence + All Rules Pass → Publishes to Azure Service Bus (auto-approval-queue)
     - Medium Confidence OR Partial Rules → Publishes to Azure Service Bus (manual-review-queue) + Calls ServiceNow API
     - Low Confidence OR Rules Fail → Publishes to Azure Service Bus (rejection-queue)
   • For manual-review cases: Calls ServiceNow REST API (POST /api/now/workflow/review) with Sets A/B/C/D for case creation
   • Logs event to Azure Cosmos DB (event sourcing pattern for audit trail)
→ Output: Routed messages to appropriate Service Bus queues, ServiceNow case created for manual reviews

[Notification Service - .NET Core 8 Microservice]
**Hosting:** Azure Kubernetes Service (AKS) with Horizontal Pod Autoscaler (HPA)
→ Input: Listens to all routing queues on Azure Service Bus (auto-approval-queue, manual-review-queue, rejection-queue) published by Business Rule Validation Engine
→ Actions:
   • Sends email notifications via SendGrid API (approval confirmations, review requests)
   • SignalR broadcasts real-time status updates to Admin Dashboard (React TypeScript) via WebSocket connections
   • Logs notification delivery status to Azure SQL (NotificationDelivery table)
→ Output: Multi-channel notifications delivered (email, SignalR push, SMS)

---

Stage 4: Auto-Approval Path (High Confidence)
#=> Functional: Consume auto-approval message → Create client profile → Assign account manager → Send confirmation

[Client Onboarding Service - .NET Core 8 Microservice]
**Hosting:** Azure Kubernetes Service (AKS) with Horizontal Pod Autoscaler (HPA)
→ Input: Consumes message from Azure Service Bus (auto-approval-queue) published by Business Rule Validation Engine
→ Actions:
   • Retrieves approved Set D data from Azure SQL (FinalizedEntities table)
   • Creates client profile in Azure SQL (ClientProfiles table) using EF Core (INSERT with transaction)
   • Generates unique Client ID (GUID)
   • Assigns account manager based on region/segment (business logic from lookup tables)
   • Saves audit log to Azure Cosmos DB (event sourcing: ClientCreated event with full Set D data)
   • Publishes success message to Azure Service Bus (notification-queue) with JSON payload
→ Output: Client profile created, confirmation notification sent

---

Stage 5: Manual Review Path (Medium/Low Confidence) & Rejection & Remediation Path
#=> Functional: Create ServiceNow case → Maker reviews & corrects → Checker approves → Create client profile → Feed corrections to ML
#=> Functional: RM reviews rejection → Correct & resubmit OR Notify client → Track remediation SLA

[ServiceNow Platform - Workflow Orchestration]
→ Input: REST API call from Business Rule Validation Engine (POST /api/now/workflow/review) with Sets A/B/C/D comparison data
→ Actions:
   • **ServiceNow workflow initiation:**
     - **Endpoint:** POST /api/now/workflow/review
     - **Purpose:** Creates review case in ServiceNow Maker-Checker queue
     - **Payload:** {submissionId, setsComparison: {A, B, C, D}, confidenceScores, validationResults}
     - **Action:** ServiceNow Flow Designer initiates maker-checker workflow
     - **Assignment:** Case automatically assigned based on load balancing + document complexity
     - **SLA Start:** ServiceNow starts SLA clock (4-hour target for maker review)
     - **Notification:** ServiceNow sends email/SMS to assigned maker with case details link
→ Output: Review case created in maker queue with SLA tracking, maker notified

[ServiceNow - Maker Review Interface (UI Builder)]
→ Input: Maker accesses case from queue
→ Actions:
   • UI displays side-by-side comparison of Sets B/C/D with confidence scores
   • Highlights fields with discrepancies (color-coded by confidence level)
   • Maker actions:
     - Accept Set D → Submits for checker approval (ServiceNow state transition)
     - Correct Set D → Updates values, submits for approval (ServiceNow update API)
     - Reject → Routes to rejection queue with reason (ServiceNow REST API call)
   • ServiceNow logs maker decision with timestamp (audit trail)
→ Output: Case moved to checker queue or rejection queue

[ServiceNow - Checker Approval Interface]
→ Input: Checker accesses case from queue
→ Actions:
   • UI displays Set D with maker notes and changes
   • Checker actions:
     - Approve → Calls Client Onboarding Service REST API (POST /api/clients/create) to create profile
     - Send back to Maker → ServiceNow workflow routes back with comments
     - Reject → Routes to rejection queue (ServiceNow state update)
   • ServiceNow tracks SLA compliance (target: 4 hours for approval)
   • Auto-escalation after timeout (2 hours → senior manager, 4 hours → VP)
→ Output: Approved cases create client profiles, rejected cases notify relationship managers

[ServiceNow - Relationship Manager Queue]
→ Input: Rejected cases routed to RM queue from Business Rule Validation Engine (via Azure Service Bus rejection-queue → ServiceNow integration)
→ Actions:
   • RM reviews rejection reasons, Sets B/C/D comparison, and maker-checker notes via ServiceNow UI
   • RM actions:
     - Make corrections and resubmit → Calls Document Processing Service API (PUT /api/submissions/{id}/resubmit with corrected Set D)
     - Notify client for document resubmission → Calls Notification Service API (POST /api/notifications/send with rejection reasons)
   • ServiceNow tracks remediation SLA (target: 24 hours) with automated reminders
→ Output: Resubmitted documents enter Stage 2, or client notified for new submission

[Client Onboarding Service - .NET Core 8 Microservice]
→ Input: REST API call from ServiceNow (POST /api/clients/create with approved Set D and maker-checker metadata) after checker approval
→ Actions:
   • Validates ServiceNow authentication token (OAuth 2.0 bearer token validation)
   • Retrieves approved Set D data from request payload (includes maker-checker audit trail)
   • Creates client profile in Azure SQL using EF Core with approved data (INSERT transaction)
   • Logs correction history to Azure Cosmos DB (event: MakerCheckerApproval with before/after values)
   • Feeds corrections back to AI model training dataset via Azure Blob Storage (continuous learning pipeline)
   • Publishes success message to Azure Service Bus (notification-queue) with JSON: {clientId, status, timestamp}
→ Output: Client profile created with audit trail

---

Stage 6: Continuous Learning & Model Improvement
#=> Functional: Retrieve corrections weekly → Analyze OCR errors → Retrain AI models → Adjust thresholds → Deploy improvements → Monitor metrics

[Document Processing Service - Python ML Pipeline]
**Hosting:** Azure Functions with Python runtime (timer-triggered, runs weekly)
→ Input: Weekly batch job (Azure Function timer trigger, cron: 0 0 * * SUN) retrieves corrections from Azure SQL (MakerCheckerCorrections table populated by ServiceNow maker-checker workflow)
→ Actions:
   • Queries correction history from Azure SQL: SELECT OriginalSetC, CorrectedSetD, FieldName, DocumentType FROM MakerCheckerCorrections WHERE CorrectionDate >= @lastWeek
   • Analyzes patterns in OCR extraction errors:
     - Calculates per-field accuracy rates (CompanyName: 92%, TaxID: 98%, BankAccount: 89%)
     - Identifies document types with low accuracy (<90%): Non-2D handwritten forms, degraded scans
     - Detects systematic errors: Common OCR misreads (0→O, 1→I, 5→S)
   • Compares original extractions (Set C) vs corrected data (Set D) from manual reviews
   • Builds training dataset:
     - Exports corrected documents to Azure Blob Storage (training-data container)
     - Formats as JSON with ground truth labels: {documentUrl, fields: [{name, value, boundingBox}]}
     - Minimum 500 samples per document type for retraining
   • Retrains Azure AI Document Intelligence custom models:
     - POST to Azure AI API: /documentModels:build with training dataset URL
     - Model training takes 15-30 minutes, validates on 20% holdout set
     - Compares new model accuracy vs current production model (must improve by >2%)
   • Adjusts confidence thresholds per document type:
     - 2D forms (structured): High >95%, Medium 85-95%, Low <85%
     - Non-2D forms (unstructured): High >90%, Medium 80-90%, Low <80%
   • Updates fuzzy matching tolerance based on historical accuracy:
     - Levenshtein distance threshold adjusted from 3→2 chars if false positives >5%
   • Deploys improved model: Updates Azure AI model endpoint reference in appsettings.json
→ Output: Improved AI models deployed to Azure AI Document Intelligence, accuracy metrics logged to Azure Monitor

[Application Insights & Azure Monitor]
→ Continuous monitoring:
   • TelemetryClient (C# SDK) tracks custom metrics via TrackMetric() calls:
     - BarcodeDecodeAccuracy (target >98%), OCRConfidenceScores (target >95%), ValidationPassRate (target >80%)
     - StraightThroughProcessingRate (auto-approval without manual review, target >80%)
   • Dependency tracking monitors:
     - Azure AI API response times (P95 <2s), SQL query performance (P95 <100ms)
     - ServiceNow API latency, Azure Blob Storage upload speeds
   • Alerting rules configured in Azure Monitor:
     - Severity 1 (Critical): Processing time >5 min, OCR accuracy <90%, auto-approval rate <70%
     - Severity 2 (Warning): SLA breaches >10%, manual review queue >50 items
   • Action groups send email/SMS to operations team for critical alerts (PagerDuty integration)
   • Power BI dashboards visualize:
     - Daily processing volume, auto-approval trends, field-level accuracy heatmaps
     - SLA compliance tracking, manual review queue backlog, maker-checker turnaround time
→ Output: Real-time observability dashboards, MTTR reduced from 4 hours to 30 minutes

```

##### FLOW 2: Payment Processing & Routing

````csharp
Stage 1: Payment Request Initiation & Document Upload
#=> Functional: User login → Enter payment details → Upload documents → Validate completeness → Store in Blob → Queue payment

[Payment Web Portal - React TypeScript Frontend + ASP.NET Core 8 Backend]
**Hosting:** Frontend on Azure Static Web Apps, Backend API on Azure App Service (Premium P1V2), All microservices on Azure Kubernetes Service (AKS) with auto-scaling

**Authentication & Authorization Flow:**
• **User Authentication:** Azure AD B2C for external users, Azure AD for internal employees
• **User Authorization:** RBAC via Azure AD groups/roles (PaymentInitiator, PaymentApprover)
• **Frontend → Backend API:** React calls Backend API with user JWT token (from Azure AD authentication)
• **Backend API → Microservices:** ASP.NET Core (App Service) acts as API Gateway, routes to AKS microservices with service JWT
• **Microservices ↔ Microservices:** Call each other via REST APIs with JWT bearer tokens for authentication
• **Microservices → Azure Resources:** Managed Identity for AKS pods accessing Blob Storage, SQL Database, Service Bus, Key Vault
→ Input: Corporate user login request
→ Actions:
   • OAuth 2.0 authentication with MFA (Azure AD integration)
   • RBAC validates user permissions for payment initiation (role: PaymentInitiator, PaymentApprover)
   • React UI renders payment form with dynamic fields based on payment type selection
→ Output: Authenticated session token (30-min TTL stored in Redis)

[Payment Web Portal - Payment Form Interface]
→ Input: User enters payment details
→ Actions:
   • User fills payment form via React UI:
     - Beneficiary information (name, account number, bank details, address)
     - Amount and currency (real-time FX rate display from external service)
     - Payment type selection (Wire/ACH/SWIFT/RTP)
     - Payment purpose/description
     - Urgency (Standard/Express/Same-Day)
   • Frontend validates inputs:
     - Amount range validation (min: $1, max: $10M per transaction)
     - Beneficiary account format validation (IBAN/SWIFT/Routing number regex)
     - Required fields validation per payment type
   • Backend validates user authorization vs payment amount:
     - Authorization rules fetched from ServiceNow (GET /api/now/table/authorization_rules) and cached in Redis (30-min TTL)
     - <$50K: Auto-approved for PaymentInitiator role
     - $50K-$500K: Requires manager approval
     - >$500K: Requires manager + VP approval
   • Fetches required document rules from Redis cache:
     - Rules source: ServiceNow Business Rules table (centralized rule repository)
     - ServiceNow Scheduled Job syncs rules to Redis every 15 minutes
     - Rules determine: Required documents by payment type, mandatory fields, file size limits, allowed formats
     - Purpose: These are document validation rules (different from authorization rules)
→ Output: Payment draft saved to Azure SQL (PaymentDrafts table with status: Draft)

[Payment Web Portal - Document Upload Interface]
→ Input: User uploads supporting documents
→ Actions:
   **Centralized Rule Management (Flow 1 & Flow 2):**
   • All business rules stored in ServiceNow as single source of truth:
     - Document requirements by payment/onboarding type
     - Validation rules (amount thresholds, field formats, duplicate detection)
     - Authorization rules (approval tiers, user permissions)
     - Routing rules (payment network selection criteria)
     - Compliance rules (sanction screening, country restrictions)
   • ServiceNow Configuration Management Database (CMDB) manages rule versioning and audit trail
   • Redis Cache acts as performance layer (15-30 min TTL) for frequently accessed rules
   • Rule updates in ServiceNow automatically invalidate Redis cache via pub/sub pattern

   • React frontend displays required documents checklist based on payment type:
     - Wire transfer → Invoice + Contract (if >$100K)
     - SWIFT international → Invoice + Bill of Lading + Tax documents
     - ACH domestic → Invoice only
     - Trade finance → Full documentation (LC, Bills, Insurance)
   • Frontend client-side validation:
     - File size <10MB per document
     - Allowed formats: PDF, JPG, PNG
     - Document count validation (all required documents present)
   • Backend pre-upload validation:
     - ASP.NET Core validates file content type (prevents malicious uploads)
     - **Document completeness check BEFORE Azure upload:**
       * Compares selected documents vs required documents from business rules (cached from Redis)
       * Validates minimum document count per payment type
       * Validates document types match requirements (Invoice, Contract, PO, etc.)
       * If validation fails:
         - Returns error to frontend with missing document list
         - Payment status remains: Draft (not saved to PendingProcessing)
         - User can add/replace documents and resubmit
         - No Azure storage costs incurred for incomplete submissions

   • Backend processes upload (only after completeness validation passes):
     - Generates unique payment reference ID (format: PAY-{yyyy}-{sequential})
     - Uploads documents to Azure Blob Storage container (payment-docs-{yyyy}/{mm})
     - Document metadata saved to Azure SQL (PaymentDocuments table):
       {paymentId, documentType, blobUrl, fileName, fileSize, uploadTimestamp, uploadedBy}
   • Updates payment status: Draft → PendingProcessing
   • Publishes message to Azure Service Bus (payment-processing-queue) with JSON payload:
     {paymentId, paymentType, amount, currency, beneficiary, urgency, blobUrls[], documentTypes[], initiatedBy, timestamp}
→ Output: Upload confirmation with payment reference ID, message queued for processing

---

Stage 2: AI Document Processing & Data Extraction
#=> Functional: Consume payment message → Retrieve documents from Blob → OCR extraction per document type → Save extracted data → Queue for validation

[Azure Blob Storage]
→ Event: Payment documents uploaded, metadata available in payment-processing-queue message

[Azure Function - Payment Document Processor (.NET Core 8)]
**Hosting:** Azure Functions Consumption Plan (auto-scales, pay-per-execution)
→ Input: Consumes message from Azure Service Bus (payment-processing-queue) via ServiceBusTrigger binding
→ Actions:
   • Retrieves payment metadata from Azure SQL (SELECT * FROM PaymentDrafts WHERE PaymentId = @paymentId)
   • Fetches document list from Azure SQL (SELECT * FROM PaymentDocuments WHERE PaymentId = @paymentId)
   • Downloads documents from Azure Blob Storage to Azure Function memory (BlobClient.DownloadAsync()):
     - **Why download?** Azure AI Document Intelligence requires document bytes or publicly accessible URLs with SAS tokens
     - Function downloads to memory stream (not to disk), processes, then discards
     - Alternative: Generate time-limited SAS token and pass URL to Azure AI (used for documents >10MB)
     - Documents stay in Blob Storage permanently for audit/compliance
→ Output: Document bytes in memory, ready for AI processing

**Payment Processing Context:**
• Corporate remittance includes both domestic and international money transfers for business operations
• Supporting documents vary by use case:
  - **Vendor payments:** Invoice + Purchase Order (most common)
  - **Trade finance:** Invoice + Contract + Bill of Lading + Letter of Credit
  - **International transfers:** Invoice + Contract + Tax documents (W-9/VAT certificates)
  - **Domestic ACH:** Invoice only (simpler requirements)
• All documents validate payment legitimacy and comply with anti-money laundering (AML) regulations

[Document Processing Service - Invoice Extraction Pipeline]
→ Input: Invoice documents (PDF/image format) from Azure Blob Storage (payment-docs container), list provided by Azure Function Payment Document Processor
→ Actions:
   • Posts invoice to Azure AI Document Intelligence REST API:
     - Endpoint: POST /formrecognizer/documentModels/prebuilt-invoice:analyze
     - Uses custom-trained invoice model ID for financial documents
   • Azure AI extracts structured data:
     - Vendor/Supplier name and address (field: VendorName, VendorAddress)
     - Invoice number and date (field: InvoiceId, InvoiceDate)
     - Invoice line items with JSON array:
       [{description, quantity, unitPrice, lineTotal}]
     - Financial totals: Subtotal, TaxAmount (broken by tax type), TotalAmount
     - Payment terms (field: PaymentTerms - parsed as "Net 30", "Net 60", "Due Date")
     - Bank account details if present (field: VendorBankAccount, IBAN/SWIFT)
   • Receives OCR results with per-field confidence scores:
     - High confidence: >95% (structured PDFs, clear printed text)
     - Medium confidence: 85-95% (scanned documents, slight degradation)
     - Low confidence: <85% (poor quality scans, handwritten notes, complex layouts)
   • Saves extracted data to Azure SQL (InvoiceExtractions table):
     {paymentId, documentId, vendorName, invoiceNumber, invoiceDate, totalAmount, currency, confidence, extractedFields JSON, extractionTimestamp}
→ Output: Invoice data extracted and stored

[Document Processing Service - Contract Extraction Pipeline]
→ Input: Contract documents (PDF format) from Azure Blob Storage (payment-docs container), list provided by Azure Function Payment Document Processor
→ Actions:
   • Posts contract to Azure AI Document Intelligence REST API:
     - Endpoint: POST /formrecognizer/documentModels/prebuilt-document:analyze
     - Uses layout + key-value extraction for unstructured contracts
   • Azure AI extracts key fields:
     - Contract parties (buyer/seller names from header/signature sections)
     - Contract number and date (field: ContractId, ContractDate)
     - Contract amount and currency (field: ContractValue, Currency)
     - Payment milestones and terms (table extraction for milestone schedules)
     - Contract reference numbers (field: ReferenceNumber)
   • Confidence scoring per field
   • Saves to Azure SQL (ContractExtractions table):
     {paymentId, documentId, buyerName, sellerName, contractNumber, contractAmount, extractedFields JSON, confidence, extractionTimestamp}
→ Output: Contract data extracted and stored

[Document Processing Service - Purchase Order Extraction Pipeline]
→ Input: Purchase order documents (PDF/Excel format) from Azure Blob Storage (payment-docs container), list provided by Azure Function Payment Document Processor
→ Actions:
   • Posts PO to Azure AI Document Intelligence:
     - Endpoint: POST /formrecognizer/documentModels/prebuilt-document:analyze
   • Extracts fields:
     - PO number and date (field: PONumber, PODate)
     - Item descriptions and quantities (table extraction)
     - Unit prices and totals (field: LineItems array, TotalAmount)
     - Vendor information (field: VendorName, VendorCode)
   • Saves to Azure SQL (POExtractions table)
→ Output: PO data extracted and stored

[Document Processing Service - Aggregation & Routing]
→ Input: Extraction results from Invoice, Contract, and PO extraction pipelines (from Azure SQL)
→ Actions:
   • Aggregates extraction results from all document types
   • Calculates overall document set confidence:
     **Confidence Basis:**
     - **Field-level confidence:** Azure AI assigns 0-1 confidence score per extracted field based on:
       * OCR text clarity (font quality, contrast, resolution)
       * Field location consistency (expected position in document template)
       * Semantic understanding (field value matches expected pattern/format)
       * Model training accuracy on similar documents
     - **Document-level confidence:** Average of all required field confidences
       * Required fields defined in business rules (VendorName, InvoiceNumber, TotalAmount, etc.)
       * Missing required field = 0% confidence for that field
     - **Overall confidence:** Weighted average across all documents in payment
       * Invoice (weight: 50%), Contract (weight: 30%), PO (weight: 20%)

     **Confidence Thresholds:**
     - Overall High: All documents >95% confidence AND all required fields extracted
     - Overall Medium: Any document 85-95% confidence OR some optional fields missing
     - Overall Low: Any document <85% confidence OR critical required fields missing
   • Updates payment status: PendingProcessing → Extracted
   • Publishes completion message to Azure Service Bus (payment-validation-queue) with JSON:
     {paymentId, overallConfidence, extractedData: {invoice, contract, po}, extractionTimestamp}
→ Output: Extraction complete, message queued for validation

---

Stage 3: Business Rule Validation & Data Reconciliation
#=> Functional: Consume validation message → Compare payment vs document data → Fetch business rules → Execute validation rules → Check compliance → Route to approval/review/rejection

[Payment Validation Service - .NET Core 8 Microservice]
**Hosting:** Azure Kubernetes Service (AKS) with Horizontal Pod Autoscaler (HPA) based on CPU/memory
**Service Bus Consumption:** .NET SDK with IMessageReceiver interface
  - Event-driven processing: ServiceBusTrigger attribute binds to queue
  - Message handling pattern: CompleteMessageAsync() for success, AbandonMessageAsync() for transient failures
  - Dead-letter queue: Messages failing after 10 delivery attempts moved to payment-validation-queue-deadletter
  - Consumer group: payment-validation-consumer with max concurrent calls: 32
→ Input: Consumes message from Azure Service Bus (payment-validation-queue) published by Document Processing Service
→ Actions:
   • Retrieves payment draft from Azure SQL (SELECT * FROM PaymentDrafts WHERE PaymentId = @paymentId)
   • Retrieves extracted document data from Azure SQL:
     - Invoice data (SELECT * FROM InvoiceExtractions WHERE PaymentId = @paymentId)
     - Contract data (SELECT * FROM ContractExtractions WHERE PaymentId = @paymentId)
     - PO data (SELECT * FROM POExtractions WHERE PaymentId = @paymentId)
→ Output: Payment and document data loaded, ready for comparison

[Payment Validation Service - Data Reconciliation Engine]
→ Input: Payment data from Azure SQL (PaymentDrafts table) and extracted document data from Azure SQL (InvoiceExtractions, ContractExtractions, POExtractions tables)
→ Actions:
   • Amount reconciliation:
     - Compare payment amount vs invoice total
     - Fuzzy match with ±2% tolerance for FX/rounding differences
     - Algorithm: |paymentAmount - invoiceTotal| / invoiceTotal <= 0.02
     - If mismatch >2%: Flag as AmountMismatch with variance percentage
   • Beneficiary name validation:
     - Compare beneficiary name (payment form) vs vendor name (invoice)
     - Levenshtein distance algorithm (threshold <5 characters)
     - Example: "Acme Corp" vs "ACME Corporation" → distance 12 → Mismatch
     - If mismatch: Flag as NameMismatch with similarity score
   • Reference number validation:
     - Compare payment purpose/description vs invoice/contract reference numbers
     - Keyword extraction and matching (regex patterns)
     - If no match: Flag as ReferenceMismatch
   • Currency consistency check:
     - Verify payment currency = invoice currency OR valid FX conversion exists
     - If multi-currency: Fetch FX rate from external service (GET /api/fx/rate?from={currency1}&to={currency2})
     - If currency mismatch without valid FX: Flag as CurrencyMismatch
   • Saves reconciliation results to Azure SQL (PaymentReconciliation table):
     {paymentId, amountMatch, beneficiaryMatch, referenceMatch, currencyMatch, reconciliationScore, mismatches JSON, timestamp}
→ Output: Reconciliation complete with match scores

[Business Rule Validation Engine - .NET Core 8 Microservice]
→ Input: Payment data from Azure SQL (PaymentDrafts table) with reconciliation results from Azure SQL (PaymentReconciliation table)
→ Actions:
   • Fetches business rules from ServiceNow REST API (GET /api/now/table/business_rules?type=payment&active=true)
   • Rules cached in Redis (15-min TTL) with key pattern: payment-rules:{paymentType}
   • Executes validation rules using .NET Rules Engine library:

     **Two-Tier Validation Approach:**
     - **Tier 1 (Frontend/Backend Basic Check):** Completed in Stage 1 Document Upload Interface
       * Quick client-side validations (file size, format, count)
       * Pre-upload completeness check (prevents incomplete submissions)
       * Purpose: Prevent unnecessary Azure storage costs and processing overhead

     - **Tier 2 (Comprehensive Microservice Validation):** Executed here in Stage 3
       * Post-extraction deep validation (validates extracted data against business rules)
       * Context-aware rules (amount thresholds, payment type-specific requirements)
       * Compliance checks (sanctions, AML, regulatory requirements)
       * Purpose: Ensure extracted document data meets business and regulatory standards

     1. Document Completeness Validation (Tier 2 - Post-Extraction):
        - Validates extracted fields from uploaded documents:
          * Invoice: VendorName, InvoiceNumber, InvoiceDate, TotalAmount extracted successfully?
          * Contract: ContractNumber, ContractAmount, Parties extracted successfully?
          * PO: PONumber, LineItems, TotalAmount extracted successfully?
        - Checks field-level confidence scores meet minimum thresholds (>85%)
        - Compares extracted data completeness vs business rule requirements
        - Example: Wire >$100K requires both Invoice AND Contract with all mandatory fields extracted
        - If incomplete extraction or low confidence: Flag as DocumentIncomplete with missing field list

     2. Country-Specific Regulatory Validation (Tier 2 - Post-Extraction):
        - Validates extracted tax documents and compliance fields:
        - US payments → Validates W-9 form extracted successfully with TIN (Tax Identification Number)
        - EU payments → Validates VAT certificate and SEPA compliance
        - High-risk countries → Validates enhanced due diligence documents
        - If non-compliant: Flag as RegulatoryNonCompliant with country code

     3. Data Mismatch Tolerance Validation:
        - Checks reconciliation results from previous step
        - Amount difference >2% → Flag for manual review
        - Beneficiary name mismatch → Flag for manual review
        - If mismatches within tolerance: Pass with warning

     4. Duplicate Payment Detection:
        - SQL query: SELECT COUNT(*) FROM CompletedPayments
          WHERE InvoiceNumber = @invoiceNumber AND PaymentDate >= DATEADD(day, -90, GETDATE())
        - Checks same beneficiary + amount + date combination (within 7 days)
        - If duplicate found: Flag as PotentialDuplicate with previous payment ID

     5. Payment Terms Validation:
        - Compares payment date vs invoice due date
        - Validates payment within due date tolerance (+/- 5 days)
        - Calculates early payment discount if applicable (e.g., 2/10 Net 30)
        - If overdue: Flag as OverduePayment with days overdue

     6. Amount Threshold Validation:
        - Validates approval requirements based on amount:
          * Domestic wire >$50K → Requires manager approval
          * International >$100K → Requires VP approval + compliance review
          * >$1M → Requires SVP approval + dual authorization
        - Sets approval workflow tier based on amount
→ Output: Validation rules executed, results aggregated

[Compliance Service - .NET Core 8 Microservice]
→ Input: Payment data from Azure SQL (PaymentDrafts table), invoked via REST API call from Business Rule Validation Engine
→ Actions:
   • Sanction list screening via external Compliance API:
     - POST /api/compliance/screen with JSON payload:
       {beneficiaryName, beneficiaryCountry, vendorName, paymentPurpose}
     - Screens against multiple sanction lists:
       * OFAC (US Office of Foreign Assets Control)
       * EU consolidated sanctions list
       * UN consolidated sanctions list
       * PEP (Politically Exposed Persons) lists
     - Receives screening results with risk score (0-100)
   • Restricted keywords screening:
     - Checks payment purpose/description for restricted industries:
       * Weapons, ammunition, defense
       * Gambling, adult entertainment (in certain jurisdictions)
       * Cryptocurrency (in restricted countries)
     - Regex pattern matching against prohibited keyword database
   • Destination country restriction validation:
     - Checks beneficiary country against embargoed jurisdictions list
     - Validates source of funds for high-risk corridors (e.g., Middle East → Europe)
   • Saves compliance screening results to Azure SQL (ComplianceScreening table):
     {paymentId, sanctionHit, riskScore, restrictedKeywords[], countryRestrictions, screeningTimestamp}
→ Output: Compliance screening complete

[Payment Service - Beneficiary Validation Module]
→ Input: Beneficiary details from payment data (Azure SQL PaymentDrafts table), invoked via REST API call from Business Rule Validation Engine
→ Actions:
   • Checks beneficiary status in vendor master:
     - SQL query: SELECT Status, BlockedReason FROM VendorMaster WHERE VendorId = @vendorId
     - Validates status = Active (not Blocked, not Suspended)
   • Bank account format validation:
     - IBAN validation (regex + checksum algorithm for EU accounts)
     - SWIFT code validation (8 or 11 characters, valid BIC format)
     - US routing number validation (9 digits, ABA checksum)
   • Cross-reference with previous successful payments:
     - SQL query: SELECT COUNT(*) FROM CompletedPayments
       WHERE BeneficiaryAccount = @account AND Status = 'Completed' AND PaymentDate >= DATEADD(month, -6, GETDATE())
     - If new beneficiary: Flag as NewBeneficiary for additional review
→ Output: Beneficiary validated

[Payment Service - Payer Account Validation Module]
→ Input: Payer account details from payment data (Azure SQL PaymentDrafts table), invoked via REST API call from Business Rule Validation Engine
→ Actions:
   • Validates available balance via Core Banking System REST API:
     - GET /api/accounts/{accountId}/balance
     - Checks available balance >= payment amount + fees
   • Credit limit validation:
     - GET /api/accounts/{accountId}/creditLimit
     - Validates current utilization + payment amount <= credit limit
   • Account status verification:
     - SQL query: SELECT Status, FrozenReason FROM Accounts WHERE AccountId = @accountId
     - Validates status = Active (not Frozen, not Closed)
   • If insufficient funds or frozen: Flag as PayerAccountIssue
→ Output: Payer account validated

[Payment Service - Fee Calculation Module]
→ Input: Payment details from Azure SQL (PaymentDrafts table: amount, currency, destination, payment type), invoked via REST API call from Business Rule Validation Engine
→ Actions:
   • Calculates wire transfer fees:
     - Domestic wire: Flat fee $25
     - International wire: $45 + correspondent bank charges
     - Fee matrix cached in Redis from fee schedule table
   • FX conversion calculation (if multi-currency):
     - Fetches real-time FX rate from external service (GET /api/fx/rate?from={from}&to={to})
     - Applies bank FX margin (e.g., 1.5% over mid-market rate)
     - Calculates total amount in destination currency
   • Correspondent bank charges estimation:
     - Queries fee schedule by destination country and bank
     - Adds estimated correspondent fees ($10-$30 for international)
   • Total cost calculation: Payment amount + fees + FX margin
   • Saves fee breakdown to Azure SQL (PaymentFees table)
→ Output: Fees calculated and added to payment

[Business Rule Validation Engine - Decision Routing Logic]
**Microservice Communication Patterns:**
  - **Synchronous (Request-Response):** REST over HTTP/HTTPS for direct service-to-service calls
    * Example: Payment Service → Compliance Service (POST /api/compliance/screen)
    * Example: Payment Service → Core Banking System (GET /api/accounts/{accountId}/balance)
    * Authentication: JWT bearer tokens for API Gateway, Managed Identity for internal AKS services
  - **Asynchronous (Event-Driven):** Azure Service Bus for decoupled processing
    * Example: Document Processing Service publishes to payment-validation-queue
    * Example: Payment Execution Service publishes to payment-confirmation-queue
    * Pattern: Producer publishes event, consumer processes independently
  - **Real-Time (Client Updates):** SignalR WebSocket connections for frontend notifications
    * Example: Payment status updates pushed to React UI
    * Example: Real-time approval workflow status
    * Connection Hub: Notification Service maintains persistent WebSocket connections per user session

→ Input: Aggregated validation results, reconciliation scores, compliance screening, account validation
→ Actions:
   • Calculates overall risk score:
     - Base score from document confidence (High: 100, Medium: 85, Low: 70)
     - Deductions: Amount mismatch -10, Name mismatch -15, Sanction hit -50
     - Compliance risk score weighted (0-100 scale)
   • Decision routing rules:
     - High Confidence (>95%) + All Rules Pass + Amount Match (<2% diff) + No Compliance Issues:
       → Publishes to Azure Service Bus (payment-approval-queue) with tier: AutoApprove
     - Medium Confidence (85-95%) OR Minor Discrepancies (2-5% diff) OR New Beneficiary:
       → Publishes to Azure Service Bus (payment-review-queue) with tier: ManualReview
     - Low Confidence (<85%) OR Rules Fail OR Sanctions Hit OR Major Discrepancies (>5%):
       → Publishes to Azure Service Bus (payment-rejection-queue) with rejection reasons
   • Updates payment status in Azure SQL: Extracted → PendingApproval / UnderReview / Rejected
   • Logs decision to Azure Cosmos DB (event sourcing: PaymentValidated event with full validation results)
   • For manual review cases: Calls ServiceNow REST API to initiate workflow:
     - **Endpoint:** POST /api/now/workflow/review
     - **Purpose:** Creates review case in ServiceNow Payment Review queue
     - **Payload:** {paymentId, validationResults, riskScore, assignmentGroup: "PaymentReviewers"}
     - **Action:** ServiceNow Flow Designer initiates maker-checker workflow
     - **Assignment:** Case automatically assigned based on load balancing + payment amount tier
     - **SLA Start:** ServiceNow starts SLA clock (2-hour response time for high-priority)
     - **Notification:** ServiceNow sends email/SMS to assigned reviewer with payment details link
→ Output: Payment routed to appropriate queue, ServiceNow review case created with SLA tracking

[Notification Service - .NET Core 8 Microservice]
→ Input: Listens to all routing queues (payment-approval-queue, payment-review-queue, payment-rejection-queue)
→ Actions:
   • Sends email notifications via SendGrid API:
     - Auto-approval: Confirmation to initiator with payment reference
     - Manual review: Notification to operations team with validation issues
     - Rejection: Detailed rejection reasons to initiator with correction instructions
   • SignalR broadcasts real-time status updates to Payment Dashboard (React TypeScript)
   • Logs notification delivery status to Azure SQL (Notifications table)
→ Output: Multi-channel notifications delivered

---

Stage 4: Routing Decision Engine
#=> Functional: Consume approved payment → Analyze payment attributes → Apply routing rules → Select payment network → Generate routing metadata → Queue for execution

[Payment Routing Service - .NET Core 8 Microservice]
→ Input: Consumes message from Azure Service Bus (payment-approval-queue for auto-approved payments)
→ Actions:
   • Retrieves payment details from Azure SQL (SELECT * FROM PaymentDrafts WHERE PaymentId = @paymentId AND Status = 'PendingApproval')
   • Loads payment attributes for routing analysis:
     {amount, currency, beneficiaryCountry, paymentType, urgency, initiatedDate}
→ Output: Payment data loaded, ready for routing analysis

[Routing Engine - Payment Attribute Analyzer]
→ Input: Payment attributes from Azure SQL (PaymentDrafts table: amount, currency, beneficiaryCountry, paymentType, urgency, initiatedDate)
→ Actions:
   • Analyzes transaction amount with threshold classification:
     - Small: <$10K (optimize for lowest cost)
     - Medium: $10K-$100K (balance cost and speed)
     - Large: $100K-$1M (optimize for reliability)
     - High-value: >$1M (requires pre-notification, dual authorization)
   • Destination country analysis:
     - Domestic US: ACH, Wire, RTP options available
     - International: SWIFT, Wire options available
     - EU SEPA region: SEPA Credit Transfer preferred
     - High-risk countries: Wire only with enhanced monitoring
   • Currency analysis:
     - USD domestic: ACH preferred for cost efficiency
     - Multi-currency: SWIFT for FX conversion support
     - EUR within SEPA: SEPA Credit Transfer
   • Urgency classification:
     - Standard (2-3 days): Optimize for cost
     - Express (same-day): Prioritize speed
     - Same-Day (within hours): RTP or Wire only
   • Regulatory requirements check:
     - Sanctions screening results (from Stage 3)
     - Country restrictions (embargo list)
     - Special monitoring requirements (high-risk corridors)
   • Network availability check:
     - Queries network status from Redis cache (updated every 5 minutes)
     - Checks for network outages or maintenance windows
     - If primary network unavailable: Selects backup network
→ Output: Payment attributes analyzed and classified

[Routing Engine - Rules Engine (.NET Core 8 with Strategy Pattern)]
→ Input: Classified payment attributes from Routing Engine Payment Attribute Analyzer
→ Actions:
   • Fetches routing rules from ServiceNow REST API (GET /api/now/table/routing_rules?active=true&orderBy=priority)
   • Rules cached in Redis (15-min TTL) with key: routing-rules:v{version}
   • Executes 50+ routing rules using first-match-wins strategy:

     **Domestic US Rules (Priority 1-20):**
     1. Domestic <$10K + Standard → ACH (cost: $0.25, settlement: 1-2 days)
     2. Domestic <$10K + Same-Day → RTP (cost: $0.50, settlement: <1 hour)
     3. Domestic $10K-$100K + Standard → ACH batch (cost: $1.50, settlement: 1-2 days)
     4. Domestic $10K-$100K + Express → Wire (cost: $25, settlement: same-day)
     5. Domestic >$100K + Any → Wire (cost: $25, settlement: same-day, tracking enabled)
     6. Domestic >$1M → Wire with pre-notification (cost: $35, requires dual authorization)

     **International Rules (Priority 21-40):**
     7. International + Standard + Non-urgent → SWIFT MT103 (cost: $45, settlement: 2-3 days)
     8. International + Express → Wire Transfer (cost: $55, settlement: same-day)
     9. International >$1M → Wire with pre-notification + SWIFT confirmation (cost: $75)
     10. EU SEPA region + EUR currency → SEPA Credit Transfer (cost: €0.50, settlement: 1 day)
     11. High-risk country → Wire only + enhanced monitoring (cost: $65, manual approval required)

     **Special Rules (Priority 41-50):**
     12. Trade finance → SWIFT with documentary collection (cost: $95, requires LC documents)
     13. Payroll batch → ACH bulk with same effective date (cost: $0.10 per transaction)
     14. Tax payments → Wire to IRS with special formatting (cost: $15)
     15. Vendor payment with discount terms → RTP if discount date within 48 hours

   • Rule evaluation using C# expression trees:
     - Condition: (amount < 10000 && isdomestic && urgency == "Standard")
     - Action: SelectNetwork(NetworkType.ACH)
     - Cost calculation: baseWireFee + (amount * feePercentage)
   • First matching rule selected, remaining rules skipped
   • If no rule matches: Default to manual routing queue
→ Output: Target payment network selected

[Routing Engine - Cost Optimization Module]
→ Input: Selected network and alternative networks from Routing Engine Rules Engine
→ Actions:
   • Calculates total cost for selected network:
     - Base network fee
     - Correspondent bank charges (if international)
     - FX conversion fees (if multi-currency)
   • Evaluates alternative networks (if urgency allows):
     - Compares ACH vs Wire for domestic >$10K
     - Compares SWIFT vs Wire for international
   • Cost-benefit analysis:
     - If cost savings >20% and urgency = Standard: Recommends cheaper network
     - If time-critical: Prioritizes speed over cost
   • Saves cost analysis to Azure SQL (RoutingCostAnalysis table)
→ Output: Optimal network selected with cost breakdown

[Routing Engine - Routing Metadata Generator]
→ Input: Selected network (ACH/Wire/SWIFT/RTP/SEPA) from Routing Engine Cost Optimization Module
→ Actions:
   • Generates network-specific routing metadata:

     **For ACH:**
     - Routing number (9 digits)
     - Account number
     - Transaction type code (22 for checking credit)
     - Company ID and batch ID
     - Effective date (next business day)

     **For Wire:**
     - Beneficiary bank SWIFT/BIC code
     - Beneficiary account number (IBAN or local format)
     - Intermediary bank details (if required)
     - Payment reference (OBI field, 140 chars max)
     - Value date (same-day or next-day)

     **For SWIFT:**
     - Message type (MT103 for customer credit transfer)
     - Sender reference (20 chars unique ID)
     - Ordering customer details (SWIFT address)
     - Beneficiary customer details (SWIFT address)
     - Details of charges (OUR/SHA/BEN)
     - Regulatory reporting fields (if required)

     **For RTP:**
     - ISO 20022 XML format (pain.001)
     - End-to-end ID (unique payment reference)
     - Purpose code (SUPP for supplier payment)
     - Remittance information (structured or unstructured)

     **For SEPA:**
     - SEPA Credit Transfer scheme
     - Debtor IBAN and BIC
     - Creditor IBAN and BIC
     - Purpose code (SEPA purpose codes)
     - End-to-end reference

   • Validates routing metadata completeness:
     - All mandatory fields present
     - Field length and format validation
     - IBAN/SWIFT checksum validation
   • Saves routing decision to Azure SQL (PaymentRouting table):
     {paymentId, selectedNetwork, routingMetadata JSON, costEstimate, routingReason, timestamp}
   • Updates payment status: PendingApproval → RoutedForExecution
   • Publishes message to Azure Service Bus (payment-execution-queue) with JSON payload:
     {paymentId, network, routingMetadata, priority, executionDate}
→ Output: Payment routed with complete metadata, queued for execution

---

Stage 5: Approval Workflow (if required)
#=> Functional: Route based on amount/risk → Send approval notifications → Track SLA → Handle timeouts → Execute approved payments

[Payment Routing Service - Approval Router]
→ Input: Routed payment from Stage 4 (if requires approval)
→ Actions:
   • Determines approval tier based on criteria:
     - <$50K + High Confidence + All Validations Pass → AutoApproved (skip approval workflow)
     - $50K-$500K → Tier 1: Manager approval required
     - $500K-$2M → Tier 2: Manager + VP approval required (sequential)
     - >$2M → Tier 3: Manager + VP + CFO approval required (sequential)
     - Sanctions hit or compliance flag → Compliance officer approval (high priority, parallel to normal flow)
     - Medium confidence or data mismatch >2% → Operations review queue
   • If AutoApproved: Publishes directly to payment-execution-queue
   • If approval required: Calls ServiceNow REST API to initiate workflow
→ Output: Approval workflow initiated or payment auto-approved

[ServiceNow Platform - Workflow Orchestration]
→ Input: REST API call from Payment Routing Service (POST /api/now/workflow/payment-approval) with JSON payload:
  {paymentId, amount, currency, beneficiary, paymentType, urgency, approvalTier, validationResults, initiatedBy}
→ Actions:
   • ServiceNow Flow Designer workflow triggered
   • Creates approval case in ServiceNow with payment details:
     - Case number generated (format: APR-{yyyy}-{sequential})
     - Priority set based on urgency (1: Same-Day, 2: Express, 3: Standard)
     - SLA timer started (target: 4 hours for Tier 1, 8 hours for Tier 2/3)
   • Routes to appropriate approval queue based on tier:
     - Tier 1: Manager queue (filtered by business unit and region)
     - Tier 2: Manager queue → VP queue (sequential workflow)
     - Tier 3: Manager queue → VP queue → CFO queue (sequential)
     - Compliance flag: Parallel route to Compliance Officer queue
   • ServiceNow JavaScript business rules determine assignee:
     - Workload distribution algorithm (assigns to approver with lowest pending count)
     - Region-based routing (US payments → US managers)
     - Backup assignment if primary approver out-of-office
→ Output: Approval case created and routed

[ServiceNow - Approval Notification Engine]
→ Input: Approval case created
→ Actions:
   • Sends multi-channel notifications to assigned approver:

     **Email notification:**
     - Sends via ServiceNow email engine (SMTP integration)
     - Email template includes:
       * Payment reference and case number
       * Amount, currency, beneficiary details
       * Payment purpose and urgency
       * Validation summary (confidence score, compliance status)
       * Link to ServiceNow approval interface
       * Deep link to mobile app (if available)
       * SLA deadline (e.g., "Approval required by 3:00 PM EST")

     **Mobile push notification:**
     - Sent via ServiceNow Mobile App push service
     - Title: "Payment Approval Required"
     - Body: "${amount} ${currency} to ${beneficiary} - Urgent: ${urgency}"
     - Deep link to approval screen in mobile app

     **In-app notification:**
     - ServiceNow portal notification bell icon
     - Real-time notification via SignalR connection
     - Priority badge for urgent/high-value payments

   • Notification delivery tracking:
     - Logs notification sent time and delivery status
     - Tracks email open rate and click-through
     - Re-sends notification if not opened within 30 minutes (for urgent payments)
→ Output: Approver notified via multiple channels

[ServiceNow - Approver Interface (UI Builder)]
→ Input: Approver accesses approval case from queue or notification link
→ Actions:
   • UI displays comprehensive payment details:
     - Payment summary card (amount, beneficiary, purpose)
     - Document preview (uploaded invoices, contracts with thumbnail gallery)
     - AI extraction results with confidence scores (color-coded: green >95%, yellow 85-95%, red <85%)
     - Validation results:
       * Data reconciliation (amount match, name match, reference match)
       * Compliance screening results (sanction hit, risk score)
       * Business rule validation (completeness, regulatory compliance)
     - Cost breakdown (fees, FX rate, total cost)
     - Routing decision (selected network, reasoning)
     - Audit trail (who initiated, when, validation history)

   • Approver actions:

     **Approve:**
     - Click "Approve" button
     - Optional: Add approval comments (e.g., "Verified with vendor via phone")
     - ServiceNow state transition: Pending → Approved
     - Calls Payment Service REST API (POST /api/payments/{paymentId}/approve) with approver details
     - If sequential approval (Tier 2/3): Routes to next approver in chain
     - If final approval: Payment proceeds to execution (Stage 6)

     **Reject:**
     - Click "Reject" button
     - Mandatory: Enter rejection reason (dropdown + free text)
     - Rejection reasons: Incorrect Amount, Wrong Beneficiary, Missing Documents, Suspected Fraud, Policy Violation
     - ServiceNow state transition: Pending → Rejected
     - Calls Payment Service REST API (POST /api/payments/{paymentId}/reject) with rejection details
     - Notification sent to payment initiator with rejection reason and correction instructions

     **Request More Information:**
     - Click "Request Info" button
     - Enter specific questions or required documents
     - ServiceNow state transition: Pending → InformationRequested
     - Notification sent to payment initiator
     - SLA timer paused until information provided
     - Once info received: Case returns to approver queue

     **Delegate:**
     - Click "Delegate" button (if approver is unavailable)
     - Select backup approver from dropdown
     - ServiceNow reassigns case with delegation note

   • ServiceNow logs approver decision:
     - Approver name, employee ID, timestamp
     - Decision (Approve/Reject/Request Info/Delegate)
     - Comments and reasoning
     - IP address and device info (for security audit)
     - Adds to approval chain audit trail
→ Output: Approval decision recorded, payment proceeds or rejected

[ServiceNow - SLA Tracking & Timeout Handling]
→ Input: Approval case with SLA timer running
→ Actions:
   • Continuous SLA monitoring:
     - ServiceNow scheduled job runs every 15 minutes
     - Checks all pending approvals with active SLA timers
     - Calculates time remaining: slaDeadline - currentTime

   • Escalation logic (tiered approach):

     **2-hour mark (50% of SLA for urgent payments):**
     - Sends reminder notification to assigned approver
     - Email: "REMINDER: Payment approval due in 2 hours"
     - Mobile push: High-priority notification

     **4-hour mark (SLA warning threshold):**
     - Auto-escalation to senior manager
     - ServiceNow workflow:
       * Creates escalation case linked to original approval
       * Assigns to senior manager (determined by org hierarchy API)
       * Sends urgent notification: "Escalated: Payment approval overdue"
     - Original approver still can approve (parallel approval paths)

     **6-hour mark (SLA breach for Tier 1):**
     - Auto-escalation to VP level
     - ServiceNow incident created (severity: Medium)
     - Operations team notified of SLA breach
     - VP can override and approve or reassign

     **8-hour mark (SLA breach for Tier 2/3):**
     - Auto-escalation to SVP/CFO level
     - ServiceNow incident escalated (severity: High)
     - Email notification to business unit head
     - Critical alert sent to payment operations management

   • SLA breach tracking:
     - Updates case with SLA breach flag
     - Records actual resolution time vs target
     - Adds to approver performance metrics
     - Quarterly SLA reports generated for management review

   • Timeout handling for abandoned approvals:
     - After 24 hours with no response: Payment auto-rejected
     - Notification sent to initiator: "Payment approval timed out - please resubmit"
     - Updates payment status: PendingApproval → TimedOut
→ Output: SLA tracked, escalations handled, timeout processed

[ServiceNow - Compliance Officer Review (Parallel Path)]
→ Input: Payments flagged for compliance review from Business Rule Validation Engine (sanctions hit, high-risk country), received via POST /api/now/workflow/compliance-review
→ Actions:
   • Creates compliance review case (high priority)
   • Compliance officer interface shows:
     - Sanction screening details (matched entity, list source, risk score)
     - Enhanced due diligence checklist
     - Source of funds documentation review
     - Beneficial ownership verification (UBO check)
   • Compliance actions:
     - Approve: Payment proceeds (compliance clearance recorded)
     - Reject: Payment blocked (SAR filing may be triggered)
     - Request Enhanced Due Diligence: Additional documentation required
   • Compliance decision logged for regulatory reporting
→ Output: Compliance clearance or rejection

[Payment Service - Approval Status Handler]
→ Input: Webhook callback from ServiceNow on approval decision (POST /api/webhooks/servicenow/approval)
→ Actions:
   • Validates ServiceNow webhook signature (OAuth 2.0 bearer token validation)
   • Parses approval decision from webhook payload:
     {caseNumber, paymentId, decision, approver, approvalTimestamp, comments, approvalChain[]}
   • Updates payment status in Azure SQL:
     - If Approved: PendingApproval → ApprovedForExecution
     - If Rejected: PendingApproval → Rejected
     - If InformationRequested: PendingApproval → OnHold
   • Saves approval chain to Azure SQL (PaymentApprovals table):
     {paymentId, approverName, approverRole, decision, timestamp, comments, approvalTier}
   • Logs approval event to Azure Cosmos DB (event sourcing: PaymentApproved event with full approval chain)

   • If final approval (all required approvers approved):
     - Publishes message to Azure Service Bus (payment-execution-queue) with JSON payload:
       {paymentId, approvalChain[], executionPriority, scheduledExecutionDate}
     - Sends confirmation notification to payment initiator:
       "Payment ${paymentId} approved - Execution scheduled for ${executionDate}"

   • If rejected:
     - Publishes message to Azure Service Bus (payment-rejection-queue)
     - Sends rejection notification to initiator with detailed reasons and next steps

   • If intermediate approval (sequential workflow):
     - Waits for next approver in chain
     - No action taken until all required approvals received
→ Output: Payment status updated, execution queued or rejection processed

---

Stage 6: Payment Execution
#=> Functional: Consume approved payment → Debit account → Transform format → Execute on network → Receive confirmation → Update status → Handle exceptions

[Payment Execution Service - .NET Core 8 Microservice]
→ Input: Consumes message from Azure Service Bus (payment-execution-queue)
→ Actions:
   • Retrieves payment details from Azure SQL:
     - SELECT * FROM PaymentDrafts WHERE PaymentId = @paymentId AND Status = 'ApprovedForExecution'
   • Retrieves routing metadata:
     - SELECT * FROM PaymentRouting WHERE PaymentId = @paymentId
   • Validates payment still valid (not expired, not already executed)
→ Output: Payment loaded, ready for execution

[Payment Execution Service - Account Debit Module]
→ Input: Approved payment from Azure SQL (PaymentDrafts table: Status = ApprovedForExecution) with payer account details
→ Actions:
   • Calls Core Banking System REST API to debit account:

     **Step 1: Reserve funds (pessimistic locking)**
     - POST /api/accounts/{accountId}/reserveFunds
     - JSON payload: {accountId, amount, currency, paymentReference, lockTimeout: 300}
     - Core Banking creates fund reservation:
       * Reduces available balance immediately
       * Creates pending transaction record
       * Sets lock with 5-minute timeout (prevents double-spending)
     - Receives reservation ID and locked balance confirmation

     **Step 2: Execute debit**
     - POST /api/accounts/{accountId}/debit
     - JSON payload:
       {reservationId, accountId, amount, currency, transactionType: 'Payment',
        beneficiary, paymentReference, executionDate, initiatedBy}
     - Core Banking processes debit:
       * Validates reservation exists and not expired
       * Debits account with transaction ID
       * Updates account balance: currentBalance - amount
       * Creates transaction history entry
       * Releases fund reservation lock
     - Receives transaction confirmation:
       {transactionId, debitedAmount, newBalance, transactionTimestamp}

   • Updates payment status in Azure SQL: ApprovedForExecution → Debited
   • Saves debit transaction details to Azure SQL (AccountTransactions table):
     {paymentId, accountId, transactionId, debitAmount, balance, timestamp}
   • Logs debit event to Azure Cosmos DB (event sourcing: AccountDebited event)

   • Error handling:
     - If insufficient funds: Throws InsufficientFundsException
     - If account frozen: Throws AccountFrozenException
     - If reservation timeout: Retries reservation with exponential backoff
→ Output: Account debited, funds locked for payment

[Payment Format Transformation Service - .NET Core 8]
→ Input: Payment data from Azure SQL (PaymentDrafts table) with routing metadata from Azure SQL (PaymentRouting table) and selected network, invoked by Payment Execution Service
→ Actions:
   • Transforms payment from internal JSON format to network-specific format:

   **For SWIFT (ISO 20022 XML - MT103/MT202):**
   - Creates XML document with ISO 20022 schema:
     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <Document xmlns="urn:iso:std:iso:20022:tech:xsd:pain.001.001.03">
       <CstmrCdtTrfInitn>
         <MsgId>{uniqueMessageId}</MsgId>
         <CreDtTm>{currentTimestamp}</CreDtTm>
         <NbOfTxs>1</NbOfTxs>
         <CtrlSum>{amount}</CtrlSum>
         <PmtInf>
           <PmtMtd>TRF</PmtMtd>
           <DbtrAcct><IBAN>{payerIBAN}</IBAN></DbtrAcct>
           <CdtrAcct><IBAN>{beneficiaryIBAN}</IBAN></CdtrAcct>
           <Amt Ccy="{currency}">{amount}</Amt>
         </PmtInf>
       </CstmrCdtTrfInitn>
     </Document>
     ```
   - Message type selection:
     * MT103: Customer credit transfer (most common)
     * MT202: Bank-to-bank transfer (for correspondent banking)
   - Populates mandatory SWIFT fields:
     * :20: Sender's reference (unique payment ID)
     * :32A: Value date and currency/amount
     * :50K: Ordering customer (payer details)
     * :59: Beneficiary customer (name, account, address)
     * :71A: Details of charges (OUR/SHA/BEN)
     * :72: Sender to receiver information (payment purpose)

   **For ACH (NACHA fixed-width format):**
   - Creates fixed-width file with record structure:
     * File Header Record (94 chars)
     * Batch Header Record (94 chars)
     * Entry Detail Record (94 chars) - payment details
     * Addenda Record (optional, 94 chars) - additional info
     * Batch Control Record (94 chars) - batch totals
     * File Control Record (94 chars) - file totals
   - Example Entry Detail Record:
     ```
     622012345678901234567890000001000000USD0123456789CompanyName        1234567890
     ^^^ ^^^^^^^^^ ^^^^^^^^^^^^^^ ^^^^^^^^^^^ ^^^ ^^^^^^^^^^                  ^^^^^^^^^
     |   |         |               |           |   |                           |
     |   |         |               |           |   Originator ID              Trace Number
     |   |         |               |           Currency
     |   |         |               Amount (with implied decimal)
     |   |         Beneficiary Account Number
     |   Routing Transit Number
     Transaction Code (22=checking credit)
     ```
   - Transaction code selection:
     * 22: Checking account credit
     * 32: Savings account credit
     * 27: Checking account debit
   - Batch processing: Groups multiple payments with same effective date

   **For Wire Transfer (Fedwire format):**
   - Creates proprietary XML/CSV format (bank-specific):
     {wireType: 'Domestic', amount, currency, beneficiaryName, beneficiaryAccount,
      beneficiaryBank: {name, routingNumber, address},
      intermediaryBank: {name, swiftCode} (if international),
      senderReference, valueDate, chargeOption: 'OUR'}
   - Fedwire tag structure:
     * {1510}: Beneficiary (name, account, bank routing)
     * {3600}: Amount (with currency code)
     * {4200}: Sender reference (payment purpose)

   **For RTP (Real-Time Payments - ISO 20022 XML pain.001):**
   - Creates ISO 20022 pain.001 XML:
     * End-to-end ID (unique payment reference)
     * Purpose code (SUPP for supplier payment, SALA for payroll)
     * Remittance information (structured or unstructured)
     * Request for execution: Immediate (within seconds)
   - Real-time validation fields:
     * Account status verification
     * Immediate funds availability confirmation

   **For SEPA Credit Transfer:**
   - Creates SEPA XML (ISO 20022 pain.001.001.03):
     * Debtor IBAN and BIC (payer bank)
     * Creditor IBAN and BIC (beneficiary bank)
     * Purpose code (SEPA-specific codes)
     * Charge bearer: SLEV (shared charges)
     * Requested execution date (D+1)

   • Validates transformed format:
     - XML schema validation (for SWIFT, RTP, SEPA)
     - Fixed-width field length validation (for ACH)
     - Mandatory field presence check
     - Character encoding validation (UTF-8 for XML, ASCII for ACH)
   • Saves transformed message to Azure Blob Storage (payment-messages container):
     {paymentId}-{network}-{timestamp}.{xml|txt}
→ Output: Payment transformed to network format, ready for transmission

[Payment Network Integration Service - .NET Core 8]
→ Input: Transformed payment message from Payment Format Transformation Service and network endpoint from Azure SQL (PaymentRouting table)
→ Actions:
   • Executes payment on target network with resilience patterns:

   **HttpClient with Polly Circuit Breaker:**
   - Circuit breaker configuration:
     * Open circuit after 5 consecutive failures
     * Half-open state after 60 seconds
     * Test with single request before closing circuit
   - Retry policy:
     * 3 retry attempts with exponential backoff
     * Delays: 2s, 10s, 30s
     * Retries only on transient errors (timeout, 5xx responses)

   **Network-specific API calls:**

   **SWIFT Network:**
   - POST https://swift.network.api/messages
   - Headers: {Authorization: Bearer {swiftToken}, Content-Type: application/xml}
   - Body: ISO 20022 XML message
   - Receives SWIFT acknowledgment:
     {ackId, status: 'ACCP' (accepted), creationDateTime, networkReference}

   **ACH Network:**
   - File upload to ACH processor SFTP server
   - Connection: sftp://ach.processor.com/{company-id}/outbound/
   - File naming: {companyId}-{date}-{batchId}.ach
   - Polls for acknowledgment file in /inbound/ directory
   - ACH response contains:
     {batchId, totalTransactions, acceptedCount, rejectedCount, settlementDate}

   **Wire Network:**
   - POST https://wire.network.api/v2/transfers
   - Headers: {X-API-Key: {apiKey}, Content-Type: application/json}
   - Body: Wire transfer JSON
   - Receives immediate response:
     {transferId, status: 'Accepted', trackingNumber, estimatedSettlement}

   **RTP Network:**
   - POST https://rtp.network.api/payments/initiate
   - Real-time response (within 15 seconds):
     {paymentId, status: 'Completed', actualSettlementTimestamp, beneficiaryConfirmation}
   - If declined: {status: 'Declined', declineReason: 'AccountClosed'}

   • Receives confirmation from payment network:
     - Transaction reference number (network-assigned unique ID)
     - Execution status:
       * Accepted: Payment accepted by network, pending settlement
       * Completed: Payment fully settled (RTP only)
       * Rejected: Payment rejected by network with reason code
       * Pending: Payment queued for batch processing (ACH)
     - Settlement date and time (estimated or actual)
     - Network fees charged (if applicable)

   • Saves network response to Azure SQL (NetworkResponses table):
     {paymentId, network, networkReferenceId, status, settlementDate, networkFees, responseTimestamp}
→ Output: Payment executed on network, confirmation received

[Payment Execution Service - Status Update & Finalization]
→ Input: Network execution response from Payment Network Integration Service (NetworkResponses table in Azure SQL)
→ Actions:
   • Updates payment status in Azure SQL based on network response:
     - If Accepted/Completed: Debited → Executing → Completed
     - If Rejected: Debited → Failed
     - If Pending: Debited → Executing (await settlement confirmation)
   • Updates account transaction status in Core Banking System:
     - POST /api/accounts/{accountId}/transactions/{transactionId}/status
     - JSON: {status: 'Completed', networkReference, settlementDate}
   • Releases fund reservation (if still locked)
   • Saves final payment state to Azure SQL (Payments table with status: Completed/Failed)
   • Logs execution event to Azure Cosmos DB (event sourcing: PaymentExecuted event with network response)
   • Publishes completion message to Azure Service Bus (payment-confirmation-queue)
→ Output: Payment status finalized, ready for confirmation

[Payment Execution Service - Exception Handling & Compensation (Saga Pattern)]
→ Input: Exception during execution from Payment Network Integration Service or Core Banking System REST API
→ Actions:
   • Handles various failure scenarios with compensating transactions:

   **Network Timeout (no response within 60 seconds):**
   - Retry logic: 3 attempts with exponential backoff (30s, 2min, 5min)
   - If all retries fail:
     * Updates payment status: Executing → TimedOut
     * Does NOT reverse debit immediately (payment may still be processing)
     * Creates manual investigation case in ServiceNow
     * Operations team queries network status manually
     * Once confirmed failed: Triggers reversal (compensating transaction)

   **Insufficient Funds (Core Banking rejects debit):**
   - Status update: ApprovedForExecution → Failed
   - Reason: InsufficientFunds
   - Compensating transaction: Release fund reservation (if exists)
   - Notification to initiator:
     "Payment failed: Insufficient funds. Current balance: ${balance}, Required: ${amount + fees}"
   - Suggests: "Please add funds and retry payment"
   - Payment remains in system (can be retried after funding account)

   **Network Rejection (invalid account, closed account, invalid routing):**
   - Status update: Executing → Failed
   - Reason: NetworkRejection with error code (e.g., R03: No Account, R02: Account Closed)
   - Compensating transaction:
     * Reverses debit via Core Banking API (POST /api/accounts/{accountId}/credit)
     * Credits account with original debited amount
     * Updates transaction: Debited → Reversed
   - Logs rejection reason to Azure SQL (PaymentFailures table)
   - Notification to initiator with detailed error:
     "Payment rejected by network: ${rejectionReason}. Account debited amount has been refunded."
   - Suggests corrective action: "Please verify beneficiary account number and retry"

   **Partial Success (accepted by network but settlement pending/failed):**
   - Status: Executing → UnderInvestigation
   - Scenario: Network accepted payment but settlement fails (rare)
   - Actions:
     * Creates ServiceNow incident (severity: High)
     * Operations team contacts network support
     * Funds remain debited until resolution
     * Daily reconciliation job checks settlement status
   - If settlement eventually succeeds: Updates to Completed
   - If settlement fails after 7 days: Triggers automatic reversal

   **Duplicate Transaction ID (idempotency check failed):**
   - Checks if payment already executed:
     - SQL query: SELECT * FROM Payments WHERE PaymentId = @paymentId AND Status = 'Completed'
   - If duplicate found:
     * Prevents reprocessing (idempotency guarantee)
     * Returns existing transaction details
     * Does NOT debit account again
     * Logs duplicate attempt to Azure Monitor (potential bug or replay attack)
   - Status: No change (original transaction remains)

   • All exceptions logged to Azure Application Insights:
     - Custom exception telemetry with payment context
     - Correlation ID for end-to-end tracing
     - Alerting rules trigger for critical failures
→ Output: Exception handled, compensating transactions executed, status updated

---

Stage 7: Confirmation & Audit Trail
#=> Functional: Consume completion message → Generate receipt → Send confirmations → Log audit trail → Update dashboards → Queue for compliance reporting

[Payment Confirmation Service - .NET Core 8 Microservice]
→ Input: Consumes message from Azure Service Bus (payment-confirmation-queue)
→ Actions:
   • Retrieves completed payment details from Azure SQL:
     - SELECT p.*, r.*, n.* FROM Payments p
       JOIN PaymentRouting r ON p.PaymentId = r.PaymentId
       JOIN NetworkResponses n ON p.PaymentId = n.PaymentId
       WHERE p.PaymentId = @paymentId AND p.Status = 'Completed'
   • Retrieves approval chain from Azure SQL:
     - SELECT * FROM PaymentApprovals WHERE PaymentId = @paymentId ORDER BY ApprovalTimestamp
→ Output: Payment data aggregated, ready for confirmation

[Payment Receipt Generator - .NET Core 8 with PDF Library]
→ Input: Completed payment details from Azure SQL (Payments, PaymentRouting, NetworkResponses tables), provided by Payment Confirmation Service
→ Actions:
   • Generates payment receipt PDF using iTextSharp/QuestPDF library:

     **Receipt structure:**
     - Header section:
       * Bank logo and letterhead
       * Document title: "Payment Confirmation"
       * Receipt number (format: RCP-{yyyy}-{sequential})
       * Generation date and time

     - Payment details section:
       * Payment reference ID
       * Transaction date and time
       * Payer account (masked: ***1234)
       * Payment amount and currency (bold, large font)
       * Beneficiary name and account (masked: ***5678)
       * Payment type (Wire/ACH/SWIFT/RTP)
       * Payment purpose/description

     - Network execution details:
       * Selected network (e.g., "SWIFT MT103")
       * Network transaction reference ID
       * Execution timestamp
       * Settlement date (estimated or actual)
       * Network status (Completed/Pending Settlement)

     - Fee breakdown table:
       | Fee Type              | Amount    |
       |-----------------------|-----------|
       | Base transfer fee     | $25.00    |
       | Correspondent charges | $15.00    |
       | FX conversion fee     | $8.50     |
       | Total fees            | $48.50    |
       | Total debited         | $10,048.50|

     - Approval chain section (if applicable):
       * Approved by: John Smith (Manager) - Jan 18, 2026 10:30 AM
       * Approved by: Jane Doe (VP) - Jan 18, 2026 11:15 AM

     - Footer section:
       * Disclaimer text ("This is a system-generated receipt...")
       * Contact information (support email, phone)
       * Legal notices and terms
       * QR code (contains payment reference for mobile tracking)

   • Saves PDF to Azure Blob Storage (payment-receipts container):
     - Path: /receipts/{yyyy}/{mm}/{paymentId}.pdf
     - Generates SAS token (7-day expiry) for secure download link
   • Saves receipt metadata to Azure SQL (PaymentReceipts table):
     {paymentId, receiptNumber, blobUrl, sasToken, generatedTimestamp}
→ Output: PDF receipt generated and stored

[Notification Service - Multi-Channel Delivery]
→ Input: Completed payment from Azure SQL (Payments table) with receipt from Azure Blob Storage (PaymentReceipts table), invoked by Payment Confirmation Service
→ Actions:
   • Sends confirmation to payment initiator:

     **Email confirmation (via SendGrid API):**
     - POST https://api.sendgrid.com/v3/mail/send
     - Email template (HTML):
       * Subject: "Payment Completed - ${paymentReference}"
       * Body:
         - Payment summary (amount, beneficiary, status)
         - "Your payment of ${amount} ${currency} to ${beneficiary} has been successfully processed."
         - Network confirmation: "Transaction reference: ${networkReferenceId}"
         - Settlement information: "Funds will be available on ${settlementDate}"
         - Call-to-action buttons:
           * "Download Receipt" (links to PDF SAS URL)
           * "View Payment Details" (deep link to portal)
       * PDF receipt attached (base64 encoded)
       * Footer with support contact information

     **Portal notification:**
     - Creates in-app notification in Payment Dashboard
     - Notification bell icon with badge count
     - Notification content:
       * Title: "Payment Completed"
       * Message: "${paymentReference} - ${amount} ${currency}"
       * Timestamp: "2 minutes ago"
       * Action: Click to view payment details
     - Real-time delivery via SignalR:
       * SignalR hub method: notificationHub.Clients.User(userId).SendAsync("PaymentCompleted", notification)
       * Frontend receives notification and displays toast message
       * Updates payment list in real-time without page refresh

     **Mobile push notification (if mobile app installed):**
     - Sends via Firebase Cloud Messaging (FCM) for Android or APNs for iOS
     - Push payload:
       {title: "Payment Completed", body: "${amount} sent to ${beneficiary}",
        data: {paymentId, type: "payment_completed"}, badge: 1}
     - Deep link opens payment details screen in mobile app

   • Notification delivery tracking:
     - Logs delivery status to Azure SQL (NotificationDelivery table):
       {paymentId, channel, sentTimestamp, deliveryStatus, openedTimestamp}
     - Tracks email open rate (SendGrid webhook)
     - Tracks link click-through (UTM parameters)
→ Output: Multi-channel confirmations sent

[Audit Trail Service - Azure Cosmos DB Event Sourcing]
→ Input: Completed payment with all processing history
→ Actions:
   • Logs complete payment lifecycle as event stream to Azure Cosmos DB:
     - Container: payment-events
     - Partition key: paymentId
     - Events stored as immutable append-only log

     **Event sequence logged:**
     1. PaymentInitiated event:
        {eventId, paymentId, eventType: 'PaymentInitiated', timestamp,
         userId, userName, amount, currency, beneficiary, urgency}

     2. DocumentsUploaded event:
        {eventId, paymentId, eventType: 'DocumentsUploaded', timestamp,
         documents: [{documentType, fileName, blobUrl, fileSize}]}

     3. AIExtractionCompleted event:
        {eventId, paymentId, eventType: 'AIExtractionCompleted', timestamp,
         extractedData: {invoice, contract}, overallConfidence, aiModelVersion}

     4. ValidationCompleted event:
        {eventId, paymentId, eventType: 'ValidationCompleted', timestamp,
         reconciliationResults: {amountMatch, beneficiaryMatch},
         ruleValidationResults: {passed[], failed[]}, complianceScreening: {riskScore, sanctionHit}}

     5. RoutingDecisionMade event:
        {eventId, paymentId, eventType: 'RoutingDecisionMade', timestamp,
         selectedNetwork, routingReason, costEstimate, alternatives[]}

     6. ApprovalRequested event:
        {eventId, paymentId, eventType: 'ApprovalRequested', timestamp,
         approvalTier, assignedApprovers[], slaDeadline}

     7. PaymentApproved event(s):
        {eventId, paymentId, eventType: 'PaymentApproved', timestamp,
         approverName, approverRole, decision, comments}

     8. AccountDebited event:
        {eventId, paymentId, eventType: 'AccountDebited', timestamp,
         accountId, debitedAmount, transactionId, newBalance}

     9. PaymentExecuted event:
        {eventId, paymentId, eventType: 'PaymentExecuted', timestamp,
         network, networkReferenceId, executionStatus, settlementDate, networkFees}

     10. PaymentCompleted event:
         {eventId, paymentId, eventType: 'PaymentCompleted', timestamp,
          finalStatus, totalProcessingTime, endToEndDuration}

     11. ConfirmationSent event:
         {eventId, paymentId, eventType: 'ConfirmationSent', timestamp,
          notificationChannels: ['email', 'portal', 'mobile'], deliveryStatus}

   • Event metadata includes:
     - User context (userId, userName, IP address, device info)
     - System context (service name, version, instance ID)
     - Correlation ID (for distributed tracing across microservices)
     - Causation ID (links events in causal chain)

   • Event schema versioning:
     - Each event has schemaVersion field (e.g., v1.0, v2.0)
     - Supports schema evolution without breaking historical queries

   • Query patterns enabled by event sourcing:
     - Reconstruct payment state at any point in time
     - Audit trail for compliance: "Show all payments approved by John Smith in Q4 2025"
     - Performance analysis: "Calculate average time between validation and approval"
     - Anomaly detection: "Find payments with unusual approval patterns"
→ Output: Complete immutable audit trail stored in Cosmos DB

[ServiceNow Integration - SLA Dashboard Update]
→ Input: Completed payment with processing timestamps
→ Actions:
   • Calls ServiceNow REST API to update SLA tracking:
     - POST /api/now/table/payment_sla_metrics
     - JSON payload:
       {paymentId, initiationTime, completionTime, totalDuration,
        approvalDuration, executionDuration, slaTarget, slaActual, slaCompliance: true/false}
   • Updates ServiceNow dashboards:
     - Real-time SLA compliance metrics (98% target)
     - Average processing time by payment type
     - Approval turnaround time by approver
     - Exception rate and failure reasons
   • Triggers SLA breach alerts if processing time exceeded target
→ Output: SLA metrics updated in ServiceNow

[Compliance Reporting Service]
→ Input: Completed payment with audit trail
→ Actions:
   • Publishes message to Azure Service Bus (compliance-reporting-queue) for regulatory reporting:
     - Daily batch job aggregates compliance data
     - Generates reports required by regulators:
       * Large transaction reports (>$10K) for FinCEN (Currency Transaction Reports)
       * International payments report for OFAC compliance
       * Suspicious activity monitoring data
     - Reports stored in Azure Blob Storage (compliance-reports container)
   • Adds payment to compliance data warehouse (Azure Synapse Analytics):
     - Enables compliance analytics and trend analysis
     - Supports regulatory audit queries
→ Output: Payment queued for compliance reporting

[Performance Monitoring - Application Insights]
→ Input: Completed payment with processing metrics
→ Actions:
   • Tracks custom metrics via TelemetryClient:
     - PaymentProcessingDuration (total end-to-end time)
     - AIExtractionDuration (document processing time)
     - ValidationDuration (rule execution time)
     - ApprovalDuration (time from request to approval)
     - ExecutionDuration (network execution time)
     - SuccessRate (completed / total payments)
   • Logs performance data:
     - P50, P95, P99 percentiles for each stage
     - Identifies performance bottlenecks
     - Tracks SLA compliance metrics
   • Triggers alerts if metrics exceed thresholds:
     - Total processing time >10 minutes → Severity 2 alert
     - Success rate <95% → Severity 1 alert
     - Approval duration >4 hours → SLA breach alert
→ Output: Performance metrics logged, dashboards updated

---

Stage 8: Continuous Learning & Feedback Loop
#=> Functional: Store extraction results → Analyze patterns → Build training data → Retrain models → Adjust thresholds → Refine rules → Monitor performance

[ML Data Collection Service - Python/C# Integration]
→ Input: Completed payments with AI extraction results and manual corrections
→ Actions:
   • Stores AI extraction results for model improvement in Azure SQL (MLTrainingData table):
     {paymentId, documentType, extractedFields JSON, confidenceScores JSON, manualCorrections JSON,
      validationOutcome, extractionTimestamp, documentQuality}
   • Captures document type classification:
     - Invoice (structured, semi-structured, unstructured)
     - Contract (standard template, custom format)
     - Purchase order (digital PDF, scanned image)
   • Logs extraction confidence per field:
     {fieldName, extractedValue, confidenceScore, wasCorrect, correctedValue, correctionReason}
→ Output: Extraction data stored for analysis

[ML Pattern Analysis Service - Python ML Pipeline]
→ Input: Weekly batch job (Azure Function timer trigger, cron: 0 0 * * SUN) retrieves training data
→ Actions:
   • Queries extraction history from Azure SQL:
     - SELECT DocumentType, FieldName, ExtractedValue, ConfidenceScore, ManualCorrection,
       ValidationOutcome, DocumentQuality
       FROM MLTrainingData
       WHERE ExtractionTimestamp >= DATEADD(day, -7, GETDATE())

   • Analyzes patterns in document processing:

     **1. Document type accuracy analysis:**
     - Calculates per-field accuracy rates:
       * InvoiceNumber: 98% (high accuracy, no action needed)
       * VendorName: 87% (medium accuracy, needs improvement)
       * LineItems: 82% (low accuracy, focus area)
     - Identifies document types with low extraction accuracy (<90%):
       * Handwritten invoices: 75% accuracy
       * Degraded scans (low DPI): 68% accuracy
       * Non-standard contract formats: 79% accuracy

     **2. Common OCR misreads detection:**
     - Pattern analysis identifies systematic errors:
       * 0 (zero) → O (letter O): 45 occurrences
       * 1 (one) → I (letter I): 38 occurrences
       * 5 (five) → S (letter S): 27 occurrences
       * 8 (eight) → B (letter B): 15 occurrences
     - Creates character confusion matrix for post-processing rules

     **3. Validation outcome correlation:**
     - Analyzes relationship between confidence scores and validation outcomes:
       * Payments with >95% confidence: 96% validation pass rate
       * Payments with 85-95% confidence: 78% validation pass rate
       * Payments with <85% confidence: 42% validation pass rate
     - Identifies false positives (high confidence but validation failed)
     - Identifies false negatives (low confidence but validation passed)

     **4. Data comparison mismatch analysis:**
     - Common mismatches between payment entry vs extracted documents:
       * Vendor name variations: "Acme Corp" vs "ACME Corporation" (52 cases)
       * Amount rounding differences: $1,234.99 vs $1,235.00 (38 cases)
       * Currency notation: "USD 1000" vs "$1,000.00" (29 cases)
     - Identifies patterns where fuzzy matching tolerance should be adjusted

     **5. Manual review outcomes analysis:**
     - Tracks manual corrections made by operations team:
       * Most frequently corrected fields: VendorAddress (67 corrections), LineItems (54 corrections)
       * Correction reasons: OCR error (45%), Ambiguous formatting (30%), Missing data (25%)
     - Identifies reviewers with high correction rates (potential training issues)
→ Output: Comprehensive analysis report with improvement recommendations

[ML Training Dataset Builder - Python]
→ Input: Analysis results and corrected documents
→ Actions:
   • Builds enhanced training dataset:

     **1. Data export:**
     - Queries corrected documents from Azure SQL:
       SELECT DocumentId, BlobUrl, DocumentType, ExtractedFields, CorrectedFields
       FROM MLTrainingData
       WHERE ManualCorrection IS NOT NULL AND CorrectionTimestamp >= DATEADD(month, -1, GETDATE())
     - Downloads original documents from Azure Blob Storage
     - Minimum dataset size: 500 samples per document type

     **2. Ground truth labeling:**
     - Creates JSON with ground truth labels using manual corrections:
       ```json
       {
         "documentUrl": "https://storage.blob.core.windows.net/...",
         "documentType": "invoice",
         "fields": [
           {"name": "VendorName", "value": "Acme Corporation",
            "boundingBox": {"x": 120, "y": 50, "width": 200, "height": 20},
            "confidence": 0.98},
           {"name": "InvoiceNumber", "value": "INV-2026-001",
            "boundingBox": {"x": 120, "y": 80, "width": 150, "height": 18},
            "confidence": 0.99},
           {"name": "TotalAmount", "value": "1234.99",
            "boundingBox": {"x": 400, "y": 500, "width": 100, "height": 22},
            "confidence": 0.96}
         ]
       }
       ```
     - Includes bounding box coordinates for layout understanding
     - Adds negative samples (fields that should not be extracted)

     **3. Data augmentation:**
     - Creates variations to improve model robustness:
       * Image rotation (±5 degrees)
       * Brightness adjustment (±20%)
       * Noise addition (simulates low-quality scans)
       * Contrast enhancement
     - Increases training dataset from 500 to 2,500 samples per document type

     **4. Dataset export:**
     - Uploads training dataset to Azure Blob Storage (ml-training-data container):
       /training-datasets/{documentType}/{yyyy-mm-dd}/dataset.json
     - Splits dataset: 80% training, 20% validation
     - Creates dataset manifest file (CSV with document URLs and labels)
→ Output: Enhanced training dataset ready for model retraining

[Azure AI Document Intelligence - Model Retraining Pipeline]
→ Input: Training dataset from Blob Storage
→ Actions:
   • Retrains custom Azure AI models:

     **1. Model training API call:**
     - POST https://ai.azure.com/formrecognizer/documentModels:build
     - Request body:
       ```json
       {
         "modelId": "custom-invoice-v2.1",
         "description": "Retrained with 1000 new samples",
         "azureBlobSource": {
           "containerUrl": "https://storage.blob.core.windows.net/ml-training-data",
           "prefix": "training-datasets/invoice/2026-01-18/"
         },
         "buildMode": "template"
       }
       ```
     - Model training process:
       * Analyzes document layouts and field patterns
       * Trains on labeled data with transfer learning from base model
       * Validates on 20% holdout set
       * Training duration: 15-30 minutes for 500 documents

     **2. Model validation:**
     - Azure AI returns validation metrics:
       {modelId, accuracy: 0.94, precision: 0.92, recall: 0.95, f1Score: 0.935}
     - Per-field performance:
       * VendorName: 94% accuracy (improved from 87%)
       * InvoiceNumber: 99% accuracy (maintained)
       * TotalAmount: 97% accuracy (improved from 91%)
       * LineItems: 89% accuracy (improved from 82%, still needs work)
     - Compares new model vs current production model:
       * Overall accuracy improvement: +3.2% (from 91% to 94.2%)
       * Meets deployment threshold (>2% improvement required)

     **3. Model deployment decision:**
     - If improvement >2%: Deploy new model to production
     - If improvement <2%: Keep current model, log metrics for review
     - If accuracy degrades: Rollback, investigate dataset quality issues

     **4. Model versioning:**
     - Saves model metadata to Azure SQL (MLModels table):
       {modelId, version, trainingDate, accuracy, datasetSize, deployedDate, status: 'Production'}
     - Updates application configuration:
       - Modifies appsettings.json: "AzureAI:DocumentModelId": "custom-invoice-v2.1"
       - Or: Updates configuration in Azure App Configuration service
     - Old model archived for rollback capability
→ Output: Improved AI model deployed to production

[Business Rule Tuning Service - .NET Core 8]
→ Input: Validation outcomes and pattern analysis
→ Actions:
   • Adjusts confidence thresholds based on historical accuracy:

     **Dynamic threshold adjustment:**
     - Invoice (structured documents):
       * Current thresholds: High >95%, Medium 85-95%, Low <85%
       * Analysis shows false positives at 93-95% (validation fails despite high confidence)
       * Adjusted thresholds: High >96%, Medium 86-96%, Low <86%

     - Contract (unstructured documents):
       * Current thresholds: High >95%, Medium 85-95%, Low <85%
       * Analysis shows many 80-85% confidence documents pass validation
       * Adjusted thresholds: High >92%, Medium 80-92%, Low <80%

     - Handwritten forms:
       * Analysis shows lower baseline accuracy
       * Adjusted thresholds: High >88%, Medium 75-88%, Low <75%
       * Automatically routes to manual review if confidence <85%

   • Updates fuzzy matching tolerance based on false positive rate:
     - Current Levenshtein distance threshold: <5 characters
     - Analysis shows false positives >5% when threshold is 5
     - Adjusted threshold: <3 characters (stricter matching)
     - Exception: Allow looser matching for known corporate suffix variations:
       ("Inc." vs "Incorporated", "Corp" vs "Corporation" - pre-approved list)

   • Refines validation rules based on exception patterns:
     - Duplicate payment detection:
       * Current window: 90 days
       * Analysis shows legitimate recurring payments every 30 days flagged as duplicates
       * Refined rule: Exclude recurring payment patterns from duplicate check
       * Add whitelist for known recurring vendors

     - Amount tolerance validation:
       * Current tolerance: ±2%
       * Analysis shows FX rounding often causes 2.1-2.5% variance (false positives)
       * Adjusted tolerance: ±3% for multi-currency payments
       * Keep ±2% for domestic same-currency payments

   • Updates business rules in ServiceNow:
     - PUT /api/now/table/business_rules/{ruleId}
     - JSON payload: {ruleId, ruleCondition, threshold, effectiveDate}
     - Invalidates Redis cache for immediate rule updates
→ Output: Business rules and thresholds optimized based on data

[Performance Monitoring Dashboard - Power BI + Azure Monitor]
→ Input: ML metrics and system performance data
→ Actions:
   • Tracks key ML/AI performance indicators:

     **Extraction accuracy trends:**
     - Line chart showing accuracy over time (daily, weekly, monthly)
     - Breakdown by document type (invoice, contract, PO)
     - Target: >95% accuracy, current: 94.2%
     - Trendline shows improvement trajectory after each retraining

     **False positive/negative rates:**
     - Sanction screening false positives: 3.2% (down from 5.1%)
     - Sanction screening false negatives: 0.1% (critical metric, target <0.5%)
     - Validation false positives: 8.5% (payments incorrectly flagged for review)
     - Tracks impact on operational efficiency

     **Straight-through processing rate:**
     - Percentage of payments processed without manual intervention
     - Target: >80%, current: 78% (improving)
     - Breakdown by payment type and amount range
     - Identifies bottlenecks (e.g., manual review queue size)

     **Manual review queue metrics:**
     - Queue size over time (target: <50 pending items)
     - Average time in queue (target: <2 hours)
     - Resolution outcomes (approved, rejected, correction required)
     - Identifies peak times and staffing needs

     **Model retraining impact:**
     - Before/after comparison charts
     - Accuracy improvement per retraining cycle
     - Cost-benefit analysis (training cost vs operational savings)

   • Azure Monitor alerting:
     - Alert if extraction accuracy drops below 90% for 3 consecutive days
     - Alert if false negative rate (sanctions) exceeds 0.5%
     - Alert if STP rate drops below 75%
     - Alert if manual review queue exceeds 100 items

   • Automated reports:
     - Weekly ML performance summary emailed to data science team
     - Monthly model accuracy trends for management review
     - Quarterly ROI analysis of AI investment
→ Output: Comprehensive ML/AI performance monitoring

---
````

#### Solution Architecture

**Architecture Layers:**

```
┌─────────────────────────────────────────────────────────────┐
│ PRESENTATION LAYER                                          │
│ - Payment Web Portal (React TypeScript)                    │
│ - Admin Dashboard (React TypeScript)                       │
│ - Mobile App Integration (REST API)                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ API GATEWAY LAYER                                           │
│ - Azure API Management                                      │
│ - OAuth 2.0 Authentication                                  │
│ - Rate Limiting & Throttling                               │
│ - API Analytics & Monitoring                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ BUSINESS LOGIC LAYER (Microservices)                       │
│                                                              │
│ ┌──────────────────┐  ┌──────────────────┐                │
│ │ Payment Service  │  │ Document Service │                │
│ │ - Validation     │  │ - AI Processing  │                │
│ │ - Routing Engine │  │ - OCR/NLP        │                │
│ │ - Fee Calculation│  │ - Data Extract   │                │
│ └──────────────────┘  └──────────────────┘                │
│                                                              │
│ ┌──────────────────┐  ┌──────────────────┐                │
│ │ Compliance Svc   │  │ Notification Svc │                │
│ │ - AML Screening  │  │ - Email/SMS      │                │
│ │ - Sanction Check │  │ - Push Notify    │                │
│ └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ INTEGRATION LAYER                                           │
│ - Azure Service Bus (Message Queue)                        │
│ - ServiceNow REST API (Workflow Integration)               │
│ - Azure AI Document Intelligence (OCR/NLP)                 │
│ - External Payment Systems (SWIFT/ACH/Wire/RTP)           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ DATA LAYER                                                  │
│ - Azure SQL Database (Transaction data, Client profiles)   │
│ - Azure Blob Storage (Document repository)                 │
│ - Redis Cache (Session, Lookup data)                       │
│ - Azure Cosmos DB (Audit logs, Event sourcing)            │
└─────────────────────────────────────────────────────────────┘
```

**Key Components:**

1. **Payment Service (.NET Core 8)**
   - RESTful API endpoints for payment operations
   - Business rule engine for routing decisions
   - Transaction state management
   - Fee calculation and FX conversion

2. **Document Processing Service (Python + .NET Core)**
   - Azure AI Document Intelligence integration
   - Data extraction and validation
   - Confidence scoring algorithm
   - Auto-approval logic

3. **Compliance Service (.NET Core 8)**
   - AML screening integration
   - Sanction list verification (OFAC, EU, UN)
   - Regulatory reporting queue
   - Audit trail generation

4. **Notification Service (.NET Core 8)**
   - Multi-channel notification (Email, SMS, Push)
   - Template management
   - Delivery tracking and retry logic

5. **Business Rule Validation Engine (.NET Core 8)**
   - Standalone validation service (not a ServiceNow component)
   - Fetches business rules from ServiceNow via REST API (rules configured in ServiceNow UI)
   - Receives AI extraction results and comparison data from Azure AI Document Intelligence
   - Executes validation rules dynamically using .NET Rules Engine library
   - Aggregates validation results (pass/fail for each rule, overall status)
   - Posts complete validation results back to ServiceNow for workflow orchestration
   - Supports data comparison and correction workflows
   - Redis caching for rules (15-minute TTL)

6. **ServiceNow Platform (External Enterprise System)**
   - Enterprise workflow orchestration and business process management platform
   - **Technology Stack**:
     - Platform: ServiceNow Cloud (SaaS)
     - Workflow Development: ServiceNow Flow Designer (low-code visual workflow builder)
     - Business Rules: JavaScript (server-side scripting engine)
     - UI Customization: ServiceNow UI Builder (drag-and-drop interface designer)
     - Forms & Tables: Declarative configuration (no-code table/form designer)
     - REST API: ServiceNow Scripted REST APIs (JavaScript-based endpoints)
   - Business rule configuration and management (UI-based rule definition by business users)
   - Workflow execution engine (auto-approval, manual review routing, multi-tier approvals)
   - Repair queue management for data corrections and exception handling
   - Approval workflows for corrections (manager/VP approval based on thresholds)
   - Multi-channel notifications (email, SMS, in-app alerts)
   - SLA tracking and timeout handling with auto-escalation
   - Complete audit trail and compliance reporting
   - Integration via REST API with internal microservices

**Data Exchange Formats:**

- **Internal Services Communication** (JSON over REST/HTTPS):
  - Payment Service ↔ Document Service: JSON (client data, document metadata)
  - Payment Service ↔ Compliance Service: JSON (AML screening requests/responses)
  - Payment Service ↔ ServiceNow: JSON via REST API (approval workflows, rule configurations)
  - Document Service ↔ Azure AI: JSON (extraction requests/responses with confidence scores)
  - All microservices ↔ Azure Service Bus: JSON messages for async processing

- **External Payment Networks** (Industry-standard formats):
  - **SWIFT**: ISO 20022 XML format (MT103 for customer credit transfers, MT202 for bank transfers)
  - **ACH**: NACHA file format (fixed-width records, batch processing)
  - **Wire Transfer**: Fedwire format (proprietary XML/CSV based on bank)
  - **RTP (Real-Time Payments)**: ISO 20022 XML format (pain.001 for payment initiation)
  - Payment Processor Service handles format transformation from internal JSON to network-specific formats

- **Core Banking System**:
  - REST API with JSON payloads for account debit/credit operations
  - Real-time synchronous calls for balance checks and transaction posting

**Integration Patterns:**

- **Event-Driven Architecture**: Payment events published to Azure Service Bus for asynchronous processing
- **Request-Response**: Synchronous calls to compliance services for real-time validation
- **Saga Pattern**: Distributed transaction management for multi-step workflows:
  - **Document Onboarding Saga**: Upload → Barcode Decode → OCR Extract → Data Validation → Comparison → Approval (compensating transactions: reverse profile creation if validation fails, restore previous state)
  - **Payment Processing Saga**: Initiate → Validate → Route → Approve → Execute → Confirm (compensating transactions: reverse debit if payment network fails, refund if settlement fails)
  - Each saga step publishes events to Azure Service Bus, saga coordinator tracks state in Azure SQL Database, timeout handlers trigger compensation logic
- **Circuit Breaker**: Resilience pattern for external system calls (SWIFT, ServiceNow)
- **CQRS**: Separate read/write models for payment queries vs command execution

#### Technical Implementation Details

**Technology Stack & Rationale:**

| Technology                     | Purpose                   | Why Chosen                                                                                  |
| ------------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| .NET Core 8 / C# 12            | Microservices backend     | High performance, async/await support, cross-platform, excellent Azure integration          |
| Python                         | AI/ML document processing | Rich ecosystem for AI/ML (Azure SDK), data processing libraries (Pandas)                    |
| Azure AI Document Intelligence | OCR/NLP                   | Enterprise-grade accuracy (>95%), pre-trained models for financial documents, PII detection |
| Azure Service Bus              | Message queue             | Guaranteed delivery, dead-letter queue, session support for ordered processing              |
| ServiceNow                     | Workflow management       | Existing enterprise platform, rich approval workflows, audit capabilities                   |
| Azure SQL Database             | Transactional data        | ACID compliance, high availability (99.99% SLA), backup/restore, encryption at rest         |
| Redis Cache                    | Session & lookup data     | Sub-millisecond latency, reduces database load by 80%, supports distributed caching         |
| React TypeScript               | Frontend                  | Type safety, component reusability, excellent developer experience, large ecosystem         |
| ZXing.Net                      | Barcode reading           | Open-source barcode library, supports QR codes, high accuracy for document barcodes         |

**Key Design Patterns Applied:**

1. **Repository Pattern**
   - Abstract data access layer
   - Unit of Work for transaction management
   - Generic repository for CRUD operations

2. **Factory Pattern**
   - Payment routing factory to select appropriate payment system
   - Document processor factory based on form type

3. **Strategy Pattern**
   - Routing strategy selection based on payment attributes
   - Validation strategy based on transaction type

4. **Chain of Responsibility**
   - Payment validation pipeline (balance → compliance → limits → approval)
   - Document processing pipeline (upload → scan → extract → validate → approve)

5. **Observer Pattern**
   - Event notification to multiple subscribers (audit, reporting, notification)

#### My Technical Contributions

**1. Payment Routing Engine Design & Implementation**

- Designed configurable business rules engine using Strategy pattern with 50+ routing rules across transaction types, amounts, geographic regions (15+ countries), and regulatory requirements (OFAC, EU sanctions)
- Implementation approach: Created .NET Rules Engine consuming rules from ServiceNow as JSON configuration (condition expressions, priority, target system), stored rules in Redis cache (15-min TTL), rules evaluated sequentially with first-match-wins logic, built routing simulation tool for pre-production testing with sample transaction sets
- **Impact**: Reduced routing errors from 12% to <1%, $180K annual savings

**2. AI Document Processing Integration & Barcode Validation**

- Barcode processing pipeline: Used ZXing.Net library with BarcodeReader class to decode QR codes from scanned forms, extracted embedded JSON data (Company Name, Tax ID, Bank Account, Submission Timestamp encrypted with AES-256), implemented barcode quality detection (if decode confidence <80%, flag for manual review)
- Data comparison engine: Built reconciliation algorithm comparing barcode data (original submission) vs OCR extracted data using Levenshtein distance for fuzzy string matching (threshold <3 characters for company names), exact match validation for Tax ID and account numbers, confidence scoring (High >95% match, Medium 85-95%, Low <85%), auto-approve if High confidence + all rules pass
- Integrated Azure AI Document Intelligence with custom validation engine: POST document to Azure AI endpoint (receives OCR results + confidence scores), fetch business rules from ServiceNow REST API filtered by form type (2D/Non-2D), execute validations using .NET Rules Engine library (Tax ID regex, duplicate SQL check, sanction API screening), aggregate results and POST back to ServiceNow for workflow orchestration
- Built feedback loop: Stored validation outcomes + AI confidence scores in SQL Database, analyzed patterns (field types with low accuracy), retrained custom models using misclassification data from manual reviews, adjusted confidence thresholds based on historical accuracy per form type
- **Impact**: Reduced onboarding time from 45 min to 4 min, 91% accuracy, 98% barcode decode success rate

**3. ServiceNow Workflow Integration**

- Implemented REST API integration: POST to ServiceNow /api/now/workflow/approve endpoint with payment metadata (amount, beneficiary, purpose), ServiceNow evaluates amount thresholds and routes to approval groups, webhook callback on approval/rejection updates payment status in SQL, timeout handler checks SLA every 15 min and escalates via ServiceNow priority update API
- **ASP.NET Core implementation**: Built middleware pipeline with custom error handling middleware (UseExceptionHandler), request/response logging middleware (UseMiddleware<LoggingMiddleware>), API versioning (MapToApiVersion v1/v2), AutoMapper for entity-to-DTO transformations (CreateMap<Payment, PaymentDto>().ForMember with custom value resolvers)
- **Impact**: Reduced approval time from 6 hours to 45 min, 98% SLA compliance

**4. Performance Optimization & Observability**

- Redis caching implementation: Lookup data (currencies, bank codes, country lists) cached at startup with background refresh, session tokens cached per user (30-min sliding expiration), business rules cached with pub/sub invalidation pattern when ServiceNow signals changes
- SQL optimization: Added composite indexes on (ClientId, Status) and (PaymentId, CreatedAt), rewrote N+1 query patterns to batch fetches using LINQ Select/Include, enabled query result caching via EF Core second-level cache
- Application Insights setup: Custom telemetry tracking using TelemetryClient, dependency tracking for external API calls, custom metrics (PaymentProcessingDuration, RoutingDecisionTime, DocumentProcessingAccuracy), alerting rules configured in Azure Monitor with action groups for email/SMS
- **Impact**: API response time 3.5s→400ms (P95), 5x transaction capacity, MTTR 45min→8min

#### NFRs & Technical Achievements

**Performance:**

- API response time: P95 <500ms
- Payment processing: 2 min average end-to-end
- Document processing: 4 min average
- Throughput: 500 txn/hour (peak: 800/hour)
- Concurrent users: 200+

**Scalability:**

- Microservices on Azure Kubernetes Service (AKS) with auto-scaling (CPU >70%, memory >80%)
- Azure Application Gateway load balancing across 3+ instances
- Redis distributed cache (80% DB load reduction)
- Azure Service Bus with queue partitioning

**Security & Compliance:**

- OAuth 2.0 + Azure AD + MFA, RBAC (5 permission levels)
- AES-256 encryption at rest, TLS 1.3 in transit, field-level PII encryption
- PCI-DSS Level 1, SOC 2 Type II, GDPR compliance
- Complete audit trail, API rate limiting (100 req/min/user)

**Business Impact:**

- 70% reduction in manual intervention
- 89% reduction in onboarding time (45 min → 4 min)
- 85% reduction in routing errors (12% → <1%)
- 87% improvement in approval efficiency (6 hours → 45 min)
- 98% SLA compliance
- $420K annual savings ($180K error corrections + $240K manual processing)
- 2x transaction capacity, zero security incidents (7 years)

===========================================================================================

## RESUME HIGHLIGHTS:

#### **Automated Document Processing Platform**

Architected AI-powered platform transforming corporate client onboarding from 30-45 minute manual reviews to <2 minutes automated processing with 95%+ accuracy.

**Architecture & Design:**

- Designed event-driven microservices architecture using Azure Service Bus, enabling independent scaling of 7 processing stages and supporting 10,000+ documents/day with 99.9% uptime
- Architected AI/ML pipeline integrating Azure Cognitive Services (Document Intelligence, Computer Vision) with custom C# orchestration for OCR extraction, achieving 95%+ accuracy with confidence-based auto-approval decisions
- Implemented continuous learning pipeline where maker-checker corrections retrain models, improving accuracy by 4% over 6 months and reducing manual review queue by 40%
- Designed workflow orchestration using Azure Durable Functions to coordinate document lifecycle (upload → OCR → validation → approval) with automatic retry and compensation for failures
- Architected secure barcode generation/decoding engine using ZXing.Net to embed and extract submission data, enabling automated data reconciliation with fuzzy matching validation
- Built flexible rule engine using C# expression trees supporting 50+ configurable validation rules (KYC, credit limits, sanctions screening) with business-configurable thresholds requiring zero code changes
- Implemented SignalR real-time notification system providing instant status updates to operations teams, eliminating manual refresh cycles and reducing issue resolution time by 60%
- Designed RESTful API layer with versioning and comprehensive error handling, integrating with 12+ downstream systems (ServiceNow, core banking, compliance) with 99.95% availability

**Technical Leadership & Best Practices:**

- Established Architecture Decision Records (ADR) practice documenting key design choices for team knowledge sharing
- Implemented comprehensive observability using Application Insights and Azure Monitor, reducing MTTR from 4 hours to 30 minutes
- Reduced Azure infrastructure costs by 25% through auto-scaling optimization, reserved instances, and blob storage lifecycle policies

**Business Impact:**

- Reduced document processing time from 30-45 minutes to <2 minutes (95% improvement)
- Enabled straight-through processing for 80% of qualified applications
- Achieved $250K annual cost savings through reduced manual effort
- Maintained 100% compliance across regulatory audits

**Technologies:** ASP.NET Core 8, C# 12, EF Core 8, Azure Functions, Azure Service Bus, Azure Cognitive Services, Azure Blob Storage, SignalR, React TypeScript, ZXing.Net

---

#### **Payment Processing & Routing System**

Architected intelligent payment platform processing corporate remittances across SWIFT, ACH, Wire, and RTP networks with AI-powered document validation and ServiceNow workflow integration.

**Architecture & Design:**

- Integrated Azure AI Document Intelligence with payment workflow to automate extraction and validation of supporting documents (invoices, contracts), reducing manual review from 15 minutes to <2 minutes per payment
- Designed tiered confidence-based processing architecture (high: auto-process, medium: manual review, low: reject) that prevented erroneous payments through automated amount/vendor validation
- Architected intelligent routing engine using Strategy pattern with C# expression trees to evaluate 50+ decision rules across payment networks, reducing routing errors from 12% to <1%
- Implemented Redis-cached routing rules (15-min TTL) with first-match-wins evaluation, achieving $180K annual savings through optimal network selection
- Designed ServiceNow REST API integration orchestrating multi-tier approval workflows with configurable thresholds ($50K-$500K) and automated escalation, achieving 98% SLA compliance
- Architected integration layer transforming payment formats (JSON to network-specific XML/fixed-width) for 15+ countries, eliminating manual format conversion that caused 12% of routing errors
- Implemented Saga pattern with compensating transactions (reverse debit on network failure, refund on settlement failure), reducing failed transactions from 8% to <0.5%
- Designed core banking integration using HttpClient with Polly circuit breaker and retry policies for real-time account operations with fund locking, preventing race conditions
- Built comprehensive audit trail using Cosmos DB event sourcing capturing complete payment lifecycle, reducing compliance audit preparation from 2 weeks to 2 hours

**Cloud Cost Optimization:**

- Optimized Cosmos DB partitioning strategy reducing RU consumption by 40% and monthly costs by $2,500
- Implemented Azure Service Bus message batching reducing messaging costs by 30%

**Business Impact:**

- Reduced transaction processing time from 4-6 hours to <2 minutes (98% improvement)
- Achieved $180K annual savings through optimal payment routing
- Reduced approval delays from 6 hours to 45 minutes
- Passed 100% of regulatory audits with zero findings

**Technologies:** ASP.NET Core 8, C# 12, EF Core 8, Azure Service Bus, Azure Cognitive Services, Azure SQL, Cosmos DB, Redis, Polly, ServiceNow API, AutoMapper

---
