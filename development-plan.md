# Dynamic Pricing Engine — Phased Development Plan

> Project: 138-dynamic-pricing-engine · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12+ | ML-heavy domain (RL agents, demand forecasting, elasticity modelling) requires first-class NumPy/SciPy/scikit-learn/PyTorch ecosystem. All competitor APIs offer Python SDKs. |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 generation aligns with standards.md; async support for high-concurrency price scraping and webhook delivery; Pydantic v2 for request/response validation. |
| Database | PostgreSQL 16 + TimescaleDB 2.x | Data model suggestion 4 (time-series/analytics-first) is the best fit — competitor prices, sales, and repricing events are fundamentally time-series workloads. TimescaleDB continuous aggregates eliminate ETL for Omnibus reference price computation and daily sales rollups. Standard relational tables for dimension data. |
| Migrations | Alembic 1.14+ | Standard SQLAlchemy migration tool; supports custom DDL for TimescaleDB hypertable creation and compression policies. |
| ORM / query builder | SQLAlchemy 2.0 (Core + ORM) | Handles both relational dimension tables and raw SQL for TimescaleDB-specific queries (time_bucket, continuous aggregates). |
| Task queue | Celery 5.4+ with Redis broker | Repricing runs, competitor scraping, demand forecast generation, and webhook delivery are all async workloads. Redis doubles as cache layer for hot price data. |
| Cache | Redis 7+ | Cache current competitor prices, product prices, and frequently-read strategy configurations. Sub-millisecond reads for repricing hot path. |
| ML / RL framework | scikit-learn + XGBoost + Stable-Baselines3 (PPO) | scikit-learn for elasticity estimation (log-log regression), XGBoost for demand forecasting, Stable-Baselines3 (PyTorch-backed) for RL pricing agents. |
| Frontend | Next.js 15 (React 19) + Tailwind CSS + shadcn/ui | Dashboard-heavy application (pricing analytics, competitor tracking, experiment results). Next.js provides SSR for initial load, React for interactive strategy builder. |
| Charting | Recharts 2.x | React-native charting for time-series price trends, demand forecasts, experiment results. Lightweight alternative to D3. |
| API authentication | OAuth 2.0 client credentials + JWT (RFC 6749, RFC 7519) | Standards-aligned (per standards.md). API clients authenticate via client_id/client_secret, receive JWT bearer tokens with scoped claims. |
| User authentication | Session-based with bcrypt | Web UI users authenticate via email/password with bcrypt hashing. OIDC federation deferred to post-MVP. |
| Webhook delivery | HMAC-SHA256 signed payloads | Industry standard (Shopify, GitHub pattern). Outbox pattern with retry queue for at-least-once delivery. |
| Containerisation | Docker + docker-compose | Multi-service architecture (API, worker, scheduler, Redis, PostgreSQL). docker-compose for local dev; Kubernetes manifests deferred. |
| Testing | pytest 8+ (backend), Vitest (frontend), Playwright (E2E) | pytest-asyncio for async FastAPI tests; pytest-httpx for HTTP mocking; factory-boy for test fixtures. |
| Code quality | Ruff (linter + formatter), mypy (type checker), ESLint + Prettier (frontend) | Ruff replaces flake8+black+isort with a single fast tool. mypy for strict type checking of API contracts. |
| Package manager | uv (Python), pnpm (Node.js) | uv for fast, reproducible Python dependency resolution. pnpm for efficient node_modules. |
| API documentation | Auto-generated OpenAPI 3.1 via FastAPI | standards.md identifies OpenAPI 3.1 as the industry standard for pricing APIs. FastAPI generates this natively. |
| Event documentation | AsyncAPI 3.x definitions | Webhook and queue event schemas documented in AsyncAPI format per standards.md. |
| Product identifiers | GS1 GTIN (ISO standard) | All product matching and cross-system integration keyed on GTIN per GS1 standard from standards.md. |
| Currency handling | ISO 4217 CHAR(3) codes + python-money | All monetary values paired with ISO 4217 currency code. python-money for arithmetic with currency awareness. |
| Competitor scraping | httpx + selectolax + rotating proxies | httpx for async HTTP; selectolax (Modest-based) for fast HTML parsing; proxy rotation for resilience. |

### Project Structure

```
dynamic-pricing-engine/
├── pyproject.toml
├── alembic.ini
├── Dockerfile
├── Dockerfile.worker
├── docker-compose.yml
├── .env.example
├── README.md
├── asyncapi.yaml
├── src/
│   └── pricing_engine/
│       ├── __init__.py
│       ├── main.py                         # FastAPI app entry point
│       ├── config.py                       # Settings via pydantic-settings
│       ├── database.py                     # SQLAlchemy engine, session factory
│       ├── dependencies.py                 # FastAPI dependency injection
│       ├── models/                         # SQLAlchemy ORM models
│       │   ├── __init__.py
│       │   ├── tenant.py
│       │   ├── user.py
│       │   ├── product.py
│       │   ├── channel.py
│       │   ├── competitor.py
│       │   ├── pricing_strategy.py
│       │   ├── experiment.py
│       │   ├── ml_model.py
│       │   └── timeseries.py              # Hypertable definitions
│       ├── schemas/                        # Pydantic request/response schemas
│       │   ├── __init__.py
│       │   ├── products.py
│       │   ├── channels.py
│       │   ├── competitors.py
│       │   ├── pricing.py
│       │   ├── experiments.py
│       │   ├── analytics.py
│       │   └── webhooks.py
│       ├── api/                            # FastAPI routers
│       │   ├── __init__.py
│       │   ├── auth.py
│       │   ├── products.py
│       │   ├── channels.py
│       │   ├── competitors.py
│       │   ├── pricing.py
│       │   ├── experiments.py
│       │   ├── analytics.py
│       │   ├── webhooks.py
│       │   └── health.py
│       ├── services/                       # Business logic layer
│       │   ├── __init__.py
│       │   ├── product_service.py
│       │   ├── competitor_service.py
│       │   ├── pricing_service.py
│       │   ├── repricing_engine.py
│       │   ├── experiment_service.py
│       │   ├── analytics_service.py
│       │   ├── webhook_service.py
│       │   └── compliance_service.py       # MAP, MSRP, Omnibus enforcement
│       ├── ml/                             # Machine learning modules
│       │   ├── __init__.py
│       │   ├── demand_forecast.py
│       │   ├── elasticity_estimator.py
│       │   ├── cross_elasticity.py
│       │   └── rl_agent.py
│       ├── integrations/                   # External platform connectors
│       │   ├── __init__.py
│       │   ├── base.py                     # Abstract channel connector
│       │   ├── shopify.py
│       │   ├── woocommerce.py
│       │   ├── amazon_sp.py
│       │   └── scraper.py                  # Competitor price scraping
│       ├── tasks/                          # Celery async tasks
│       │   ├── __init__.py
│       │   ├── celery_app.py
│       │   ├── scraping.py
│       │   ├── repricing.py
│       │   ├── forecasting.py
│       │   ├── webhooks.py
│       │   └── scheduled.py
│       └── utils/
│           ├── __init__.py
│           ├── currency.py
│           ├── gtin.py
│           └── security.py
├── migrations/
│   └── versions/
├── tests/
│   ├── conftest.py
│   ├── factories.py                       # factory-boy model factories
│   ├── fixtures/                          # Static test data files
│   ├── unit/
│   │   ├── test_pricing_rules.py
│   │   ├── test_elasticity.py
│   │   ├── test_compliance.py
│   │   ├── test_demand_forecast.py
│   │   └── test_currency.py
│   ├── integration/
│   │   ├── test_api_products.py
│   │   ├── test_api_pricing.py
│   │   ├── test_repricing_engine.py
│   │   ├── test_competitor_scraping.py
│   │   └── test_webhooks.py
│   └── e2e/
│       ├── test_full_repricing_flow.py
│       └── test_experiment_lifecycle.py
└── frontend/
    ├── package.json
    ├── next.config.ts
    ├── tailwind.config.ts
    ├── src/
    │   ├── app/
    │   │   ├── layout.tsx
    │   │   ├── page.tsx
    │   │   ├── dashboard/
    │   │   ├── products/
    │   │   ├── competitors/
    │   │   ├── pricing/
    │   │   ├── experiments/
    │   │   └── settings/
    │   ├── components/
    │   │   ├── ui/                         # shadcn/ui components
    │   │   ├── charts/
    │   │   ├── pricing/
    │   │   └── layout/
    │   ├── lib/
    │   │   ├── api-client.ts
    │   │   └── utils.ts
    │   └── types/
    │       └── api.ts                     # Generated from OpenAPI spec
    └── tests/
```

---

## Phase 1: Foundation & Data Layer

### Purpose
Establish the project skeleton, database schema, configuration system, and core data access layer. After this phase, the API server boots, connects to PostgreSQL/TimescaleDB, runs migrations, and exposes health-check endpoints. All subsequent phases build on this foundation.

### Tasks

#### 1.1 — Project Scaffolding & Configuration

**What**: Create the Python package, dependency files, Docker setup, and configuration system.

**Design**:

```python
# src/pricing_engine/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Application
    app_name: str = "Dynamic Pricing Engine"
    debug: bool = False
    log_level: str = "INFO"

    # Database
    database_url: str = "postgresql+asyncpg://pricing:pricing@localhost:5432/pricing_engine"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"

    # Auth
    jwt_secret_key: str = "CHANGE-ME-IN-PRODUCTION"
    jwt_algorithm: str = "HS256"
    jwt_expiry_minutes: int = 60

    # Pricing defaults
    default_currency: str = "USD"
    repricing_interval_minutes: int = 60
    omnibus_lookback_days: int = 30
    max_price_change_pct: float = 0.15

    model_config = {"env_prefix": "PRICING_", "env_file": ".env"}
```

```yaml
# docker-compose.yml services
services:
  api:
    build: { context: ., dockerfile: Dockerfile }
    ports: ["8000:8000"]
    depends_on: [db, redis]
    env_file: .env

  worker:
    build: { context: ., dockerfile: Dockerfile.worker }
    depends_on: [db, redis]
    env_file: .env

  scheduler:
    build: { context: ., dockerfile: Dockerfile.worker }
    command: celery -A pricing_engine.tasks.celery_app beat
    depends_on: [redis]
    env_file: .env

  db:
    image: timescale/timescaledb:latest-pg16
    volumes: [pgdata:/var/lib/postgresql/data]
    environment:
      POSTGRES_DB: pricing_engine
      POSTGRES_USER: pricing
      POSTGRES_PASSWORD: pricing

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

**Testing**:
- `Unit: Settings loads from env vars with PRICING_ prefix → all fields populated`
- `Unit: Settings with missing DATABASE_URL → uses default localhost connection`
- `Unit: Settings with invalid log_level value → ValidationError`
- `Integration: docker-compose up → all 5 services healthy within 30 seconds`

#### 1.2 — Database Schema & Migrations

**What**: Implement the complete database schema (dimension tables + TimescaleDB hypertables) via Alembic migrations.

**Design**:

Adopts Data Model Suggestion 4 (Time-Series / Analytics-First) as the primary schema, with select elements from Suggestion 3 (Hybrid JSONB) for flexible rule parameters and channel configuration.

Dimension tables (standard PostgreSQL):
```python
# src/pricing_engine/models/tenant.py
from sqlalchemy import Column, String, Boolean, DateTime, func
from sqlalchemy.dialects.postgresql import UUID, JSONB
import uuid

