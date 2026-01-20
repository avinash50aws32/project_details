# PROJECT DETAILS - THOUGHTWORKS

## AVINASH LONDHE

---

## THOUGHTWORKS - PUNE, INDIA (Aug 2013 – Oct 2016)

---

### SYSTEM 1: DYNAMIC PRICING ENGINE (McKinsey - Pricing/Consulting)

#### Project Overview

Intelligent pricing recommendation system leveraging AWS cloud services, big data processing, and machine learning to provide real-time optimal pricing recommendations across retail stores using historical data, seasonal trends, geographic factors, and competitor pricing analysis.

#### Business Context & Problem Statement

**Business Challenge:**

- McKinsey consulting client required data-driven pricing optimization across retail stores
- Manual pricing decisions based on intuition leading to suboptimal revenue
- No systematic analysis of seasonal patterns, geographic competition, or historical performance
- Inability to process and analyze millions of price points across locations, products, and time periods
- Lack of real-time pricing recommendations for flash sales or urgent adjustments
- Limited visibility into pricing effectiveness and competitor positioning

**Pain Points:**

- Manual pricing strategy taking weeks to update across product categories
- No centralized view of pricing data across geographic locations
- Seasonal pricing adjustments reactive rather than predictive
- Competitor pricing intelligence gathered manually and sporadically
- Flash sale pricing decisions ad-hoc without data analysis
- Historical pricing performance not systematically tracked or analyzed

**Stakeholder Requirements:**

- Retail operations needed real-time pricing recommendations with configurable parameters
- Analytics team required ML-powered prediction models using historical data
- Regional managers needed geographic-specific pricing considering local competition
- Finance team needed pricing effectiveness tracking and revenue impact analysis
- IT operations needed scalable system handling millions of price points daily

#### Functional Flow & Process Design

**FLOW 1: Real-time Data Ingestion & Storage**

```
Stage 1: Multi-Source Price Data Collection
→ Retail stores publish price data to AWS Kinesis Data Streams in real-time:
   • Product details: Product ID, category, SKU, description
   • Pricing data: Current price, cost, margin, discount percentage
   • Location data: Store ID, geographic region, city, postal code
   • Temporal data: Timestamp, season, day of week, holiday indicator
   • Sales data: Units sold, revenue, inventory level
→ Kinesis receives millions of price points daily from multiple geographic locations
→ Data partitioned by geographic region for parallel processing

Stage 2: Data Validation & Transformation (AWS Lambda)
→ Lambda functions triggered by Kinesis stream events
→ Data validation checks:
   • Mandatory field presence (product ID, price, location, timestamp)
   • Data type validation (numeric prices, valid timestamps, valid location codes)
   • Business rule validation (price > cost, valid discount percentages)
   • Outlier detection (prices >3 standard deviations from category mean)
→ Data transformation:
   • Currency normalization to USD base
   • Date/time standardization (UTC timezone)
   • Geographic code standardization
   • Product category hierarchies applied
→ Invalid records routed to error queue for manual review

Stage 3: Multi-Tier Storage Strategy
→ Raw data lake (AWS S3):
   • Store all raw price points partitioned by location/date/product
   • S3 lifecycle policy: 90 days hot storage → 1 year Glacier → delete
   • Parquet format for efficient columnar storage and compression
→ Structured analytics database (SQL Server):
   • Store aggregated pricing metrics and historical trends
   • Fact tables: Daily price changes, sales transactions, inventory snapshots
   • Dimension tables: Products, stores, dates, regions, categories
   • Indexed on product_id, location_id, date for fast query performance
→ Fast query layer (AWS Elasticsearch):
   • Real-time indexing of current prices and recent trends
   • Support multi-dimensional queries: geography × seasonality × product category × time
   • Sub-second query response for API layer
```

**FLOW 2: Big Data Processing & Feature Engineering**

```
Stage 1: Historical Data Processing (AWS EMR + Scala)
→ Scheduled batch jobs (nightly) processing historical pricing data
→ Scala Spark applications on EMR cluster:
   • Read Parquet files from S3 data lake (millions of records)
   • Aggregate pricing statistics by product-location-season combinations:
     - Average price, median, min, max, standard deviation
     - Sales volume correlations with price points
     - Competitor price differentials
     - Seasonal price elasticity metrics
   • Calculate geographic pricing patterns:
     - Urban vs rural pricing trends
     - Regional competition intensity scores
     - Location-specific demand seasonality
   • Generate time-series features:
     - 7-day, 30-day, 90-day rolling averages
     - Week-over-week price change velocity
     - Holiday season impact factors

Stage 2: ML Feature Store Creation
→ Processed features stored in S3 feature store:
   • Product features: Category, brand, cost structure, historical elasticity
   • Location features: Competition density, demographic segments, purchasing power
   • Temporal features: Season, day-of-week patterns, holiday proximity
   • Historical performance: Revenue trends, margin optimization, inventory turnover
→ Features versioned and tracked for model reproducibility
→ Feature quality metrics calculated (completeness, consistency, freshness)
```

**FLOW 3: ML Model Training & Pricing Optimization**

```
Stage 1: Model Training Pipeline (AWS SageMaker)
→ Training data preparation:
   • Combine features from feature store with historical sales outcomes
   • Create training/validation/test splits (70/15/15)
   • Feature scaling and normalization
→ Model algorithms:
   • Gradient Boosting (XGBoost) for price-demand prediction
   • Time series forecasting (Prophet) for seasonal patterns
   • Regression models for competitor price response
→ Hyperparameter tuning using SageMaker automatic model tuning
→ Model evaluation metrics:
   • Mean Absolute Percentage Error (MAPE) for demand prediction
   • Revenue optimization score (predicted revenue vs actual)
   • A/B test performance in pilot stores
→ Model versioning and registry in SageMaker Model Registry

Stage 2: Pricing Algorithm & Recommendation Engine
→ .NET-based pricing recommendation service consuming ML models:
   • Input: Product, location, target date, business constraints
   • Fetch relevant features from Elasticsearch and feature store
   • Call SageMaker endpoint for ML prediction (predicted demand at price points)
   • Apply business rules:
     - Minimum margin requirements
     - Maximum discount limits
     - Competitor positioning constraints
     - Inventory clearance priorities
   • Generate pricing recommendations:
     - Optimal price maximizing revenue
     - Alternative scenarios (volume vs margin optimization)
     - Confidence intervals and risk assessment
     - Price elasticity estimates
→ Recommendation caching in Redis (5-minute TTL for frequently queried products)

Stage 3: Real-time Price Monitoring (AWS Kinesis Analytics)
→ Real-time analytics on incoming price stream:
   • Detect sudden price changes (>15% movement)
   • Identify competitor pricing actions requiring immediate response
   • Flash sale opportunity detection (inventory clearance + high demand)
   • Geographic pricing anomalies (price gaps across nearby stores)
→ Generate immediate recommendations for urgent adjustments
→ Alert notifications via AWS SNS to pricing managers
```

