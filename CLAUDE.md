# Vault: Multi-Tenant Knowledge Management System

**UnifiedMemory.ai** - Secure, Encrypted, Searchable Knowledge Base

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Overview](#architecture-overview)
3. [vault-product (Backend API)](#vault-product-backend-api)
4. [vault-platform (Frontend)](#vault-platform-frontend)
5. [vault-api-gateway (API Gateway)](#vault-api-gateway-api-gateway)
6. [Integration & Data Flow](#integration--data-flow)
7. [Security Model](#security-model)
8. [Technology Stack Summary](#technology-stack-summary)
9. [Development Setup](#development-setup)
10. [Deployment Architecture](#deployment-architecture)
11. [Key Features & Capabilities](#key-features--capabilities)
12. [Repository Structure](#repository-structure)
13. [Contributing & Resources](#contributing--resources)

---

## Project Overview

**Vault** is a production-ready, multi-tenant SaaS platform for secure knowledge management with end-to-end encryption and semantic search capabilities. The system enables organizations to store, search, and manage sensitive information while maintaining enterprise-grade security and compliance standards.

### Core Value Proposition

- **End-to-End Encryption**: All knowledge stored with AES-256-GCM encryption, keys managed via AWS KMS
- **Semantic Search**: Vector-based search with OpenAI embeddings and freshness reranking
- **Multi-Tenant Isolation**: Complete data isolation with row-level security (RLS)
- **PII Protection**: Automatic detection and redaction of sensitive personal information
- **Automated Key Rotation**: Zero-downtime encryption key rotation with versioning
- **Subscription Management**: Flexible billing with personal and organization subscription tiers

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User/Client                             │
└───────────────────────────────┬─────────────────────────────────┘
                                │ HTTPS + JWT
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    vault-platform (Frontend)                     │
│  Next.js 15 │ Clerk Auth │ Stripe Billing │ React 19            │
│  - User authentication and session management                    │
│  - Organization & subscription management                        │
│  - API key management UI                                         │
│  - Dashboard and project interface                               │
└───────────────────────────────┬─────────────────────────────────┘
                                │ JWT Token
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                 vault-api-gateway (API Gateway)                  │
│  Zuplo v6 │ JWT Validation │ HMAC Signing │ Webhooks            │
│  - Validates Clerk JWT tokens                                    │
│  - Adds signed headers (X-User-Id, X-Org-Id, X-Auth-Signature) │
│  - Webhook verification and forwarding                           │
│  - Rate limiting and API composition                             │
└───────────────────────────────┬─────────────────────────────────┘
                                │ Signed Headers
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                   vault-product (Backend API)                    │
│  FastAPI │ PostgreSQL │ Qdrant │ AWS KMS │ OpenAI               │
│  - Multi-tenant data storage with RLS                            │
│  - AES-256-GCM encryption/decryption                             │
│  - Semantic vector search                                        │
│  - PII detection and redaction                                   │
│  - Automated key rotation                                        │
│  - RBAC/ABAC authorization                                       │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                    ┌───────────┴────────────┐
                    ↓                        ↓
        ┌─────────────────────┐  ┌─────────────────────┐
        │   PostgreSQL        │  │     Qdrant          │
        │   + pgvector        │  │  Vector Database    │
        │   (Metadata + RLS)  │  │  (Encrypted Notes)  │
        └─────────────────────┘  └─────────────────────┘
```

### Repository Structure

This is a Git repository with three submodules:

- **vault-product**: Python FastAPI backend providing the core API
- **vault-platform**: Next.js frontend for user interface and subscription management
- **vault-api-gateway**: Zuplo API gateway configuration linking frontend to backend

---

## Architecture Overview

### Three-Tier Architecture

The Vault system employs a modern three-tier architecture with clear separation of concerns:

1. **Presentation Layer** (vault-platform)
   - User interface and experience
   - Authentication flows (OAuth2 via Clerk)
   - Subscription and billing management (Stripe)
   - Client-side routing and state management

2. **API Gateway Layer** (vault-api-gateway)
   - Authentication validation (JWT)
   - Request signing and verification (HMAC)
   - API composition and routing
   - Webhook processing
   - Rate limiting and caching

3. **Application Layer** (vault-product)
   - Business logic and data operations
   - Encryption/decryption services
   - Semantic search and vector operations
   - Multi-tenant data isolation
   - Key management and rotation

### Complete Request Lifecycle

```
1. User Login
   └─> vault-platform → Clerk Auth → JWT issued

2. Authenticated API Request
   └─> vault-platform (JWT in header)
       └─> vault-api-gateway
           ├─> Validates JWT signature
           ├─> Extracts user_id, org_id from claims
           ├─> Computes HMAC: SHA256(org_id:user_id:timestamp, secret)
           ├─> Adds headers:
           │   - X-User-Id: <user_id>
           │   - X-Org-Id: <org_id>
           │   - X-Auth-Timestamp: <unix_time>
           │   - X-Auth-Signature: <hmac>
           └─> vault-product
               ├─> Validates HMAC signature
               ├─> Checks org path parameter matches X-Org-Id
               ├─> Enforces RBAC/ABAC permissions
               ├─> Executes business logic
               └─> Returns encrypted/decrypted data

3. Response Flow
   └─> vault-product → vault-api-gateway → vault-platform → User
```

### Multi-Tenancy Model

**Hierarchical Structure**:
```
Organization (Tenant)
  ├─> Projects (Containers for knowledge)
  │     ├─> Notes (Encrypted content with vectors)
  │     └─> Topics (Categories for organization)
  ├─> Users (Membership with roles)
  ├─> API Keys (Programmatic access with scopes)
  └─> Encryption Keys (Org-specific DEKs wrapped by KMS)
```

**Isolation Mechanisms**:
- PostgreSQL Row-Level Security (RLS) policies
- Organization-scoped encryption keys (one DEK per org)
- Qdrant collection separation (per-org or per-project strategies)
- API-level authorization checks (RBAC + ABAC)

### Security Layers

1. **Transport Security**: HTTPS/TLS for all communications
2. **Authentication**: Clerk OAuth2 + JWT validation
3. **Request Integrity**: HMAC-SHA256 signatures on gateway headers
4. **Authorization**: Role-based (RBAC) + Attribute-based (ABAC) access control
5. **Data Encryption**: AES-256-GCM at rest, AWS KMS for key wrapping
6. **Webhook Verification**: Svix signature validation for platform webhooks

---

## vault-product (Backend API)

### Overview

The backend API is a Python FastAPI application providing secure, multi-tenant knowledge management with end-to-end encryption and semantic search capabilities.

**Repository**: `vault-product/`
**Entry Point**: `vault-product/app/main.py`
**Language**: Python 3.12

### Technology Stack

| Category | Technology | Version |
|----------|-----------|---------|
| Framework | FastAPI | 0.115.6 |
| Web Server | Uvicorn | 0.34.0 |
| Data Validation | Pydantic | 2.11+ |
| Database | PostgreSQL (pgvector) | 16 |
| ORM | SQLAlchemy (async) | 2.0.44 |
| Migrations | Alembic | 1.17.1 |
| Vector DB | Qdrant | 1.12.1 |
| Encryption | cryptography | 44.0.2 |
| Key Management | AWS KMS (boto3) | 1.40.70 |
| Embeddings | OpenAI | 1.59.7 |
| PII Detection | Presidio | 2.2.355 |
| Observability | OpenTelemetry | 1.20+ |
| Metrics | Prometheus Client | 0.21.1 |
| Testing | pytest | 8.3.4 |

### Key Features

#### 1. End-to-End Encryption
- **Algorithm**: AES-256-GCM (authenticated encryption with 96-bit nonces)
- **Key Management**: AWS KMS wraps organization-specific DEKs (Data Encryption Keys)
- **AAD Binding**: Additional Authenticated Data includes `org_id|project_id|note_id` to prevent ciphertext substitution attacks
- **Dev Mode**: PBKDF2-based key derivation for local development without AWS
- **Implementation**: `vault-product/app/encryption/vault_encryptor.py`

#### 2. Semantic Search
- **Embeddings**: OpenAI text-embedding-3-small (1536 dim) or text-embedding-3-large (3072 dim)
- **Vector Storage**: Qdrant HNSW index for efficient similarity search
- **Reranking**: Time-decay algorithm boosts recent notes (freshness reranking)
- **Filtering**: ABAC-based filtering with topic/tag allow/deny lists
- **Optional Decryption**: Search can return encrypted or plaintext results based on permissions

#### 3. Automated Key Rotation
- **Zero-Downtime**: New DEKs generated and wrapped by KMS, old keys remain accessible for reads
- **Batch Re-encryption**: Planner discovers orgs with rotation policies, executor re-encrypts all notes
- **Version Tracking**: `org_keys` table tracks current version, `org_key_history` stores historical keys
- **Signing Keys**: Ed25519 keypairs for JWT signing auto-rotate monthly
- **Implementation**: `vault-product/app/services/key_rotation.py`

#### 4. PII Detection & Redaction
- **Library**: Microsoft Presidio analyzer + anonymizer
- **Detected Types**: EMAIL_ADDRESS, PHONE_NUMBER, US_SSN, CREDIT_CARD, PERSON (NER)
- **Redaction Strategy**: Replace with tokens like `[EMAIL]`, `[PHONE]`, `[SSN]`, `[CARD]`, `[NAME]`
- **Timing**: Applied before embedding generation to prevent PII in vectors
- **Implementation**: `vault-product/app/services/pii_redaction.py`

#### 5. Near-Duplicate Detection
- **Algorithm**: SimHash fingerprinting on redacted text
- **Matching**: PostgreSQL trigram similarity (pg_trgm) for fuzzy matching
- **Threshold**: Configurable (default 0.60, recommended 0.90 for production)
- **Response**: HTTP 409 Conflict if duplicate detected
- **Implementation**: `vault-product/app/services/dedup_notes.py`

#### 6. RBAC/ABAC Authorization
- **Organization Roles**: org_admin, admin, member
- **Project Roles**: project_admin, writer, reader, auditor
- **ABAC Rules**: Per-project topic allow/deny lists, tag allow/deny lists
- **API Key Scopes**: Restrict programmatic access to specific projects/topics/tags
- **Implementation**: `vault-product/app/auth/rbac_resolver.py`, `vault-product/app/services/filters.py`

### API Endpoints Overview

#### Organizations
- `POST /v1/orgs` - Create organization (webhook-only in production)
- `GET /v1/orgs/{org}` - Get organization details
- `GET /v1/orgs` - List user's organizations

#### Projects
- `POST /v1/orgs/{org}/projects` - Create project
- `GET /v1/orgs/{org}/projects/{proj}` - Get project details
- `GET /v1/orgs/{org}/projects` - List projects

#### Knowledge Management
- `POST /v1/orgs/{org}/projects/{proj}/notes` - Create note (with near-duplicate detection)
- `GET /v1/orgs/{org}/projects/{proj}/notes/{note_id}` - Get note
- `DELETE /v1/orgs/{org}/projects/{proj}/notes/{note_id}` - Delete note
- `POST /v1/orgs/{org}/projects/{proj}/topics` - Create topic
- `GET /v1/orgs/{org}/projects/{proj}/topics` - List topics
- `POST /v1/orgs/{org}/projects/{proj}/search` - Semantic search with freshness reranking

#### API Keys
- `POST /v1/api-keys` - Create API key
- `GET /v1/api-keys` - List API keys
- `DELETE /v1/api-keys/{key_id}` - Revoke API key

#### Authentication
- `POST /v1/auth/api-keys/tokens` - Exchange API key for JWT token
- `GET /v1/auth/api-keys/jwks` - Public JWKS endpoint (no auth required)

#### Key Rotation
- `POST /v1/orgs/{org}/rotate-keys` - Initiate DEK rotation
- `GET /v1/orgs/{org}/rotation-status` - Check rotation status

#### Webhooks
- `POST /v1/webhooks/platform/users` - Platform user sync webhook
- `POST /v1/webhooks/platform/orgs` - Platform org sync webhook
- `POST /v1/webhooks/platform/memberships` - Platform membership sync webhook

#### Health & Monitoring
- `GET /health` - Service health status
- `GET /health/db` - Database connectivity check
- `GET /health/api-keys` - API key subsystem health
- `GET /metrics` - Prometheus metrics endpoint

#### MCP Integration
- `POST /mcp` - Model Context Protocol HTTP transport
- `GET /mcp/echo` - Test endpoint showing org_id/user_id/key_scope context

### Database Models

**Core Models** (`vault-product/app/models/`):

| Model | Purpose | Key Fields |
|-------|---------|-----------|
| `Organization` | Top-level tenant | `id`, `name`, `region`, `status`, `current_key_version` |
| `Project` | Knowledge container | `id`, `org_id`, `name`, `slug`, `status` |
| `User` | User identity | `id`, `auth_provider`, `external_id`, `user_type` |
| `Topic` | Note categorization | `id`, `project_id`, `name`, `description` |
| `OrgMembership` | User-org roles | `user_id`, `org_id`, `role` |
| `ProjectMembership` | User-project roles | `user_id`, `project_id`, `role` |
| `ProjectACL` | ABAC rules | `project_id`, `allowed_topics`, `denied_topics` |
| `APIKey` | Programmatic access | `id`, `prefix`, `hashed_secret`, `scope`, `revocation_counter` |
| `OrgKeys` | Current encryption keys | `org_id`, `wrapped_dek`, `kms_key_id`, `version` |
| `OrgKeyHistory` | Historical keys | `org_id`, `version`, `wrapped_dek`, `deprecated_at` |
| `OrgRotationPolicy` | Key rotation schedule | `org_id`, `enabled`, `rotation_interval_days` |
| `KeyRotationJob` | Rotation executions | `id`, `org_id`, `status`, `notes_rotated` |
| `SigningKey` | JWT signing keys | `id`, `key_id`, `public_key_pem`, `status` |
| `AuditLog` | Compliance audit trail | `id`, `org_id`, `user_id`, `action`, `timestamp` |

### Deployment

#### Docker Configuration
**File**: `vault-product/Dockerfile`

Multi-stage build:
1. **Builder Stage**: Install Python dependencies with build tools
2. **Runtime Stage**: Minimal Python 3.12-slim with PostgreSQL client

**Exposed Port**: 8000
**Command**: `uvicorn app.main:app --host 0.0.0.0 --port 8000`

#### Docker Compose Services
**File**: `vault-product/docker-compose.yml`

- **PostgreSQL** (`pgvector/pgvector:pg16`) - Port 5432, volume: postgres_data
- **Qdrant** (custom build) - REST: 6333, gRPC: 6334, volume: qdrant_data
- **Jaeger** (observability) - UI: 16686, OTLP: 4318 (HTTP), 4317 (gRPC)

#### Production Infrastructure
- **Platform**: Kubernetes (K3s) on Oracle Cloud Infrastructure (OCI)
- **Compute**: ARM64 Ampere instances
- **Networking**: Tailscale VPN (no public endpoint)
- **Registry**: OCI Container Registry (OCIR)
- **Scaling**: Horizontal Pod Autoscaler (HPA)
- **CI/CD**: GitHub Actions (`.github/workflows/build-and-push-images.yml`)

### Configuration

**Key Environment Variables** (`vault-product/.env.example`):

```bash
# Database
DATABASE_URL=postgresql+asyncpg://vault_user:password@localhost:5432/vault
POSTGRES_POOL_SIZE=10
POSTGRES_MAX_OVERFLOW=20

# Vector Database
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_GRPC_PORT=6334
QDRANT_COLLECTION_STRATEGY=single_org

# Encryption
KMS_MASTER_KEY_REF=alias/vault-master-key
AWS_KMS_ENABLED=true
ENCRYPTION_DEV_MODE=false

# OpenAI
OPENAI_API_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSION=1536

# Authentication
AUTH_MODE=gateway-hmac
GATEWAY_AUTH_SECRET=...
GATEWAY_SIGNATURE_TTL_SECONDS=300
WEBHOOK_SECRET=...

# Application
APP_ENV=production
DEBUG=false
LOG_LEVEL=INFO
API_PORT=8000

# Observability
OTEL_ENABLED=true
OTEL_SERVICE_NAME=vault-api
OTEL_ENDPOINT=http://jaeger:4318
```

### Testing

**Test Structure**: `vault-product/tests/`

- **Unit Tests** (`tests/unit/`): Encryption, authentication, business logic, filters
- **Integration Tests** (`tests/integration/`): RBAC enforcement, key rotation, end-to-end flows

**Run Tests**:
```bash
pytest --cov=app --cov-report=html
pytest tests/integration/test_key_rotation.py
./scripts/quick_test.sh  # End-to-end walkthrough
```

---

## vault-platform (Frontend)

### Overview

The frontend is a modern Next.js 15 SaaS web application providing user authentication, organization management, subscription billing, and API access management.

**Repository**: `vault-platform/`
**Entry Point**: `vault-platform/app/layout.tsx`
**Language**: TypeScript 5

### Technology Stack

| Category | Technology | Version |
|----------|-----------|---------|
| Framework | Next.js | 15 |
| UI Library | React | 19 |
| Language | TypeScript | 5 |
| Styling | Tailwind CSS | 3.4 |
| UI Components | ShadCN UI | Latest |
| Animation | Framer Motion | 12 |
| Authentication | Clerk | Latest |
| Payments | Stripe | 19.3.0 |
| Database | PostgreSQL | Latest |
| ORM | Prisma | 6.19.0 |
| Package Manager | pnpm | Latest |
| Deployment | Vercel/Docker | - |

### Key Features

#### 1. User Authentication (Clerk OAuth2)
- **Provider**: Clerk authentication platform
- **Methods**: Email/password, OAuth social logins (Google, GitHub, etc.)
- **Session Management**: JWT tokens with automatic refresh
- **Organization Support**: Built-in organization switcher
- **Middleware Protection**: Routes protected via `clerkMiddleware`
- **Implementation**: `vault-platform/middleware.ts`

#### 2. Organization Management
- **Creation**: Users can create organizations with unique slugs
- **Membership**: Invite members with role-based permissions
- **Switching**: Context switcher for personal vs organization accounts
- **Settings**: Organization profile, branding, member management
- **Implementation**: `vault-platform/app/organization/`

#### 3. Subscription Billing (Stripe)
- **Dual Model**: Users can have BOTH personal AND organization subscriptions simultaneously
- **Plans**: Free, Pro, Enterprise tiers
- **Billing**: Stripe Checkout for new subscriptions, Customer Portal for management
- **Webhooks**: Async fire-and-forget processing with idempotency
- **Metadata Sync**: Subscription info synced to Clerk metadata for client-side access
- **Implementation**: `vault-platform/app/api/stripe/`

#### 4. API Key Management UI
- **Creation**: Generate API keys with custom scopes (project/topic/tag restrictions)
- **Display**: Show prefix and secret once at creation
- **Listing**: View all active keys with metadata
- **Revocation**: Instant key revocation
- **Implementation**: `vault-platform/app/components/account/ApiKeysClient.tsx`

#### 5. State Management (Clerk Metadata)
- **Public Metadata**: Subscription tier, status, pricing (client + server accessible)
- **Private Metadata**: Feature flags (server-side only for security)
- **No Redux/Zustand**: Leverages Clerk's built-in metadata system
- **Context**: Determined by Clerk's org switcher (current `orgId`)

### Page Structure

**Main Pages** (`vault-platform/app/`):

| Route | Component | Purpose |
|-------|-----------|---------|
| `/` | `page.tsx` | Landing page (Strapi CMS content) |
| `/pricing` | `pricing/page.tsx` | Pricing table (public) |
| `/account` | `account/page.tsx` | Dashboard (protected) |
| `/organization/create` | `organization/create/page.tsx` | Create organization |
| `/organization/list` | `organization/list/page.tsx` | List organizations |
| `/organization/profile` | `organization/profile/page.tsx` | Organization settings |
| `/learn` | `learn/page.tsx` | Learning center |
| `/billing/*` | `billing/` | Post-transaction pages |
| `/[slug]` | `[slug]/page.tsx` | Dynamic CMS pages |

### Database Schema (Prisma)

**File**: `vault-platform/prisma/schema.prisma`

**7 Core Models**:

1. **Customer** - Links Clerk users/orgs to Stripe customers
   - `clerkUserId` XOR `clerkOrgId` (mutually exclusive)
   - `stripeCustomerId` (unique)

2. **Product** - Stripe product catalog
   - `features` (JSONB) - Machine-readable feature keys
   - `marketingFeatures` (JSONB) - Human-readable descriptions
   - `productType` - 'personal' or 'organization'

3. **Price** - Pricing for products
   - `unitAmount` (cents), `currency`, `type` ('one_time' or 'recurring')
   - `recurringInterval` ('day', 'week', 'month', 'year')
   - `trialPeriodDays`

4. **Subscription** - Active/canceled subscriptions
   - `status` - 'active', 'trialing', 'past_due', 'canceled', etc.
   - `currentPeriodStart/End`, `cancelAtPeriodEnd`

5. **SubscriptionPriceHistory** - Audit trail of plan changes
   - `changeReason` - 'created', 'upgraded', 'downgraded', 'plan_switch'

6. **WebhookEvent** - Idempotency tracking
   - `stripeEventId` (unique), `processedAt`, `errorMessage`

7. **PriceChangeEvent** - Product/price catalog changes
   - `previousAttributes` (JSONB)

### API Routes

#### Stripe API (`vault-platform/app/api/stripe/`)
- `GET /api/stripe/products` - Fetch active products with features (cached 5 min)
- `POST /api/stripe/create-checkout-session` - Initiate Stripe checkout
- `POST /api/stripe/create-portal-session` - Stripe billing portal
- `GET /api/stripe/subscription-details` - Current subscription info
- `POST /api/stripe/preview-proration` - Calculate upgrade/downgrade costs
- `POST /api/stripe/update-subscription` - Change plan
- `POST /api/stripe/webhook` - Webhook receiver (async processing)

#### UnifiedMemory API (`vault-platform/app/api/unifiedmemory/`)
- `GET|POST /api/unifiedmemory/projects` - Project CRUD
- `GET|POST|DELETE /api/unifiedmemory/api-keys` - API key management
- `GET|POST /api/unifiedmemory/projects/[projectId]/topics` - Topic management

**Pattern**: Server-side helpers forward Clerk auth context via headers:
- `X-Org-Id`, `X-User-Id`, `Authorization: Bearer {token}`
- Base URL: Configured via environment variable

### Webhook System

**Architecture**: Modular, router-based in `vault-platform/lib/stripe-webhooks/`

**Processing Flow**:
1. Verify Stripe signature
2. Return 200 immediately (prevent retries)
3. Process asynchronously (fire-and-forget)
4. Check idempotency via `webhook_events` table
5. Update database via Prisma
6. Sync subscription to Clerk metadata

**Handled Events**:
- `checkout.session.completed` - Create subscription records
- `customer.subscription.*` - Sync to Clerk metadata
- `price.*` - Update price catalog
- `product.*` - Update products, sync features to all subscribers

### Deployment

#### Docker Configuration
**File**: `vault-platform/Dockerfile`

Multi-stage build (deps → builder → runner):
- Node 20-slim, pnpm package manager
- Prisma client generation during build
- Non-root user (nextjs:1001) for security
- Exposed port: 3000

#### Recommended Platform
- **Vercel**: Optimized for Next.js with automatic builds, edge functions, analytics
- **Docker**: Alternative for self-hosted deployments

### Configuration

**Key Environment Variables** (`vault-platform/.env.example`):

```bash
# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/account

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...

# Database
DATABASE_URI=postgresql://user:password@host:5432/db

# Application
NEXT_PUBLIC_APP_URL=https://app.example.com

# UnifiedMemory API
API_URL=https://api-gateway-url.dev
UNIFIED_MEMORY_TOKEN_TEMPLATE=zuplo

# Strapi CMS
STRAPI_URL=http://localhost:1337
STRAPI_API_TOKEN=...
```

### Development

**Scripts**:
```bash
npm run dev          # Prisma migrations + Stripe webhook forwarding + Next.js dev server
npm run build        # Prisma generate + Next.js build
npm start            # Production server
npm run lint         # ESLint
```

**Development Helper**: `vault-platform/scripts/dev.sh`
- Validates environment variables
- Runs Prisma migrations
- Starts Stripe CLI webhook forwarding
- Starts Next.js dev server
- Graceful shutdown handling

---

## vault-api-gateway (API Gateway)

### Overview

The API gateway is a Zuplo-based configuration that provides secure authentication, request signing, and API composition between the frontend and backend.

**Repository**: `vault-api-gateway/`
**Platform**: Zuplo v6 (Edge API Gateway)
**Language**: TypeScript (ES2022, WebWorker runtime)

### Technology Stack

| Category | Technology | Version |
|----------|-----------|---------|
| Platform | Zuplo | 6 |
| Language | TypeScript | ES2022 |
| Runtime | WebWorker | - |
| OpenAPI | Specification | 3.1.0 |

### Key Features

#### 1. JWT Validation (Clerk)
- **Policy**: `clerk-jwt-auth-inbound` (built-in)
- **Frontend URL**: `https://clear-caiman-45.clerk.accounts.dev`
- **Audience**: Validates JWT audience claim
- **Strict Mode**: Rejects unauthenticated requests
- **Claim Extraction**: Extracts `sub` (user_id) and `data.o.o_id` (org_id)

#### 2. HMAC Signature Generation
- **Module**: `vault-api-gateway/modules/append-gateway-headers.ts`
- **Algorithm**: HMAC-SHA256(org_id:user_id:timestamp, secret)
- **Headers Added**:
  - `X-User-Id` - From JWT sub claim
  - `X-Org-Id` - From JWT custom data or fallback to user_id
  - `X-Auth-Timestamp` - Unix timestamp
  - `X-Auth-Signature` - HMAC signature for tamper-proofing

#### 3. Webhook Verification (Svix)
- **Module**: `vault-api-gateway/modules/clerk-webhook-verify.ts`
- **Library**: Svix webhook verification
- **Headers**: `svix-id`, `svix-timestamp`, `svix-signature`
- **Response**: 400 for missing headers, 401 for invalid signature

#### 4. Outbound Webhook Signing
- **Module**: `vault-api-gateway/modules/append-webhook-signature.ts`
- **Algorithm**: HMAC-SHA256(timestamp.payload)
- **Headers Added**:
  - `X-Webhook-Signature` - Hex-encoded HMAC
  - `X-Webhook-Timestamp` - Unix timestamp

### Middleware Policies

**File**: `vault-api-gateway/config/policies.json`

**6 Total Policies**:

| Name | Type | Module | Purpose |
|------|------|--------|---------|
| `mock-api-inbound` | Built-in | @zuplo/runtime | Mock API testing |
| `clerk-jwt-auth-inbound` | Built-in | @zuplo/runtime | JWT validation |
| `append-gateway-headers` | Custom | append-gateway-headers.ts | Add signed headers |
| `clerk-webhook-verify` | Custom | clerk-webhook-verify.ts | Verify Svix signatures |
| `append-web-hook-signature` | Custom | append-webhook-signature.ts | Sign outbound webhooks |
| `debug-outbound` | Custom | debug-outbound.ts | Log responses |

**Policy Chain**:
```
Request → clerk-jwt-auth-inbound → append-gateway-headers → [Route Handler] → debug-outbound → Response
```

### Routes Overview

**File**: `vault-api-gateway/config/routes.oas.json` (6573 lines)

**24 Main Endpoints** (grouped by domain):

#### Health & Debug
- `GET /health`, `GET /health/db`, `GET /health/api-keys`, `GET /health/debug`, `GET /`

#### Authentication
- `POST /v1/auth/api-keys/token` - Issue JWT for API key (gateway-only)
- `GET /v1/auth/api-keys/jwks` - Public JWKS endpoint (cached 5 min)

#### API Keys
- `POST|GET|DELETE /v1/orgs/{org}/api-keys/*` - API key CRUD operations

#### Organizations & Projects
- `POST|GET /v1/orgs/*` - Organization management
- `POST|GET /v1/orgs/{org}/projects/*` - Project management

#### Knowledge Management
- `POST|GET|DELETE /v1/orgs/{org}/projects/{proj}/notes/*` - Note operations
- `POST /v1/orgs/{org}/projects/{proj}/search` - Semantic search
- `POST|GET /v1/orgs/{org}/projects/{proj}/topics/*` - Topic management

#### Key Rotation
- `POST|GET /v1/orgs/{org}/rotation/*` - Key rotation operations

#### Webhooks
- `POST /v1/webhooks/platform` - Platform webhook receiver

**Handler**: All routes use `urlForwardHandler` with `baseUrl: ${env.BASE_API_URL}`

### Authentication Flow

**Primary Auth Method**:
```
1. Frontend (vault-platform) obtains JWT from Clerk
2. Request sent to gateway with Authorization: Bearer <jwt>
3. Gateway validates JWT (clerk-jwt-auth-inbound)
4. Gateway extracts user_id, org_id from JWT claims
5. Gateway computes HMAC signature
6. Gateway adds signed headers (X-User-Id, X-Org-Id, X-Auth-Signature, X-Auth-Timestamp)
7. Backend (vault-product) validates HMAC signature
8. Backend enforces authorization based on headers
```

**API Key Token Endpoint**:
- **Endpoint**: `POST /v1/auth/api-keys/token`
- **Requirements**: mTLS + HMAC signature
- **Rate Limit**: 100 req/s per gateway instance
- **Returns**: Short-lived JWT with org_id, user_id, scope, revocation counter

**JWKS Endpoint**:
- **Endpoint**: `GET /v1/auth/api-keys/jwks`
- **Cache**: 5 minutes with ETags
- **Key Statuses**: active, deprecated, compromised

### Configuration

**Environment Variables** (`vault-api-gateway/env.example`):

```bash
# Backend Service
BASE_API_URL=https://backend-service-url.com

# Authentication
GATEWAY_SECRET=<hmac-secret-shared-with-backend>
CLERK_WEBHOOK_SECRET=<clerk-webhook-verification-secret>
```

### Development

**Scripts**:
```bash
npm run dev   # Start Zuplo dev server (port 9000)
npm run test  # Run tests
npm run docs  # Documentation workspace
```

**Debug Configuration**: VS Code launch config available (`.vscode/launch.json`)

---

## Integration & Data Flow

### Complete Request Flow

#### 1. User Authentication
```
User → vault-platform → Clerk Auth → JWT issued
```

#### 2. Authenticated API Request
```
vault-platform (JWT in Authorization header)
    ↓
vault-api-gateway
    ├─> Validates JWT signature (clerk-jwt-auth-inbound)
    ├─> Extracts identity from JWT claims:
    │   - sub → user_id
    │   - data.o.o_id → org_id
    ├─> Computes HMAC: SHA256(org_id:user_id:timestamp, GATEWAY_SECRET)
    ├─> Adds headers:
    │   - X-User-Id: <user_id>
    │   - X-Org-Id: <org_id>
    │   - X-Auth-Timestamp: <unix_time>
    │   - X-Auth-Signature: <hmac>
    ├─> Forwards request to ${BASE_API_URL}
    ↓
vault-product
    ├─> Validates HMAC signature using shared GATEWAY_SECRET
    ├─> Checks org path parameter matches X-Org-Id (prevents hijacking)
    ├─> Enforces RBAC/ABAC permissions
    ├─> Executes business logic (encryption, search, etc.)
    └─> Returns response
    ↓
vault-api-gateway (debug-outbound logs response)
    ↓
vault-platform
    ↓
User
```

### Webhook Synchronization

#### Platform → Gateway → Product
```
Platform Event (e.g., user signup in Auth0)
    ↓
vault-platform triggers webhook
    ↓ POST /v1/webhooks/platform
vault-api-gateway
    ├─> Verifies Svix signature (clerk-webhook-verify)
    ├─> Adds outbound signature:
    │   - X-Webhook-Signature: HMAC-SHA256(timestamp.payload)
    │   - X-Webhook-Timestamp: <unix_time>
    └─> Forwards to vault-product
    ↓
vault-product
    ├─> Verifies X-Webhook-Signature
    ├─> Creates/updates user in database
    └─> Returns 204 No Content
```

### API Key JWT Flow

#### Token Issuance
```
Client (service principal)
    ↓ POST /v1/auth/api-keys/token (API key credentials)
vault-api-gateway (rate limit: 100 req/s)
    ↓
vault-product
    ├─> Verifies API key (prefix + hashed secret match)
    ├─> Checks key status (not revoked, not expired)
    ├─> Generates JWT signed with Ed25519 private key:
    │   - org_id, user_id/service_principal_id
    │   - scope (project/topic/tag restrictions)
    │   - revocation_counter (for cache invalidation)
    │   - expiry (short-lived, e.g., 1 hour)
    └─> Returns JWT
    ↓
Client stores JWT
    ↓
Client uses JWT in subsequent requests (Authorization: Bearer <jwt>)
```

#### Token Verification
```
Client request with JWT
    ↓
vault-api-gateway validates JWT (clerk-jwt-auth-inbound)
    ↓
vault-product receives signed headers
    ├─> Checks revocation_counter against cached value
    ├─> If stale, returns 409 REV_COUNTER_STALE (force refresh)
    └─> Enforces scope restrictions (ABAC filters)
```

### Key Rotation Coordination

#### DEK Rotation
```
vault-product (scheduled job or manual trigger)
    ├─> Planner discovers orgs with rotation policies due
    ├─> Executor generates new DEK
    ├─> AWS KMS wraps new DEK
    ├─> Increment org key version
    ├─> Store new key in org_keys table
    ├─> Move old key to org_key_history table
    ├─> Re-encrypt all notes:
    │   ├─> Fetch encrypted note from Qdrant
    │   ├─> Decrypt with old DEK (via org_key_history)
    │   ├─> Encrypt with new DEK
    │   └─> Update Qdrant with new ciphertext
    └─> Zero-downtime: old keys remain accessible for reads
```

#### Signing Key Rotation (Monthly)
```
vault-product (automated monthly job)
    ├─> Generate new Ed25519 keypair
    ├─> Store in signing_keys table with status='active'
    ├─> Mark old key as status='deprecated' (grace period for existing JWTs)
    ├─> After grace period, mark as status='compromised' (force invalidation)
    └─> Update JWKS endpoint (/v1/auth/api-keys/jwks)
    ↓
vault-api-gateway polls JWKS every 5 minutes
    └─> Updates cached signing keys
```

---

## Security Model

### Authentication Layers

#### 1. Clerk OAuth2 (Primary User Auth)
- **Method**: OAuth2 authorization code flow with PKCE
- **Providers**: Email/password, Google, GitHub, etc.
- **Token Type**: JWT with RS256 signature
- **Session**: Stored in Clerk's infrastructure, refreshed automatically
- **Middleware**: `clerkMiddleware` in vault-platform, `clerk-jwt-auth-inbound` in gateway

#### 2. Gateway HMAC Signatures (Tamper-Proofing)
- **Algorithm**: HMAC-SHA256
- **Message**: `org_id:user_id:timestamp`
- **Secret**: Shared between gateway and backend (GATEWAY_SECRET)
- **Headers**: `X-Auth-Signature`, `X-Auth-Timestamp`
- **TTL**: Configurable (default 300 seconds) to prevent replay attacks
- **Validation**: Backend verifies signature and timestamp freshness

#### 3. API Key JWT Tokens (Programmatic Access)
- **Generation**: Backend issues short-lived JWTs for API keys
- **Signing**: Ed25519 private key (auto-rotated monthly)
- **Claims**: org_id, user_id, scope, revocation_counter, expiry
- **Verification**: Gateway validates using public JWKS endpoint
- **Revocation**: Revocation counter invalidates cached JWTs immediately

### Authorization Model

#### Organization Roles (org_membership)
- **org_admin**: Full organization access, can manage members and settings
- **admin**: Administrative access, can manage projects and users
- **member**: Standard member, access based on project permissions

#### Project Roles (project_membership)
- **project_admin**: Full project access, can manage settings and members
- **writer**: Can create and update notes
- **reader**: Read-only access to notes
- **auditor**: Read-only with audit trail visibility

#### Attribute-Based Access Control (ABAC)
- **Per-Project Rules** (project_acl table):
  - `allowed_topics`: Whitelist of topics user can access
  - `denied_topics`: Blacklist of topics user cannot access
  - `allowed_tags`: Whitelist of tags user can access
  - `denied_tags`: Blacklist of tags user cannot access
- **API Key Scopes**: JSON object restricting access:
  ```json
  {
    "projects": ["proj_123", "proj_456"],
    "topics": ["all"] | ["topic_1", "topic_2"],
    "tags": ["all"] | ["tag_1", "tag_2"]
  }
  ```
- **Enforcement**: Applied during search queries and note access

### Encryption

#### At-Rest Encryption (Backend Only)
- **Algorithm**: AES-256-GCM (authenticated encryption)
- **Key Management**:
  - Master Key: AWS KMS (production) or PBKDF2 (dev)
  - DEKs: One per organization, wrapped by master key
  - Version Tracking: Current version in `org_keys`, historical in `org_key_history`
- **Nonce**: 96-bit (12 bytes) random per encryption
- **AAD**: `org_id|project_id|note_id` binds ciphertext to context
- **Storage**: Encrypted notes stored in Qdrant, metadata in PostgreSQL

#### In-Transit Encryption
- **HTTPS/TLS**: All communications between components
- **Certificate Management**: Automated via cloud providers (Vercel, OCI, Zuplo Cloud)

#### Client-Side Encryption
- **Status**: Not implemented (backend encryption only)
- **Consideration**: Future enhancement for zero-knowledge architecture

### Security Headers

**Gateway → Backend**:
- `X-User-Id`: Authenticated user identifier
- `X-Org-Id`: Organization context
- `X-Auth-Signature`: HMAC-SHA256 signature
- `X-Auth-Timestamp`: Request timestamp
- `Authorization`: Bearer JWT token

**Platform → Gateway (Webhooks)**:
- `svix-id`: Webhook unique identifier
- `svix-timestamp`: Webhook timestamp
- `svix-signature`: Svix HMAC signature

**Gateway → Backend (Webhooks)**:
- `X-Webhook-Signature`: Gateway HMAC signature
- `X-Webhook-Timestamp`: Forwarding timestamp

### CORS Policies

**Configuration**: `"corsPolicy": "none"` on all gateway routes
- No cross-origin requests by default
- Assumes same-origin frontend or explicit client configuration
- Production: Frontend and gateway on same domain or whitelisted origins

---

## Technology Stack Summary

| Aspect | vault-product | vault-platform | vault-api-gateway |
|--------|---------------|----------------|-------------------|
| **Language** | Python 3.12 | TypeScript 5 | TypeScript (ES2022) |
| **Framework** | FastAPI 0.115.6 | Next.js 15 | Zuplo v6 |
| **UI Library** | N/A | React 19 | N/A |
| **Web Server** | Uvicorn 0.34.0 | Node.js (Next.js) | Edge Runtime |
| **Database** | PostgreSQL 16 + pgvector | PostgreSQL | N/A |
| **ORM** | SQLAlchemy 2.0.44 (async) | Prisma 6.19.0 | N/A |
| **Migrations** | Alembic 1.17.1 | Prisma Migrate | N/A |
| **Vector DB** | Qdrant 1.12.1 | N/A | N/A |
| **Authentication** | Gateway HMAC validation | Clerk (OAuth2) | Clerk JWT validation |
| **Payments** | N/A | Stripe 19.3.0 | N/A |
| **Encryption** | cryptography 44.0.2 (AES-256-GCM) | N/A | N/A |
| **Key Management** | AWS KMS (boto3 1.40.70) | N/A | N/A |
| **Embeddings** | OpenAI 1.59.7 | N/A | N/A |
| **PII Detection** | Presidio 2.2.355 | N/A | N/A |
| **Styling** | N/A | Tailwind CSS 3.4 | N/A |
| **UI Components** | N/A | ShadCN UI + Radix UI | N/A |
| **Animation** | N/A | Framer Motion 12 | N/A |
| **Observability** | OpenTelemetry 1.20+, Prometheus | Vercel Analytics | Zuplo Logs |
| **Testing** | pytest 8.3.4 | N/A | Zuplo Test Runner |
| **Code Quality** | black, ruff, mypy | ESLint | TypeScript strict mode |
| **Package Manager** | pip | pnpm | npm |
| **Container** | Docker (multi-stage) | Docker (multi-stage) | N/A (Zuplo Cloud) |
| **Deployment** | Kubernetes (K3s on OCI ARM64) | Vercel / Docker | Zuplo Cloud (Edge) |
| **CI/CD** | GitHub Actions | N/A (Vercel auto-deploy) | N/A (Zuplo auto-deploy) |

### Key Libraries

**vault-product**:
- FastAPI, Uvicorn, Pydantic (web framework)
- SQLAlchemy, Alembic, asyncpg (database)
- Qdrant client (vector search)
- cryptography, boto3 (encryption)
- OpenAI (embeddings)
- Presidio (PII detection)
- OpenTelemetry, Prometheus (observability)
- pytest, pytest-asyncio (testing)

**vault-platform**:
- Next.js, React (framework)
- Clerk (authentication)
- Stripe (payments)
- Prisma (database ORM)
- Tailwind CSS, ShadCN UI, Radix UI (styling/components)
- Framer Motion (animation)
- Lucide React (icons)

**vault-api-gateway**:
- Zuplo runtime (gateway platform)
- Svix (webhook verification)
- Custom TypeScript modules (HMAC, signing)

---

## Development Setup

### Prerequisites

**Required**:
- Node.js 20+ (for vault-platform)
- Python 3.12+ (for vault-product)
- Docker & Docker Compose (for databases)
- Git (for submodule management)

**Optional**:
- AWS CLI (for KMS in vault-product production)
- Stripe CLI (for webhook testing in vault-platform)
- kubectl (for Kubernetes deployment)

### Initial Repository Setup

```bash
# Clone repository with submodules
git clone --recurse-submodules <repository-url>
cd vault

# Or if already cloned
git submodule update --init --recursive

# Pull latest changes for all submodules
git submodule update --remote
```

### vault-product Setup

```bash
cd vault-product

# 1. Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Copy and configure environment
cp .env.example .env
# Edit .env with your configuration:
#   - OPENAI_API_KEY
#   - AWS credentials (or ENCRYPTION_DEV_MODE=true)
#   - Database credentials

# 4. Start infrastructure services
docker-compose up -d postgres qdrant jaeger

# 5. Wait for PostgreSQL to be ready (check logs)
docker-compose logs -f postgres

# 6. Run database migrations
alembic upgrade head

# 7. (Optional) Set up Qdrant collection for testing org
python qdrant_setup.py --org-id test_org

# 8. Start API server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# API available at http://localhost:8000
# OpenAPI docs at http://localhost:8000/docs
# Jaeger UI at http://localhost:16686
```

**Testing**:
```bash
# Run all tests with coverage
pytest --cov=app --cov-report=html

# Run integration tests
pytest tests/integration/

# Run quick end-to-end test
./scripts/quick_test.sh
```

### vault-platform Setup

```bash
cd vault-platform

# 1. Install dependencies
pnpm install

# 2. Copy and configure environment
cp .env.example .env.local
# Edit .env.local with your configuration:
#   - Clerk keys (NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY, CLERK_SECRET_KEY)
#   - Stripe keys (STRIPE_SECRET_KEY, NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY)
#   - Database URI (DATABASE_URI)
#   - API Gateway URL (API_URL)
#   - Strapi URL (STRAPI_URL) - optional

# 3. Set up database
pnpm prisma generate
pnpm prisma migrate dev

# 4. (Optional) Seed database with Stripe products
# This requires Stripe account and products configured

# 5. Start development server (includes Stripe webhook forwarding)
pnpm run dev

# Or start components separately:
# pnpm run next:dev        # Next.js only
# stripe listen --forward-to localhost:3000/api/stripe/webhook

# Application available at http://localhost:3000
```

**Testing**:
```bash
# Lint code
pnpm run lint

# Build for production (test)
pnpm run build
```

### vault-api-gateway Setup

```bash
cd vault-api-gateway

# 1. Install dependencies
npm install

# 2. Copy and configure environment
cp env.example .env
# Edit .env with your configuration:
#   - BASE_API_URL (vault-product URL)
#   - GATEWAY_SECRET (shared HMAC secret)
#   - CLERK_WEBHOOK_SECRET

# 3. Start development server
npm run dev

# Gateway available at http://localhost:9000
# OpenAPI docs at http://localhost:9000/docs
```

**Testing**:
```bash
# Run tests
npm run test
```

### Environment Variable Requirements

**Minimum Required (Development)**:

**vault-product**:
- `DATABASE_URL` - PostgreSQL connection
- `QDRANT_HOST`, `QDRANT_PORT` - Qdrant connection
- `OPENAI_API_KEY` - OpenAI embeddings
- `ENCRYPTION_DEV_MODE=true` - Local encryption (skip AWS KMS)
- `AUTH_MODE=headers` - Development auth mode

**vault-platform**:
- `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY` - Clerk auth
- `STRIPE_SECRET_KEY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` - Stripe billing
- `DATABASE_URI` - PostgreSQL connection
- `API_URL` - Gateway URL (for API calls)

**vault-api-gateway**:
- `BASE_API_URL` - Backend URL (vault-product)
- `GATEWAY_SECRET` - HMAC secret (must match vault-product)
- `CLERK_WEBHOOK_SECRET` - Webhook verification

### Quick Start (All Services)

```bash
# Terminal 1: Start vault-product infrastructure
cd vault-product
docker-compose up -d

# Terminal 2: Start vault-product API
cd vault-product
source venv/bin/activate
alembic upgrade head
uvicorn app.main:app --reload

# Terminal 3: Start vault-api-gateway
cd vault-api-gateway
npm run dev

# Terminal 4: Start vault-platform
cd vault-platform
pnpm run dev

# Access points:
# - Frontend: http://localhost:3000
# - API Gateway: http://localhost:9000
# - Backend API: http://localhost:8000
# - Jaeger UI: http://localhost:16686
```

---

## Deployment Architecture

### Production Infrastructure

#### vault-product (Backend API)

**Platform**: Kubernetes (K3s) on Oracle Cloud Infrastructure (OCI)

**Infrastructure Details**:
- **Compute**: ARM64 Ampere instances (cost-effective, energy-efficient)
- **Networking**: Tailscale VPN (no public endpoint, zero-trust network)
- **Container Registry**: OCI Container Registry (OCIR)
- **Cluster**: K3s (lightweight Kubernetes distribution)
- **Scaling**: Horizontal Pod Autoscaler (HPA) based on CPU/memory

**CI/CD Pipeline** (`.github/workflows/build-and-push-images.yml`):
1. Trigger: Push to main, Git tags (v*), manual workflow dispatch
2. Build: Multi-stage Docker build
3. Push: Two images to OCIR:
   - `api_image`: Main API service
   - `backup_image`: Background jobs (key rotation, backups)
4. Tag: Short git SHA (main) or release version (tags)
5. Artifacts: `image-manifest.json` with fully-qualified image references

**Deployment**:
```bash
# Access control plane via Tailscale
ssh <tailscale-ip>

# Deploy via kubectl
kubectl apply -f k8s/vault-api-deployment.yaml
kubectl apply -f k8s/vault-api-service.yaml
kubectl apply -f k8s/vault-api-hpa.yaml

# Rolling update
kubectl set image deployment/vault-api vault-api=<new-image>
kubectl rollout status deployment/vault-api
```

**Databases**:
- **PostgreSQL**: Managed service or self-hosted with replication
- **Qdrant**: Self-hosted in K8s with persistent volumes

#### vault-platform (Frontend)

**Recommended Platform**: Vercel

**Vercel Benefits**:
- Automatic builds from Git pushes
- Edge network with global CDN
- Automatic HTTPS with custom domains
- Environment variable management
- Preview deployments for PRs
- Vercel Analytics built-in

**Vercel Deployment**:
1. Connect GitHub repository to Vercel
2. Configure environment variables in Vercel dashboard
3. Set build command: `pnpm run build`
4. Set output directory: `.next`
5. Automatic deployments on push to main

**Alternative: Docker Deployment**:
```bash
# Build Docker image
docker build -t vault-platform:latest .

# Run container
docker run -p 3000:3000 \
  -e NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=... \
  -e CLERK_SECRET_KEY=... \
  -e DATABASE_URI=... \
  vault-platform:latest
```

#### vault-api-gateway (API Gateway)

**Platform**: Zuplo Cloud (Edge Gateway)

**Zuplo Benefits**:
- Global edge deployment (200+ locations)
- Automatic scaling
- Built-in rate limiting and caching
- OpenAPI-first configuration
- Integrated observability

**Deployment**:
1. Push changes to Git repository
2. Zuplo automatically detects changes
3. Deploys to edge locations within minutes
4. No manual configuration required

**Environment Configuration**:
- Set environment variables in Zuplo dashboard
- Configure per-environment (dev, staging, production)

### Environment Separation

#### Development
- **vault-product**: Local Docker Compose (PostgreSQL, Qdrant, Jaeger)
- **vault-platform**: Local Next.js dev server (http://localhost:3000)
- **vault-api-gateway**: Local Zuplo dev server (http://localhost:9000)
- **Auth**: Development Clerk instance
- **Payments**: Stripe test mode
- **Encryption**: PBKDF2 (no AWS KMS required)

#### Staging
- **vault-product**: Kubernetes staging namespace, staging database
- **vault-platform**: Vercel preview deployment
- **vault-api-gateway**: Zuplo staging environment
- **Auth**: Staging Clerk instance
- **Payments**: Stripe test mode
- **Encryption**: AWS KMS (staging key)

#### Production
- **vault-product**: Kubernetes production namespace, production database with replication
- **vault-platform**: Vercel production deployment
- **vault-api-gateway**: Zuplo production environment
- **Auth**: Production Clerk instance
- **Payments**: Stripe live mode
- **Encryption**: AWS KMS (production key with rotation)

### Monitoring & Observability

#### vault-product
- **Distributed Tracing**: OpenTelemetry with Jaeger backend
  - Automatic instrumentation for FastAPI, SQLAlchemy, httpx
  - Custom spans for encryption, embedding, search operations
- **Metrics**: Prometheus client exposes metrics at `/metrics`
  - Request counts, latency histograms, error rates
  - Custom metrics: encryption operations, key rotations, search queries
- **Logging**: Structured JSON logs with correlation IDs
- **Alerts**: Prometheus Alertmanager for critical issues

#### vault-platform
- **Analytics**: Vercel Analytics (page views, performance)
- **Error Tracking**: Sentry (recommended, not configured)
- **Logging**: Vercel logs (stdout/stderr)

#### vault-api-gateway
- **Logging**: Zuplo built-in logging with request/response capture
- **Analytics**: Zuplo dashboard (traffic, latency, errors)
- **Alerts**: Zuplo alerting for rate limits, errors, latency spikes

### Backup & Disaster Recovery

**Database Backups** (vault-product):
- **PostgreSQL**: Automated daily backups via `pg_dump`
- **Storage**: Oracle Cloud Object Storage (OCI)
- **Retention**: 30 days
- **Script**: `vault-product/scripts/backup_db.sh`

**Database Backups** (vault-platform):
- **PostgreSQL**: Managed service automatic backups (if using managed DB)
- **Retention**: 7-30 days based on tier

**Qdrant Backups**:
- **Method**: Snapshot API with collection export
- **Frequency**: Weekly
- **Storage**: OCI Object Storage

**Disaster Recovery Plan**:
1. Restore PostgreSQL from latest backup
2. Restore Qdrant collections from snapshot
3. Redeploy services via CI/CD pipeline
4. Validate encryption keys accessible via AWS KMS
5. Run health checks and smoke tests
6. Update DNS if necessary

---

## Key Features & Capabilities

### Core Features

- **Multi-Tenant Isolation with RLS**: Complete data separation between organizations using PostgreSQL row-level security policies, ensuring one tenant cannot access another's data

- **End-to-End Encryption with Automated Key Rotation**: AES-256-GCM encryption for all stored knowledge, AWS KMS-managed encryption keys with zero-downtime rotation, version tracking, and historical key access

- **Semantic Search with Freshness Reranking**: Vector-based search using OpenAI embeddings (1536 or 3072 dimensions), stored in Qdrant with HNSW indexing, time-decay algorithm boosts recent notes for relevance

- **PII Detection and Redaction**: Microsoft Presidio-based automatic detection of emails, phone numbers, SSNs, credit cards, and names before embedding generation, preventing sensitive data in vectors

- **Near-Duplicate Detection**: SimHash fingerprinting with PostgreSQL trigram similarity matching, prevents duplicate content with configurable thresholds (HTTP 409 on duplicates)

- **OAuth2 Authentication**: Clerk-based authentication with social logins (Google, GitHub), organization support, automatic JWT refresh, and middleware-protected routes

- **Subscription Management with Dual Model**: Flexible billing where users can have BOTH personal AND organization subscriptions simultaneously, Stripe integration with async webhook processing

- **API Key Management with Scopes**: Generate programmatic access keys with granular scopes (project/topic/tag restrictions), JWT-based tokens with revocation counters for instant invalidation

- **Webhook Synchronization**: Fire-and-forget async processing with idempotency tracking, platform-to-product user/org/membership sync, HMAC-verified for security

- **MCP (Model Context Protocol) Support**: Auto-generates LLM-compatible tools from all 23+ API endpoints, preserves OpenAPI schemas and documentation, HTTP transport for LLM integrations

- **Distributed Tracing and Metrics**: OpenTelemetry instrumentation for FastAPI, SQLAlchemy, and httpx, Jaeger UI for trace visualization, Prometheus metrics export for monitoring

### Security Features

- **Three-Layer Authentication**: Clerk OAuth2 for users, Gateway HMAC for request integrity, API key JWTs for programmatic access

- **RBAC/ABAC Authorization**: Organization roles (org_admin, admin, member), Project roles (project_admin, writer, reader, auditor), Attribute-based topic/tag allow/deny lists

- **Request Signing**: HMAC-SHA256 signatures on gateway headers prevent tampering, timestamp validation prevents replay attacks (configurable TTL)

- **Webhook Verification**: Svix signature validation for inbound webhooks, Gateway HMAC signatures for outbound webhooks to backend

- **Encryption Key Management**: One DEK per organization wrapped by AWS KMS master key, AAD binding to org/project/note context prevents ciphertext substitution

- **Audit Trail**: Immutable audit logs for compliance, tracks all data access and modifications with user/timestamp

### Developer Features

- **OpenAPI Documentation**: Auto-generated interactive docs at `/docs`, complete with request/response schemas and examples

- **Type Safety**: Python type hints with mypy checking, TypeScript strict mode in frontend and gateway

- **Testing Infrastructure**: Comprehensive pytest suite with unit and integration tests, Docker Compose for test dependencies

- **Local Development**: No cloud dependencies required (PBKDF2 encryption, mock webhooks), hot-reload for all services

- **CI/CD Pipelines**: GitHub Actions for backend builds and deploys, Vercel auto-deploy for frontend, Zuplo auto-deploy for gateway

- **Observability**: Structured JSON logging with correlation IDs, distributed tracing with OpenTelemetry, Prometheus metrics export

### Operational Features

- **Zero-Downtime Key Rotation**: Automated DEK rotation with version tracking, historical keys remain accessible for decryption, batch re-encryption with progress tracking

- **Horizontal Scaling**: Stateless API design enables horizontal pod autoscaling, connection pooling for databases, async request processing

- **Rate Limiting**: Configurable per-endpoint rate limits in gateway, prevents abuse and DDoS attacks

- **Caching**: Gateway-level caching for JWKS endpoint (5 min TTL), Stripe product catalog caching in platform (5 min TTL)

- **Health Checks**: Multi-layer health endpoints for service, database, and subsystems, Kubernetes liveness and readiness probes

- **Feature Flags**: Clerk private metadata for server-side feature flags, Stripe Product Features API for subscription-based features

---

## Repository Structure

```
vault/
├── .gitmodules                       # Git submodule configuration
├── claude.md                         # This file - comprehensive project documentation
│
├── vault-product/                    # Python Backend API
│   ├── app/
│   │   ├── main.py                  # FastAPI application entry point
│   │   ├── core/
│   │   │   ├── config.py           # Pydantic settings management
│   │   │   └── secrets.py          # AWS Secrets Manager integration
│   │   ├── db/
│   │   │   ├── postgres.py         # PostgreSQL connection and session
│   │   │   └── qdrant.py           # Qdrant client initialization
│   │   ├── models/                  # SQLAlchemy ORM models
│   │   │   ├── org.py
│   │   │   ├── project.py
│   │   │   ├── user.py
│   │   │   ├── api_key.py
│   │   │   ├── org_keys.py
│   │   │   └── ...
│   │   ├── schemas/                 # Pydantic request/response schemas
│   │   ├── api/                     # API route handlers
│   │   │   ├── orgs.py
│   │   │   ├── projects.py
│   │   │   ├── notes.py
│   │   │   ├── search.py
│   │   │   ├── api_keys.py
│   │   │   ├── key_rotation.py
│   │   │   ├── webhooks.py
│   │   │   └── auth/
│   │   ├── services/                # Business logic services
│   │   │   ├── ingestion.py        # Note ingestion pipeline
│   │   │   ├── embedding.py        # OpenAI embedding generation
│   │   │   ├── pii_redaction.py    # Presidio PII detection
│   │   │   ├── dedup_notes.py      # SimHash duplicate detection
│   │   │   ├── key_rotation.py     # DEK rotation executor
│   │   │   └── filters.py          # ABAC filter builder
│   │   ├── encryption/
│   │   │   └── vault_encryptor.py  # AES-256-GCM encryption
│   │   ├── auth/
│   │   │   ├── gateway_hmac.py     # HMAC signature validation
│   │   │   └── rbac_resolver.py    # Role-based access control
│   │   └── middleware/
│   │       ├── auth.py             # Authentication middleware
│   │       └── tracing.py          # OpenTelemetry tracing
│   ├── alembic/                     # Database migrations
│   │   └── versions/
│   ├── tests/
│   │   ├── unit/                    # Unit tests
│   │   └── integration/             # Integration tests
│   ├── scripts/
│   │   ├── quick_test.sh           # End-to-end test script
│   │   ├── verify_rotation.sh      # Key rotation verification
│   │   └── backup_db.sh            # Database backup script
│   ├── requirements.txt             # Python dependencies
│   ├── Dockerfile                   # Multi-stage Docker build
│   ├── docker-compose.yml           # Local development services
│   ├── .env.example                 # Example environment variables
│   └── README.md                    # Backend-specific documentation
│
├── vault-platform/                  # Next.js Frontend
│   ├── app/
│   │   ├── layout.tsx              # Root layout with ClerkProvider
│   │   ├── page.tsx                # Landing page (Strapi CMS)
│   │   ├── pricing/page.tsx        # Pricing table
│   │   ├── account/page.tsx        # Dashboard (protected)
│   │   ├── organization/
│   │   │   ├── create/page.tsx
│   │   │   ├── list/page.tsx
│   │   │   └── profile/page.tsx
│   │   ├── billing/
│   │   └── api/
│   │       ├── stripe/             # Stripe API routes
│   │       │   ├── products/route.ts
│   │       │   ├── create-checkout-session/route.ts
│   │       │   ├── webhook/route.ts
│   │       │   └── ...
│   │       └── unifiedmemory/      # UnifiedMemory API proxy
│   │           ├── projects/route.ts
│   │           └── api-keys/route.ts
│   ├── components/
│   │   ├── landing/                # Marketing components
│   │   ├── account/                # Dashboard components
│   │   ├── ui/                     # ShadCN UI components
│   │   └── PricingTable.tsx        # Pricing table component
│   ├── lib/
│   │   ├── subscription-features.ts # Feature checking helpers
│   │   ├── clerk-metadata-sync.ts  # Clerk metadata sync
│   │   ├── stripe-db-helpers.ts    # Stripe database operations
│   │   ├── stripe-webhooks/        # Webhook handlers
│   │   │   ├── index.ts
│   │   │   ├── event-router.ts
│   │   │   ├── async-processor.ts
│   │   │   └── handlers/
│   │   ├── unifiedmemory.ts        # UnifiedMemory API client
│   │   └── strapi.ts               # Strapi CMS client
│   ├── prisma/
│   │   └── schema.prisma           # Database schema (7 models)
│   ├── scripts/
│   │   └── dev.sh                  # Development orchestration
│   ├── package.json                # Node dependencies
│   ├── Dockerfile                  # Multi-stage Docker build
│   ├── .env.example                # Example environment variables
│   └── README.md                   # Frontend-specific documentation
│
└── vault-api-gateway/              # Zuplo API Gateway
    ├── config/
    │   ├── routes.oas.json         # OpenAPI 3.1.0 routes (24 endpoints)
    │   └── policies.json           # 6 middleware policies
    ├── modules/                     # Custom TypeScript middleware
    │   ├── append-gateway-headers.ts    # HMAC signature generation
    │   ├── clerk-webhook-verify.ts      # Svix webhook verification
    │   ├── append-webhook-signature.ts  # Outbound webhook signing
    │   ├── debug-outbound.ts            # Response logging
    │   └── zuplo.runtime.ts             # OAuth plugin init
    ├── docs/                        # API documentation workspace
    ├── package.json                 # Node dependencies (Zuplo runtime)
    ├── tsconfig.json                # TypeScript config (ES2022, WebWorker)
    ├── zuplo.jsonc                  # Zuplo metadata
    ├── env.example                  # Example environment variables
    └── README.md                    # Gateway-specific documentation
```

---

## Contributing & Resources

### Detailed Documentation

Each submodule has its own comprehensive README with setup instructions and technical details:

- **Backend API**: `vault-product/README.md`
- **Frontend**: `vault-platform/README.md`
- **API Gateway**: `vault-api-gateway/README.md`

### Development Workflow

1. **Feature Development**:
   - Create feature branch from main
   - Develop and test locally
   - Write unit and integration tests
   - Submit pull request with description

2. **Code Quality**:
   - **vault-product**: Run `black`, `ruff`, `mypy` before commit
   - **vault-platform**: Run `pnpm run lint` before commit
   - All tests must pass (`pytest`, `pnpm test`)

3. **Commit Messages**:
   - Use conventional commits format: `type(scope): description`
   - Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`
   - Example: `feat(api): add near-duplicate detection for notes`

4. **Pull Requests**:
   - Link related issues
   - Include screenshots for UI changes
   - Update documentation if needed
   - Ensure CI/CD pipelines pass

### Architecture Decisions

Key architectural decisions and trade-offs are documented in:
- ADRs (Architecture Decision Records) in `docs/adr/` (if applicable)
- Inline code comments for complex logic
- This claude.md file for high-level architecture

### Support & Contact

- **Issues**: GitHub Issues in respective submodule repositories
- **Discussions**: GitHub Discussions for questions and feature requests
- **Security**: Report security vulnerabilities via email (not public issues)

### Versioning

- **Semantic Versioning**: Major.Minor.Patch (e.g., v1.2.3)
- **Release Tags**: Git tags on main branch (e.g., `v1.0.0`)
- **Changelog**: Maintained in `CHANGELOG.md` per submodule

### License

See `LICENSE` file in repository root or individual submodules for licensing information.

---

**Last Updated**: 2025-12-09
**Documentation Version**: 1.0
**Project Status**: Production Ready

---

For questions or contributions, please refer to the individual submodule README files or open an issue in the respective repository.
