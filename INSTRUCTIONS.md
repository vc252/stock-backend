# Stockpiece Backend Rebuild - Project Instructions

## Executive Summary

This document outlines the complete architectural plan for rebuilding the Stockpiece backend, a manga stock trading simulation platform designed to handle 100K+ concurrent users.

### Technology Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Framework | NestJS (TypeScript) | Team familiarity, faster iteration, excellent structure |
| Database | PostgreSQL 16 | ACID transactions, strong relationships, audit requirements |
| Cache | Redis 7 | Sessions, leaderboards, rate limiting, job queues |
| ORM | Prisma | Type-safe, excellent developer experience |
| Queue | BullMQ | Background job processing on Redis |
| Auth | Passport.js + JWT | Industry standard, NestJS integration |
| Logging | Pino | Structured JSON logging, high performance |
| File Storage | Cloudinary | Keep existing infrastructure |

---

## Architecture Overview

### Pattern: Modular Monolith with Event Sourcing

```
┌──────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY LAYER                            │
│  Rate Limiting → Authentication → Validation                         │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      APPLICATION MODULES                             │
│  Auth │ User │ Stock │ Trading │ Wallet │ Window │ Admin │ Analytics│
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       SHARED SERVICES                                │
│  Event Bus │ Cache │ Queue │ Algorithm Engine │ Config │ Logger     │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                    │
│  PostgreSQL (Primary Data) │ Redis (Cache/Sessions/Queues)          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Module Responsibilities

### Auth Module
- User registration/login
- JWT access + refresh token management
- Admin authentication (separate flow)
- Session management via Redis

### User Module
- Profile management
- Portfolio views (read-only)
- Leaderboard queries
- Avatar/settings

### Wallet Module ⚠️ CRITICAL
- **ALL balance changes MUST go through WalletService**
- Credit operations: bonuses, sells, refunds
- Debit operations: purchases
- Transaction history with full audit trail
- Optimistic locking for concurrency

### Stock Module
- Stock CRUD (admin only)
- Stock listings and search
- Price history queries
- Stock metadata

### Trading Module ⚠️ CRITICAL
- Buy/Sell operations (atomic transactions)
- Order validation
- Portfolio updates
- Emits events for audit trail

### Window Module
- Trading window state machine
- Schedule management
- Window type support (chapter/bonus/special)
- Settlement orchestration

### Algorithm Module
- Pluggable price calculation engines
- Algorithm registry
- Admin-tunable parameters
- A/B testing support

### Admin Module
- User management
- Stock management
- Window controls
- Manual interventions

### Audit Module
- Event sourcing for critical paths
- Change data capture
- Manipulation detection
- Compliance reports

### Analytics Module
- Real-time market stats
- Historical analysis
- Leaderboard management
- Admin dashboards

---

## Trading Window State Machine

```
CREATED → SCHEDULED → OPEN → TRADING ⇄ PAUSED
                                ↓
                          TRADE_CLOSE → SETTLING → SETTLED → CLOSED
                                ↓
                            CANCELLED
```

### Valid Transitions
- `CREATED` → `SCHEDULED` (schedule)
- `SCHEDULED` → `OPEN` (open)
- `OPEN` → `TRADING` (startTrading)
- `TRADING` → `PAUSED` (pause)
- `PAUSED` → `TRADING` (resume)
- `TRADING` → `TRADE_CLOSE` (closeTrading)
- `TRADE_CLOSE` → `SETTLING` (startSettlement)
- `SETTLING` → `SETTLED` (completeSettlement)
- `SETTLED` → `CLOSED` (close)
- Any non-terminal → `CANCELLED` (cancel)

---

## Database Schema Reference

### Core Tables

```
users
├── id (UUID, PK)
├── username (unique)
├── email (unique)
├── password_hash
├── avatar_url
├── role (USER/ADMIN)
├── is_verified
├── created_at
└── updated_at

wallets
├── id (UUID, PK)
├── user_id (FK → users)
├── balance (DECIMAL)
├── locked_balance (DECIMAL)
├── version (optimistic lock)
├── created_at
└── updated_at