class Tenant(Base):
    __tablename__ = "tenants"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False)
    slug = Column(String(100), nullable=False, unique=True)
    plan = Column(String(50), nullable=False, default="free")
    base_currency = Column(String(3), nullable=False, default="USD")
    timezone = Column(String(50), nullable=False, default="UTC")
    settings = Column(JSONB, nullable=False, default=dict)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

```python
# src/pricing_engine/models/product.py
class Product(Base):
    __tablename__ = "products"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.id", ondelete="CASCADE"), nullable=False)
    category_id = Column(UUID(as_uuid=True), ForeignKey("product_categories.id", ondelete="SET NULL"))
    sku = Column(String(100), nullable=False)
    gtin = Column(String(14))                          # GS1 GTIN
    name = Column(String(500), nullable=False)
    brand = Column(String(255))
    cost_price = Column(Numeric(12, 4))
    cost_currency = Column(String(3), default="USD")   # ISO 4217
    status = Column(String(20), nullable=False, default="active")
    tags = Column(ARRAY(Text), default=list)
    attributes = Column(JSONB, nullable=False, default=dict)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    __table_args__ = (UniqueConstraint("tenant_id", "sku"),)
```

Hypertable definitions (TimescaleDB):
```python
# src/pricing_engine/models/timeseries.py
class TSCompetitorPrice(Base):
    __tablename__ = "ts_competitor_prices"

    observed_at = Column(DateTime(timezone=True), nullable=False, primary_key=True)
    tenant_id = Column(UUID(as_uuid=True), nullable=False)
    competitor_id = Column(UUID(as_uuid=True), nullable=False)
    product_id = Column(UUID(as_uuid=True), nullable=False)
    match_id = Column(UUID(as_uuid=True), nullable=False)
    price = Column(Numeric(12, 4), nullable=False)
    currency = Column(String(3), nullable=False, default="USD")
    price_converted = Column(Numeric(12, 4))
    was_in_stock = Column(Boolean)
    is_promotional = Column(Boolean, nullable=False, default=False)
    shipping_cost = Column(Numeric(12, 4))
    source = Column(String(20), default="scrape")

class TSPriceChange(Base):
    __tablename__ = "ts_price_changes"

    changed_at = Column(DateTime(timezone=True), nullable=False, primary_key=True)
    tenant_id = Column(UUID(as_uuid=True), nullable=False)
    channel_product_id = Column(UUID(as_uuid=True), nullable=False)
    product_id = Column(UUID(as_uuid=True), nullable=False)
    channel_id = Column(UUID(as_uuid=True), nullable=False)
    old_price = Column(Numeric(12, 4))
    new_price = Column(Numeric(12, 4), nullable=False)
    currency = Column(String(3), nullable=False, default="USD")
    reason = Column(String(50), nullable=False)
    rule_id = Column(UUID(as_uuid=True))
    strategy_id = Column(UUID(as_uuid=True))
    experiment_id = Column(UUID(as_uuid=True))
    changed_by = Column(UUID(as_uuid=True))

class TSSale(Base):
    __tablename__ = "ts_sales"

    transaction_at = Column(DateTime(timezone=True), nullable=False, primary_key=True)
    tenant_id = Column(UUID(as_uuid=True), nullable=False)
    product_id = Column(UUID(as_uuid=True), nullable=False)
    channel_id = Column(UUID(as_uuid=True), nullable=False)
    quantity = Column(Integer, nullable=False)
    unit_price = Column(Numeric(12, 4), nullable=False)
    total_price = Column(Numeric(14, 4), nullable=False)
    currency = Column(String(3), nullable=False, default="USD")
    cost_price = Column(Numeric(12, 4))
    margin = Column(Numeric(12, 4))
    discount_amount = Column(Numeric(12, 4), default=0)
    customer_segment = Column(String(100))
    external_order_id = Column(String(255))
```

Alembic migration for hypertable creation:
```python
# migrations/versions/002_create_hypertables.py
def upgrade():
    # Create hypertables after table creation
    op.execute("SELECT create_hypertable('ts_competitor_prices', 'observed_at', chunk_time_interval => INTERVAL '1 day')")
    op.execute("SELECT create_hypertable('ts_price_changes', 'changed_at', chunk_time_interval => INTERVAL '7 days')")
    op.execute("SELECT create_hypertable('ts_sales', 'transaction_at', chunk_time_interval => INTERVAL '7 days')")
    op.execute("SELECT create_hypertable('ts_demand_forecasts', 'generated_at', chunk_time_interval => INTERVAL '7 days')")
    op.execute("SELECT create_hypertable('ts_inventory_snapshots', 'snapshot_at', chunk_time_interval => INTERVAL '1 day')")
    op.execute("SELECT create_hypertable('ts_repricing_events', 'evaluated_at', chunk_time_interval => INTERVAL '1 day')")

    # Compression policies
    op.execute("""
        ALTER TABLE ts_competitor_prices SET (
            timescaledb.compress,
            timescaledb.compress_segmentby = 'tenant_id, product_id, competitor_id',
            timescaledb.compress_orderby = 'observed_at DESC'
        )
    """)
    op.execute("SELECT add_compression_policy('ts_competitor_prices', INTERVAL '7 days')")
    # ... similar for other hypertables
```

**Testing**:
- `Integration: alembic upgrade head → all tables created, hypertables confirmed via timescaledb_information.hypertables`
- `Integration: alembic downgrade base → all tables dropped cleanly`
- `Unit: Product model with valid GTIN → validates 8-14 digit format`
- `Unit: Product model with duplicate (tenant_id, sku) → IntegrityError`
- `Integration: insert into ts_competitor_prices → row queryable via time_bucket`

#### 1.3 — Core API Server & Health Endpoints

**What**: Set up the FastAPI application with middleware, error handling, tenant-scoped dependency injection, and health endpoints.

**Design**:

```python
# src/pricing_engine/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(
    title="Dynamic Pricing Engine",
    version="0.1.0",
    description="AI-driven pricing optimization API",
    openapi_version="3.1.0",
)

app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])
```

```python
# src/pricing_engine/api/health.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter(tags=["health"])

@router.get("/health")
async def health_check() -> dict:
    return {"status": "healthy", "version": "0.1.0"}

@router.get("/health/ready")
async def readiness_check(db: AsyncSession = Depends(get_db)) -> dict:
    await db.execute(text("SELECT 1"))
    return {"status": "ready", "database": "connected"}
```

```python
# src/pricing_engine/dependencies.py
from fastapi import Depends, Header, HTTPException
from uuid import UUID

async def get_current_tenant(x_tenant_id: str = Header(...)) -> UUID:
    """Extract tenant ID from request header. Production uses JWT claims."""
    try:
        return UUID(x_tenant_id)
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid tenant ID")

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        yield session
```

**Testing**:
- `Integration: GET /health → 200, {"status": "healthy"}`
- `Integration: GET /health/ready with DB up → 200, {"status": "ready"}`
- `Integration: GET /health/ready with DB down → 503`
- `Integration: GET /docs → 200, OpenAPI 3.1 spec served`
- `Unit: get_current_tenant with valid UUID header → returns UUID`
- `Unit: get_current_tenant with invalid header → raises 400`

#### 1.4 — Authentication & API Client Management

**What**: Implement JWT-based API authentication with client credentials flow (RFC 6749) and user session auth for the web UI.

**Design**:

```python
# src/pricing_engine/schemas/auth.py
from pydantic import BaseModel
from datetime import datetime

class TokenRequest(BaseModel):
    client_id: str
    client_secret: str
    grant_type: str = "client_credentials"

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int
    scope: str

class UserLogin(BaseModel):
    email: str
    password: str

class JWTPayload(BaseModel):
    sub: str                  # client_id or user_id
    tenant_id: str
    scopes: list[str]
    exp: datetime
    iat: datetime
    token_type: str           # "api_client" or "user"
```

```python
# src/pricing_engine/api/auth.py
@router.post("/auth/token", response_model=TokenResponse)
async def create_token(request: TokenRequest, db: AsyncSession = Depends(get_db)) -> TokenResponse:
    client = await verify_client_credentials(db, request.client_id, request.client_secret)
    if not client:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    token = create_jwt(client_id=client.client_id, tenant_id=str(client.tenant_id), scopes=client.scopes)
    return TokenResponse(access_token=token, expires_in=settings.jwt_expiry_minutes * 60, scope=" ".join(client.scopes))

@router.post("/auth/login")
async def login(request: UserLogin, db: AsyncSession = Depends(get_db)) -> dict:
    user = await authenticate_user(db, request.email, request.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid email or password")
    token = create_jwt(client_id=str(user.id), tenant_id=str(user.tenant_id), scopes=["*"], token_type="user")
    return {"access_token": token, "user": {"id": str(user.id), "email": user.email, "name": user.name, "role": user.role}}
```

**Testing**:
- `Integration: POST /auth/token with valid client credentials → 200, JWT returned`
- `Integration: POST /auth/token with wrong secret → 401`
- `Integration: POST /auth/token with inactive client → 401`
- `Unit: JWT payload contains correct tenant_id, scopes, and expiry`
- `Unit: expired JWT → authentication fails`
- `Unit: JWT with insufficient scope → 403 on protected endpoint`
- `Integration: POST /auth/login with valid user → 200, JWT + user object returned`
- `Integration: POST /auth/login with wrong password → 401`

---

## Phase 2: Product Catalogue & Channel Management

### Purpose
Implement CRUD operations for products, categories, channels, and channel-product mappings. After this phase, users can manage their product catalogue, connect sales channels (Shopify, Amazon, etc.), and map products to channels with MAP/MSRP constraints. This is the foundational entity layer that all pricing logic operates on.

### Tasks

#### 2.1 — Product & Category CRUD

**What**: REST API for managing products and hierarchical categories with GS1 GTIN support.

**Design**:

```python
# src/pricing_engine/schemas/products.py
from pydantic import BaseModel, Field
from typing import Optional
from uuid import UUID
from decimal import Decimal

class ProductCreate(BaseModel):
    sku: str = Field(max_length=100)
    gtin: Optional[str] = Field(None, pattern=r"^\d{8,14}$")  # GS1 GTIN validation
    name: str = Field(max_length=500)
    brand: Optional[str] = None
    category_id: Optional[UUID] = None
    cost_price: Optional[Decimal] = Field(None, max_digits=12, decimal_places=4)
    cost_currency: str = Field(default="USD", pattern=r"^[A-Z]{3}$")  # ISO 4217
    tags: list[str] = []
    attributes: dict = {}

class ProductResponse(BaseModel):
    id: UUID
    sku: str
    gtin: Optional[str]
    name: str
    brand: Optional[str]
    category: Optional[CategorySummary]
    cost_price: Optional[Decimal]
    cost_currency: str
    status: str
    tags: list[str]
    attributes: dict
    created_at: datetime
    updated_at: datetime

class ProductListResponse(BaseModel):
    items: list[ProductResponse]
    total: int
    page: int
    page_size: int

class CategoryCreate(BaseModel):
    name: str = Field(max_length=255)
    slug: str = Field(max_length=255, pattern=r"^[a-z0-9\-]+$")
    parent_id: Optional[UUID] = None
    attributes: dict = {}
```

```python
# src/pricing_engine/api/products.py
router = APIRouter(prefix="/api/v1/products", tags=["products"])

@router.post("/", response_model=ProductResponse, status_code=201)
async def create_product(
    product: ProductCreate,
    tenant_id: UUID = Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
) -> ProductResponse: ...

@router.get("/", response_model=ProductListResponse)
async def list_products(
    tenant_id: UUID = Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=200),
    status: Optional[str] = None,
    category_id: Optional[UUID] = None,
    brand: Optional[str] = None,
    search: Optional[str] = None,
) -> ProductListResponse: ...

@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(product_id: UUID, ...) -> ProductResponse: ...

@router.patch("/{product_id}", response_model=ProductResponse)
async def update_product(product_id: UUID, update: ProductUpdate, ...) -> ProductResponse: ...

@router.delete("/{product_id}", status_code=204)
async def delete_product(product_id: UUID, ...) -> None: ...

@router.post("/import", response_model=ImportResult)
async def bulk_import_products(file: UploadFile, ...) -> ImportResult: ...
```

