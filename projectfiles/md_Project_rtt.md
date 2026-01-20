# PROJECT DETAILS - RTT (ROYAL BANK OF SCOTLAND / THOUGHTWORKS)

## AVINASH LONDHE

---

## INDEPENDENT PROJECT / CONSULTING ENGAGEMENT - (Oct 2024 – Present)

---

### PROJECT: VENDOR PORTAL - B2B VENDOR & REQUIREMENT MATCHING PLATFORM

#### Project Overview

Enterprise B2B SaaS platform enabling companies to post requirements, manage vendor profiles, and leverage AI-powered matching to connect requirement providers with qualified vendors. Multi-tenant architecture supporting complex role-based access control, document management, and intelligent recommendation system for automated vendor-requirement matching.

#### Business Context & Problem Statement

**Business Challenge:**

- Traditional vendor discovery and requirement fulfillment processes relied on manual networking, inefficient email exchanges, and fragmented communication channels
- Companies struggled to find qualified vendors matching their specific requirements across multiple domains (technology, consulting, staffing, services)
- Vendors lacked visibility into relevant opportunities and spent significant time manually searching for matching requirements
- No systematic way to manage vendor profiles, track applications, or maintain audit trails for compliance
- Manual shortlisting and evaluation processes consumed excessive time for procurement teams
- Lack of intelligent matching algorithms resulted in poor vendor-requirement fit and high rejection rates

**Pain Points:**

- Manual vendor discovery taking 2-4 weeks per requirement across multiple sources (LinkedIn, job boards, referrals)
- 60% mismatch rate between vendor capabilities and requirement specifications due to poor screening
- Limited visibility for vendors into relevant opportunities matching their skill sets and capacity
- Document management challenges with scattered files across email attachments, shared drives, and manual tracking
- No standardized process for requirement posting, application tracking, and vendor response management
- Compliance and audit trail gaps due to informal communication channels and undocumented decisions

**Stakeholder Requirements:**

- Companies needed dual-role capability to act as both requirement providers (posting needs) and suppliers (responding to opportunities)
- Procurement teams required AI-powered vendor recommendations based on profile matching with 80%+ relevance accuracy
- Vendors needed intelligent requirement discovery with automated shortlisting capabilities
- Compliance teams required comprehensive audit trails, document management, and role-based access controls
- Operations teams needed multi-tenant architecture supporting scalable growth with 1000+ companies and 10,000+ users
- IT teams required cloud-native deployment with containerization, CI/CD pipelines, and infrastructure as code

#### Functional Flow & Process Design

**FLOW 1: Company Registration & Multi-Profile Management**

```
Stage 1: Company Onboarding
→ User initiates company registration via web portal (Angular UI)
→ Provide company details:
   • Company name, industry, size, location, contact information
   • Tax ID, registration number for verification
   • Select subscription plan (Basic, Professional, Enterprise)
→ User becomes default Company Admin for new company
→ System validates company uniqueness (duplicate check on tax ID, name)
→ OAuth integration support (Google, Microsoft) for streamlined authentication

Stage 2: Profile Creation & Management
→ Company Admin creates company profiles (multiple profiles per company):
   • Default profile: Core company capabilities and services
   • Specialized profiles: Domain-specific expertise (e.g., Cloud Migration, Data Analytics, AI/ML)
   • Profile attributes: Skills, certifications, industry experience, team size, hourly rates
→ Profile approval workflow:
   • Company User creates draft profiles
   • Company Admin reviews and approves before publication
→ Supporting documents uploaded to Azure Blob Storage:
   • Certifications, case studies, client testimonials, whitepapers
   • Document types validated (PDF, DOCX, max 10MB per file)

Stage 3: Role-Based Access Control Setup
→ Three-tier role hierarchy:
   • app_admin: Platform administrator (all access)
   • company_admin: Company management, user approval, profile/requirement approval
   • company_user: Create requirements/profiles, search, shortlist, communicate
→ Users can belong to multiple companies with one role per company
→ RoleActionPermission table enforces feature-level access control
→ AccessControlService validates permissions at API and service layer
```

**FLOW 2: Requirement Posting & Vendor Discovery**

```
Stage 1: Requirement Creation
→ Company Admin posts requirement via web portal:
   • Requirement details: Title, description, skills required, budget range, duration, location
   • Category classification: Technology, Consulting, Staffing, Services
   • Priority level: Low, Medium, High, Urgent
   • Deadline for applications
→ Supporting documents attached (RFP, specifications, SOW templates)
→ Requirement saved in PostgreSQL database with full-text search indexing

Stage 2: AI-Powered Vendor Matching
→ System triggers AI recommendation engine (TensorFlow.js / OpenAI API):
   • Analyze requirement text using NLP (extract keywords, required skills, domain)
   • Query vendor profiles matching criteria:
     - Skill overlap (minimum 70% match threshold)
     - Industry experience alignment
     - Geographic proximity (if location-specific)
     - Availability and capacity indicators
   • Machine learning model scores vendor-requirement fit (0-100 confidence score)
→ Top 20 recommended vendors ranked by relevance score
→ Recommendation results cached for performance (Redis)

Stage 3: Vendor Discovery & Application
→ Vendors search requirements via multiple channels:
   • Full-text search across requirement titles and descriptions
   • Filter by category, location, budget range, deadline
   • AI recommendations: "Requirements matching your profile"
→ Vendor reviews requirement details and supporting documents
→ Vendor selects appropriate profile to apply (if multiple profiles available)
→ Application submission (RequirementProfile entity):
   • Cover letter or proposal
   • Upload supporting documents (capability statement, previous work samples)
   • Estimated timeline and pricing
→ Application tracked in system with status workflow:
   • Submitted → Under Review → Shortlisted → Approved / Rejected / Request More Info
```