wallet_transactions
├── id (UUID, PK)
├── wallet_id (FK → wallets)
├── type (CREDIT/DEBIT)
├── amount (DECIMAL)
├── balance_before
├── balance_after
├── reason (enum)
├── reference_type
├── reference_id
├── metadata (JSONB)
├── idempotency_key
└── created_at

stocks
├── id (UUID, PK)
├── name (unique)
├── symbol (unique)
├── display_name
├── image_url
├── description
├── is_active
├── created_at
└── updated_at

trading_windows
├── id (UUID, PK)
├── type (CHAPTER/BONUS/WEEK)
├── reference_number
├── status (state machine)
├── starts_at
├── trading_starts_at
├── trading_ends_at
├── settlement_starts_at
├── settlement_ends_at
├── ends_at
├── metadata (JSONB)
├── created_at
└── updated_at

trades
├── id (UUID, PK)
├── user_id (FK → users)
├── stock_id (FK → stocks)
├── window_id (FK → trading_windows)
├── type (BUY/SELL)
├── quantity
├── price_per_unit
├── total_value
├── wallet_tx_id (FK → wallet_transactions)
├── status
├── idempotency_key
└── created_at

audit_events
├── id (UUID, PK)
├── event_type
├── aggregate_type
├── aggregate_id
├── actor_id
├── actor_type (USER/ADMIN/SYSTEM)
├── payload (JSONB)
├── metadata (JSONB)
├── ip_address
├── user_agent
├── window_id (nullable)
└── created_at
```

---

## Rollback Levels

### Level 1: Single Trade Rollback
- Scenario: Admin needs to reverse a specific trade
- Process: Compensating wallet transaction, portfolio update, mark trade as rolled back

### Level 2: User Session Rollback
- Scenario: Reverse all trades from a user in a window
- Process: Execute Level 1 for each trade in reverse order

### Level 3: Full Window Rollback
- Scenario: Major issue, reset entire window
- Process: Restore from pre-window snapshots
- Requires: Dual-admin approval

### Level 4: Price Correction
- Scenario: Algorithm error, prices calculated wrong
- Process: Recalculate prices, update records, no trade rollback

---

## Implementation Phases

### Phase 0: Foundation Setup [Week 1]
- NestJS project scaffolding
- PostgreSQL + Redis setup (Docker)
- Prisma configuration
- Logging, error handling, Swagger

### Phase 1: Core User & Auth [Weeks 2-3]
- User CRUD
- JWT authentication
- Admin authentication
- Rate limiting on auth endpoints

### Phase 2: Wallet & Stock Management [Weeks 4-5]
- Centralized wallet service
- Stock CRUD
- Price history

### Phase 3: Trading Engine [Weeks 6-8]
- Portfolio module
- Buy/sell operations (atomic)
- Trade validation
- Event publishing

### Phase 4: Window Lifecycle [Weeks 9-10]
- State machine implementation
- Scheduled transitions
- Pre-window snapshots
- Admin controls

### Phase 5: Algorithm & Settlement [Weeks 11-12]
- Pluggable algorithm engine
- Settlement process
- Price calculation audit

### Phase 6: Admin & Analytics [Weeks 13-14]
- Admin tools
- Coupon/referral system
- Leaderboard
- Analytics APIs

### Phase 7: Performance & Scale [Weeks 15-16]
- Caching strategy
- Rate limiting enhancement
- Load testing (target: 100K concurrent)

### Phase 8: Rollback & Recovery [Week 17]
- All rollback levels
- Recovery tools
- Dual-admin approval flows

### Phase 9: Polish & Launch [Weeks 18-19]
- Security audit
- Test coverage
- Documentation
- Migration scripts

---

## Key Architectural Principles

### 1. Single Source of Truth
- Wallet service for ALL balance changes
- Window service for ALL state transitions
- Event bus for ALL cross-module communication

### 2. Event-Driven Audit
- Every state change emits an event
- Audit module subscribes to all events
- Complete reconstruction possible from events

### 3. Idempotency Everywhere
- All mutations have idempotency keys
- Safe to retry any operation
- No duplicate transactions possible

### 4. Transactional Boundaries
- Database transactions wrap related changes
- Compensating transactions for failures
- Clear rollback procedures

### 5. Configuration Over Code
- Algorithm parameters in database
- Window schedules configurable
- Feature flags for gradual rollout

### 6. Separation of Concerns
- Controllers: HTTP handling only
- Services: Business logic
- Repositories: Data access
- No business logic in controllers

---

## Project Structure

```
src/
├── main.ts
├── app.module.ts
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   ├── pipes/
│   ├── dto/
│   │   ├── api-response.dto.ts
│   │   └── pagination.dto.ts
│   └── constants/
│       ├── error-codes.ts
│       └── app.constants.ts
├── config/
│   ├── config.module.ts
│   ├── database.config.ts
│   ├── redis.config.ts
│   └── app.config.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── modules/
│   ├── auth/
│   ├── user/
│   ├── wallet/
│   ├── stock/
│   ├── trading/
│   ├── window/
│   ├── algorithm/
│   ├── admin/
│   ├── audit/
│   └── analytics/
└── shared/
    ├── services/
    │   ├── logger.service.ts
    │   └── cache.service.ts
    └── events/
        └── event-emitter.module.ts
