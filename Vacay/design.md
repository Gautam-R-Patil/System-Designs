# Vacay – Autonomous AI Concierge & Execution Platform
## System Design Document

## 1. Architecture Overview

Vacay is built as a serverless, event-driven system on AWS, leveraging managed AI services for LLM and vision capabilities. The architecture follows a multi-agent pattern with a Manager LLM orchestrating specialized tool agents for booking, communication, and payment processing.

**Core Principles**:
- Serverless-first for cost efficiency and scalability
- Event-driven architecture for loose coupling
- Managed AI services for rapid development
- Security and privacy by design
- Graceful degradation with human escalation

## 2. High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface Layer                     │
│  ┌──────────────────┐         ┌─────────────────────────────┐  │
│  │ WhatsApp Business│◄────────┤   React Native App (Future) │  │
│  │      API         │         └─────────────────────────────┘  │
│  └────────┬─────────┘                                           │
└───────────┼─────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Amazon API Gateway (REST + WebSocket)                   │  │
│  │  - Authentication (Cognito)                              │  │
│  │  - Rate limiting                                         │  │
│  │  - Request validation                                    │  │
│  └────────┬─────────────────────────────────────────────────┘  │
└───────────┼─────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Orchestration Layer                           │
│  ┌──────────────────┐         ┌─────────────────────────────┐  │
│  │  AWS Step        │         │   n8n Workflow Engine       │  │
│  │  Functions       │◄────────┤   (Voice, External APIs)    │  │
│  └────────┬─────────┘         └─────────────────────────────┘  │
└───────────┼─────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Agent Core Layer                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Manager LLM (Amazon Bedrock)                │  │
│  │  - Claude 3.5 Sonnet for reasoning                       │  │
│  │  - Tool calling & decision orchestration                 │  │
│  │  - Confidence scoring                                    │  │
│  └────────┬─────────────────────────────────────────────────┘  │
│           │                                                      │
│  ┌────────┴──────────┬──────────────┬──────────────┬─────────┐ │
│  ▼                   ▼              ▼              ▼         ▼ │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───┐│
│ │ Vision   │  │ Booking  │  │  Wallet  │  │  Voice   │  │...││
│ │ Agent    │  │ Agent    │  │  Agent   │  │  Agent   │  │   ││
│ └──────────┘  └──────────┘  └──────────┘  └──────────┘  └───┘│
└─────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Data & State Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  DynamoDB    │  │  PostgreSQL  │  │  S3 (Temp Storage)   │ │
│  │  (State)     │  │  (Relational)│  │  (Photos, Logs)      │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   External Integration Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Booking     │  │  Payment     │  │  Voice/TTS/STT       │ │
│  │  APIs        │  │  Gateway     │  │  Services            │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## 3. Component Breakdown

### 3.1 User Interface Components

**WhatsApp Business API**
- Primary user interaction channel
- Handles photo uploads, text messages, quick replies
- Webhook integration with API Gateway
- Message templates for notifications

**React Native App (Future)**
- Rich UI for itinerary visualization
- Photo upload and management
- Wallet management
- Trip tracking dashboard

### 3.2 Backend Services

**API Gateway Service** (AWS Lambda + API Gateway)
- Handles incoming WhatsApp webhooks
- Routes requests to appropriate services
- Authentication via AWS Cognito
- Rate limiting and request validation

**Session Manager Service** (AWS Lambda + DynamoDB)
- Manages user conversation state
- Tracks multi-turn interactions
- Stores context for agent decisions
- Session timeout: 30 minutes

**Photo Processing Service** (AWS Lambda + S3)
- Receives photo uploads
- Stores temporarily in S3 (24-hour lifecycle)
- Triggers vision model inference
- Aggregates preference data

### 3.3 AI/ML Components

**Vision Model Service** (Amazon SageMaker)
- Hosted vision model (e.g., CLIP, custom fine-tuned model)
- Extracts visual preferences from photos
- Outputs structured JSON with confidence scores
- Batch processing for efficiency

**Manager LLM Service** (Amazon Bedrock)
- Claude 3.5 Sonnet for high-level reasoning
- Generates itineraries based on preferences
- Orchestrates tool agents
- Evaluates booking options
- Determines escalation needs

**Voice Services** (Amazon Polly + Transcribe)
- Text-to-speech for outbound calls
- Speech-to-text for vendor responses
- Multi-language support
- Integration with n8n for call management