**FLOW 4: API Services & Pricing Delivery**

```
Stage 1: Pricing Recommendation API (ASP.NET Web API)
→ RESTful API endpoints:
   • GET /api/pricing/recommend/{productId}/{locationId}
     - Returns optimal price with confidence score, alternatives, elasticity
   • POST /api/pricing/batch-recommend
     - Bulk pricing recommendations for product catalogs
   • GET /api/pricing/analysis/{productId}
     - Historical pricing performance, competitor comparison, trends
   • GET /api/pricing/scenarios
     - Scenario modeling (what-if pricing simulations)
→ API authentication via API keys with rate limiting (1000 requests/hour)
→ Response caching in Redis for frequently accessed products

Stage 2: Dashboard & Visualization (Power BI + React)
→ Power BI dashboards connecting to SQL Server data marts:
   • Pricing effectiveness: Revenue impact, margin trends, price elasticity
   • Geographic pricing: Regional price maps, competition heat maps
   • Seasonal analysis: Holiday pricing performance, trend forecasts
   • Product category insights: Category-level pricing optimization
→ React.js admin portal for pricing managers:
   • Real-time price monitoring across stores
   • Recommendation review and approval workflow
   • Manual override capabilities with audit logging
   • A/B test management and results tracking
→ SignalR integration for real-time dashboard updates
```