```

---

## Getting Started

### Prerequisites
- Node.js 20+
- Docker & Docker Compose
- PostgreSQL 16 (via Docker)
- Redis 7 (via Docker)

### Initial Setup

```bash
# Create new NestJS project
npx @nestjs/cli new stockpiece-v3

# Install core dependencies
npm install @nestjs/config @nestjs/swagger
npm install prisma @prisma/client
npm install ioredis bullmq
npm install argon2
npm install @nestjs/passport passport passport-jwt
npm install class-validator class-transformer
npm install pino nestjs-pino

# Dev dependencies
npm install -D @types/passport-jwt
```

### Docker Compose

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: stockpiece
      POSTGRES_PASSWORD: stockpiece
      POSTGRES_DB: stockpiece
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Initialize Prisma

```bash
npx prisma init
# Edit prisma/schema.prisma with initial tables
npx prisma migrate dev --name init
```

---

## API Versioning

All API endpoints should be prefixed with `/api/v1/`

Example endpoints:
- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `GET /api/v1/users/me`
- `GET /api/v1/stocks`
- `POST /api/v1/trades/buy`
- `GET /api/v1/market/status`
- `POST /api/v1/admin/windows`

---

## Environment Variables

```env
# Application
NODE_ENV=development
PORT=3000
API_PREFIX=api/v1

# Database
DATABASE_URL=postgresql://stockpiece:stockpiece@localhost:5432/stockpiece

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# JWT
JWT_SECRET=your-secret-key
JWT_ACCESS_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=7d

# Cloudinary (existing)
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

# Rate Limiting
RATE_LIMIT_TTL=60
RATE_LIMIT_MAX=100
```

---

## Testing Requirements

- Unit test coverage: >80%
- Integration tests for all API endpoints
- E2E tests for critical flows:
  - User registration → login → trading
  - Window lifecycle
  - Settlement process
  - Rollback operations
- Load testing: Verify 100K concurrent user capability

---

## Performance Targets

- API response time: <100ms (p95)
- Database query time: <50ms (p95)
- Trading operation: <200ms (end-to-end)
- Leaderboard refresh: <1 second
- Settlement process: <5 minutes for full window

---

## Security Checklist

- [ ] Input validation on all endpoints
- [ ] SQL injection prevention (Prisma parameterized queries)
- [ ] XSS prevention (no HTML in responses)
- [ ] Rate limiting on all endpoints
- [ ] JWT token rotation
- [ ] Password hashing with Argon2
- [ ] Admin endpoints behind separate auth
- [ ] Audit logging for all admin actions
- [ ] HTTPS in production
- [ ] Secrets in environment variables only