### 3.4 Tool Agents

**Booking Agent** (AWS Lambda)
- Integrates with hotel/activity booking APIs
- Handles API authentication and retries
- Validates booking parameters
- Stores booking confirmations

**Wallet Agent** (AWS Lambda + PostgreSQL)
- Manages user wallet balance
- Processes transactions
- Handles holds and releases
- Generates transaction receipts

**Location Agent** (AWS Lambda)
- Tracks user location (with consent)
- Provides navigation assistance
- Monitors trip progress
- Triggers location-based notifications

**Escalation Agent** (AWS Lambda + SNS)
- Monitors agent confidence scores
- Triggers human expert notifications
- Manages escalation queue
- Tracks resolution status

### 3.5 Data Storage

**DynamoDB Tables**
- `user-sessions`: Conversation state and context
- `agent-state`: Current agent execution state
- `trip-state`: Real-time trip orchestration data
- `escalation-queue`: Pending human interventions

**PostgreSQL Database**
- `users`: User profiles and preferences
- `trips`: Trip details and itineraries
- `bookings`: Booking records and confirmations
- `wallet_transactions`: Payment history
- `audit_logs`: Decision and action logs

**S3 Buckets**
- `vacay-photos-temp`: Temporary photo storage (24h lifecycle)
- `vacay-logs`: Application and agent logs
- `vacay-backups`: Database backups

## 4. AI Agent Architecture

### 4.1 Manager LLM Design

**Role**: High-level reasoning and orchestration

**Capabilities**:
- Analyzes user preferences and constraints
- Generates structured itineraries
- Decides which tools to invoke and when
- Evaluates multiple booking options
- Assesses confidence in decisions
- Determines when to escalate

**Tool Calling Interface**:
```json
{
  "tool": "book_hotel",
  "parameters": {
    "location": "Goa, India",
    "check_in": "2026-03-15",
    "check_out": "2026-03-18",
    "preferences": {
      "style": "boutique",
      "budget": "mid-range",
      "amenities": ["pool", "beach-access"]
    }
  },
  "confidence": 0.92,
  "reasoning": "User's photos show preference for boutique beach properties"
}
```

**Decision Loop**:
1. Receive user input or event
2. Retrieve context from state store
3. Reason about next action
4. Generate tool call with confidence score
5. Execute tool via orchestration layer
6. Evaluate result
7. Update state
8. Repeat or respond to user

### 4.2 Confidence Scoring

**Confidence Thresholds**:
- High confidence (>= 0.85): Proceed autonomously
- Medium confidence (0.70 - 0.84): Proceed with logging
- Low confidence (< 0.70): Escalate to human

**Factors Affecting Confidence**:
- Clarity of user preferences
- Availability of booking options
- API response quality
- Historical success rate
- Complexity of request

### 4.3 Tool Agent Pattern

Each tool agent follows a standard interface:

```python
class ToolAgent:
    def execute(self, parameters: dict) -> ToolResult:
        """Execute tool action"""
        pass
    
    def validate(self, parameters: dict) -> bool:
        """Validate parameters before execution"""
        pass
    
    def rollback(self, execution_id: str) -> bool:
        """Rollback action if needed"""
        pass
```

## 5. Vision Model Flow

### 5.1 Photo Ingestion Pipeline

```
User uploads photos via WhatsApp
    ↓
API Gateway receives webhook
    ↓
Photos stored in S3 (temp bucket)
    ↓
Lambda triggers SageMaker inference
    ↓
Vision model processes batch
    ↓
Extracts features and preferences
    ↓
Aggregates into taste profile
    ↓
Stores in PostgreSQL
    ↓
Notifies user via WhatsApp
```

### 5.2 Preference Extraction

**Visual Features Detected**:
- Architecture style (modern, traditional, rustic)
- Environment (beach, mountains, urban, rural)
- Activity types (adventure, relaxation, cultural)
- Accommodation style (luxury, boutique, budget)
- Dining preferences (fine dining, casual, street food)
- Color palettes and aesthetic preferences

**Output Format**:
```json
{
  "user_id": "user_123",
  "taste_profile": {
    "accommodation": {
      "style": "boutique",
      "confidence": 0.89,
      "examples": ["beach_resort", "heritage_hotel"]
    },
    "activities": {
      "types": ["water_sports", "cultural_tours"],
      "confidence": 0.85
    },
    "environment": {
      "preference": "coastal",
      "confidence": 0.92
    },
    "budget_indicator": "mid_to_high",
    "aesthetic": {
      "colors": ["blue", "white", "earth_tones"],
      "style": "minimalist_modern"
    }
  },
  "created_at": "2026-02-15T10:30:00Z"
}
```