**Testing**:
- `Integration: POST /api/v1/products with valid data → 201, product created with UUID`
- `Integration: POST /api/v1/products with duplicate (tenant, sku) → 409 Conflict`
- `Integration: POST /api/v1/products with invalid GTIN (5 digits) → 422, validation error`
- `Integration: GET /api/v1/products?search=widget → filtered results returned`
- `Integration: GET /api/v1/products?category_id=<uuid> → only products in that category`
- `Integration: PATCH /api/v1/products/<id> with cost_price update → 200, price updated`
- `Integration: DELETE /api/v1/products/<id> → 204, product soft-deleted`
- `Integration: POST /api/v1/products/import with CSV → bulk insert, ImportResult with counts`
- `Unit: GTIN validator accepts 8, 12, 13, 14 digit inputs`
- `Unit: GTIN validator rejects 7 and 15 digit inputs`
- `Unit: ISO 4217 currency validator accepts "USD", "EUR", rejects "USDX", "us"`

#### 2.2 — Channel Management & Product Mapping

**What**: CRUD for sales channels (Shopify, WooCommerce, Amazon, etc.) and mapping products to channels with external IDs, MAP/MSRP constraints, and compliance fields.

**Design**:

```python
# src/pricing_engine/schemas/channels.py
class ChannelCreate(BaseModel):
    name: str = Field(max_length=255)
    type: Literal["shopify", "woocommerce", "amazon", "ebay", "pos", "custom"]
    config: dict = {}                                    # Channel-specific config (JSONB)
    jurisdictions: list[str] = []                        # ISO 3166 codes

class ChannelProductMapping(BaseModel):
    product_id: UUID
    external_ids: dict = {}                              # {"product_id": "...", "variant_id": "...", "asin": "..."}
    current_price: Optional[Decimal] = None
    currency: str = "USD"
    constraints: PriceConstraints = PriceConstraints()
    compliance: dict = {}                                # Jurisdiction-specific compliance fields

class PriceConstraints(BaseModel):
    map_price: Optional[Decimal] = None                  # Minimum Advertised Price
    msrp: Optional[Decimal] = None                       # Manufacturer Suggested Retail Price
    price_floor: Optional[Decimal] = None
    price_ceiling: Optional[Decimal] = None
    rounding: Literal["none", "charm_99", "round_up", "round_nearest"] = "none"

class ChannelResponse(BaseModel):
    id: UUID
    name: str
    type: str
    config: dict
    jurisdictions: list[str]
    sync_enabled: bool
    last_sync_at: Optional[datetime]
    product_count: int
    created_at: datetime
```

```python
# src/pricing_engine/api/channels.py
router = APIRouter(prefix="/api/v1/channels", tags=["channels"])

@router.post("/", response_model=ChannelResponse, status_code=201)
async def create_channel(channel: ChannelCreate, ...) -> ChannelResponse: ...

@router.get("/", response_model=list[ChannelResponse])
async def list_channels(...) -> list[ChannelResponse]: ...

@router.post("/{channel_id}/products", response_model=ChannelProductResponse, status_code=201)
async def map_product_to_channel(channel_id: UUID, mapping: ChannelProductMapping, ...) -> ChannelProductResponse: ...

@router.get("/{channel_id}/products", response_model=ChannelProductListResponse)
async def list_channel_products(channel_id: UUID, ...) -> ChannelProductListResponse: ...

@router.patch("/{channel_id}/products/{product_id}", response_model=ChannelProductResponse)
async def update_channel_product(channel_id: UUID, product_id: UUID, update: ChannelProductUpdate, ...) -> ChannelProductResponse: ...
```

**Testing**:
- `Integration: POST /api/v1/channels with type "shopify" → 201, channel created`
- `Integration: POST /api/v1/channels/{id}/products → 201, product mapped with external IDs`
- `Integration: POST /api/v1/channels/{id}/products with duplicate product → 409 Conflict`
- `Integration: PATCH channel_product with MAP price set → 200, constraints stored`
- `Unit: PriceConstraints validates map_price < msrp when both present`
- `Unit: jurisdiction codes validated against ISO 3166 pattern`
- `Integration: GET /api/v1/channels/{id}/products → paginated list with constraint details`

---

## Phase 3: Competitor Monitoring

### Purpose
Enable tracking of competitor prices — the most fundamental input to any dynamic pricing engine. After this phase, users can register competitors, match them to products, record price observations (manual or scraped), and query competitor price history via TimescaleDB.

### Tasks

#### 3.1 — Competitor & Product Match Management

**What**: CRUD for competitors and the product-matching subsystem that links competitor offerings to internal products.

**Design**:

```python
# src/pricing_engine/schemas/competitors.py
class CompetitorCreate(BaseModel):
    name: str = Field(max_length=255)
    config: dict = {}    # website_url, scrape_frequency_hours, type, markets

class CompetitorProductMatch(BaseModel):
    product_id: UUID
    match_data: MatchData

class MatchData(BaseModel):
    external_url: Optional[str] = None
    external_product_id: Optional[str] = None
    match_method: Literal["manual", "gtin", "title_match", "ml_matched"] = "manual"
    confidence: float = Field(ge=0.0, le=1.0, default=1.0)

class CompetitorPriceObservation(BaseModel):
    match_id: UUID
    price: Decimal = Field(gt=0, max_digits=12, decimal_places=4)
    currency: str = Field(default="USD", pattern=r"^[A-Z]{3}$")
    was_in_stock: Optional[bool] = None
    is_promotional: bool = False
    shipping_cost: Optional[Decimal] = None
    source: Literal["scrape", "api", "manual"] = "manual"
    observed_at: Optional[datetime] = None  # defaults to now()
```

```python
# src/pricing_engine/api/competitors.py
router = APIRouter(prefix="/api/v1/competitors", tags=["competitors"])

@router.post("/", response_model=CompetitorResponse, status_code=201)
async def create_competitor(competitor: CompetitorCreate, ...) -> CompetitorResponse: ...

@router.post("/{competitor_id}/matches", response_model=MatchResponse, status_code=201)
async def create_product_match(competitor_id: UUID, match: CompetitorProductMatch, ...) -> MatchResponse: ...

@router.post("/{competitor_id}/prices", status_code=201)
async def record_price_observation(competitor_id: UUID, observation: CompetitorPriceObservation, ...) -> dict: ...

@router.post("/{competitor_id}/prices/batch", status_code=201)
async def record_price_observations_batch(competitor_id: UUID, observations: list[CompetitorPriceObservation], ...) -> dict: ...

@router.get("/{competitor_id}/prices", response_model=CompetitorPriceHistory)
async def get_competitor_price_history(
    competitor_id: UUID,
    product_id: Optional[UUID] = None,
    start_date: Optional[datetime] = None,
    end_date: Optional[datetime] = None,
    bucket: Literal["raw", "hourly", "daily", "weekly"] = "daily",
    ...,
) -> CompetitorPriceHistory: ...
```

**Testing**:
- `Integration: POST competitor → 201, competitor created`
- `Integration: POST product match with GTIN method → 201, match stored with confidence`
- `Integration: POST price observation → 201, inserted into ts_competitor_prices hypertable`
- `Integration: POST batch of 100 price observations → 201, all inserted efficiently`
- `Integration: GET price history with bucket=daily → aggregated daily prices via time_bucket`
- `Integration: GET price history with date range → only observations in range returned`
- `Unit: price observation with negative price → 422 validation error`
- `Unit: match confidence outside 0-1 range → 422 validation error`

#### 3.2 — Competitor Price Scraping Engine

**What**: Async web scraper that fetches competitor prices on configurable schedules, with proxy rotation, rate limiting, and error handling.

**Design**:

```python
# src/pricing_engine/integrations/scraper.py
from dataclasses import dataclass
import httpx
from selectolax.parser import HTMLParser

@dataclass
class ScrapeResult:
    match_id: UUID
    price: Decimal | None
    currency: str
    was_in_stock: bool | None
    is_promotional: bool
    shipping_cost: Decimal | None
    raw_html_hash: str
    error: str | None = None

class CompetitorScraper:
    def __init__(self, proxy_pool: ProxyPool, rate_limiter: RateLimiter):
        self.proxy_pool = proxy_pool
        self.rate_limiter = rate_limiter
        self.client = httpx.AsyncClient(timeout=30, follow_redirects=True)

    async def scrape_url(self, url: str, selectors: PriceSelectors) -> ScrapeResult:
        """Fetch URL via rotating proxy, extract price using CSS selectors."""
        ...

    async def scrape_batch(self, matches: list[ScrapeTarget], concurrency: int = 10) -> list[ScrapeResult]:
        """Scrape multiple URLs concurrently with rate limiting."""
        ...

@dataclass
class PriceSelectors:
    price_selector: str           # CSS selector for price element
    currency_selector: str | None
    stock_selector: str | None
    promo_selector: str | None
    price_regex: str = r"[\d,]+\.?\d*"  # Extract numeric price from text
```

```python
# src/pricing_engine/tasks/scraping.py
@celery_app.task(bind=True, max_retries=3, default_retry_delay=300)
def scrape_competitor_prices(self, competitor_id: str, tenant_id: str) -> dict:
    """Celery task: scrape all product matches for a competitor."""
    ...

@celery_app.task
def schedule_all_scraping() -> None:
    """Periodic task: enqueue scraping for all competitors whose scrape interval has elapsed."""
    ...
```

**Testing**:
- `Unit: scrape_url with mocked HTML containing "$29.99" → ScrapeResult with price=29.99`
- `Unit: scrape_url with mocked HTML missing price element → ScrapeResult with error message`
- `Unit: price_regex extracts "1,299.99" from "Price: $1,299.99 USD"`
- `Integration (mocked HTTP): scrape_batch with 5 URLs → 5 ScrapeResults, rate limiting respected`
- `Integration (mocked HTTP): scrape_url with 403 response → retries with different proxy`
- `Integration (mocked HTTP): scrape_url with timeout → ScrapeResult with error, no crash`
- `Unit: schedule_all_scraping identifies competitors whose interval has elapsed`

#### 3.3 — Continuous Aggregates for Competitor Analytics

**What**: Create TimescaleDB continuous aggregates for hourly competitor price summaries and a view that powers the competitor analytics dashboard.

**Design**:

```sql
-- Hourly competitor price summary (from data model suggestion 4)
CREATE MATERIALIZED VIEW cagg_competitor_prices_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', observed_at) AS bucket,
    tenant_id,
    product_id,
    competitor_id,
    AVG(price_converted) AS avg_price,
    MIN(price_converted) AS min_price,
    MAX(price_converted) AS max_price,
    last(price_converted, observed_at) AS latest_price,
    last(was_in_stock, observed_at) AS latest_in_stock,
    COUNT(*) AS observation_count
FROM ts_competitor_prices
GROUP BY bucket, tenant_id, product_id, competitor_id
WITH NO DATA;
```

```python
# src/pricing_engine/services/competitor_service.py
class CompetitorService:
    async def get_competitive_position(
        self, tenant_id: UUID, product_id: UUID, days: int = 30
    ) -> CompetitivePosition:
        """Compute where our price sits relative to competitors."""
        ...

    async def detect_price_changes(
        self, tenant_id: UUID, threshold_pct: float = 0.05, hours: int = 24
    ) -> list[CompetitorPriceAlert]:
        """Detect competitor price movements exceeding threshold."""
        ...
```