**FLOW 3: Profile Sharing & Direct Engagement**

```
Stage 1: Direct Profile Sharing
→ Vendor (Provider) selects profile to share with target company (Recipient)
→ System creates SharedProfile entity linking:
   • Provider company + Profile + Recipient company
   • Share date, status (Active, Expired, Revoked)
→ Recipient company receives notification (email + in-app)
→ Shared profile visible only to recipient company (privacy enforcement)

Stage 2: Shortlisting Workflows
→ Two-way shortlisting capability:
   • Vendors shortlist requirements for future tracking (ShortlistedRequirement)
   • Requirement owners shortlist vendor profiles (ShortlistedProfile)
→ Shortlist actions tracked for analytics and recommendation improvement
→ Shortlisted items displayed in dedicated dashboard views

Stage 3: Communication & Collaboration
→ In-app messaging system for requirement discussions
→ Email notifications for key events:
   • Application received
   • Application status change (shortlisted, approved, rejected)
   • New shared profile
   • Requirement deadline approaching
→ Audit trail maintained for all communications (UsageLog table)
```

**FLOW 4: Document Management & Security**

```
Stage 1: Document Upload & Storage
→ Documents linked to entities via DocumentType:
   • Profile documents (certifications, case studies)
   • Requirement documents (RFP, specifications)
   • Application documents (proposals, work samples)
   • Invoice documents (payment records)
→ Files uploaded to Azure Blob Storage with secure URLs
→ Document metadata stored in PostgreSQL (filename, size, upload date, uploader)

Stage 2: Access Control & Permissions
→ Document visibility enforced by AccessControlService:
   • Profile documents: Visible to profile owner and companies with shared profile access
   • Requirement documents: Visible to requirement owner and applicant vendors
   • Application documents: Visible to requirement owner and applicant vendor only
→ JWT token validation on document download requests
→ Audit log for all document access (who, when, which document)

Stage 3: Search & Discovery
→ Full-text search across document metadata and content (future: OCR for PDF content)
→ Filter documents by type, date range, associated entity
→ Document versioning support (future enhancement)
```

#### Solution Architecture

**Architecture Style:** Microservices-ready monolith with clean separation of concerns, designed for future microservices decomposition

**Technology Stack:**

- **Frontend:** Angular 16+ with TypeScript, Angular Material UI components, RxJS for reactive state management, SCSS for styling
- **Backend:** ASP.NET Core 8+ with C# 12, Entity Framework Core (Code-First), AutoMapper for DTO mapping, MediatR for CQRS pattern
- **Database:** PostgreSQL (primary), Redis (caching layer for AI recommendations and session state)
- **Authentication:** JWT-based authentication with ASP.NET Identity, OAuth 2.0 integration (Google, Microsoft)
- **AI/ML:** TensorFlow.js for client-side inference, OpenAI API for semantic matching and NLP processing
- **Cloud Storage:** Azure Blob Storage for document management with secure SAS tokens
- **DevOps:** Docker containerization, Kubernetes orchestration, GitHub Actions CI/CD, Terraform for IaC
- **Monitoring:** Azure Application Insights (future), structured logging with Serilog

**Architecture Patterns Applied:**

- **Clean Architecture:** Separation of domain, application, infrastructure, and presentation layers
- **Repository Pattern:** Data access abstraction with generic repository and unit of work
- **CQRS Pattern:** Command/Query separation using MediatR for complex business operations
- **Dependency Injection:** Native ASP.NET Core DI container for loose coupling
- **DTO Pattern:** Separation of domain entities and API contracts with AutoMapper
- **Role-Based Access Control (RBAC):** RoleActionPermission table with AccessControlService enforcement at service and controller layers

**Deployment Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                     Load Balancer (Azure LB)                │
└────────────────────┬────────────────────────────────────────┘
                     │
     ┌───────────────┴───────────────┐
     │                               │
┌────▼─────────┐              ┌─────▼────────┐
│ Angular SPA  │              │ Angular SPA  │
│ (Container)  │              │ (Container)  │
└────┬─────────┘              └─────┬────────┘
     │                               │
     └───────────────┬───────────────┘
                     │ HTTPS / REST API
     ┌───────────────▼───────────────┐
     │                               │
┌────▼──────────────┐       ┌────────▼─────────┐
│ ASP.NET Core API  │       │ ASP.NET Core API │
│ (Container)       │       │ (Container)      │
└────┬──────────────┘       └────────┬─────────┘
     │                               │
     └───────────────┬───────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌─────▼─────┐   ┌────▼─────────┐