## 6. Manager LLM Decision Engine

### 6.1 Itinerary Generation Process

**Input**:
- Taste profile from vision model
- User constraints (dates, budget, destination)
- Real-time availability data

**Process**:
1. Analyze taste profile and constraints
2. Query knowledge base for destination info
3. Generate day-by-day structure
4. Select accommodations matching preferences
5. Choose activities aligned with interests
6. Plan dining experiences
7. Optimize timing and logistics
8. Generate 2-3 alternatives for key decisions

**Output**: Structured itinerary JSON

### 6.2 Booking Decision Logic

**Decision Factors**:
- Preference match score
- Price vs budget
- Availability
- Reviews and ratings
- Location convenience
- Cancellation policy

**Decision Tree**:
```
Is preference match >= 0.85?
  Yes → Is price within budget?
    Yes → Is availability confirmed?
      Yes → Book (confidence: high)
      No → Find alternative
    No → Find cheaper alternative
  No → Find better match or escalate
```

### 6.3 Escalation Logic

**Escalation Triggers**:
- Confidence score < 0.70
- Booking API failures (3+ retries)
- Voice call failures (2+ attempts)
- Ambiguous user requests
- Budget conflicts
- Safety concerns

**Escalation Process**:
1. Log escalation reason and context
2. Notify human expert via SNS
3. Add to escalation queue (DynamoDB)
4. Notify user of handoff
5. Pause autonomous actions
6. Wait for expert resolution

## 7. Tool Layer Architecture

### 7.1 Wallet Tool

**Operations**:
- `check_balance`: Query current balance
- `hold_funds`: Reserve funds for pending booking
- `charge`: Deduct funds for confirmed booking
- `release_hold`: Release reserved funds
- `refund`: Return funds for cancellation

**Implementation**:
- PostgreSQL for transaction records
- ACID compliance for consistency
- Idempotency keys for safety
- Audit logging for all operations

### 7.2 Booking API Tool

**Supported APIs** (MVP):
- Hotel booking: Booking.com API, Agoda API
- Activities: GetYourGuide API, Viator API
- Transport: Uber API (for local transport)

**Retry Strategy**:
- Exponential backoff: 1s, 2s, 4s
- Max retries: 3
- Fallback to alternative vendors
- Escalate after exhaustion

### 7.3 Voice Call Tool (via n8n)

**n8n Workflow**:
1. Receive call request from Manager LLM
2. Generate TTS audio (Amazon Polly)
3. Initiate call via Twilio/Exotel
4. Stream audio to vendor
5. Capture vendor response
6. Transcribe via Amazon Transcribe
7. Return transcript to Manager LLM

**Call Script Generation**:
- Manager LLM generates call script
- Includes booking details and questions
- Handles common responses
- Escalates on confusion

## 8. Human Escalation Architecture

### 8.1 Expert Dashboard

**Components**:
- Queue view: List of escalated cases
- Case detail: Full context and history
- Action panel: Take over agent actions
- Communication: Direct chat with user
- Resolution tracking: Mark as resolved

**Technology**:
- React web app
- Real-time updates via WebSocket
- Hosted on AWS Amplify

### 8.2 Handoff Protocol

**Agent → Human**:
1. Agent detects escalation condition
2. Generates summary with context
3. Adds to escalation queue
4. Sends SNS notification to on-call expert
5. Notifies user via WhatsApp
6. Pauses autonomous actions

**Human → Agent**:
1. Expert resolves issue
2. Updates state with resolution
3. Marks case as resolved
4. Agent resumes if appropriate
5. Logs resolution for learning

## 9. Data Flow

### 9.1 Real-Time Data Flow

**User Message → Response**:
```
WhatsApp → API Gateway → Lambda → DynamoDB (state)
    ↓
Manager LLM (Bedrock) → Tool Agent → External API
    ↓
DynamoDB (update state) → Lambda → API Gateway → WhatsApp
```

**Latency Budget**:
- API Gateway: 50ms
- Lambda cold start: 500ms (warm: 10ms)
- DynamoDB read/write: 10ms
- Bedrock inference: 2000ms
- Tool execution: 1000ms
- Total: ~3.5s (target: < 5s)