#### Solution Architecture

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│ DATA INGESTION LAYER                                        │
│ - AWS Kinesis Data Streams (Real-time price data)         │
│ - AWS Lambda (.NET Core) - Validation & Transformation    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STORAGE LAYER                                               │
│ - AWS S3 (Raw data lake - Parquet format)                 │
│ - SQL Server (Structured analytics - fact/dimension tables)│
│ - AWS Elasticsearch (Fast query layer - real-time index)  │
│ - Redis Cache (API response cache, session data)          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ BIG DATA PROCESSING LAYER                                   │
│ - AWS EMR (Elastic MapReduce) - Scala Spark               │
│ - Feature Engineering (Product/Location/Temporal features) │
│ - Historical Aggregation (Statistics, patterns, trends)    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ML & ANALYTICS LAYER                                        │
│ - AWS SageMaker (ML model training & hosting)             │
│ - AWS Kinesis Analytics (Real-time stream processing)     │
│ - .NET Pricing Algorithm (Recommendation engine)          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ API & PRESENTATION LAYER                                    │
│ - ASP.NET Web API (Pricing recommendations, analytics)    │
│ - React.js Dashboard (Admin portal, real-time monitoring) │
│ - Power BI (Executive dashboards, pricing analytics)      │
│ - AWS SNS (Alert notifications)                           │
└─────────────────────────────────────────────────────────────┘
```

**Key Components:**

1. **Data Ingestion Pipeline (AWS Kinesis + Lambda .NET Core)**

   - Real-time streaming ingestion handling millions of price points daily
   - Lambda functions for validation, transformation, and routing
   - Partitioning by geographic region for parallel processing
   - Error handling with dead-letter queue for failed records

2. **Big Data Processing (AWS EMR + Scala Spark)**

   - Distributed processing of historical pricing data (TB-scale)
   - Feature engineering for ML models (50+ features per product-location)
   - Aggregation pipelines calculating pricing statistics and trends
   - Parquet columnar format for efficient storage and query performance

3. **ML Model Training & Hosting (AWS SageMaker)**

   - Gradient Boosting models (XGBoost) for demand prediction
   - Time series forecasting (Prophet) for seasonal patterns
   - Hyperparameter tuning with automatic model optimization
   - Model versioning and A/B testing infrastructure
   - Real-time inference endpoints with auto-scaling

4. **Pricing Recommendation Engine (.NET Framework + ASP.NET Web API)**

   - ML model integration consuming SageMaker endpoints
   - Business rule engine applying margin and discount constraints
   - Scenario modeling and what-if analysis
   - Redis caching for performance optimization
   - RESTful API with authentication and rate limiting

5. **Fast Query Layer (AWS Elasticsearch Service)**

   - Real-time indexing of current prices and trends
   - Multi-dimensional aggregation queries (geography × product × season × time)
   - Sub-second query response supporting API layer
   - Full-text search for product and location lookups

6. **Real-time Analytics (AWS Kinesis Analytics)**

   - Stream processing for immediate pricing anomaly detection
   - Competitor action monitoring and alert generation
   - Flash sale opportunity identification
   - AWS SNS integration for notifications

7. **Visualization Layer (Power BI + React.js + SignalR)**
   - Power BI dashboards for executive pricing analytics
   - React.js admin portal with real-time monitoring
   - SignalR for live price updates and notifications
   - Manual override workflow with audit trails

#### Technical Implementation Details

**Technology Stack & Rationale:**

| Technology                 | Purpose                    | Why Chosen                                                                                     |
| -------------------------- | -------------------------- | ---------------------------------------------------------------------------------------------- |
| AWS Kinesis Data Streams   | Real-time data ingestion   | Scalable streaming (millions records/day), reliable delivery, Kinesis Analytics integration    |
| AWS Lambda (.NET Core)     | Event processing           | Serverless auto-scaling, event-driven architecture, cost-effective for variable workloads      |
| AWS S3 + Parquet           | Data lake storage          | Unlimited scalability, low cost, Parquet columnar format for efficient analytics               |
| AWS EMR + Scala Spark      | Big data processing        | Distributed processing for TB-scale data, Spark performance for aggregations, Scala efficiency |
| SQL Server                 | Structured analytics       | ACID compliance, complex query support, Power BI native integration                            |
| AWS Elasticsearch Service  | Fast query layer           | Sub-second multi-dimensional queries, full-text search, real-time indexing                     |
| AWS SageMaker              | ML model training/hosting  | Managed ML infrastructure, auto-scaling inference, built-in algorithms (XGBoost, Prophet)      |
| ASP.NET Web API + .NET 4.5 | Pricing API services       | High performance, excellent AWS SDK, synchronous/asynchronous patterns, enterprise stability   |
| Redis Cache                | API response cache         | Sub-millisecond latency, reduces backend load, supports distributed caching                    |
| React.js + TypeScript      | Admin dashboard            | Component reusability, real-time updates with SignalR, type safety                             |
| Power BI                   | Executive dashboards       | Rich visualizations, SQL Server native connector, scheduled refresh                            |
| AWS SNS                    | Notification service       | Multi-channel delivery (email/SMS), reliable message delivery, AWS ecosystem integration       |
| AWS Kinesis Analytics      | Real-time stream analytics | SQL-based stream processing, integration with Kinesis Streams, real-time anomaly detection     |

**Key Design Patterns Applied:**

1. **Lambda Architecture**

   - Batch layer: Historical processing with EMR
   - Speed layer: Real-time processing with Kinesis Analytics
   - Serving layer: Combined view via Elasticsearch and SQL Server

2. **CQRS (Command Query Responsibility Segregation)**

   - Write path: Kinesis → Lambda → S3/SQL
   - Read path: Elasticsearch for queries, Redis for caching
   - Separate optimization for ingestion vs query performance

3. **Event-Driven Architecture**

   - Kinesis events trigger Lambda processing
   - S3 events trigger EMR batch jobs
   - Price change events trigger SNS notifications

4. **Feature Store Pattern**

   - Centralized feature repository in S3
   - Versioned features for model reproducibility
   - Feature quality monitoring and validation

5. **Cache-Aside Pattern**
   - Check Redis cache first
   - On miss, query Elasticsearch/SQL
   - Update cache with TTL

#### My Technical Contributions

**1. Real-time Data Ingestion Pipeline Architecture**

- Designed Kinesis Data Streams architecture with partitioning strategy by geographic region enabling parallel processing
- Implemented AWS Lambda functions (.NET Core) for data validation and transformation:
  - Validation logic: Mandatory field checks, data type validation, business rules (price > cost, valid discounts)
  - Transformation: Currency normalization to USD, timezone standardization (UTC), geographic code mapping
  - Error handling: Invalid records routed to SQS dead-letter queue with detailed error messages
- Built multi-tier storage strategy: Raw data lake (S3 Parquet), structured analytics (SQL Server), fast queries (Elasticsearch)
- **Impact**: Ingestion capacity 2M price points/day, data available for analysis within 5 minutes, 99.5% data quality

**2. Big Data Processing with Scala on AWS EMR**

- Developed Scala Spark applications for historical data aggregation and feature engineering:
  - Read Parquet files from S3 (optimized partition pruning, predicate pushdown)
  - Calculated pricing statistics: Average, median, percentiles by product-location-season combinations
  - Generated time-series features: Rolling windows (7/30/90 days), week-over-week velocity, seasonal indices
  - Geographic analytics: Competition density scores, regional demand patterns, price elasticity by location
- Implemented EMR cluster auto-scaling based on workload (5-20 nodes dynamically)
- Optimized Spark jobs: Broadcast joins for dimension tables, repartitioning strategies, caching intermediate results
- **Impact**: Processing 500GB+ historical data in <2 hours, feature store with 50+ engineered features, 85% cost reduction via spot instances

**3. ML Model Integration with AWS SageMaker**

- Integrated SageMaker ML models with .NET pricing recommendation engine:
  - Built SageMaker training pipeline: Data prep from feature store, model training (XGBoost for demand prediction, Prophet for seasonality)
  - Deployed models to real-time inference endpoints with auto-scaling (target: 70% CPU utilization)
  - .NET API integration: REST calls to SageMaker endpoints with retry logic, exponential backoff, circuit breaker pattern
- Implemented recommendation algorithm combining ML predictions with business rules:
  - ML prediction: Demand forecast at various price points
  - Business constraints: Minimum margin thresholds, maximum discount limits, competitor positioning
  - Optimization: Revenue maximization vs volume maximization scenarios
- Built A/B testing framework comparing ML recommendations vs existing pricing in pilot stores
- **Impact**: 12-18% revenue increase in pilot stores, ML model MAPE <8%, 92% recommendation acceptance rate

**4. Fast Query Layer with Elasticsearch**

- Designed Elasticsearch index schema optimized for multi-dimensional pricing queries:
  - Document structure: Product, location, price, date, category hierarchy, competitor data
  - Mapping strategy: Keyword fields for exact matching, text fields for search, nested objects for hierarchies
  - Index partitioning: Monthly indices with alias for seamless rollover
- Implemented real-time indexing pipeline: Kinesis → Lambda → Elasticsearch (bulk API, batch size 1000)
- Built aggregation queries supporting dashboard requirements:
  - Geographic aggregations: Bucket by region → stats aggregation (avg price, percentiles)
  - Time-based aggregations: Date histogram with moving averages
  - Category drill-down: Terms aggregation with sub-aggregations
- Performance optimization: Index tuning (refresh interval 5s), query caching, replica shards for read scaling
- **Impact**: Query response time <200ms (P95), supports 500+ queries/min, 99.9% availability

**5. Pricing Recommendation API Development**

- Built RESTful API using ASP.NET Web API (.NET Framework 4.5) with comprehensive endpoints:
  - GET /api/pricing/recommend: Real-time pricing recommendations with confidence scores and alternatives
  - POST /api/pricing/batch-recommend: Bulk recommendations for product catalogs (up to 1000 products)
  - GET /api/pricing/analysis: Historical performance, competitor comparison, elasticity metrics
  - GET /api/pricing/scenarios: What-if simulations with custom constraints
- Implemented Redis caching strategy:
  - Frequently accessed products cached (TTL 5 min), cache-aside pattern
  - Cache invalidation via Redis pub/sub when prices updated
  - Distributed caching across multiple Redis nodes for high availability
- API performance optimization:
  - Async/await patterns for non-blocking I/O
  - Entity Framework query optimization (Select projections, Include eager loading)
  - Response compression (gzip) reducing payload size by 70%
- **Impact**: API response time P95 <300ms, throughput 1000 req/min, cache hit rate 85%, API uptime 99.95%

**6. Real-time Monitoring & Alerting**

- Implemented AWS Kinesis Analytics SQL queries for real-time anomaly detection:
  - Price change velocity detection (>15% movement within 1 hour)
  - Competitor action detection (price gaps exceeding thresholds)
  - Flash sale opportunity identification (high demand + excess inventory)
- Built AWS SNS notification system:
  - Email alerts to pricing managers with actionable recommendations
  - SMS alerts for critical pricing anomalies requiring immediate response
  - Topic subscription filtering by product category and geographic region
- Created React.js real-time dashboard with SignalR:
  - Live price updates pushed via WebSocket
  - Visual alerts for anomalies (color-coded by severity)
  - Drill-down capabilities for detailed analysis
- **Impact**: Alert response time <5 minutes, 40% faster pricing decision-making, 25% increase in flash sale revenue

#### NFRs & Technical Achievements

**Performance:**

- Data ingestion: 2M+ price points/day with <5 min latency
- Big data processing: 500GB+ historical data in <2 hours
- ML model inference: P95 <100ms via SageMaker endpoints
- API response time: P95 <300ms, throughput 1000 req/min
- Elasticsearch queries: P95 <200ms for multi-dimensional aggregations

**Scalability:**

- Kinesis auto-sharding handling 10x traffic spikes
- EMR cluster auto-scaling: 5-20 nodes based on workload
- SageMaker endpoints auto-scaling: 2-10 instances (CPU target 70%)
- Elasticsearch cluster: 5 data nodes with replica shards
- Redis cache: Distributed across 3 nodes with automatic failover

**Business Impact:**

- 12-18% revenue increase in pilot stores using ML recommendations
- 92% recommendation acceptance rate by pricing managers
- 40% faster pricing decision-making (alert response <5 min)
- 25% increase in flash sale revenue with real-time monitoring
- 85% infrastructure cost reduction using AWS spot instances and auto-scaling
- ML model accuracy: MAPE <8% for demand prediction
- System uptime: 99.95% availability

---

### SYSTEM 2: DIGITAL ADVERTISING ANALYTICS & DYNAMIC PRICING (Transport For London - Transportation Analytics)

#### Project Overview

Computer vision-based advertising effectiveness measurement and dynamic pricing system using AWS AI services (Rekognition) for crowd detection, demographic analysis, and attention metrics from cameras at digital advertising hoardings across London transportation network.

#### Business Context & Problem Statement

**Business Challenge:**

- Transport For London (TFL) sold advertising space on digital hoardings without data-driven pricing
- Traditional advertising pricing based on location and fixed rates regardless of actual audience reach
- No systematic measurement of advertising effectiveness or viewer engagement
- Inability to optimize pricing based on real-time audience metrics
- Limited visibility into demographic composition and attention patterns
- Advertisers demanding ROI metrics and performance-based pricing

**Pain Points:**

- Fixed advertising rates not reflecting actual viewer reach or engagement
- No audience measurement system for digital hoardings
- Manual demographic estimates without data validation
- Pricing decisions based on historical averages rather than real-time data
- Advertisers unable to optimize campaign timing for target demographics
- Missed revenue opportunities during peak audience periods

**Stakeholder Requirements:**

- Advertising sales team needed data-driven pricing based on measured audience metrics
- Operations team required real-time crowd analytics and attention measurement system
- Finance team needed dynamic pricing algorithm maximizing revenue per advertising slot
- Advertisers demanded demographic analytics and attention metrics for ROI justification
- IT operations needed scalable system handling 100+ camera feeds across London

#### Functional Flow & Process Design

**FLOW 1: Camera Integration & Video Stream Processing**

```
Stage 1: Camera Infrastructure & IoT Integration
→ Digital hoarding locations across London equipped with cameras and sensors:
   • High-resolution cameras capturing crowd movement (1080p, 30fps)
   • Sensors detecting ambient conditions (lighting, weather, temperature)
   • Edge devices preprocessing video streams (frame extraction, compression)