│PostgreSQL│    │   Redis   │   │Azure Blob    │
│ Cluster  │    │   Cache   │   │Storage       │
└──────────┘    └───────────┘   └──────────────┘
```

#### Technical Implementation Highlights

**Backend Architecture (ASP.NET Core):**

- **Controllers:** RESTful API endpoints with attribute routing, model validation, and HTTP verb-specific actions
- **Services:** Business logic layer with transactional boundaries and error handling
- **Repositories:** Generic repository pattern for CRUD operations with EF Core
- **DTOs:** Request/Response models separate from domain entities, mapped with AutoMapper
- **Middleware:** Custom JWT authentication middleware, exception handling middleware, request/response logging
- **Validation:** FluentValidation for complex business rule validation at API boundary
- **Unit of Work:** Transaction management across multiple repository operations

**Frontend Architecture (Angular):**

- **Components:** Smart/presentational component separation for reusability
- **Services:** API client services using HttpClient with interceptors for JWT injection
- **Guards:** Route guards for authentication and role-based authorization
- **Resolvers:** Data pre-loading for smoother navigation experience
- **State Management:** RxJS BehaviorSubjects for reactive state across components
- **Material Design:** Angular Material components for consistent, accessible UI

**AI/ML Integration:**

- **Vendor-Requirement Matching Algorithm:**
  - NLP text analysis: Extract skills, domain keywords from requirement description
  - TF-IDF vectorization: Convert text to numerical feature vectors
  - Cosine similarity calculation: Measure profile-requirement similarity (0-1 score)
  - Weighted scoring: Skill match (50%), industry experience (25%), location (15%), availability (10%)
  - Threshold filtering: Return vendors with confidence score > 0.7
- **Recommendation Caching:** Redis cache for 24-hour TTL to optimize performance
- **Feedback Loop (Future):** Track user actions (shortlist, apply, approve) to improve ML model

**Security Implementation:**

- **Authentication:** JWT tokens with 1-hour expiration, refresh token mechanism (future)
- **Authorization:** Custom authorization policies based on RoleActionPermission table
- **Data Protection:** Sensitive data encrypted at rest (database), in transit (HTTPS/TLS 1.3)
- **CORS:** Configured for specific frontend origins only
- **SQL Injection Prevention:** Parameterized queries via EF Core, no raw SQL
- **XSS Prevention:** Angular sanitization, Content Security Policy headers

**DevOps & CI/CD:**

- **Branching Strategy:** main → staging → uat → test → develop → feature/\*
- **CI Pipeline:** GitHub Actions on PR (build, test, code quality checks)
- **CD Pipeline:** Automated deployment to dev/test/staging/prod environments
- **Containerization:** Multi-stage Docker builds for optimized image size
- **Orchestration:** Kubernetes deployment with rolling updates, health checks, resource limits
- **Infrastructure:** Terraform scripts for Azure resources (AKS, PostgreSQL, Blob Storage, Redis)

#### My Contribution (Architecture & Design)

- Architected microservices-ready clean architecture with clear separation of concerns enabling future scalability and maintainability
- Designed multi-tenant B2B platform architecture supporting complex role-based access control across 1000+ companies with hierarchical permission model
- Designed AI-powered vendor-requirement matching system using NLP, TF-IDF vectorization, and cosine similarity algorithms with 80%+ relevance accuracy
- Architected document management system with Azure Blob Storage integration, secure SAS token generation, and role-based document visibility
- Designed PostgreSQL database schema with 25+ normalized tables supporting multi-tenancy, audit trails, and complex relationships
- Implemented CQRS pattern using MediatR for separation of read/write operations in complex business workflows
- Designed RESTful API architecture with 50+ endpoints following REST principles, proper HTTP verbs, and consistent response formats
- Architected JWT-based authentication system with OAuth 2.0 integration for Google and Microsoft identity providers
- Designed DevOps pipeline with GitHub Actions CI/CD, Docker containerization, Kubernetes orchestration, and Terraform IaC
- Implemented Repository and Unit of Work patterns for clean data access layer abstraction with Entity Framework Core
- Designed comprehensive audit trail system tracking user actions, document access, and state transitions for compliance
- Architected caching strategy using Redis for AI recommendation results, session state, and frequently accessed data
- Designed branching strategy supporting multiple environments (dev, test, uat, staging, prod) with controlled promotion workflow
- Implemented full-text search capabilities across requirements, profiles, companies, and documents using PostgreSQL text search
- Designed notification system architecture supporting email and in-app notifications for 20+ event types
- Architected Angular frontend with component-based architecture, reactive state management, and Material Design consistency

**Technologies Applied:** ASP.NET Core 8, C# 12, Angular 16+, TypeScript, Entity Framework Core, PostgreSQL, Redis, Azure Blob Storage, TensorFlow.js, OpenAI API, Docker, Kubernetes, Terraform, GitHub Actions, JWT, OAuth 2.0, MediatR (CQRS), AutoMapper, Angular Material, RxJS, Serilog

**Project Status:** Active development, production-ready MVP completed, ongoing feature enhancements

---

Enterprise-wide budget tracking and forecasting platform consolidating financial data from multiple sources (SharePoint, Clarity PPM, Azure DevOps) to provide real-time visibility into $6M+ annual portfolio with automated reconciliation, resource-cost validation, and Power BI integration.

#### Business Context & Problem Statement

**Business Challenge:**

- Bank program management tracked budgets across multiple projects with data scattered across SharePoint lists, Clarity PPM system, and Azure DevOps (ADO) project tracking
- Manual consolidation process required 2+ days per month with significant data accuracy risks
- Budget vs actuals reconciliation was error-prone and time-consuming
- Forecast variance analysis required manual Excel manipulation
- No real-time visibility into financial health across portfolio
- Reporting delays impacted decision-making and resource allocation

**Pain Points:**

- Manual data extraction from 3 separate systems consuming 16+ hours per month
- Excel-based consolidation prone to formula errors and version control issues
- Budget overruns discovered too late due to monthly reporting cycle
- Inconsistent data formats requiring manual cleansing and transformation
- Limited forecasting capability based on historical trends
- No automated alerts for variance thresholds or budget risks

**Stakeholder Requirements:**

- Program managers needed real-time budget visibility across all projects
- Finance team required automated reconciliation reducing manual effort by 80%
- Leadership needed forecast accuracy improvements for resource planning
- Project teams needed self-service budget reports without IT dependency
- Compliance required complete audit trails for all financial data changes

#### Functional Flow & Process Design

**FLOW 1: Automated Data Extraction & Consolidation**

```
Stage 1: Data Source Integration
→ SharePoint REST API Integration:
   • Connect to SharePoint lists storing budget allocations
   • Extract: Project code, budget amount, fiscal year, cost center, approval status
   • Filter by fiscal year (e.g., FY2017)
   • Handle pagination for large datasets (1000+ projects)
