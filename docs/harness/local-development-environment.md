# Local Development Environment: Docker Compose Parity

**Principle**: Every component that exists in production has a local equivalent via Docker Compose. Local dev should match production as closely as possible.

---

## The Parity Principle

```
Local Development Environment      Production Infrastructure
────────────────────────────────  ──────────────────────────
docker compose up                  Terraform deployment
  ├── backend container              ├── Backend service (Container App)
  ├── frontend container             ├── Frontend service (CDN)
  ├── database (Postgres)            ├── Database (Managed SQL)
  ├── cache (Redis)                  └── Cache (Azure Cache for Redis)
  └── monitoring (logs, metrics)
```

---

## Why Parity Matters

1. **Confidence**: If it works locally, it works in production
2. **Debugging**: Replicate production issues locally
3. **Performance**: Catch bottlenecks before deployment
4. **Cost**: No need for dev/test cloud environments (save money)
5. **Speed**: Deploy in minutes (local), not hours (cloud)

---

## Template: Service Compose Files

### Service 1: Backend

**File: `backend/compose.yaml`**

```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=development
      - DATABASE_URL=postgresql://user:password@db:5432/app
      - REDIS_URL=redis://cache:6379
      - LOG_LEVEL=debug
    depends_on:
      - db
      - cache
    volumes:
      - ./src:/app/src              # Hot reload for development
    networks:
      - backend

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=app
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - backend

  # Optional: Monitoring
  logs:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    networks:
      - backend

volumes:
  postgres_data:

networks:
  backend:
```

**Development commands:**
```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f app

# Run migrations
docker compose exec app python migrations/apply.py

# Stop
docker compose down

# Clean (remove volumes)
docker compose down -v
```

**Maps to production:**
| Local | Production |
|-------|-----------|
| `app` (Python container) | Container App (managed) |
| `db` (Postgres local) | Azure SQL Database (managed) |
| `cache` (Redis local) | Azure Cache for Redis (managed) |
| `logs` (Loki local) | Application Insights (cloud) |

---

### Service 2: Frontend

**File: `frontend/compose.yaml`**

```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8000
      - REACT_APP_ENV=development
    depends_on:
      - backend  # For integration testing
    volumes:
      - ./src:/app/src               # Hot reload
    networks:
      - frontend

  # Optional: Integration test runner
  integration-tests:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - TEST_API_URL=http://localhost:8000
    depends_on:
      - app
    networks:
      - frontend

networks:
  frontend:
```

**Development commands:**
```bash
# Start frontend dev server
docker compose up -d

# Run tests
docker compose run integration-tests npm test

# Build production image
docker compose build --no-cache

# View live logs
docker compose logs -f app
```

**Maps to production:**
| Local | Production |
|-------|-----------|
| `app` (React dev server) | Static assets on CDN |
| Dev hot-reload | Optimized build on CDN |

---

### Service 3: Infrastructure (Terraform Validation)

**File: `infrastructure/compose.yaml`**

```yaml
version: '3.9'

services:
  # Terraform validation without cloud credentials
  terraform:
    image: hashicorp/terraform:latest
    working_dir: /terraform
    volumes:
      - ./terraform:/terraform
    entrypoint: terraform
    command: validate

  # (Optional) Terraform planning container
  terraform-plan:
    image: hashicorp/terraform:latest
    working_dir: /terraform
    volumes:
      - ./terraform:/terraform
      - ~/.azure:/root/.azure  # Azure credentials (if needed for planning)
    environment:
      - ARM_SUBSCRIPTION_ID=${AZURE_SUBSCRIPTION_ID}
      - ARM_TENANT_ID=${AZURE_TENANT_ID}
      - ARM_CLIENT_ID=${AZURE_CLIENT_ID}
      - ARM_CLIENT_SECRET=${AZURE_CLIENT_SECRET}
    entrypoint: terraform
    command: plan
```

**Development commands:**
```bash
# Validate Terraform syntax
docker compose run terraform

# Plan (requires credentials)
docker compose run terraform-plan

# Format check
docker compose run terraform fmt -check terraform/
```

---

## Full Stack: Unified Compose

**File: `infrastructure/compose.yaml` (orchestrates all services)**

For integration testing, orchestrate all services together:

```yaml
version: '3.9'

services:
  # Backend
  backend:
    build:
      context: ../backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/app
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache
    networks:
      - app-network

  # Frontend
  frontend:
    build:
      context: ../frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://backend:8000
    depends_on:
      - backend
    networks:
      - app-network

  # Shared Database
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=app
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  # Shared Cache
  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

  # Monitoring (optional)
  monitoring:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
```

**Start entire stack:**
```bash
cd infrastructure
docker compose up -d

# Verify all services
docker compose ps

# Access apps
# Frontend: http://localhost:3000
# Backend API: http://localhost:8000
# Prometheus: http://localhost:9090
```

---

## Adding New Services: Composition Pattern

**When adding a new service (e.g., Analytics):**

### Step 1: Update main `infrastructure/compose.yaml`