```python
@dataclass
class CompetitivePosition:
    product_id: UUID
    our_price: Decimal
    competitor_min: Decimal
    competitor_max: Decimal
    competitor_avg: Decimal
    our_rank: int             # 1 = cheapest
    total_competitors: int
    competitors: list[CompetitorPriceSummary]
```

**Testing**:
- `Integration: insert 100 competitor prices, query cagg_competitor_prices_hourly → aggregated rows exist`
- `Integration: get_competitive_position with 3 competitors → correct rank, min, max, avg`
- `Integration: detect_price_changes with competitor drop > 5% → alert generated`
- `Integration: detect_price_changes with no significant changes → empty list`
- `Unit: CompetitivePosition our_rank=1 when our price is lowest`

---

## Phase 4: Rule-Based Repricing Engine

### Purpose
Implement the core repricing engine that evaluates pricing rules against product/competitor/inventory data and recommends price changes. This is the "heart" of the product — the mechanism that turns market data into pricing actions. After this phase, users can create pricing strategies with layered rules and run repricing evaluations.

### Tasks

#### 4.1 — Pricing Strategy & Rule Management

**What**: CRUD for pricing strategies and their rules, with JSONB-based rule parameters validated by JSON Schema.

**Design**:

```python
# src/pricing_engine/schemas/pricing.py
class PricingStrategyCreate(BaseModel):
    name: str = Field(max_length=255)
    description: Optional[str] = None
    priority: int = Field(default=100, ge=1, le=1000)
    schedule: Optional[ScheduleConfig] = None

class ScheduleConfig(BaseModel):
    type: Literal["manual", "recurring", "event_triggered"]
    frequency: Optional[Literal["hourly", "daily", "weekly"]] = None
    active_hours: Optional[dict] = None    # {"start": "06:00", "end": "22:00"}
    timezone: str = "UTC"

class PricingRuleCreate(BaseModel):
    name: str = Field(max_length=255)
    rule_type: Literal[
        "cost_plus", "competitor_match", "competitor_beat",
        "margin_floor", "margin_target", "map_floor",
        "msrp_ceiling", "omnibus_floor", "inventory_markdown",
        "demand_based", "time_decay", "clearance"
    ]
    priority: int = Field(default=100, ge=1, le=1000)
    scope: RuleScope
    params: dict          # Validated per rule_type via JSON Schema
    constraints: RuleConstraints = RuleConstraints()

class RuleScope(BaseModel):
    type: Literal["all", "category", "products", "brand", "tags", "filter"]
    values: Optional[list[str]] = None
    skus: Optional[list[str]] = None
    min_price: Optional[Decimal] = None
    max_price: Optional[Decimal] = None

class RuleConstraints(BaseModel):
    enforce_map: bool = True
    enforce_msrp_ceiling: bool = True
    enforce_omnibus: bool = True
    min_margin_pct: Optional[float] = None
    max_price_change_pct: Optional[float] = None
    rounding: Literal["none", "charm_99", "round_up", "round_nearest"] = "none"
```

```python
# Rule parameter schemas (validated at creation time)
RULE_PARAM_SCHEMAS: dict[str, dict] = {
    "cost_plus": {
        "type": "object",
        "properties": {
            "markup_pct": {"type": "number", "minimum": 0, "maximum": 10},
            "include_shipping": {"type": "boolean", "default": False},
        },
        "required": ["markup_pct"],
    },
    "competitor_beat": {
        "type": "object",
        "properties": {
            "offset_pct": {"type": "number", "minimum": -0.5, "maximum": 0},
            "competitor_ids": {"type": "array", "items": {"type": "string", "format": "uuid"}},
            "use_lowest": {"type": "boolean", "default": True},
            "ignore_out_of_stock": {"type": "boolean", "default": True},
        },
        "required": ["offset_pct"],
    },
    "inventory_markdown": {
        "type": "object",
        "properties": {
            "thresholds": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "days_of_supply_above": {"type": "integer", "minimum": 1},
                        "markdown_pct": {"type": "number", "minimum": 0, "maximum": 1},
                    },
                    "required": ["days_of_supply_above", "markdown_pct"],
                },
            },
        },
        "required": ["thresholds"],
    },
}
```

**Testing**:
- `Integration: POST strategy → 201, strategy created with priority`
- `Integration: POST rule with type "cost_plus" and valid params → 201`
- `Integration: POST rule with type "cost_plus" and missing markup_pct → 422, JSON Schema validation error`
- `Integration: POST rule with type "competitor_beat" and offset_pct > 0 → 422`
- `Integration: GET strategy with rules → rules ordered by priority`
- `Unit: RuleScope type "category" with empty values → validation error`
- `Unit: RuleConstraints min_margin_pct > 1.0 → validation error`

#### 4.2 — Repricing Engine Core

**What**: The rule evaluation engine that takes a product, its market context (competitor prices, inventory, demand), and applies a strategy's rules to compute a recommended price.

**Design**:

```python
# src/pricing_engine/services/repricing_engine.py
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class PricingContext:
    """All data needed to price a single product on a single channel."""
    product_id: UUID
    channel_product_id: UUID
    current_price: Decimal
    cost_price: Decimal
    currency: str
    map_price: Decimal | None
    msrp: Decimal | None
    competitor_prices: list[CompetitorPriceSnapshot]
    inventory: InventorySnapshot | None
    elasticity: float | None
    demand_forecast: DemandForecastSnapshot | None
    omnibus_reference_price: Decimal | None

@dataclass
class CompetitorPriceSnapshot:
    competitor_id: UUID
    competitor_name: str
    price: Decimal
    was_in_stock: bool
    is_promotional: bool
    observed_at: datetime

@dataclass
class PriceRecommendation:
    channel_product_id: UUID
    current_price: Decimal
    recommended_price: Decimal
    final_price: Decimal              # After constraints (MAP, MSRP, Omnibus, rounding)
    price_change_pct: float
    rule_id: UUID
    rule_type: str
    constraints_applied: list[str]
    reasoning: str                     # Human-readable explanation

class RepricingEngine:
    def evaluate_product(self, context: PricingContext, strategy: PricingStrategy) -> PriceRecommendation | None:
        """Evaluate all rules in priority order, return first matching recommendation."""
        for rule in sorted(strategy.rules, key=lambda r: r.priority):
            if not self._matches_scope(rule.scope, context):
                continue
            raw_price = self._compute_rule_price(rule, context)
            if raw_price is None:
                continue
            final_price = self._apply_constraints(raw_price, context, rule.constraints)
            return PriceRecommendation(
                channel_product_id=context.channel_product_id,
                current_price=context.current_price,
                recommended_price=raw_price,
                final_price=final_price,
                price_change_pct=float((final_price - context.current_price) / context.current_price),
                rule_id=rule.id,
                rule_type=rule.rule_type,
                constraints_applied=self._list_applied_constraints(raw_price, final_price, context, rule.constraints),
                reasoning=self._generate_reasoning(rule, raw_price, final_price, context),
            )
        return None

    def _compute_rule_price(self, rule: PricingRule, ctx: PricingContext) -> Decimal | None:
        match rule.rule_type:
            case "cost_plus":
                return ctx.cost_price * (1 + Decimal(str(rule.params["markup_pct"])))
            case "competitor_beat":
                comp_price = self._get_reference_competitor_price(rule.params, ctx.competitor_prices)
                if comp_price is None:
                    return None
                return comp_price * (1 + Decimal(str(rule.params["offset_pct"])))
            case "competitor_match":
                return self._get_reference_competitor_price(rule.params, ctx.competitor_prices)
            case "margin_floor":
                return ctx.cost_price * (1 + Decimal(str(rule.params.get("min_margin_pct", 0.10))))
            case "inventory_markdown":
                return self._compute_inventory_markdown(rule.params, ctx)
            case _:
                return None

    def _apply_constraints(self, price: Decimal, ctx: PricingContext, constraints: RuleConstraints) -> Decimal:
        """Apply MAP floor, MSRP ceiling, Omnibus floor, margin floor, max change, and rounding."""
        result = price
        if constraints.enforce_map and ctx.map_price and result < ctx.map_price:
            result = ctx.map_price
        if constraints.enforce_msrp_ceiling and ctx.msrp and result > ctx.msrp:
            result = ctx.msrp
        if constraints.enforce_omnibus and ctx.omnibus_reference_price and result < ctx.omnibus_reference_price:
            result = ctx.omnibus_reference_price
        if constraints.min_margin_pct and ctx.cost_price:
            margin_floor = ctx.cost_price * (1 + Decimal(str(constraints.min_margin_pct)))
            if result < margin_floor:
                result = margin_floor
        if constraints.max_price_change_pct:
            max_delta = ctx.current_price * Decimal(str(constraints.max_price_change_pct))
            result = max(ctx.current_price - max_delta, min(result, ctx.current_price + max_delta))
        result = self._apply_rounding(result, constraints.rounding)
        return result
```

**Testing**:
- `Unit: cost_plus rule with 40% markup on $10 cost → $14.00`
- `Unit: competitor_beat rule with -2% offset, competitor at $27.49 → $26.94`
- `Unit: competitor_beat with all competitors out of stock and ignore_out_of_stock=True → None`
- `Unit: inventory_markdown with 120 days supply, threshold at 90 days (10% markdown) → price reduced 10%`
- `Unit: MAP constraint raises $18.00 to $24.99 MAP floor → $24.99, constraints_applied includes "map_floor"`
- `Unit: MSRP constraint lowers $39.99 to $34.99 MSRP ceiling → $34.99`
- `Unit: Omnibus constraint raises price to reference price when below → constraints_applied includes "omnibus_floor"`
- `Unit: max_price_change_pct of 15% caps a 30% price increase → change limited to 15%`
- `Unit: charm_99 rounding on $26.94 → $26.99`
- `Unit: round_nearest rounding on $26.94 → $27.00`
- `Unit: rules evaluated in priority order, first match wins`
- `Unit: no rules match product scope → None returned`

#### 4.3 — Repricing Run Orchestration

**What**: Orchestrate full repricing runs across all products in a strategy's scope, log results to ts_repricing_events and ts_price_changes, and optionally push price updates to channels.

**Design**:

```python
# src/pricing_engine/services/pricing_service.py
@dataclass
class RepricingRunResult:
    run_id: UUID
    strategy_id: UUID
    products_evaluated: int
    prices_changed: int
    prices_unchanged: int
    errors: int
    duration_seconds: float
    recommendations: list[PriceRecommendation]

class PricingService:
    async def execute_repricing_run(
        self,
        tenant_id: UUID,
        strategy_id: UUID,
        dry_run: bool = False,
        product_ids: list[UUID] | None = None,
    ) -> RepricingRunResult:
        """Execute a repricing run for a strategy across its scoped products."""
        ...

    async def build_pricing_context(
        self, tenant_id: UUID, channel_product_id: UUID
    ) -> PricingContext:
        """Assemble all data needed to price a product: cost, competitors, inventory, compliance."""
        ...
```

```python
# src/pricing_engine/api/pricing.py
@router.post("/api/v1/strategies/{strategy_id}/run", response_model=RepricingRunResultResponse)
async def run_repricing(
    strategy_id: UUID,
    dry_run: bool = Query(False),
    product_ids: Optional[list[UUID]] = Body(None),
    ...,
) -> RepricingRunResultResponse: ...

@router.get("/api/v1/repricing/history", response_model=RepricingHistoryResponse)
async def get_repricing_history(
    start_date: Optional[datetime] = None,
    end_date: Optional[datetime] = None,
    product_id: Optional[UUID] = None,
    strategy_id: Optional[UUID] = None,
    changed_only: bool = False,
    ...,
) -> RepricingHistoryResponse: ...
```