→ Clarity PPM API Integration:
   • Query Clarity for actual costs and resource hours
   • Extract: Project code, actual costs, actual hours, period (monthly breakdown)
   • Date range: FY start to current date (e.g., Apr 2016 - Dec 2017)
   • Aggregate by project and cost category
→ Azure DevOps (ADO) REST API Integration:
   • Fetch work item data from ADO (sprints, iterations, user stories, tasks)
   • Extract: Project key, iteration path, sprint capacity, team members assigned, work item status, completed work
   • Query resource assignments: GET /_apis/wit/workitems with expand=Relations to get assigned-to users
   • Extract team capacity: GET /{project}/_apis/work/teamsettings/iterations/{iterationId}/capacities
   • Calculate iteration variance (planned vs actual completion dates)
   • Identify at-risk projects based on sprint velocity decline or capacity mismatches

Stage 2: Data Transformation & Validation
→ Data Standardization:
   • Normalize project codes across systems (SharePoint: "PRJ-001", Clarity: "001", ADO: "PROJ-001")
   • Convert currencies to base currency (USD)
   • Standardize date formats (ISO 8601)
   • Clean text fields (trim spaces, remove special characters)
   • Normalize resource names across ADO and Clarity (handle aliases, email variations)
→ Data Quality Checks:
   • Validate mandatory fields (project code, budget amount)
   • Check for duplicate entries
   • Verify budget > 0 and actuals >= 0
   • Flag projects with actuals > budget (variance threshold)
   • Identify missing data (projects in SharePoint but not in Clarity or ADO)
→ Business Logic Calculations:
   • Budget remaining = Current budget - Actual costs
   • Variance % = (Budget remaining / Current budget) * 100
   • Burn rate = Actual costs / Fiscal year progress %
   • Expected spend = Current budget * FY progress %
   • Spend variance = Actual costs - Expected spend
   • At-risk flag = Spend variance > 10% of budget

Stage 3: Reconciliation & Variance Analysis
→ Budget vs Actuals Reconciliation:
   • Match projects across all three systems (SharePoint, Clarity, ADO)
   • Identify unmatched projects (budget allocated but no actuals, actuals without budget)
   • Calculate aggregate variances by cost center, program, project manager
   • Detect budget re-forecasts (budget changes mid-year)
→ **Resource & Cost Reconciliation (ADO vs Clarity)**:
   • Extract resources from ADO: Team members assigned to work items, capacity allocations
   • Extract resources from Clarity: Resources with time entries (actuals), hourly rates, total cost
   • Cross-match resources: Identify resources in ADO but NOT logging time in Clarity (ghost resources)
   • Identify resources in Clarity but NOT in ADO (undocumented work, possible contractors)
   • Calculate cost impact:
     - Ghost resources: Capacity allocated in ADO × avg hourly rate = potential wasted capacity cost
     - Undocumented resources: Clarity actuals without ADO assignment = untracked work cost
   • Generate alerts: Email to project managers with resource mismatches, cost impact summary ($X wasted, $Y untracked)
→ Forecast Variance Analysis:
   • Compare latest forecast vs original budget
   • Calculate forecast accuracy (actual vs forecast variance)
   • Identify forecast drift patterns (consistent over/under estimates)
   • Project completion forecast based on burn rate
→ Exception Handling:
   • Flag high-variance projects (>20% over/under budget)
   • Alert on missing data or sync issues
   • Log data quality issues for manual review
   • Create exception report for finance team

Stage 4: Power BI Data Preparation & Export
→ Data Model Optimization:
   • Create fact table (consolidated financial data)
   • Create dimension tables (projects, cost centers, fiscal periods)
   • Define relationships (project code foreign keys)
   • Add calculated columns for Power BI consumption
→ Export to SQL Server:
   • Truncate staging tables
   • Bulk insert consolidated data (pandas to_sql)
   • Create indexes on key columns (project_code, fiscal_year, cost_center)
   • Update metadata table with last refresh timestamp
→ Power BI Refresh Trigger:
   • Call Power BI REST API to trigger dataset refresh
   • Schedule daily refresh at 6 AM (Azure Function timer)
   • Send completion notification to stakeholders
   • Log execution metrics (records processed, duration, errors)