→ AWS IoT Core integration:
   • MQTT protocol for lightweight sensor data transmission
   • Device shadow for camera configuration management
   • Device certificates for secure authentication
→ Camera metadata:
   • Location ID (station name, platform, geographic coordinates)
   • Hoarding details (size, position, orientation)
   • Operational status (active, maintenance, offline)

Stage 2: Video Stream Ingestion & Frame Extraction
→ AWS Kinesis Video Streams receives continuous camera feeds:
   • Separate stream per camera location (100+ streams)
   • Stream retention: 7 days for reprocessing capability
   • HLS output for video playback and audit review
→ AWS Lambda functions triggered periodically (every 30 seconds):
   • Extract representative frames from video stream
   • Frame selection logic: Motion detection algorithm prioritizing frames with people
   • Image quality validation (blur detection, adequate lighting)
   • Upload selected frames to S3 for AI processing

Stage 3: Computer Vision Processing (AWS Rekognition)
→ AWS Rekognition DetectFaces API analyzes extracted frames:
   • Face detection: Identify all faces in frame with bounding boxes
   • Count people: Total individuals visible in frame
   • Demographic analysis: Age range (7 age groups), gender (Male/Female/Unknown)
   • Emotional analysis: Emotions detected (Happy, Sad, Angry, Confused, Calm, etc.)
→ AWS Rekognition Custom Labels for attention measurement:
   • Custom model trained to detect "looking at hoarding" behavior:
     - Training dataset: 10K+ labeled images (looking vs not-looking)
     - Face orientation analysis (facing hoarding vs other directions)
     - Dwell time estimation using temporal frame sequences
   • Attention confidence score: High (>90%), Medium (70-90%), Low (<70%)
→ Output data structure:
   • Location ID, timestamp, frame reference
   • Total people count, attention count (people looking at hoarding)
   • Demographics: Age distribution, gender distribution
   • Attention metrics: Dwell time estimate, attention confidence scores
```

**FLOW 2: Analytics Pipeline & Metrics Calculation**

```
Stage 1: Real-time Analytics (AWS Kinesis Analytics)
→ Kinesis Analytics application processing vision data stream:
   • Windowed aggregations: 5-minute, 15-minute, 1-hour windows
   • Calculate metrics per location:
     - Average crowd size (people count per frame)
     - Attention rate (% of people looking at hoarding)
     - Demographic composition (age/gender distributions)
     - Peak traffic times (timestamps with highest crowds)
   • Detect anomalies: Sudden crowd surges, equipment failures (no faces detected)
→ Output to AWS Elasticsearch for real-time dashboards

Stage 2: Historical Data Processing (AWS EMR + Scala)
→ Batch processing of historical vision data (nightly jobs):
   • Aggregate metrics by location, date, hour-of-day, day-of-week
   • Calculate attention patterns:
     - Time-of-day attention curves (morning vs evening rush)
     - Day-of-week patterns (weekday vs weekend variations)
     - Seasonal trends (summer vs winter audience behavior)
   • Demographic segmentation analysis:
     - Age group attention rates (teenagers vs professionals vs seniors)
     - Gender-specific engagement patterns
     - Demographic composition by location type (business district vs residential)
   • Location effectiveness scoring:
     - Attention rate benchmarking across locations
     - Demographic targeting effectiveness scores
     - Audience reach potential (crowd size × attention rate)
→ Results stored in SQL Server data marts for Power BI reporting

Stage 3: Time Series Forecasting (AWS Forecast)
→ Predict audience metrics for future time periods:
   • Input features: Historical crowd data, day-of-week, hour-of-day, holidays, weather
   • Forecast algorithms: DeepAR+ for time series prediction
   • Output predictions (7-day forecast, hourly granularity):
     - Expected crowd size with confidence intervals
     - Predicted attention rate
     - Demographic composition forecast