**Testing**:
- `Integration: execute_repricing_run with 3 products → 3 evaluated, repricing events logged to ts_repricing_events`
- `Integration: execute_repricing_run dry_run=True → recommendations generated, no price changes written`
- `Integration: execute_repricing_run with price change → ts_price_changes row inserted, channel_products.current_price updated`
- `Integration: execute_repricing_run with strategy targeting category → only category products evaluated`
- `Integration: GET /api/v1/repricing/history with changed_only=True → only price-change events`
- `E2E: create strategy + rules → run repricing → verify prices changed correctly`

---

## Phase 5: Compliance & Price History

### Purpose
Implement regulatory compliance features — EU Omnibus Directive 30-day reference pricing, MAP enforcement audit trails, and MSRP compliance reporting. After this phase, the engine ensures all price changes respect regulatory and brand constraints, and provides auditable price history.

### Tasks

#### 5.1 — Omnibus Reference Price Computation

**What**: Continuous aggregate and service layer for computing the 30-day lowest price per channel-product, as required by the EU Omnibus Directive.

**Design**:

```sql
-- From data model suggestion 4
CREATE MATERIALIZED VIEW cagg_omnibus_reference
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', changed_at) AS bucket,
    tenant_id,
    channel_product_id,
    product_id,
    MIN(new_price) AS daily_min_price
FROM ts_price_changes
GROUP BY bucket, tenant_id, channel_product_id, product_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_omnibus_reference',
    start_offset => INTERVAL '32 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

```python
# src/pricing_engine/services/compliance_service.py
class ComplianceService:
    async def get_omnibus_reference_price(
        self, channel_product_id: UUID, lookback_days: int = 30
    ) -> Decimal | None:
        """Return the lowest price in the lookback window."""
        ...

    async def check_omnibus_compliance(
        self, tenant_id: UUID, channel_id: UUID
    ) -> list[OmnibusViolation]:
        """Scan all listed products for Omnibus violations."""
        ...

    async def check_map_compliance(
        self, tenant_id: UUID, channel_id: UUID
    ) -> list[MAPViolation]:
        """Check all channel products against MAP constraints."""
        ...

@dataclass
class OmnibusViolation:
    channel_product_id: UUID
    product_sku: str
    current_price: Decimal
    omnibus_reference_price: Decimal
    jurisdiction: str
    violation_type: str     # "below_reference" or "missing_reference"
```

```python
# src/pricing_engine/api/pricing.py
@router.get("/api/v1/compliance/omnibus", response_model=list[OmnibusViolationResponse])
async def check_omnibus_compliance(channel_id: Optional[UUID] = None, ...) -> list: ...

@router.get("/api/v1/compliance/map", response_model=list[MAPViolationResponse])
async def check_map_compliance(channel_id: Optional[UUID] = None, ...) -> list: ...
```

**Testing**:
- `Integration: insert 30 days of price changes, get_omnibus_reference_price → returns minimum`
- `Integration: product with current_price below 30-day min → OmnibusViolation returned`
- `Integration: product with no price history → violation_type = "missing_reference"`
- `Integration: product with current_price above 30-day min → no violation`
- `Integration: MAP violation check with price below MAP → MAPViolation returned`
- `Unit: lookback window correctly excludes day 31`
- `Unit: Omnibus does not apply to channels without EU jurisdictions`

#### 5.2 — Price History API & Analytics

**What**: Time-series price history queries for products across channels, with support for daily/weekly/monthly rollups.

**Design**:

```python
# src/pricing_engine/api/analytics.py
@router.get("/api/v1/analytics/price-history/{product_id}")
async def get_price_history(
    product_id: UUID,
    channel_id: Optional[UUID] = None,
    start_date: datetime = Query(default_factory=lambda: datetime.now() - timedelta(days=90)),
    end_date: datetime = Query(default_factory=datetime.now),
    bucket: Literal["raw", "daily", "weekly", "monthly"] = "daily",
    ...,
) -> PriceHistoryResponse: ...

class PriceHistoryResponse(BaseModel):
    product_id: UUID
    data_points: list[PriceDataPoint]
    summary: PriceHistorySummary

class PriceDataPoint(BaseModel):
    timestamp: datetime
    price: Decimal
    min_price: Optional[Decimal] = None
    max_price: Optional[Decimal] = None
    change_count: int = 0

class PriceHistorySummary(BaseModel):
    current_price: Decimal
    period_min: Decimal
    period_max: Decimal
    period_avg: Decimal
    total_changes: int
    omnibus_reference_price: Optional[Decimal] = None
```

**Testing**:
- `Integration: GET price-history with bucket=daily → aggregated daily data points`
- `Integration: GET price-history with bucket=raw → all individual price changes`
- `Integration: GET price-history with date range → only points in range`
- `Integration: summary includes correct min, max, avg across period`
- `Integration: omnibus_reference_price included when channel is in EU jurisdiction`

---

## Phase 6: E-Commerce Platform Integrations

### Purpose
Connect the pricing engine to real e-commerce platforms (Shopify, WooCommerce) and marketplaces (Amazon SP-API) for bidirectional price synchronisation. After this phase, price changes computed by the repricing engine are pushed to live stores, and product/inventory data is pulled in.

### Tasks

#### 6.1 — Integration Framework & Base Connector

**What**: Abstract base class for channel connectors with a standard interface for pulling products/prices and pushing price updates.

**Design**:

```python
# src/pricing_engine/integrations/base.py
from abc import ABC, abstractmethod

class ChannelConnector(ABC):
    def __init__(self, channel_id: UUID, config: dict, credentials: bytes):
        self.channel_id = channel_id
        self.config = config
        self.credentials = self._decrypt_credentials(credentials)

    @abstractmethod
    async def pull_products(self) -> list[ExternalProduct]: ...

    @abstractmethod
    async def pull_prices(self, external_ids: list[str]) -> list[ExternalPrice]: ...

    @abstractmethod
    async def push_price(self, external_id: str, new_price: Decimal, currency: str) -> PushResult: ...

    @abstractmethod
    async def push_prices_batch(self, updates: list[PriceUpdate]) -> list[PushResult]: ...

    @abstractmethod
    async def verify_connection(self) -> ConnectionStatus: ...

@dataclass
class PushResult:
    external_id: str
    success: bool
    old_price: Decimal | None
    new_price: Decimal
    error: str | None = None

@dataclass
class ConnectionStatus:
    connected: bool
    store_name: str | None
    product_count: int | None
    error: str | None = None
```

**Testing**:
- `Unit: ChannelConnector subclass must implement all abstract methods`
- `Unit: _decrypt_credentials with valid encrypted bytes → credentials dict`
- `Unit: PushResult.success=False includes error message`

#### 6.2 — Shopify Connector

**What**: Shopify GraphQL Admin API connector for product sync and price updates (per standards.md, GraphQL is required for new integrations from 2025).

**Design**:

```python
# src/pricing_engine/integrations/shopify.py
class ShopifyConnector(ChannelConnector):
    """Shopify GraphQL Admin API connector."""

    async def pull_products(self) -> list[ExternalProduct]:
        query = """
        query($cursor: String) {
            products(first: 50, after: $cursor) {
                edges { node { id title variants(first: 10) {
                    edges { node { id sku price barcode } }
                } } }
                pageInfo { hasNextPage endCursor }
            }
        }
        """
        ...

    async def push_price(self, external_variant_id: str, new_price: Decimal, currency: str) -> PushResult:
        mutation = """
        mutation($input: ProductVariantInput!) {
            productVariantUpdate(input: $input) {
                productVariant { id price }
                userErrors { field message }
            }
        }
        """
        ...

    async def setup_price_webhook(self) -> str:
        """Register PRODUCTS_UPDATE webhook for price change notifications."""
        ...
```

**Testing**:
- `Integration (mocked): pull_products → 50 products parsed from GraphQL response`
- `Integration (mocked): push_price → mutation sent, PushResult with success=True`
- `Integration (mocked): push_price with userError → PushResult with success=False, error message`
- `Integration (mocked): pull_products with pagination → all pages fetched`
- `Unit: Shopify GraphQL product response → ExternalProduct with sku, price, barcode mapped`
- `Integration (mocked): setup_price_webhook → webhook registered, URL returned`

#### 6.3 — Amazon SP-API Connector

**What**: Amazon Selling Partner API connector for competitive pricing queries and automated repricing (per standards.md, uses LWA OAuth 2.0).

**Design**:

```python
# src/pricing_engine/integrations/amazon_sp.py
class AmazonSPConnector(ChannelConnector):
    """Amazon SP-API Product Pricing connector."""

    async def get_competitive_pricing(self, asins: list[str]) -> list[AmazonCompetitivePrice]:
        """GET /products/pricing/v0/competitivePrice (up to 20 ASINs per batch)."""
        ...

    async def get_featured_offer_expected_price(self, asins: list[str]) -> list[AmazonFOEP]:
        """GET /batches/products/pricing/2022-05-01/offer/featuredOfferExpectedPrice (up to 20 ASINs, 40 FOEP requests)."""
        ...

    async def push_price(self, external_id: str, new_price: Decimal, currency: str) -> PushResult:
        """Update price via Listings Items API."""
        ...

    async def authenticate(self) -> str:
        """Login with Amazon (LWA) OAuth 2.0 token refresh."""
        ...
```

**Testing**:
- `Integration (mocked): get_competitive_pricing with 5 ASINs → 5 AmazonCompetitivePrice results`
- `Integration (mocked): get_competitive_pricing batches 25 ASINs into 2 requests (20 + 5)`
- `Integration (mocked): push_price via Listings API → PushResult with success`
- `Integration (mocked): authenticate with expired refresh token → new access token obtained`
- `Unit: AmazonSP rate limiting respects 0.5 requests/second default`

#### 6.4 — Price Sync Scheduler & Push Pipeline

**What**: Scheduled tasks that synchronise prices between the engine and connected channels — pulling current prices, detecting drift, and pushing repricing decisions.

**Design**:

```python
# src/pricing_engine/tasks/repricing.py
@celery_app.task
def sync_channel_prices(channel_id: str, tenant_id: str) -> dict:
    """Pull current prices from channel, detect drift from engine's records, update channel_products."""
    ...

@celery_app.task
def push_repricing_results(run_id: str) -> dict:
    """Push approved price changes from a repricing run to their respective channels."""
    ...

@celery_app.task
def schedule_channel_syncs() -> None:
    """Periodic task: enqueue sync for channels whose sync interval has elapsed."""
    ...
```

**Testing**:
- `Integration (mocked): sync_channel_prices detects price drift → channel_products.current_price updated`
- `Integration (mocked): push_repricing_results with 3 approved changes → 3 push attempts, results logged`
- `Integration (mocked): push_repricing_results with push failure → error logged, status set to "failed"`
- `Unit: schedule_channel_syncs skips channels with sync_enabled=False`

---

## Phase 7: A/B Testing & Experimentation

### Purpose
Enable pricing experiments to measure the impact of price changes on revenue and margin before rolling them out broadly. After this phase, users can create experiments, assign product-channel combinations to variants, and analyse results with statistical significance testing.

### Tasks

#### 7.1 — Experiment Lifecycle Management

**What**: CRUD for experiments with variant configuration, product assignment, and state machine (draft -> running -> paused -> completed/cancelled).

**Design**:

```python
# src/pricing_engine/schemas/experiments.py
class ExperimentCreate(BaseModel):
    name: str = Field(max_length=255)
    hypothesis: Optional[str] = None
    config: ExperimentConfig