```

**FLOW 2: Budget Forecasting & Predictive Analytics**

```
Stage 1: Historical Data Analysis
→ Collect 3-year historical budget and actuals data
→ Calculate historical burn rates by project type (infrastructure, application, operations)
→ Identify seasonal patterns (Q1 slow start, Q4 spend surge)
→ Analyze accuracy of past forecasts (forecast vs actual variance)

Stage 2: Forecast Model Application
→ Linear trend extrapolation:
   • Project remaining spend based on current burn rate
   • Adjust for known upcoming milestones (hardware purchases, vendor payments)
→ Scenario analysis:
   • Best case: 90% budget utilization (under-spend)
   • Most likely: 100% budget utilization (on-target)
   • Worst case: 110% budget utilization (over-spend)
→ Forecast confidence scoring:
   • High confidence: Projects >60% complete with stable burn rate
   • Medium confidence: Projects 30-60% complete
   • Low confidence: Projects <30% complete or high variance

Stage 3: Risk Identification & Alerting
→ Identify at-risk projects:
   • Spend variance >10% of budget
   • Burn rate >1.2x expected (overspending trend)
   • Milestone slippage >2 weeks (project delays)
→ Generate automated alerts:
   • Email to project manager and finance lead
   • Dashboard notification (red/amber/green indicators)
   • Weekly executive summary report
```

#### Solution Architecture

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│ DATA SOURCES                                                │
│ - SharePoint Lists (Budget allocations, forecasts)         │
│ - Clarity PPM (Actual costs, resource hours)               │
│ - JIRA (Milestone tracking, project status)                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ETL PROCESSING LAYER (Python)                               │
│                                                              │
│ ┌──────────────────┐  ┌──────────────────┐                │
│ │ Data Extractors  │  │ Data Transformers│                │
│ │ - SharePoint API │  │ - Normalization  │                │
│ │ - Clarity API    │  │ - Validation     │                │
│ │ - JIRA API       │  │ - Calculations   │                │
│ └──────────────────┘  └──────────────────┘                │
│                                                              │
│ ┌──────────────────┐  ┌──────────────────┐                │
│ │ Reconciliation   │  │ Forecast Engine  │                │
│ │ Engine           │  │ - Trend Analysis │                │
│ │ - Matching       │  │ - Scenario Model │                │
│ │ - Variance       │  │ - Risk Scoring   │                │
│ └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ DATA WAREHOUSE (SQL Server)                                 │
│ - ConsolidatedFinancials (fact table)                       │
│ - Projects, CostCenters, FiscalPeriods (dimensions)        │
│ - ExecutionLog (audit trail)                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ VISUALIZATION LAYER (Power BI)                              │
│ - Budget vs Actuals Dashboard                               │
│ - Forecast Variance Reports                                 │
│ - Risk & Alert Summaries                                    │
│ - Executive Rollup Views                                    │
└─────────────────────────────────────────────────────────────┘
```

**Key Components:**

1. **SharePoint API Connector (Python + SharePoint REST API)**

   - Authenticates using OAuth 2.0 with service account credentials
   - Queries SharePoint lists via REST API endpoints
   - Handles pagination (1000 items per batch)
   - Extracts budget allocations, forecast updates, approval status

2. **Clarity PPM API Connector (Python + Clarity SOAP API)**

   - Integrates with Clarity v15.x SOAP web services
   - Queries project financial actuals (costs and hours by period)
   - Aggregates data by project code and fiscal period
   - Handles multi-currency conversions to base currency (GBP)

3. **Azure DevOps (ADO) API Connector (Python + ADO REST API)**

   - Authenticates using Personal Access Token (PAT)
   - Fetches work items, iterations, team capacity, and resource assignments
   - Extracts team members assigned to projects and their capacity allocations
   - Calculates iteration variance (planned vs actual completion)
   - Identifies blocked or at-risk projects based on velocity trends

4. **Resource Reconciliation Engine (Python + Pandas)**

   - Cross-matches resources between ADO and Clarity using name/email normalization
   - Identifies ghost resources: Resources in ADO but not logging time in Clarity
   - Identifies undocumented work: Resources in Clarity but not assigned in ADO
   - Calculates cost impact of resource mismatches (wasted capacity vs untracked work)
   - Generates resource mismatch alerts with cost impact analysis

5. **Data Consolidation Engine (Python + Pandas)**

   - Merges data from three sources using project code as key
   - Performs data quality validation and cleansing
   - Calculates financial metrics (variance, burn rate, forecast)
   - Generates exception reports for data quality issues

6. **Forecast & Analytics Engine (Python + NumPy)**

   - Applies statistical models for budget forecasting
   - Performs scenario analysis (best/most likely/worst case)
   - Calculates forecast confidence scores
   - Generates risk alerts based on thresholds

7. **SQL Server Data Warehouse**

   - Stores consolidated financial data in star schema design
   - Fact table: ConsolidatedFinancials (budget, actuals, variance by period)
   - Dimensions: Projects, CostCenters, FiscalPeriods, ProjectManagers, Resources
   - Resource mismatch table: Tracks ghost resources and undocumented work with cost impact
   - Audit: ExecutionLog tracks all ETL runs with timestamps and metrics

8. **Scheduling & Orchestration (Azure Functions)**
   - Daily automated execution at 6 AM GMT
   - Error handling with email notifications
   - Execution logging for audit and troubleshooting
   - Power BI dataset refresh trigger after successful load

#### Technical Implementation Details

**Technology Stack & Rationale:**