→ Used by pricing algorithm for forward-looking slot recommendations
```

**FLOW 3: Dynamic Pricing Engine**

```
Stage 1: Pricing Model Development
→ .NET pricing algorithm calculates advertising rates based on:
   • Measured attention metrics:
     - Attention count per hour (people looking at hoarding)
     - Attention rate (% of audience engaged)
     - Dwell time (average duration of attention)
   • Peak hour premiums:
     - Morning rush (7-9 AM): 1.4x base rate
     - Evening rush (5-7 PM): 1.5x base rate
     - Off-peak (10 AM-4 PM, 8 PM-11 PM): 1.0x base rate
     - Late night (11 PM-6 AM): 0.6x base rate
   • Demographic targeting effectiveness:
     - Premium for high-value demographics (working professionals 25-45)
     - Discount for broad-audience campaigns (lower targeting requirement)
   • Location attributes:
     - High-traffic stations: 1.3x multiplier
     - Business districts: 1.2x multiplier
     - Residential areas: 0.9x multiplier
   • Seasonal patterns:
     - Holiday season (Nov-Dec): 1.3x base rate
     - Summer months (Jun-Aug): 0.95x base rate (lower commuter traffic)
→ Pricing formula:
   Base Rate × Peak Hour Factor × Demographic Factor × Location Factor × Seasonal Factor

Stage 2: Real-time Pricing Calculation (ASP.NET Web API)
→ RESTful API endpoints for pricing queries:
   • GET /api/pricing/slot/{locationId}/{timeSlot}
     - Returns dynamic price for specific location and time slot
     - Includes expected audience metrics (crowd size, demographics, attention rate)
   • POST /api/pricing/optimize
     - Input: Advertiser constraints (budget, target demographics, date range)
     - Output: Optimal time slot recommendations maximizing ROI
   • GET /api/pricing/forecast/{locationId}
     - 7-day pricing forecast with confidence intervals
→ Pricing cache in Redis (15-minute TTL) with hourly recalculation

Stage 3: Advertiser Notification System (AWS SNS)
→ Alert advertisers about optimal time slots and pricing changes:
   • Email notifications:
     - Optimal time slot availability (matching target demographics)
     - Dynamic price drops (e.g., last-minute inventory clearance)
     - Campaign performance reports (attention metrics, ROI analysis)
   • SMS alerts for time-sensitive opportunities (flash inventory sales)
→ Notification preferences configurable per advertiser account
```

**FLOW 4: Reporting & Dashboard Integration**

```
Stage 1: SQL Server Data Marts
→ Aggregated tables for reporting:
   • AdvertisingMetrics: Location, date, hour, crowd size, attention count, attention rate, demographics
   • PricingHistory: Historical pricing by location and time slot
   • CampaignPerformance: Advertiser campaign metrics, ROI, attention achieved
   • LocationProfiles: Location characteristics, average metrics, effectiveness scores
→ Indexed on location_id, date, hour for fast query performance

Stage 2: Power BI Dashboards
→ Real-time advertising effectiveness dashboard:
   • Location performance map: Geographic heat map of attention rates
   • Time-based analytics: Attention patterns by hour-of-day, day-of-week
   • Demographic insights: Age/gender composition across locations
   • Campaign ROI tracking: Advertiser spend vs attention metrics achieved
→ Scheduled refresh: Every 1 hour for near-real-time visibility

Stage 3: Advertiser Self-Service Portal (React.js)
→ React dashboard for advertisers:
   • Campaign performance: Real-time attention metrics, audience demographics reached
   • Pricing explorer: Interactive tool to query pricing by location and time
   • Booking interface: Reserve time slots with dynamic pricing display
   • Historical analytics: Past campaign performance and ROI analysis
→ SignalR integration for live metric updates during active campaigns
```

#### Solution Architecture

**Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│ CAMERA & IoT LAYER                                          │
│ - Digital Hoardings with Cameras (100+ locations)         │
│ - AWS IoT Core (MQTT, device management, certificates)    │
│ - Edge Devices (Frame extraction, compression)            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ VIDEO PROCESSING LAYER                                      │
│ - AWS Kinesis Video Streams (Continuous video feeds)      │
│ - AWS Lambda (Frame extraction, quality validation)       │
│ - AWS S3 (Frame storage for AI processing)                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ AI & COMPUTER VISION LAYER                                  │
│ - AWS Rekognition (Face detection, demographics, emotions)│
│ - AWS Rekognition Custom Labels (Attention measurement)   │
│ - Kinesis Data Streams (Vision analytics events)          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ANALYTICS & PROCESSING LAYER                                │
│ - AWS Kinesis Analytics (Real-time aggregations)          │
│ - AWS EMR + Scala (Batch processing, historical trends)   │
│ - AWS Forecast (Time series prediction)                   │
│ - Elasticsearch (Real-time query layer)                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ PRICING & API LAYER                                         │
│ - .NET Dynamic Pricing Engine (Rate calculation)          │
│ - ASP.NET Web API (Pricing queries, optimization)         │
│ - Redis Cache (Pricing cache, 15-min TTL)                 │
│ - AWS SNS (Advertiser notifications)                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ PRESENTATION LAYER                                          │
│ - SQL Server (Data marts for reporting)                   │
│ - Power BI (Executive dashboards, location analytics)     │
│ - React.js (Advertiser self-service portal)               │
│ - SignalR (Real-time campaign metric updates)             │
└─────────────────────────────────────────────────────────────┘
```

**Key Components:**

1. **IoT Integration Layer (AWS IoT Core + Edge Devices)**

   - MQTT protocol for lightweight camera data transmission
   - Device shadow for remote camera configuration
   - X.509 certificates for device authentication
   - Edge preprocessing reducing bandwidth (local frame extraction)

2. **Video Stream Management (AWS Kinesis Video Streams + Lambda)**

   - 100+ concurrent video streams (one per camera location)
   - Lambda-based frame extraction (every 30 seconds)
   - Motion detection prioritizing frames with people
   - S3 storage with lifecycle policy (7 days retention)

3. **Computer Vision Processing (AWS Rekognition + Custom Labels)**

   - DetectFaces API for crowd counting and demographics
   - Custom Labels model for attention measurement (looking vs not-looking)
   - Emotion analysis and face orientation detection
   - Batch processing via Lambda (async invocation)

4. **Real-time Analytics (AWS Kinesis Analytics)**

   - SQL-based stream processing (windowed aggregations)
   - Metrics calculation: Crowd size, attention rate, demographics
   - Anomaly detection: Equipment failures, crowd surges
   - Output to Elasticsearch for dashboards

5. **Big Data Processing (AWS EMR + Scala Spark)**

   - Historical trend analysis (time-of-day, seasonal patterns)
   - Demographic segmentation (age/gender attention rates)
   - Location effectiveness scoring
   - Parquet storage in S3 for efficient queries

6. **Time Series Forecasting (AWS Forecast)**

   - DeepAR+ algorithm for 7-day hourly forecasts
   - Features: Historical crowd data, day-of-week, holidays, weather
   - Confidence intervals for pricing algorithm