class ExperimentConfig(BaseModel):
    traffic_pct: float = Field(ge=0, le=100, default=50)
    variants: list[VariantConfig]
    scope: RuleScope
    channels: list[UUID]
    success_metric: Literal["revenue", "margin", "margin_per_unit", "conversion_rate", "units_sold"]
    min_sample_size: int = Field(ge=100, default=500)
    significance_level: float = Field(ge=0.01, le=0.10, default=0.05)

class VariantConfig(BaseModel):
    name: str = Field(max_length=100)
    weight: float = Field(ge=0, le=100)
    price_modifier_pct: float = Field(ge=-0.50, le=0.50)
    strategy_id: Optional[UUID] = None

# State machine: draft -> running -> paused -> completed | cancelled
VALID_TRANSITIONS = {
    "draft": ["running", "cancelled"],
    "running": ["paused", "completed", "cancelled"],
    "paused": ["running", "completed", "cancelled"],
    "completed": [],
    "cancelled": [],
}
```

```python
# src/pricing_engine/api/experiments.py
@router.post("/api/v1/experiments", response_model=ExperimentResponse, status_code=201)
async def create_experiment(experiment: ExperimentCreate, ...) -> ExperimentResponse: ...

@router.post("/api/v1/experiments/{experiment_id}/start")
async def start_experiment(experiment_id: UUID, ...) -> ExperimentResponse: ...

@router.post("/api/v1/experiments/{experiment_id}/pause")
async def pause_experiment(experiment_id: UUID, ...) -> ExperimentResponse: ...

@router.post("/api/v1/experiments/{experiment_id}/complete")
async def complete_experiment(experiment_id: UUID, ...) -> ExperimentResponse: ...

@router.get("/api/v1/experiments/{experiment_id}/results")
async def get_experiment_results(experiment_id: UUID, ...) -> ExperimentResultsResponse: ...
```

**Testing**:
- `Integration: create experiment → 201, status = "draft"`
- `Integration: start experiment → status transitions to "running"`
- `Integration: start already running experiment → 409 Conflict`
- `Integration: complete experiment → results computed and stored`
- `Unit: invalid state transition (completed -> running) → raises ValueError`
- `Unit: variant weights must sum to 100 → validation error if not`
- `Unit: price_modifier_pct outside [-0.50, 0.50] → validation error`

#### 7.2 — Experiment Results & Statistical Analysis

**What**: Service that computes experiment results with statistical significance testing (chi-squared test for conversion rates, t-test for continuous metrics).

**Design**:

```python
# src/pricing_engine/services/experiment_service.py
from scipy import stats

@dataclass
class VariantResult:
    variant_name: str
    impressions: int
    conversions: int
    revenue: Decimal
    margin: Decimal
    units_sold: int
    avg_selling_price: Decimal
    conversion_rate: float
    margin_per_unit: Decimal

@dataclass
class ExperimentResults:
    experiment_id: UUID
    variants: list[VariantResult]
    winner: str | None
    p_value: float
    is_significant: bool
    confidence_level: float
    lift_pct: float | None           # Lift of winner over control
    recommendation: str              # "Deploy variant_a" or "No significant difference"

class ExperimentService:
    async def compute_results(self, experiment_id: UUID) -> ExperimentResults:
        """Aggregate experiment observations and compute statistical significance."""
        ...

    def _compute_significance(
        self, control: VariantResult, treatment: VariantResult, metric: str, alpha: float
    ) -> tuple[float, bool]:
        """Return (p_value, is_significant) for the given metric comparison."""
        if metric == "conversion_rate":
            # Chi-squared test for proportions
            contingency = [[control.conversions, control.impressions - control.conversions],
                          [treatment.conversions, treatment.impressions - treatment.conversions]]
            chi2, p_value, _, _ = stats.chi2_contingency(contingency)
            return p_value, p_value < alpha
        # t-test for continuous metrics (margin, revenue per unit)
        ...
```

**Testing**:
- `Unit: compute_significance with significantly different conversion rates → is_significant=True`
- `Unit: compute_significance with similar rates and small sample → is_significant=False`
- `Unit: winner selection picks variant with better metric when significant`
- `Unit: recommendation = "No significant difference" when p_value > alpha`
- `Integration: compute_results aggregates ts_experiment_observations correctly`
- `Integration: completed experiment stores results in experiments.results JSONB`

---

## Phase 8: Demand Forecasting & Elasticity

### Purpose
Add ML-powered demand forecasting and price elasticity estimation. These models feed into the repricing engine to enable demand-aware pricing (rather than purely competitor-reactive pricing). After this phase, the engine can predict how demand will change at different price points and optimise for margin or revenue.

### Tasks

#### 8.1 — Demand Forecast Model

**What**: Time-series demand forecasting using XGBoost with features from sales history, price, seasonality, and competitor pricing.

**Design**:

```python
# src/pricing_engine/ml/demand_forecast.py
import xgboost as xgb
import numpy as np
from datetime import date

@dataclass
class ForecastFeatures:
    price: float
    day_of_week: int
    month: int
    is_holiday: bool
    competitor_avg_price: float
    competitor_min_price: float
    inventory_days_supply: float
    price_change_recency_days: int
    rolling_7d_units: float
    rolling_30d_units: float

@dataclass
class ForecastResult:
    forecast_date: date
    horizon_days: int
    predicted_units: float
    confidence_lower: float
    confidence_upper: float
    model_version: str
    feature_importance: dict[str, float]

class DemandForecaster:
    def __init__(self, model_path: str | None = None):
        self.model: xgb.XGBRegressor | None = None
        if model_path:
            self.model = xgb.XGBRegressor()
            self.model.load_model(model_path)

    def train(self, features: np.ndarray, targets: np.ndarray, params: dict | None = None) -> dict:
        """Train XGBoost model on historical sales data. Returns metrics dict."""
        default_params = {
            "n_estimators": 500,
            "max_depth": 8,
            "learning_rate": 0.05,
            "subsample": 0.8,
            "colsample_bytree": 0.8,
            "objective": "reg:squarederror",
        }
        params = {**default_params, **(params or {})}
        self.model = xgb.XGBRegressor(**params)
        self.model.fit(features, targets, eval_set=[(features, targets)], verbose=False)
        ...

    def predict(self, features: ForecastFeatures) -> ForecastResult:
        """Generate demand forecast for a single product-channel-date."""
        ...

    def predict_at_price(self, features: ForecastFeatures, prices: list[float]) -> list[ForecastResult]:
        """Predict demand at multiple price points (for price optimisation)."""
        ...
```

```python
# src/pricing_engine/tasks/forecasting.py
@celery_app.task
def generate_forecasts(tenant_id: str, product_ids: list[str] | None = None) -> dict:
    """Generate 7-day demand forecasts for all active products (or specified subset)."""
    ...

@celery_app.task
def retrain_forecast_model(tenant_id: str) -> dict:
    """Retrain the demand forecast model using the latest 365 days of sales data."""
    ...
```

**Testing**:
- `Unit: train with synthetic sales data → model metrics include MAE, RMSE, R-squared`
- `Unit: predict with trained model → ForecastResult with predicted_units > 0`
- `Unit: predict_at_price with 5 price points → 5 ForecastResults, demand decreases as price increases`
- `Unit: feature_importance sums to approximately 1.0`
- `Unit: confidence_lower < predicted_units < confidence_upper`
- `Integration: generate_forecasts → forecast rows inserted into ts_demand_forecasts`
- `Fixture: test with fixtures/demand_training_data.csv → model trains and predicts within expected ranges`

#### 8.2 — Price Elasticity Estimation

**What**: Estimate price elasticity of demand using log-log regression on historical price-sales data, with cross-elasticity support.

**Design**:

```python
# src/pricing_engine/ml/elasticity_estimator.py
from sklearn.linear_model import LinearRegression
import numpy as np

@dataclass
class ElasticityResult:
    product_id: UUID
    channel_id: UUID | None
    price_elasticity: float         # Typically negative (e.g., -1.5)
    confidence_interval: float
    sample_size: int
    method: str                     # "log_log_regression"
    r_squared: float
    is_elastic: bool                # abs(elasticity) > 1
    cross_elasticities: list[CrossElasticity]

@dataclass
class CrossElasticity:
    related_product_id: UUID
    related_product_sku: str
    elasticity: float               # Positive = substitute, negative = complement
    relationship: Literal["substitute", "complement"]

class ElasticityEstimator:
    def estimate(
        self, prices: np.ndarray, quantities: np.ndarray, min_samples: int = 30
    ) -> ElasticityResult | None:
        """Estimate price elasticity via log-log regression: ln(Q) = a + b*ln(P)."""
        if len(prices) < min_samples:
            return None
        log_prices = np.log(prices)
        log_quantities = np.log(quantities + 1)  # +1 to handle zero sales
        model = LinearRegression()
        model.fit(log_prices.reshape(-1, 1), log_quantities)
        elasticity = float(model.coef_[0])
        r_squared = float(model.score(log_prices.reshape(-1, 1), log_quantities))
        ...

    def estimate_cross_elasticity(
        self, product_prices: np.ndarray, related_quantities: np.ndarray
    ) -> float:
        """Estimate cross-price elasticity between two products."""
        ...
```

**Testing**:
- `Unit: estimate with synthetic elastic demand (b=-1.5) → elasticity close to -1.5`
- `Unit: estimate with synthetic inelastic demand (b=-0.5) → is_elastic=False`
- `Unit: estimate with fewer than min_samples → returns None`
- `Unit: cross_elasticity positive for substitute products`
- `Unit: cross_elasticity negative for complement products`
- `Integration: estimate from ts_sales + ts_price_changes data → ElasticityResult stored in elasticity_estimates`

#### 8.3 — Demand-Based Pricing Rule Integration

**What**: Integrate demand forecasts and elasticity estimates into the repricing engine as a new "demand_based" rule type.

**Design**:

```python
# Addition to src/pricing_engine/services/repricing_engine.py
class RepricingEngine:
    def _compute_rule_price(self, rule: PricingRule, ctx: PricingContext) -> Decimal | None:
        match rule.rule_type:
            # ... existing rules ...
            case "demand_based":
                return self._compute_demand_based_price(rule.params, ctx)

    def _compute_demand_based_price(self, params: dict, ctx: PricingContext) -> Decimal | None:
        if ctx.elasticity is None or ctx.demand_forecast is None:
            return None
        target = params.get("target_metric", "margin")  # "margin" or "revenue"
        sensitivity = params.get("price_sensitivity", 0.5)  # 0 = ignore demand, 1 = fully demand-driven
        max_increase = Decimal(str(params.get("max_increase_pct", 0.15)))
        max_decrease = Decimal(str(params.get("max_decrease_pct", 0.25)))

        # Compute optimal price based on elasticity
        # For margin optimization: P* = cost / (1 + 1/elasticity) (Lerner index)
        if target == "margin" and ctx.cost_price and ctx.elasticity < -1:
            optimal = ctx.cost_price / (1 + Decimal("1") / Decimal(str(ctx.elasticity)))
        else:
            optimal = ctx.current_price

        # Blend with current price based on sensitivity
        blended = ctx.current_price + (optimal - ctx.current_price) * Decimal(str(sensitivity))

        # Apply max change bounds
        lower = ctx.current_price * (1 - max_decrease)
        upper = ctx.current_price * (1 + max_increase)
        return max(lower, min(blended, upper))