| Technology            | Purpose                  | Why Chosen                                                                          |
| --------------------- | ------------------------ | ----------------------------------------------------------------------------------- |
| Python 3.9            | ETL processing           | Rich ecosystem for data processing (Pandas, NumPy), excellent API integration       |
| Pandas                | Data transformation      | Efficient DataFrame operations, built-in data cleaning, statistical functions       |
| NumPy                 | Numerical computations   | Fast array operations, statistical analysis, forecasting calculations               |
| SharePoint REST API   | Budget data extraction   | Native integration with SharePoint 2013+, OAuth 2.0 authentication                  |
| Clarity PPM SOAP API  | Actuals extraction       | Official Clarity integration, comprehensive financial data access                   |
| Azure DevOps REST API | Resource & work tracking | Official ADO API, team capacity queries, work item assignments, iteration data      |
| Azure Functions       | Serverless orchestration | Event-driven, cost-effective, auto-scaling, built-in timer triggers                 |
| SQL Server 2016       | Data warehouse           | Enterprise-grade database, Power BI native connector, robust indexing               |
| Power BI Desktop      | Visualization            | Rich visualizations, DAX calculations, scheduled refresh, self-service capabilities |

#### My Technical Contributions

**1. Multi-Source Data Integration & Resource Tracking Architecture**

- Designed and implemented ETL pipeline integrating 3 disparate systems (SharePoint, Clarity PPM, Azure DevOps)
- Built reusable API connectors with error handling and retry logic:
  - SharePoint REST API with OAuth 2.0 authentication and token refresh
  - Clarity PPM SOAP API with XML parsing and multi-currency conversion
  - Azure DevOps REST API with PAT authentication, pagination handling for work items, team capacity queries
- Implemented ADO resource extraction: GET /\_apis/wit/workitems to fetch assigned resources, GET /\_apis/work/teamsettings/iterations/{id}/capacities for team capacity
- **Impact**: Reduced data extraction time from 4 hours to 15 minutes, eliminated manual errors, added resource tracking capability

**2. Resource & Cost Reconciliation Engine**

- Built resource cross-matching algorithm between ADO and Clarity:
  - Normalized resource names handling aliases, email variations (john.doe vs jdoe@company.com)
  - Fuzzy matching with Levenshtein distance for name variations (threshold <3 chars)
  - Extracted ADO team members with capacity allocations (hours/week) from iteration APIs
  - Extracted Clarity resources with actual time entries and hourly rates
- Implemented cost impact analysis:
  - **Ghost resources**: Resources in ADO but NOT logging time in Clarity
    - Calculation: ADO capacity allocation × average hourly rate × weeks = wasted capacity cost
    - Example: 5 ghost resources × 40 hours/week × $75/hour × 12 weeks = $180K potential waste
  - **Undocumented work**: Resources in Clarity but NOT assigned in ADO
    - Calculation: Clarity actual hours × hourly rate = untracked work cost
    - Example: 200 hours × $85/hour = $17K untracked consultant work
- Created automated alerts with cost impact summaries sent to project managers and finance leads
- **Impact**: Identified $450K in resource inefficiencies (ghost resources + undocumented work), improved resource utilization by 18%

**2. Data Quality & Validation Framework**

- Developed comprehensive validation framework checking:
  - Mandatory field presence (project code, budget amount, resource assignments)
  - Data type validation (numeric amounts, date formats)
  - Business rule validation (budget > 0, actuals >= 0, variance thresholds)
  - Cross-system consistency (project exists in all 3 systems, resources properly assigned)
- Built automated data cleansing routines:
  - Project code normalization (handling different formats across systems: "PRJ-001" vs "001" vs "PROJ-001")
  - Resource name standardization (aliases, email domains, name variations)
  - Currency standardization to USD base currency
  - Date format standardization (ISO 8601)
  - Text field cleaning (trim, lowercase, special character removal)
- Implemented exception reporting for data quality issues with actionable recommendations
- **Impact**: Improved data accuracy from 85% to 99%, reduced manual data cleanup by 90%

**3. Financial Reconciliation & Resource-Cost Analytics Engine**

- Designed 3-way reconciliation algorithm matching budget, actuals, and iteration data from ADO
- Implemented financial calculations:
  - Budget variance analysis (remaining budget, variance percentage)
  - Burn rate calculation (actual spend vs expected spend by period)
  - Forecast variance (current forecast vs original budget)
  - At-risk project identification (spend variance >10%, ADO iteration velocity decline >20%)
  - Resource utilization metrics (ADO capacity vs Clarity actuals)
- Built forecast engine with statistical models:
  - Linear trend extrapolation based on historical burn rates
  - Scenario analysis (best/most likely/worst case spending)
  - Confidence scoring based on project completion % and variance stability
- Created automated alerting for budget risks with email notifications
- **Impact**: Improved forecast accuracy by 40%, identified budget risks 3 weeks earlier on average, quantified $450K resource inefficiency

**4. SQL Server Data Warehouse Design**

- Architected star schema optimized for Power BI reporting:
  - Fact table (ConsolidatedFinancials) with 50K+ rows
  - 5 dimension tables (Projects, CostCenters, FiscalPeriods, ProjectManagers, Resources)
  - ResourceMismatch table tracking ghost resources and undocumented work with cost impact
  - Clear relationships enabling drill-down analysis