7. **Dynamic Pricing Engine (.NET Framework + ASP.NET Web API)**

   - Real-time rate calculation based on attention metrics
   - Peak hour premiums and demographic targeting factors
   - Optimization API for advertiser ROI maximization
   - Redis caching (15-min TTL) for performance

8. **Visualization & Reporting (Power BI + React.js + SignalR)**
   - Power BI dashboards for operations and sales teams
   - React.js advertiser portal with real-time campaign metrics
   - SignalR for live attention updates during campaigns
   - SQL Server data marts with indexed queries

#### Technical Implementation Details

**Technology Stack & Rationale:**

| Technology                      | Purpose                     | Why Chosen                                                                                |
| ------------------------------- | --------------------------- | ----------------------------------------------------------------------------------------- |
| AWS IoT Core                    | Camera/sensor integration   | MQTT lightweight protocol, device shadow for remote management, AWS ecosystem integration |
| AWS Kinesis Video Streams       | Video ingestion             | Purpose-built for video streaming, 7-day retention, HLS output for playback               |
| AWS Lambda (.NET Core)          | Frame extraction            | Event-driven processing, auto-scaling for 100+ camera streams, cost-effective             |
| AWS Rekognition                 | Face detection/demographics | Pre-trained models (>99% accuracy), demographic analysis, emotion detection               |
| AWS Rekognition Custom Labels   | Attention measurement       | Custom model training for domain-specific detection (looking at hoarding)                 |
| AWS Kinesis Analytics           | Real-time stream analytics  | SQL-based processing, windowed aggregations, low latency (<1 minute)                      |
| AWS EMR + Scala Spark           | Batch processing            | Distributed processing for historical data, Scala performance, Parquet integration        |
| AWS Forecast                    | Time series forecasting     | Managed ML service, DeepAR+ algorithm, handles seasonality and holidays                   |
| Elasticsearch                   | Real-time query layer       | Sub-second queries, multi-dimensional aggregations, real-time dashboards                  |
| SQL Server                      | Data marts                  | ACID compliance, Power BI native connector, complex analytical queries                    |
| ASP.NET Web API + .NET 4.5      | Pricing API                 | High performance, synchronous/asynchronous patterns, excellent AWS SDK integration        |
| Redis Cache                     | Pricing cache               | Sub-millisecond latency, reduces compute load, distributed caching                        |
| React.js + TypeScript + SignalR | Advertiser portal           | Real-time updates, component reusability, WebSocket for live metrics                      |
| Power BI                        | Executive dashboards        | Rich visualizations, SQL Server integration, scheduled refresh                            |
| AWS SNS                         | Notifications               | Multi-channel (email/SMS), topic subscriptions, delivery tracking                         |
| AWS S3                          | Frame storage               | Unlimited scalability, lifecycle policies, integration with Rekognition                   |

**Key Design Patterns Applied:**

1. **Edge Computing Pattern**

   - Local frame extraction reducing bandwidth
   - Edge preprocessing before cloud processing
   - Reduces latency and cloud compute costs

2. **Event-Driven Architecture**

   - Kinesis Video Streams trigger Lambda frame extraction
   - Rekognition results trigger analytics pipeline
   - SNS notifications triggered by pricing events

3. **Lambda Architecture**

   - Batch layer: EMR processing historical data
   - Speed layer: Kinesis Analytics for real-time metrics
   - Serving layer: Elasticsearch + SQL Server

4. **Cache-Aside Pattern**

   - Redis cache for pricing queries
   - 15-minute TTL with hourly recalculation
   - Cache invalidation on pricing model updates

5. **Time Series Prediction Pattern**
   - AWS Forecast for demand prediction
   - Features combining historical data + external factors
   - Confidence intervals for risk assessment

#### My Technical Contributions

**1. AWS IoT Core Integration & Video Stream Management**

- Designed IoT architecture connecting 100+ camera locations across London:
  - MQTT protocol configuration with topic hierarchy (location/device/telemetry)
  - Device shadow implementation for remote camera configuration (resolution, frame rate, operation schedule)
  - X.509 certificate provisioning and rotation for secure authentication
- Implemented AWS Kinesis Video Streams ingestion pipeline:
  - Separate stream per camera location with automatic stream creation
  - Stream retention policy: 7 days for reprocessing, HLS output for audit playback
  - Lambda-triggered frame extraction every 30 seconds with motion detection algorithm
- Built frame quality validation: Blur detection using Laplacian variance, adequate lighting checks, orientation validation
- **Impact**: 100+ camera feeds processing 24/7, 99.2% uptime, 85% bandwidth reduction via edge preprocessing

**2. Computer Vision with AWS Rekognition Custom Labels**

- Trained custom attention measurement model using AWS Rekognition Custom Labels:
  - Dataset preparation: 10K+ labeled images (people looking at hoarding vs other directions)
  - Annotation: Bounding boxes + binary classification (attention: yes/no)
  - Model training: AutoML feature with hyperparameter tuning
  - Model evaluation: Precision 91%, Recall 88%, F1-score 89.5%
- Integrated Rekognition APIs into Lambda processing pipeline:
  - DetectFaces API: Face count, demographics (age range, gender), emotion analysis
  - Custom Labels inference: Attention detection with confidence scores
  - Batch processing optimization: Process up to 10 frames per Lambda invocation
- Built face tracking algorithm using temporal frame sequences estimating dwell time (3+ consecutive frames = engaged)
- **Impact**: Attention measurement accuracy 89.5%, processing latency <5 seconds per frame, demographic insights for 95% of detected faces

**3. Real-time Analytics with AWS Kinesis Analytics**

- Developed Kinesis Analytics SQL application for stream processing:
  - Windowed aggregations: Tumbling windows (5-min), sliding windows (15-min, 1-hour)
  - Calculated metrics: Average crowd size, attention rate (%), demographic distributions
  - Anomaly detection: Alerts when no faces detected for >10 minutes (equipment failure)
- Implemented Elasticsearch integration for real-time dashboards:
  - Kinesis Analytics output to Kinesis Data Firehose → Elasticsearch
  - Index design: Daily indices with date-based aliases
  - Mapping optimization: Keyword fields for location/demographics, numeric fields for metrics
- Built alerting logic: SNS notifications for crowd surge events (>3x average) or equipment failures
- **Impact**: Real-time metrics latency <1 minute, 99.8% data processing accuracy, automated alerting reducing equipment downtime 65%

**4. Historical Trend Analysis with Scala on AWS EMR**