### 9.2 Batch Data Flow

**Photo Processing**:
```
S3 upload → EventBridge → Lambda → SageMaker (batch)
    ↓
Aggregate results → PostgreSQL → Notify user
```

**Processing Time**: < 10 seconds for 50 photos

## 10. AWS Infrastructure Design

### 10.1 Compute Resources

**AWS Lambda Functions**:
- `api-handler`: API Gateway webhook handler (512MB, 30s timeout)
- `session-manager`: Session state management (256MB, 10s)
- `photo-processor`: Photo upload handler (1GB, 60s)
- `agent-orchestrator`: Manager LLM orchestration (1GB, 300s)
- `booking-agent`: Booking API integration (512MB, 60s)
- `wallet-agent`: Wallet operations (256MB, 10s)
- `escalation-handler`: Escalation management (256MB, 10s)

**Amazon ECS** (for long-running processes):
- n8n workflow engine (Fargate, 2 vCPU, 4GB RAM)
- Expert dashboard backend (Fargate, 1 vCPU, 2GB RAM)

### 10.2 Database Configuration

**Amazon RDS (PostgreSQL)**:
- Instance: db.t3.medium (2 vCPU, 4GB RAM)
- Storage: 100GB SSD
- Multi-AZ: No (MVP), Yes (production)
- Backup: Daily automated backups

**Amazon DynamoDB**:
- On-demand pricing for MVP
- Tables: user-sessions, agent-state, trip-state, escalation-queue
- Point-in-time recovery enabled
- Global secondary indexes for queries

### 10.3 AI/ML Infrastructure

**Amazon Bedrock**:
- Model: Claude 3.5 Sonnet
- On-demand pricing
- No infrastructure management

**Amazon SageMaker**:
- Endpoint: ml.g4dn.xlarge (1 GPU)
- Auto-scaling: 1-3 instances
- Model: Custom vision model or CLIP

**Amazon Polly & Transcribe**:
- Pay-per-use
- No infrastructure management

### 10.4 Networking

**VPC Configuration**:
- Private subnets for Lambda, RDS, ECS
- Public subnets for NAT Gateway, ALB
- Security groups for service isolation

**API Gateway**:
- REST API for WhatsApp webhooks
- WebSocket API for real-time updates
- Custom domain with SSL

## 11. Security & Isolation

### 11.1 Authentication & Authorization

**User Authentication**:
- WhatsApp phone number as primary identifier
- AWS Cognito for session management
- JWT tokens for API access

**Service Authentication**:
- IAM roles for Lambda functions
- Secrets Manager for API keys
- KMS for encryption keys

### 11.2 Data Encryption

**At Rest**:
- S3: Server-side encryption (SSE-S3)
- RDS: Encryption enabled
- DynamoDB: Encryption enabled
- Secrets Manager: KMS encryption

**In Transit**:
- TLS 1.2+ for all API calls
- HTTPS only
- VPC endpoints for AWS services

### 11.3 Privacy Controls

**Photo Data**:
- 24-hour lifecycle policy on S3
- Automatic deletion after processing
- No long-term storage of raw images

**PII Protection**:
- Minimal collection
- Encrypted storage
- Access logging
- User data deletion on request

## 12. State Management

### 12.1 Session State (DynamoDB)

**user-sessions table**:
- Partition key: `user_id`
- Sort key: `session_id`
- TTL: 30 minutes
- Attributes: conversation_history, current_step, context

### 12.2 Agent State (DynamoDB)

**agent-state table**:
- Partition key: `trip_id`
- Attributes: current_phase, pending_actions, confidence_scores, tool_results

### 12.3 Trip State (DynamoDB)

**trip-state table**:
- Partition key: `trip_id`
- Attributes: itinerary, bookings, location, status, alerts

## 13. Failure Handling Strategy

### 13.1 Retry Policies

**API Calls**:
- Exponential backoff: 1s, 2s, 4s
- Max retries: 3
- Circuit breaker after 5 consecutive failures

**LLM Inference**:
- Retry on timeout or 5xx errors
- Max retries: 2
- Fallback to simpler prompt

**Booking Operations**:
- Retry with alternative vendors
- Escalate after 3 failures

### 13.2 Graceful Degradation

**Vision Model Unavailable**:
- Fall back to manual preference input
- Use previous taste profile if available