- Implemented efficient data loading:
  - Bulk insert using pandas to_sql with batch size optimization (chunks of 1000 rows)
  - Truncate and load strategy for daily full refresh
  - Indexing strategy reducing query times from 8s to 400ms (composite indexes on frequently queried columns)
- Created audit trail capturing all ETL execution metrics (records processed, duration, errors, resource mismatches)
- **Impact**: Power BI report refresh time reduced from 5 minutes to 30 seconds, added resource cost analysis capability

**5. Power BI Integration & Automation**

- Integrated Python ETL with Power BI using SQL Server as intermediary
- Configured automated dataset refresh via Power BI REST API
- Scheduled daily execution using Windows Task Scheduler (later migrated to Azure Functions)
- Implemented execution monitoring with email notifications for success/failure
- Created technical documentation for Power BI developers consuming the data
- **Impact**: Enabled self-service reporting for 50+ stakeholders, eliminated manual data preparation

**6. Process Optimization & Performance Tuning**

- Optimized API calls using batch requests and parallel processing (ThreadPoolExecutor)
- Implemented caching for static reference data (cost centers, project managers)
- Reduced database writes using bulk insert instead of row-by-row inserts
- Added incremental processing for large historical datasets
- Monitored execution metrics and tuned bottlenecks:
  - API calls: Reduced from 45 min to 8 min using batch requests
  - Data transformation: Reduced from 15 min to 3 min using vectorized Pandas operations
  - Database load: Reduced from 10 min to 2 min using bulk insert
- **Impact**: Total pipeline execution reduced from 2+ days to 2 hours (96% reduction)

#### NFRs & Technical Achievements

**Performance:**

- Pipeline execution: 2 hours end-to-end
- Data extraction: 15 min for 3 sources
- Data transformation: 3 min for 50K+ records
- Database load: 2 min bulk insert
- Power BI refresh: 30s

**Scalability & Reliability:**

- Handles 50K+ financial records per run, 1,000+ active projects
- 3 years historical data loaded in <5 min
- 99.5% pipeline availability, automatic retry (3 attempts, exponential backoff)
- Power BI supports 100+ concurrent users

**Business Impact:**

- 92% reduction in manual effort (2 days → 2 hours monthly)
- 40% forecast accuracy improvement (variance 25% → 15%)
- 99% data accuracy eliminating manual reconciliation errors
- $450K resource inefficiencies identified (ghost resources + undocumented work)
- £120K annual savings (2 FTE equivalents)
- 50+ stakeholders enabled with self-service reporting
- 3 weeks earlier budget risk identification

---

## TECHNOLOGY STACK SUMMARY & SKILLS TO BRUSH UP

Based on the technical contributions across both projects (Financial Consolidation + Process Mining), here's a comprehensive list of technologies and components to review:

### ✅ **Python & Data Engineering Stack (Strong - Maintain Proficiency)**

**Core Python Libraries:**

- ✓ Pandas (DataFrames, merge operations, groupby, rolling windows, data cleansing)
- ✓ NumPy (statistical calculations, percentiles, array operations, forecasting)
- ✓ requests (REST API integration with OAuth 2.0, error handling, pagination)
- ✓ FuzzyWuzzy (Levenshtein distance for fuzzy matching, name normalization)

**API Integration:**

- ✓ SharePoint REST API (OAuth 2.0 authentication, list queries, pagination)
- ✓ Clarity PPM SOAP API (WSDL, XML parsing, zeep library, multi-currency)
- ✓ Azure DevOps REST API (PAT authentication, work items, team capacity, iterations)
- 🔄 **Azure Functions Python runtime** (timer triggers, HTTP triggers, deployment)

**Recommended Practice:**

- Build end-to-end ETL pipeline: Extract (3 APIs) → Transform (Pandas) → Load (SQL)
- Practice fuzzy matching with Levenshtein distance for name/code normalization
- Implement resource reconciliation algorithm (cross-system matching)
- Azure Functions with timer trigger for scheduled ETL jobs

---

### 🔄 **Azure Cloud Services (Good - Refresh Deployment)**

**Serverless & Compute:**

- ✓ Azure Functions (timer triggers for scheduled jobs, HTTP triggers for APIs)
- 🔄 **Azure App Service** (hosting Python Flask/Django apps)

**Integration:**

- ✓ Azure DevOps REST API (work items, iterations, team settings, capacities)
- 🔄 **Azure Logic Apps** (workflow orchestration alternative to Functions)

**Recommended Refresh:**

- Deploy Azure Functions with Python 3.9+ runtime
- Practice Azure DevOps API queries (work items, team capacity extraction)
- Azure Functions consumption plan vs premium plan trade-offs

---

### ✅ **SQL Server & Data Warehousing (Strong - Maintain)**

**Database Design:**

- ✓ Star schema design (fact tables, dimension tables, relationships)
- ✓ Indexing strategy (composite indexes, query optimization)
- ✓ Bulk insert operations (pandas to_sql, batch processing)
- ✓ Audit logging (execution metrics, ETL run tracking)

**SQL Queries:**

- ✓ Complex JOIN operations (3-way matching, cross-system reconciliation)
- ✓ Aggregations and GROUP BY (variance calculations, rollups)
- ✓ Window functions (rolling calculations, trends)

**Recommended Practice:**

- Design star schema for financial reporting (facts + dimensions)
- Optimize query performance with proper indexing
- Practice bulk insert with pandas to_sql (chunking, batch sizes)

---

### ✅ **Power BI (Strong - Maintain)**