```

**Testing**:
- `Unit: demand_based with elasticity=-1.5, cost=$10 → optimal price computed via Lerner index`
- `Unit: demand_based with sensitivity=0 → returns current price (no change)`
- `Unit: demand_based with sensitivity=1 → returns optimal price`
- `Unit: demand_based with elasticity > -1 (inelastic) → falls back to current price`
- `Unit: demand_based with missing elasticity → returns None`
- `Unit: max_increase_pct caps upward movement`
- `Unit: max_decrease_pct caps downward movement`

---

## Phase 9: Webhooks & Event Notifications

### Purpose
Implement outbound webhook delivery for pricing events, enabling downstream systems to react to price changes, competitor alerts, and experiment completions. Uses HMAC-SHA256 signing per industry standard (Shopify/GitHub pattern).

### Tasks

#### 9.1 — Webhook Subscription Management

**What**: CRUD for webhook subscriptions with event type filtering and HMAC secret management.

**Design**:

```python
# src/pricing_engine/schemas/webhooks.py
class WebhookSubscriptionCreate(BaseModel):
    event_type: Literal[
        "price.changed", "price.recommendation.generated",
        "competitor.price_drop", "competitor.price_increase",
        "experiment.started", "experiment.completed",
        "compliance.omnibus_violation", "compliance.map_violation",
        "repricing.run.completed", "repricing.run.failed",
    ]
    target_url: str = Field(max_length=1000, pattern=r"^https://")
    headers: dict[str, str] = {}       # Custom headers to include

class WebhookPayload(BaseModel):
    id: str                            # Delivery ID
    event_type: str
    timestamp: datetime
    tenant_id: str
    data: dict

# Delivery with exponential backoff: 10s, 30s, 90s, 270s, 810s (5 retries)
RETRY_DELAYS = [10, 30, 90, 270, 810]
```

```python
# src/pricing_engine/services/webhook_service.py
class WebhookService:
    async def create_subscription(self, tenant_id: UUID, sub: WebhookSubscriptionCreate) -> WebhookSubscription:
        """Create subscription with auto-generated HMAC-SHA256 signing secret."""
        ...

    async def deliver(self, tenant_id: UUID, event_type: str, data: dict) -> None:
        """Enqueue webhook delivery for all matching subscriptions."""
        ...

    def _sign_payload(self, payload: bytes, secret: str) -> str:
        """HMAC-SHA256 signature: sha256=<hex_digest>"""
        return "sha256=" + hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()
```

**Testing**:
- `Integration: create subscription → 201, secret returned once, HMAC-SHA256 signing works`
- `Integration: deliver price.changed event → webhook_deliveries row created, HTTP POST sent`
- `Integration (mocked HTTP): delivery to 200 endpoint → response_status=200, delivered_at set`
- `Integration (mocked HTTP): delivery to 500 endpoint → retry scheduled with exponential backoff`
- `Integration (mocked HTTP): delivery after 5 retries → status set to "failed"`
- `Unit: _sign_payload produces valid HMAC-SHA256 hex digest`
- `Unit: target_url must start with https:// → validation error for http://`

#### 9.2 — Event Emission Integration

**What**: Integrate webhook emission into the repricing engine, competitor monitoring, experiment lifecycle, and compliance checks.

**Design**:

```python
# Event emission points:
# - RepricingEngine.evaluate_product() → "price.changed" when price changes
# - CompetitorService.detect_price_changes() → "competitor.price_drop" / "competitor.price_increase"
# - ExperimentService.complete_experiment() → "experiment.completed"
# - ComplianceService.check_omnibus_compliance() → "compliance.omnibus_violation"
# - PricingService.execute_repricing_run() → "repricing.run.completed" / "repricing.run.failed"
```

**Testing**:
- `Integration: price change triggers price.changed webhook delivery`
- `Integration: competitor price drop > 5% triggers competitor.price_drop webhook`
- `Integration: experiment completion triggers experiment.completed webhook with results`
- `Integration: no subscriptions for event type → no deliveries created`

---

## Phase 10: Analytics Dashboard (Frontend)

### Purpose
Build the web-based analytics dashboard that gives pricing managers visibility into competitor positioning, repricing activity, experiment results, and compliance status. The frontend consumes the API built in previous phases.

### Tasks

#### 10.1 — Frontend Scaffolding & Layout

**What**: Set up the Next.js project with authentication, navigation, and base layout components.

**Design**:

```typescript
// frontend/src/types/api.ts (generated from OpenAPI spec)
interface Product {
  id: string;
  sku: string;
  gtin: string | null;
  name: string;
  brand: string | null;
  cost_price: number | null;
  status: 'active' | 'discontinued' | 'seasonal';
  tags: string[];
}

interface PriceDataPoint {
  timestamp: string;
  price: number;
  min_price: number | null;
  max_price: number | null;
  change_count: number;
}

interface CompetitivePosition {
  product_id: string;
  our_price: number;
  competitor_min: number;
  competitor_max: number;
  competitor_avg: number;
  our_rank: number;
  total_competitors: number;
}
```

```typescript
// frontend/src/lib/api-client.ts
class PricingAPIClient {
  constructor(private baseUrl: string, private token: string) {}

  async getProducts(params: ProductListParams): Promise<ProductListResponse> { ... }
  async getPriceHistory(productId: string, params: PriceHistoryParams): Promise<PriceHistoryResponse> { ... }
  async getCompetitivePosition(productId: string): Promise<CompetitivePosition> { ... }
  async getExperimentResults(experimentId: string): Promise<ExperimentResults> { ... }
  async getRepricingHistory(params: RepricingHistoryParams): Promise<RepricingHistoryResponse> { ... }
  async getComplianceStatus(): Promise<ComplianceStatusResponse> { ... }
}
```

**Testing**:
- `Unit: API client correctly constructs request URLs with query parameters`
- `Unit: API client attaches Bearer token to all requests`
- `E2E (Playwright): login page → enter credentials → redirected to dashboard`
- `E2E (Playwright): navigation menu shows all sections`

#### 10.2 — Pricing Dashboard & Charts

**What**: Main dashboard with price history charts, competitive position visualisation, and repricing activity feed.

**Design**:

Key dashboard views:
- **Price Overview**: Time-series chart of own prices vs. competitor prices (Recharts LineChart)
- **Competitive Position**: Horizontal bar chart showing price rank per product
- **Repricing Activity Feed**: Scrollable list of recent price changes with rule and reason
- **Revenue Impact**: Area chart of revenue/margin trends correlated with price changes
- **Compliance Status**: Summary cards showing Omnibus violations, MAP violations

```typescript
// frontend/src/components/charts/PriceHistoryChart.tsx
interface PriceHistoryChartProps {
  data: PriceDataPoint[];
  competitorData: Record<string, PriceDataPoint[]>;
  mapPrice: number | null;
  msrp: number | null;
  omnibusReference: number | null;
}

// Renders:
// - Line for own price (primary color)
// - Lines for each competitor (muted colors)
// - Dashed horizontal lines for MAP, MSRP, Omnibus reference
// - Shaded region between MAP and MSRP (valid pricing zone)
```

**Testing**:
- `E2E (Playwright): dashboard loads → price history chart renders with data`
- `E2E (Playwright): select product from dropdown → chart updates with product's data`
- `E2E (Playwright): hover over chart point → tooltip shows price, date, reason`
- `E2E (Playwright): compliance card shows "2 Omnibus violations" when violations exist`
- `Unit (Vitest): PriceHistoryChart renders MAP line when mapPrice is non-null`
- `Unit (Vitest): PriceHistoryChart omits MAP line when mapPrice is null`

#### 10.3 — Strategy Builder UI

**What**: Visual interface for creating and editing pricing strategies with drag-and-drop rule ordering.

**Design**:

The strategy builder allows users to:
1. Create a new strategy with name and schedule
2. Add rules via a form that adapts to the selected rule_type
3. Reorder rules by priority (drag-and-drop)
4. Preview rule scope (which products match)
5. Run a dry-run to see recommended price changes before activation

```typescript
// frontend/src/components/pricing/RuleEditor.tsx
interface RuleEditorProps {
  ruleType: RuleType;
  params: Record<string, unknown>;
  constraints: RuleConstraints;
  scope: RuleScope;
  onChange: (rule: PricingRuleCreate) => void;
}

// Dynamic form fields based on rule_type:
// cost_plus → markup_pct slider, include_shipping toggle
// competitor_beat → offset_pct slider, competitor selection, use_lowest toggle
// inventory_markdown → threshold table (days_of_supply, markdown_pct pairs)
// demand_based → target_metric dropdown, sensitivity slider, max bounds
```

**Testing**:
- `E2E (Playwright): create strategy → add cost_plus rule → form shows markup_pct field`
- `E2E (Playwright): change rule type to competitor_beat → form fields update dynamically`
- `E2E (Playwright): drag rule B above rule A → priorities swap`
- `E2E (Playwright): click "Preview" → API call to dry-run endpoint, results displayed`
- `E2E (Playwright): click "Activate" → strategy status changes to active`

---

## Phase 11: Reinforcement Learning Pricing Agent

### Purpose
Implement the AI-native differentiator: an RL pricing agent that learns optimal pricing through experimentation rather than manual rule authoring. The agent uses Proximal Policy Optimization (PPO) to learn price adjustments from state-action-reward tuples stored in TimescaleDB. This is the feature that separates this engine from incumbent rule-based platforms.

### Tasks

#### 11.1 — RL Environment & State Space

**What**: Define the RL environment (Gymnasium-compatible) that models the pricing decision as a Markov Decision Process.

**Design**:

```python
# src/pricing_engine/ml/rl_agent.py
import gymnasium as gym
import numpy as np
from stable_baselines3 import PPO

class PricingEnvironment(gym.Env):
    """
    Gymnasium environment for dynamic pricing.

    State space (10 dimensions):
        - current_price (normalized)
        - cost_price (normalized)
        - competitor_min_price (normalized)
        - competitor_avg_price (normalized)
        - inventory_days_supply (normalized)
        - demand_trend (7-day rolling change)
        - day_of_week (one-hot: 0-6)
        - is_weekend (binary)
        - elasticity_estimate
        - price_vs_competitor_pct (our price / competitor avg - 1)

    Action space (discrete, 7 actions):
        -10%, -5%, -2%, 0%, +2%, +5%, +10%

    Reward: margin earned in the period (revenue - cost)
    """

    PRICE_ADJUSTMENTS = [-0.10, -0.05, -0.02, 0.0, 0.02, 0.05, 0.10]

    def __init__(self, product_id: UUID, channel_id: UUID, config: dict):
        super().__init__()
        self.observation_space = gym.spaces.Box(low=-np.inf, high=np.inf, shape=(10,), dtype=np.float32)
        self.action_space = gym.spaces.Discrete(len(self.PRICE_ADJUSTMENTS))
        ...

    def step(self, action: int) -> tuple[np.ndarray, float, bool, bool, dict]:
        adjustment_pct = self.PRICE_ADJUSTMENTS[action]
        new_price = self.current_price * (1 + adjustment_pct)
        # Apply constraints (MAP, MSRP, Omnibus)
        new_price = self._apply_constraints(new_price)
        # Compute reward (margin from sales in this step)
        reward = self._compute_reward(new_price)
        # Log to ts_rl_episodes
        self._log_episode_step(action, reward)
        ...

    def reset(self, **kwargs) -> tuple[np.ndarray, dict]:
        """Reset to current market state for the product."""
        ...

class RLPricingAgent:
    def __init__(self, model_path: str | None = None):
        self.model: PPO | None = None
        if model_path:
            self.model = PPO.load(model_path)

    def train(self, env: PricingEnvironment, total_timesteps: int = 100_000) -> dict:
        """Train PPO agent on the pricing environment."""
        self.model = PPO(
            "MlpPolicy", env,
            learning_rate=3e-4,
            n_steps=2048,
            batch_size=64,
            n_epochs=10,
            gamma=0.95,
            verbose=1,
        )
        self.model.learn(total_timesteps=total_timesteps)
        ...

    def recommend_price(self, state: np.ndarray) -> tuple[float, int]:
        """Return (adjustment_pct, action_index) for the current state."""
        action, _ = self.model.predict(state, deterministic=True)
        return self.PRICE_ADJUSTMENTS[int(action)], int(action)
```