**Manager LLM Unavailable**:
- Queue requests for later processing
- Notify user of delay
- Escalate to human for urgent cases

**Booking API Unavailable**:
- Try alternative vendors
- Escalate to human for manual booking

### 13.3 Rollback Mechanisms

**Booking Rollback**:
- Cancel booking via API
- Release wallet hold
- Update state to reflect cancellation

**Transaction Rollback**:
- Database transactions for atomicity
- Compensating transactions for distributed operations

## 14. Monitoring & Observability

### 14.1 Metrics (CloudWatch)

**System Metrics**:
- Lambda invocations, duration, errors
- API Gateway requests, latency, 4xx/5xx
- DynamoDB read/write capacity, throttles
- RDS CPU, memory, connections

**Business Metrics**:
- Itineraries generated per hour
- Booking success rate
- Escalation rate
- User satisfaction scores

### 14.2 Logging (CloudWatch Logs)

**Log Groups**:
- `/aws/lambda/*`: Lambda function logs
- `/aws/apigateway/*`: API Gateway logs
- `/vacay/agent-decisions`: Agent decision logs
- `/vacay/audit`: Audit trail

**Log Retention**: 30 days (MVP), 90 days (production)

### 14.3 Tracing (AWS X-Ray)

**Traced Operations**:
- End-to-end request flow
- LLM inference calls
- Database queries
- External API calls

**Trace Analysis**:
- Identify bottlenecks
- Debug failures
- Optimize performance

### 14.4 Alerting (CloudWatch Alarms + SNS)

**Critical Alerts**:
- Lambda error rate > 5%
- API Gateway 5xx rate > 1%
- RDS CPU > 80%
- Booking success rate < 90%

**Warning Alerts**:
- Lambda duration > 10s
- DynamoDB throttles > 0
- Escalation rate > 15%

## 15. Deployment Strategy

### 15.1 Infrastructure as Code

**AWS CDK** (TypeScript):
- Define all infrastructure in code
- Version control with Git
- Automated deployments via CI/CD

**Stack Organization**:
- `networking-stack`: VPC, subnets, security groups
- `data-stack`: RDS, DynamoDB, S3
- `compute-stack`: Lambda, ECS
- `ai-stack`: Bedrock, SageMaker
- `api-stack`: API Gateway, Cognito

### 15.2 CI/CD Pipeline (GitHub Actions)

**Pipeline Stages**:
1. Code checkout
2. Run tests (unit, integration)
3. Build Lambda packages
4. Deploy to staging environment
5. Run smoke tests
6. Manual approval
7. Deploy to production
8. Run post-deployment tests

### 15.3 Environment Strategy

**Environments**:
- `dev`: Developer testing (shared)
- `staging`: Pre-production testing
- `production`: Live environment

**Configuration Management**:
- AWS Systems Manager Parameter Store
- Environment-specific parameters
- Secrets in AWS Secrets Manager

## 16. MVP Implementation Phases

### Phase 1: Core Infrastructure (Week 1)
- Set up AWS account and VPC
- Deploy RDS and DynamoDB
- Configure API Gateway and Lambda
- Implement WhatsApp webhook integration
- Basic user authentication

### Phase 2: Vision & Preference System (Week 1-2)
- Deploy SageMaker endpoint
- Implement photo upload pipeline
- Build preference extraction logic
- Create taste profile storage
- Test with sample photos

### Phase 3: Manager LLM & Itinerary Generation (Week 2)
- Integrate Amazon Bedrock
- Implement itinerary generation logic
- Build tool calling interface
- Create decision logging
- Test with sample preferences

### Phase 4: Booking Integration (Week 2-3)
- Integrate hotel booking API (1 vendor)
- Integrate activity booking API (1 vendor)
- Implement wallet system
- Build booking confirmation flow
- Test end-to-end booking

### Phase 5: Voice & Escalation (Week 3)
- Set up n8n workflow engine
- Integrate Polly and Transcribe
- Implement voice call flow
- Build escalation logic
- Create expert dashboard (basic)

### Phase 6: Testing & Polish (Week 3-4)
- End-to-end testing
- Performance optimization
- Security hardening
- Documentation
- Demo preparation

### Phase 7: Hackathon Demo (Week 4)
- Prepare demo script
- Set up demo environment
- Create presentation materials
- Practice demo flow
- Submit to hackathon