**Power BI Development:**

- ✓ Power BI Desktop (report design, data modeling, DAX)
- ✓ DirectQuery vs Import mode (trade-offs, performance)
- ✓ DAX measures (CALCULATE, DIVIDE, aggregations, time intelligence)
- ✓ Power BI REST API (dataset refresh automation, scheduled jobs)

**Recommended Refresh:**

- Power BI dataflows for ETL transformation
- Advanced DAX patterns (dynamic segmentation, time-based calculations)
- Row-level security (RLS) for multi-tenant reports

---

### 🔄 **Process Mining & Log Analysis (Specialized - Maintain if Targeting)**

**Process Mining Tools:**

- ✓ Regex parsing for log extraction (event identification, timestamp extraction)
- ✓ Flow construction algorithms (trace building, state transitions)
- ✓ Cycle time calculations (duration analysis, bottleneck identification)
- ✓ Statistical analysis (min/max/avg, quartiles, box plots)

**Visualization:**

- ✓ Matplotlib (pie charts, box plots, bar charts, PNG exports)
- ✓ Excel exports (openpyxl, conditional formatting, styled reports)

**Recommended Practice (if targeting process mining roles):**

- Build log parser with regex for custom log formats
- Implement flow construction algorithm (event ordering, trace building)
- Calculate process metrics (cycle time, active vs waiting time)

---

### ⚠️ **Gaps to Address (High Priority)**

**Testing & Quality:**

- ⚠️ **pytest** (Python unit testing framework) - NOT in documentation
- ⚠️ **unittest.mock** (mocking for API integrations) - NOT in documentation
- ⚠️ **Code coverage tools** (pytest-cov) - NOT in documentation

**Recommended Learning:**

- Write pytest tests for API extractors (mock API responses)
- Test data validation logic with parameterized tests (@pytest.mark.parametrize)
- Achieve 80%+ code coverage with pytest-cov

**Time Investment:** 2-3 days

- Day 1: pytest basics, test fixtures, parametrized tests
- Day 2: unittest.mock for API mocking, testing ETL pipelines
- Day 3: Code coverage analysis, CI/CD integration

---

**DevOps & CI/CD:**

- 🔄 **Azure DevOps Pipelines** (YAML pipelines, CI/CD for Python)
- 🔄 **GitHub Actions** (alternative CI/CD platform)
- ⚠️ **Terraform / ARM Templates** (Infrastructure as Code) - NOT in documentation

**Recommended Learning:**

- Create Azure DevOps pipeline for Python ETL deployment
- Write Terraform scripts for Azure Functions provisioning
- Implement CI/CD with automated testing + deployment

**Time Investment:** 3-4 days

- Day 1-2: Azure DevOps YAML pipelines (build, test, deploy)
- Day 3-4: Terraform for Azure infrastructure provisioning

---

### 📋 **Priority Action Plan (Next 5-7 Days)**

**High Priority (3-4 days):**

1. ⚠️ **pytest + unittest.mock** (Python testing) - 2-3 days
2. 🔄 **Azure Functions deployment** (Python runtime, timer triggers) - 1 day

**Medium Priority (2-3 days):** 3. 🔄 **Azure DevOps API deep dive** (work items, capacity queries) - 1 day 4. 🔄 **Azure DevOps Pipelines** (CI/CD for Python) - 2 days

**Lower Priority (if time permits):** 5. 🔄 **Terraform basics** (Infrastructure as Code) - 1-2 days

**Total Time Investment:** 5-7 days to close critical gaps

---

### 📊 **Skills Readiness Assessment**

| Category                  | Current Level | Target Level | Gap              | Days to Close |
| ------------------------- | ------------- | ------------ | ---------------- | ------------- |
| Python/Pandas/NumPy       | 95%           | 95%          | None             | Maintain      |
| API Integration           | 90%           | 95%          | ADO API practice | 1 day         |
| SQL Server/Data Warehouse | 90%           | 95%          | Maintain         | Maintain      |
| Power BI                  | 85%           | 90%          | Advanced DAX     | 1 day         |
| Azure Functions           | 70%           | 85%          | Deployment       | 1 day         |
| Testing (pytest)          | 40%           | 85%          | Unit tests       | 2-3 days      |
| DevOps (CI/CD)            | 50%           | 80%          | Pipelines, IaC   | 3-4 days      |
| **Overall Readiness**     | **80%**       | **90%**      | **Total**        | **5-7 days**  |

---

### 🎯 **Interview Preparation Focus**

**Be Ready to Discuss:**

1. ETL pipeline design for multi-source integration (SharePoint, Clarity, Azure DevOps)
2. Resource reconciliation algorithm (ghost resources, undocumented work, cost impact)
3. Data quality validation framework (mandatory fields, business rules, cross-system consistency)
4. Power BI integration with Python ETL (SQL Server as intermediary, automated refresh)
5. Azure Functions timer triggers for scheduled ETL jobs
6. Star schema design for financial reporting (fact/dimension tables, indexing strategy)
7. Process mining with regex log parsing and flow construction

**Practice Coding Questions:**

- Implement fuzzy matching algorithm for resource name normalization (Levenshtein distance)
- Build ETL pipeline with Pandas (extract → transform → load pattern)
- Write pytest tests for API extractor with mocked responses
- Design star schema for financial consolidation (fact + dimensions)
- Calculate financial metrics (variance, burn rate, forecast) with Pandas

---