```yaml
services:
  # ... existing services ...

  analytics:
    build:
      context: ../analytics
      dockerfile: Dockerfile
    ports:
      - "8001:8001"
    environment:
      - CLICKHOUSE_URL=http://clickhouse:8123
      - BACKEND_URL=http://backend:8000
    depends_on:
      - clickhouse
      - backend
    networks:
      - app-network

  # New database for analytics service
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
    networks:
      - app-network

volumes:
  postgres_data:
  clickhouse_data:
```

### Step 2: Each service has own `compose.yaml` for isolation

**File: `analytics/compose.yaml`**

```yaml
version: '3.9'

services:
  app:
    build: .
    ports:
      - "8001:8001"
    environment:
      - CLICKHOUSE_URL=clickhouse://clickhouse:9000
    depends_on:
      - clickhouse
    networks:
      - analytics

  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
    networks:
      - analytics

networks:
  analytics:
```

**Allows testing in isolation:**
```bash
cd analytics
docker compose up  # Just analytics service

cd ..
docker compose up  # All services integrated
```

---

## Environment Variables: Local vs Production

**Principle**: Same service code, different environment files

### `.env.local` (development)
```bash
ENVIRONMENT=development
DATABASE_URL=postgresql://user:password@db:5432/app
REDIS_URL=redis://cache:6379
API_LOG_LEVEL=debug
CORS_ORIGINS=http://localhost:3000
```

### `.env.production` (cloud deployment)
```bash
ENVIRONMENT=production
DATABASE_URL=postgresql://[managed-db-connection-string]
REDIS_URL=redis://[managed-redis-connection-string]
API_LOG_LEVEL=info
CORS_ORIGINS=https://app.customer.com
```

**Codex generates both** when creating service repos.

---

## Dockerfile Strategy: Local-to-Production Parity

**Principle**: Same Dockerfile runs locally and in cloud

```dockerfile
# backend/Dockerfile

FROM python:3.11-slim as base

WORKDIR /app

# Dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Source code
COPY src ./src

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Same Dockerfile:**
- `docker compose up` → runs locally
- `docker push` → runs in cloud Container App
- No special "production" Dockerfile

---

## Data Seeding: Reproducible State

**When developing, need realistic data.**

### Option 1: SQL seed file

**File: `backend/seeds/development.sql`**
```sql
INSERT INTO users (id, email, name) VALUES
  (1, 'alice@example.com', 'Alice'),
  (2, 'bob@example.com', 'Bob');

INSERT INTO projects (id, user_id, name) VALUES
  (1, 1, 'Project A'),
  (2, 1, 'Project B');
```

**Load in compose:**
```yaml
db:
  image: postgres:15-alpine
  volumes:
    - ./seeds/development.sql:/docker-entrypoint-initdb.d/01-seed.sql
```

### Option 2: Initialization script

**File: `backend/scripts/seed.py`**
```python
import os
from sqlalchemy import create_engine
from models import User, Project

engine = create_engine(os.getenv("DATABASE_URL"))
session = Session(engine)

session.add(User(email="alice@example.com", name="Alice"))
session.add(User(email="bob@example.com", name="Bob"))
session.commit()
```

**Run after compose startup:**
```bash
docker compose up -d
sleep 5  # wait for DB to be ready
docker compose exec backend python scripts/seed.py
```

---

## Testing Locally: Parity with CI/CD

**Every PR runs the same tests locally and in CI:**

```bash
# Backend tests
cd backend
docker compose run app pytest tests/

# Frontend tests
cd frontend
docker compose run app npm test

# Infrastructure validation
cd infrastructure
docker compose run terraform validate
docker compose run terraform-plan  # Requires credentials
```

**Same test containers run in GitHub Actions CI/CD**

---

## Performance: Optimization for Development

### Memory constraints

If running multiple services on dev laptop:
```yaml
# Limit memory per service
services:
  db:
    image: postgres:15-alpine
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

### Disk space

```bash
# Clean up old images
docker image prune -a

# Remove unused volumes
docker volume prune

# See current usage
docker system df
```

---

## Troubleshooting Common Issues

### Port conflicts
```bash
# If port 5432 already in use
docker compose down  # Stop other services

# Or change port mapping
# In compose.yaml: 5433:5432
```

### Database connection errors
```bash
# Wait for DB to be ready
docker compose logs db
sleep 10
docker compose up -d  # Restart app after DB is healthy
```

### Permission errors in volumes
```bash
# If mounted volumes have permission issues
sudo chown -R $(id -u):$(id -g) .

# Or in compose, run as specific user
services:
  app:
    user: "1000:1000"  # Match your user ID
```

---

## See Also

- **[customer-repository-structure.md](../deployment/customer-repository-structure.md)** — How services are organized
- **[execution-streams-codex-vs-harness.md](../architecture/execution-streams-codex-vs-harness.md)** — Where services are generated
- **[dev-house-operational-infrastructure.md](../architecture/dev-house-operational-infrastructure.md)** — Where Dev-House itself runs