**Testing**:
- `Unit: PricingEnvironment.step with action 0 (-10%) → price decreased by 10%`
- `Unit: PricingEnvironment.step applies MAP floor constraint`
- `Unit: PricingEnvironment.reset returns 10-dimensional state vector`
- `Unit: state vector normalized values are within expected ranges`
- `Unit: reward = margin_per_unit * predicted_units (positive for profitable price)`
- `Integration: train with 1000 steps → model saved, metrics returned`
- `Integration: recommend_price with trained model → returns valid action`
- `Integration: episode steps logged to ts_rl_episodes hypertable`

#### 11.2 — RL Agent Training Pipeline

**What**: Celery tasks for training RL agents from historical data, evaluating agent performance, and deploying trained models.

**Design**:

```python
# src/pricing_engine/tasks/rl_training.py
@celery_app.task
def train_rl_agent(tenant_id: str, product_id: str, channel_id: str, config: dict) -> dict:
    """Train an RL agent for a specific product-channel combination."""
    ...

@celery_app.task
def evaluate_rl_agent(agent_id: str, test_days: int = 30) -> dict:
    """Back-test an RL agent against historical data to compute expected reward."""
    ...

@celery_app.task
def deploy_rl_agent(agent_id: str) -> dict:
    """Activate trained agent as a pricing rule in a strategy."""
    ...
```

**Testing**:
- `Integration: train_rl_agent → model artifact saved to configured path, ml_models row created`
- `Integration: evaluate_rl_agent → back-test results with avg reward, comparison to baseline`
- `Integration: deploy_rl_agent → agent added as "rl_based" rule type in pricing strategy`
- `Unit: training with insufficient data (< 30 days) → raises error with clear message`

#### 11.3 — RL-Based Pricing Rule Integration

**What**: Add an "rl_based" rule type to the repricing engine that delegates price computation to a trained RL agent.

**Design**:

```python
# Addition to repricing_engine.py
class RepricingEngine:
    def _compute_rule_price(self, rule: PricingRule, ctx: PricingContext) -> Decimal | None:
        match rule.rule_type:
            # ... existing rules ...
            case "rl_based":
                return self._compute_rl_price(rule.params, ctx)

    def _compute_rl_price(self, params: dict, ctx: PricingContext) -> Decimal | None:
        agent_id = params.get("agent_id")
        if not agent_id:
            return None
        agent = self._load_rl_agent(agent_id)
        state = self._build_rl_state(ctx)
        adjustment_pct, action = agent.recommend_price(state)
        recommended = ctx.current_price * (1 + Decimal(str(adjustment_pct)))
        # Log the RL decision for training feedback
        self._log_rl_decision(agent_id, ctx, action, adjustment_pct)
        return recommended
```

**Testing**:
- `Unit: rl_based rule with trained agent → returns adjusted price`
- `Unit: rl_based rule with missing agent_id → returns None`
- `Unit: rl_based rule logs decision to ts_rl_episodes`
- `Integration: repricing run with rl_based rule → RL agent consulted, constraints applied to recommendation`

---

## Phase 12: Advanced Features & Production Hardening

### Purpose
Add remaining differentiating features (natural-language strategy authoring, customer segment pricing) and production-ready infrastructure (rate limiting, monitoring, backup, security hardening). After this phase, the engine is ready for production deployment.

### Tasks

#### 12.1 — Natural-Language Pricing Strategy Authoring

**What**: LLM-powered interface that translates natural-language pricing intent into executable rule configurations.

**Design**:

```python
# src/pricing_engine/services/nl_strategy.py
class NLStrategyService:
    async def parse_pricing_intent(self, tenant_id: UUID, intent: str) -> list[PricingRuleCreate]:
        """
        Parse natural language like:
        "Beat the lowest competitor by 2% on electronics, but never go below 15% margin"

        Returns a list of PricingRuleCreate schemas ready for API submission.
        """
        system_prompt = """You are a pricing strategy translator. Convert the user's pricing intent
        into a JSON array of pricing rules. Each rule must have:
        - name: descriptive name
        - rule_type: one of [cost_plus, competitor_beat, competitor_match, margin_floor, ...]
        - scope: {type, values}
        - params: rule-type-specific parameters
        - constraints: {enforce_map, min_margin_pct, max_price_change_pct, rounding}

        Available rule types and their params schemas:
        ...
        """
        ...

    async def explain_strategy(self, strategy_id: UUID) -> str:
        """Generate a natural-language explanation of what a strategy does."""
        ...
```

```python
# src/pricing_engine/api/pricing.py
@router.post("/api/v1/strategies/from-text", response_model=StrategyPreview)
async def create_strategy_from_text(
    request: NLStrategyRequest,
    ...,
) -> StrategyPreview:
    """Convert natural-language pricing intent into a strategy preview."""
    ...

class NLStrategyRequest(BaseModel):
    intent: str = Field(max_length=1000)
    preview_only: bool = True
```

**Testing**:
- `Integration (mocked LLM): "beat competitors by 2%" → competitor_beat rule with offset_pct=-0.02`
- `Integration (mocked LLM): "never go below 15% margin" → margin_floor rule with min_margin_pct=0.15`
- `Integration (mocked LLM): ambiguous input → returns rules with human-review flag`
- `Unit: explain_strategy generates readable English summary of rule chain`
- `Integration: preview_only=True → rules returned but not persisted`
- `Integration: preview_only=False → strategy and rules created in database`

#### 12.2 — Rate Limiting & API Security

**What**: Rate limiting, request validation, and OWASP API Security Top 10 mitigations.

**Design**:

```python
# src/pricing_engine/middleware/rate_limit.py
from fastapi import Request
from redis.asyncio import Redis

class RateLimiter:
    def __init__(self, redis: Redis, default_limit: int = 100, window_seconds: int = 60):
        ...

    async def check(self, key: str) -> tuple[bool, dict]:
        """Return (allowed, headers) where headers include X-RateLimit-* fields."""
        ...

# Rate limits by endpoint category:
RATE_LIMITS = {
    "auth": (10, 60),        # 10 requests per minute (brute-force protection)
    "read": (200, 60),       # 200 reads per minute
    "write": (50, 60),       # 50 writes per minute
    "repricing": (10, 60),   # 10 repricing runs per minute
    "scraping": (5, 60),     # 5 scraping triggers per minute
}
```

**Testing**:
- `Integration: 11th auth request in 1 minute → 429 Too Many Requests`
- `Integration: rate limit headers (X-RateLimit-Limit, X-RateLimit-Remaining) present in response`
- `Unit: tenant A's rate limit does not affect tenant B`
- `Integration: request without Authorization header on protected endpoint → 401`
- `Integration: SQL injection attempt in query parameter → parameterised query, no injection`

#### 12.3 — Monitoring, Logging & Health Metrics

**What**: Structured logging, Prometheus metrics, and application health monitoring.

**Design**:

```python
# Key Prometheus metrics:
# pricing_repricing_runs_total (counter, labels: tenant, strategy, status)
# pricing_repricing_duration_seconds (histogram)
# pricing_prices_changed_total (counter, labels: tenant, channel, rule_type)
# pricing_competitor_scrape_duration_seconds (histogram)
# pricing_competitor_scrape_errors_total (counter, labels: competitor, error_type)
# pricing_webhook_deliveries_total (counter, labels: event_type, status)
# pricing_api_request_duration_seconds (histogram, labels: method, path, status)
# pricing_active_experiments_gauge (gauge, labels: tenant)
```

```python
# src/pricing_engine/middleware/logging.py
import structlog

def setup_logging():
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.JSONRenderer(),
        ],
    )
```

**Testing**:
- `Integration: GET /metrics → Prometheus text format with all expected metrics`
- `Integration: repricing run increments pricing_repricing_runs_total counter`
- `Integration: API request logs include request_id, tenant_id, duration_ms`
- `Unit: structured log entries are valid JSON with required fields`

#### 12.4 — Inventory-Aware Pricing Integration

**What**: Pull inventory data from channels and feed into pricing rules (inventory markdown, perishable goods time decay).

**Design**:

```python
# src/pricing_engine/services/inventory_service.py
class InventoryService:
    async def sync_inventory(self, channel_id: UUID) -> dict:
        """Pull inventory levels from channel and update ts_inventory_snapshots."""
        ...

    async def compute_days_of_supply(self, product_id: UUID, channel_id: UUID) -> float | None:
        """Days of supply = current inventory / avg daily sales (30-day rolling)."""
        ...
```

**Testing**:
- `Integration (mocked): sync_inventory from Shopify → inventory_levels updated`
- `Unit: days_of_supply with 100 units and avg 10/day → 10.0 days`
- `Unit: days_of_supply with zero avg sales → None (avoid division by zero)`
- `Integration: inventory_markdown rule uses live days_of_supply value`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Layer          ─── required by everything
    │
Phase 2: Product Catalogue & Channels    ─── requires Phase 1
    │
Phase 3: Competitor Monitoring            ─── requires Phase 2
    │
Phase 4: Rule-Based Repricing Engine      ─── requires Phase 2, Phase 3
    │
Phase 5: Compliance & Price History       ─── requires Phase 4
    │
Phase 6: E-Commerce Integrations         ─── requires Phase 2, Phase 4
    │    ├── Phase 7: A/B Testing          ─── requires Phase 4; can parallel with Phase 6
    │    └── Phase 8: Demand & Elasticity  ─── requires Phase 3, Phase 4; can parallel with Phase 6, Phase 7
    │
Phase 9: Webhooks & Events               ─── requires Phase 4; can parallel with Phase 6-8
    │
Phase 10: Analytics Dashboard            ─── requires Phases 3-5 (API endpoints); can start after Phase 5
    │
Phase 11: RL Pricing Agent               ─── requires Phase 4, Phase 8
    │
Phase 12: Advanced & Production          ─── requires all prior phases
```

### Parallelism Opportunities

- **Phases 6, 7, 8, 9** can be developed concurrently after Phase 4 is complete (they share Phase 4 as a dependency but are independent of each other).
- **Phase 10** (frontend) can start after Phase 5 and proceed in parallel with Phases 6-9, consuming API endpoints as they become available.
- **Phase 11** (RL agent) requires Phase 8 (elasticity/demand data) but can begin environment design in parallel with Phase 8 development.

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass with > 90% line coverage for new code.
3. All integration tests pass against a real PostgreSQL/TimescaleDB instance (via docker-compose).
4. `ruff check` reports zero errors.
5. `mypy --strict` passes on all new modules.
6. Docker build succeeds for all modified services.
7. Alembic migrations run cleanly (upgrade and downgrade).
8. API endpoints appear in auto-generated OpenAPI 3.1 spec at `/docs`.
9. New configuration options documented in `.env.example`.
10. No security credentials committed to source (checked via pre-commit hook).
11. Performance: API endpoints respond in < 200ms at p95 for single-product queries.
12. Feature works end-to-end via manual smoke test or E2E test.