- Developed Scala Spark applications for batch processing of historical vision data:
  - Read Parquet files from S3 (partition pruning by date, predicate pushdown)
  - Aggregation pipelines: Metrics by location × date × hour × day-of-week
  - Time-based pattern extraction:
    - Morning rush (7-9 AM) vs evening rush (5-7 PM) attention rate differences
    - Weekday vs weekend crowd size patterns
    - Seasonal variations (summer lower commuter traffic)
  - Demographic segmentation analysis:
    - Age group attention rates (Gen Z 18-24 vs Millennials 25-40 vs Gen X 41-56)
    - Gender-specific engagement patterns by location type (business vs residential)
    - Location effectiveness scoring: Attention rate × crowd size = audience reach score
- Optimized Spark jobs: Broadcast joins for location metadata, repartitioning strategies, intermediate result caching
- **Impact**: Processing 200GB+ historical data in <1.5 hours, identified 35% attention rate variation by time-of-day, location effectiveness benchmarking enabling data-driven site selection

**5. Dynamic Pricing Algorithm Implementation**

- Built .NET-based dynamic pricing engine calculating advertising rates:
  - Base pricing formula: Attention Count × Base Rate × Peak Hour Factor × Demographic Factor × Location Factor × Seasonal Factor
  - Peak hour premiums: Morning rush 1.4x, evening rush 1.5x, off-peak 1.0x, late night 0.6x
  - Demographic targeting effectiveness: Working professionals (25-45) premium 1.3x, broad audience 1.0x
  - Location multipliers: High-traffic stations 1.3x, business districts 1.2x, residential 0.9x
  - Seasonal adjustments: Holiday season 1.3x, summer months 0.95x
- Integrated AWS Forecast predictions for forward-looking pricing:
  - 7-day hourly forecast of crowd size and attention rate
  - Confidence intervals (P10, P50, P90) for risk assessment
  - Pricing adjustments based on predicted demand
- Implemented ASP.NET Web API for pricing queries:
  - GET /api/pricing/slot: Real-time pricing with expected audience metrics
  - POST /api/pricing/optimize: ROI maximization algorithm for advertiser constraints
  - Redis caching with 15-minute TTL, hourly recalculation
- **Impact**: 22-28% revenue increase vs fixed pricing, 88% advertiser satisfaction (data-driven pricing), pricing API P95 response time <250ms

**6. Advertiser Portal & Real-time Reporting**

- Developed React.js advertiser self-service portal with TypeScript:
  - Campaign dashboard: Real-time attention metrics, demographic reach, ROI tracking
  - Pricing explorer: Interactive location/time slot pricing query tool
  - Booking interface: Reserve time slots with dynamic pricing display, payment integration
  - Historical analytics: Past campaign performance, A/B test results
- Implemented SignalR WebSocket integration for live metric updates:
  - Server-side hub broadcasting attention metrics every 30 seconds during active campaigns
  - Client-side connection management with automatic reconnection logic
  - Real-time charts updating attention count, demographic distribution
- Built Power BI dashboards for operations and sales teams:
  - Location performance map: Geographic heat map of attention rates across London
  - Time-based analytics: Attention patterns by hour/day with historical comparison
  - Campaign ROI dashboard: Advertiser spend vs audience metrics achieved
- **Impact**: 95% advertiser adoption of self-service portal, 50% reduction in support queries, real-time visibility increasing campaign optimization by 30%

#### NFRs & Technical Achievements

**Performance:**

- Frame processing: 100+ cameras, 30-second intervals, <5 seconds latency
- Rekognition inference: P95 <3 seconds for face detection + custom model
- Real-time analytics: Kinesis Analytics processing latency <1 minute
- Pricing API: P95 response time <250ms, throughput 500 req/min
- Elasticsearch queries: P95 <300ms for multi-dimensional aggregations

**Scalability:**

- Video streams: 100+ concurrent Kinesis Video Streams (24/7 operation)
- Lambda auto-scaling: 1000+ concurrent invocations for frame processing
- EMR cluster: Auto-scaling 5-15 nodes based on batch job workload
- Rekognition: Asynchronous batch processing handling traffic spikes
- Redis cache: Distributed across 3 nodes with automatic failover

**Business Impact:**

- 22-28% revenue increase vs fixed pricing model
- 89.5% attention measurement accuracy (F1-score)
- 88% advertiser satisfaction with data-driven pricing transparency
- 30% campaign optimization improvement via real-time metrics
- 65% equipment downtime reduction with automated failure alerts
- 95% advertiser portal adoption rate
- 50% reduction in sales support queries (self-service booking)
- System uptime: 99.2% availability across 100+ camera locations

---

### SYSRTEM 3: ASSET MANAGEMENT WITH SERVICENOW

## TECHNOLOGY STACK SUMMARY & SKILLS TO BRUSH UP

### ✅ **AWS Cloud Services (Strong - Maintain Proficiency)**

**Core AWS Services:**

- ✓ AWS Kinesis Data Streams (real-time data ingestion, millions records/day)
- ✓ AWS Kinesis Video Streams (100+ concurrent video feeds)
- ✓ AWS Kinesis Analytics (SQL-based stream processing, windowed aggregations)
- ✓ AWS Lambda (.NET Core + Python, event-driven processing, auto-scaling)
- ✓ AWS S3 (data lake storage, Parquet format, lifecycle policies)
- ✓ AWS EMR (Elastic MapReduce with Scala Spark, TB-scale processing)
- ✓ AWS SageMaker (ML model training/hosting, XGBoost, Prophet, auto-scaling endpoints)
- ✓ AWS Rekognition (face detection, demographics, emotions, Custom Labels)
- ✓ AWS IoT Core (MQTT, device shadow, X.509 certificates)
- ✓ AWS Elasticsearch Service (real-time indexing, multi-dimensional queries)
- ✓ AWS SNS (multi-channel notifications, topic subscriptions)
- ✓ AWS Forecast (time series prediction with DeepAR+)

**Recommended Practice:**

- Build Kinesis Data Streams pipeline with Lambda processing (.NET Core)
- Train custom Rekognition model using Custom Labels (image classification)
- Deploy SageMaker model with real-time inference endpoint
- Create Kinesis Analytics SQL application with windowed aggregations
- Set up AWS IoT Core with MQTT and device shadow

---

### ✅ **Big Data & Analytics (Strong - Maintain)**

**Technologies:**

- ✓ Scala Spark on AWS EMR (distributed processing, aggregations, feature engineering)
- ✓ Parquet columnar format (efficient storage, predicate pushdown, partition pruning)
- ✓ AWS Elasticsearch (real-time indexing, aggregation queries, full-text search)
- ✓ SQL Server (data marts, star schema, Power BI integration)
- ✓ Redis Cache (distributed caching, pub/sub, sub-millisecond latency)

**Recommended Practice:**

- Write Scala Spark application processing Parquet files from S3
- Build Elasticsearch index schema with aggregation queries
- Design star schema for analytics (fact/dimension tables)
- Implement Redis caching strategy (cache-aside pattern, TTL, invalidation)

---

### ✅ **.NET Development (Strong - Maintain)**

**Core Technologies:**

- ✓ ASP.NET Web API (.NET Framework 4.5 + .NET Core)
- ✓ C# (async/await patterns, LINQ, Entity Framework)
- ✓ AWS SDK for .NET (Kinesis, S3, SageMaker, Rekognition, IoT Core)
- ✓ Redis integration (.NET client, caching patterns)

**Recommended Practice:**

- Build ASP.NET Web API with AWS service integrations
- Implement async/await patterns for non-blocking I/O
- Use Entity Framework with query optimization (Select, Include, caching)
- Integrate Redis caching with cache-aside pattern

---

### 🔄 **Machine Learning & Computer Vision (Good - Refresh)**

**AWS ML Services:**

- ✓ AWS SageMaker (model training, hyperparameter tuning, inference endpoints)
- ✓ AWS Rekognition Custom Labels (custom model training, image classification)
- ✓ AWS Forecast (time series prediction, DeepAR+ algorithm)

**Recommended Refresh:**

- Train XGBoost model on SageMaker with hyperparameter tuning
- Build custom Rekognition model for domain-specific object detection
- Create AWS Forecast predictor with external features (holidays, weather)

**Time Investment:** 2-3 days

- Day 1: SageMaker training pipeline (data prep, training, deployment)
- Day 2: Rekognition Custom Labels (dataset annotation, training, inference)
- Day 3: AWS Forecast setup and evaluation

---

### ✅ **Frontend & Visualization (Strong - Maintain)**

**Technologies:**

- ✓ React.js + TypeScript (component architecture, hooks, state management)
- ✓ SignalR (WebSocket real-time updates, hub connections)
- ✓ Power BI (dashboards, DAX, SQL Server integration, scheduled refresh)

**Recommended Practice:**

- Build React.js dashboard with TypeScript and real-time SignalR updates
- Create Power BI dashboard with DAX measures and time intelligence
- Implement WebSocket connection management with reconnection logic

---

### ⚠️ **Gaps to Address (High Priority)**

**Testing & Quality:**

- ⚠️ **pytest** (Python unit testing) - Needed for Python Lambda functions
- ⚠️ **xUnit** (C# unit testing) - Needed for .NET Web API testing
- ⚠️ **Mocking frameworks** (Moq for C#, unittest.mock for Python)

**Recommended Learning:**

- Write xUnit tests for ASP.NET Web API controllers (mock dependencies)
- Write pytest tests for Python Lambda functions (mock AWS services)
- Achieve 80%+ code coverage

**Time Investment:** 2-3 days

- Day 1: xUnit basics, test fixtures, mock AWS SDK calls
- Day 2: pytest for Lambda functions, mocking boto3 AWS SDK
- Day 3: Code coverage analysis, CI/CD integration

---

**DevOps & CI/CD:**

- 🔄 **AWS CodePipeline** (CI/CD for Lambda + API deployments)
- 🔄 **Jenkins** (mentioned but not detailed in documentation)
- ⚠️ **Infrastructure as Code** (CloudFormation, Terraform) - NOT in documentation

**Recommended Learning:**

- Create AWS CodePipeline for Lambda deployment (source, build, deploy)
- Write CloudFormation templates for infrastructure provisioning
- Set up Jenkins pipeline for .NET API builds

**Time Investment:** 3-4 days

- Day 1-2: AWS CodePipeline for Lambda + API Gateway
- Day 3-4: CloudFormation templates for AWS resources

---

### 📋 **Priority Action Plan (Next 7-10 Days)**

**High Priority (4-5 days):**

1. ⚠️ **xUnit + Moq** (C# API testing) - 2 days
2. ⚠️ **pytest + mocking** (Python Lambda testing) - 2 days
3. 🔄 **AWS CodePipeline** (CI/CD setup) - 1 day

**Medium Priority (3-4 days):** 4. 🔄 **SageMaker refresh** (ML model deployment) - 1 day 5. 🔄 **Rekognition Custom Labels** (custom model training) - 1 day 6. 🔄 **CloudFormation basics** (Infrastructure as Code) - 2 days

**Lower Priority (if time permits):** 7. 🔄 **Scala Spark** (refresh syntax, practice EMR jobs) - 1-2 days

**Total Time Investment:** 7-10 days to close critical gaps

---

### 📊 **Skills Readiness Assessment**

| Category                  | Current Level | Target Level | Gap              | Days to Close |
| ------------------------- | ------------- | ------------ | ---------------- | ------------- |
| AWS Core Services         | 90%           | 95%          | CodePipeline     | 1 day         |
| AWS ML Services           | 80%           | 90%          | Refresh practice | 2 days        |
| .NET/C# Development       | 90%           | 95%          | xUnit testing    | 2 days        |
| Python Development        | 85%           | 90%          | pytest testing   | 2 days        |
| Scala Spark               | 75%           | 85%          | Syntax refresh   | 2 days        |
| React/TypeScript/SignalR  | 85%           | 90%          | Maintain         | Maintain      |
| Power BI                  | 85%           | 90%          | Advanced DAX     | 1 day         |
| Testing (xUnit/pytest)    | 40%           | 85%          | Unit testing     | 4 days        |
| DevOps (CodePipeline/IaC) | 50%           | 80%          | CI/CD, IaC       | 3 days        |
| **Overall Readiness**     | **80%**       | **90%**      | **Total**        | **7-10 days** |

---

### 🎯 **Interview Preparation Focus**

**Be Ready to Discuss:**

1. Real-time data ingestion with AWS Kinesis Data Streams (millions records/day)
2. Big data processing with Scala Spark on AWS EMR (TB-scale, Parquet optimization)
3. ML model training and deployment with AWS SageMaker (XGBoost, Prophet, auto-scaling)
4. Computer vision with AWS Rekognition Custom Labels (attention measurement, 89.5% accuracy)
5. Dynamic pricing algorithm (.NET, Redis caching, API optimization)
6. Real-time analytics with AWS Kinesis Analytics (SQL-based stream processing)
7. IoT integration with AWS IoT Core (MQTT, device shadow, 100+ cameras)
8. React.js + SignalR for real-time dashboards (WebSocket, live metric updates)

**Practice Coding Questions:**

- Implement Lambda function processing Kinesis stream (.NET Core or Python)
- Write Scala Spark application aggregating Parquet files from S3
- Build ASP.NET Web API with Redis caching (cache-aside pattern)
- Design pricing algorithm with multiple factors (peak hours, demographics, location)
- Implement real-time dashboard with React.js + SignalR WebSocket updates

---
