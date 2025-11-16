# CLAUDE.md - Dify Development Guide for AI Assistants

## Project Overview

Dify is an open-source platform for developing LLM applications with an intuitive interface combining agentic AI workflows, RAG pipelines, agent capabilities, and model management.

**Current Version:** 1.10.0

**Key Technologies:**
- Backend: Python 3.11-3.12, Flask 3.1.2, SQLAlchemy 2.0.29, Celery 5.5.2
- Frontend: Next.js 15.5, React 19.1, TypeScript 5.9.3, Node.js >=22.11.0
- Data: PostgreSQL with pgvector, Redis, 20+ vector database integrations
- Package Managers: UV (Python backend), pnpm 10.22.0 (Node.js frontend)

## Repository Structure

The codebase is organized as a monorepo:

```
/home/user/dify/
├── api/              # Python Flask backend (Domain-Driven Design)
├── web/              # Next.js 15 frontend (TypeScript & React 19)
├── docker/           # Docker deployment configurations
├── docs/             # Multi-language documentation (20+ languages)
├── sdks/             # Client SDKs (Python, Node.js, PHP)
├── dev/              # Development tools and scripts
└── .github/          # CI/CD workflows and templates
```

### Backend Structure (`/api`)

**Layered Architecture (DDD + Clean Architecture):**

- **`/api/core`** - Domain Layer (business logic)
  - `agent/` - Agent execution engine
  - `workflow/` - Workflow graph engine with nodes
  - `rag/` - RAG pipeline (datasource, extractor, embedding, rerank)
  - `model_runtime/` - LLM provider integrations
  - `tools/` - Built-in and custom tools
  - `plugin/`, `mcp/` - Plugin and MCP systems
  - `entities/`, `schemas/` - Domain models

- **`/api/controllers`** - Presentation Layer (API routes)
  - `console/` - Admin console endpoints
  - `web/` - Public web app endpoints
  - `service_api/` - External integration API

- **`/api/models`** - Data Layer (SQLAlchemy ORM models)
  - Database models: `account.py`, `dataset.py`, `workflow.py`, etc.

- **`/api/services`** - Application Service Layer
  - Business logic services bridging controllers and domain
  - Examples: `app_generate_service.py`, `workflow_draft_variable_service.py`

- **`/api/repositories`** - Data Access Layer
  - Repository pattern implementations for data persistence

- **`/api/tasks`** - Background Jobs (Celery)
  - Asynchronous task processing

- **`/api/tests`** - Test Suite
  - `unit_tests/` - Pytest unit tests (Arrange-Act-Assert)
  - `integration_tests/` - CI-only integration tests

- **`/api/migrations`** - Database migrations (Alembic)

### Frontend Structure (`/web`)

**Next.js App Router with File-Based Routing:**

- **`/web/app`** - Pages and layouts
  - `(commonLayout)/` - Main app routes
    - `apps/`, `datasets/`, `tools/`, `plugins/`
  - `signin/`, `signup/`, `account/` - Auth routes

- **`/web/app/components`** - React components
  - `base/` - Reusable UI components
  - `plugins/`, `explore/`, `rag-pipeline/` - Feature components

- **`/web/service`** - API service layer (HTTP clients using `ky`)
  - Type-safe API calls with explicit return types

- **`/web/context`** - React Context providers
  - Global state management using Context API

- **`/web/models`** - TypeScript type definitions
  - Domain model types

- **`/web/hooks`** - Custom React hooks
  - Reusable hook library

- **`/web/utils`** - Utility functions

- **`/web/i18n`** - Internationalization files
  - 20+ languages with TypeScript exports

- **`/web/__tests__`** - Jest test suite

## Development Setup

### Quick Start

```bash
# One-command setup
make dev-setup

# This runs:
# 1. make prepare-docker  - Start middleware (PostgreSQL, Redis, Weaviate)
# 2. make prepare-web     - Install pnpm dependencies
# 3. make prepare-api     - Install UV dependencies and run migrations
```

### Manual Setup

**Backend:**
```bash
cd api
cp .env.example .env
uv sync --dev
uv run flask db upgrade
```

**Frontend:**
```bash
cd web
cp .env.example .env
pnpm install
```

**Middleware (Docker):**
```bash
cd docker
cp middleware.env.example middleware.env
docker compose -f docker-compose.middleware.yaml up -d
```

### Running the Application

**Backend API Server:**
```bash
dev/start-api
# Or: uv run --project api flask run --host 0.0.0.0 --port=5001 --debug
```

**Celery Worker:**
```bash
dev/start-worker --queues dataset,workflow
```

**Frontend Dev Server:**
```bash
cd web
pnpm dev
```

## Backend Development Workflow

### Code Quality Commands

**IMPORTANT:** Before submission, all backend modifications must pass:

```bash
make lint        # Format, check with fixes, and run import linter
make type-check  # Run basedpyright type checking
uv run --project api bash dev/pytest/pytest_unit_tests.sh  # Run unit tests
```

**Individual Commands:**
```bash
make format     # Format code with ruff
make check      # Check code with ruff (no fixes)
```

### Running Tests

**Unit Tests (Required before submission):**
```bash
uv run --project api bash dev/pytest/pytest_unit_tests.sh
```

**Other Test Suites:**
```bash
dev/pytest/pytest_workflow.sh      # Workflow tests
dev/pytest/pytest_tools.sh         # Tool tests
dev/pytest/pytest_testcontainers.sh # TestContainers tests
```

**Integration Tests:** CI-only, not expected to run locally

### CLI Commands

All backend CLI commands use `uv run`:
```bash
uv run --project api <command>
```

### Architecture Conventions

**Domain-Driven Design:**
- Clear layer separation: models → entities → services → controllers
- Domain entities are independent of storage mechanisms
- Use dependency injection through constructors

**Models (SQLAlchemy 2.0):**
```python
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

class Account(db.Model):
    id: Mapped[str] = mapped_column(StringUUID, server_default=sa.text("uuid_generate_v4()"))
    name: Mapped[str] = mapped_column(String(255))
    email: Mapped[str] = mapped_column(String(255))
    password: Mapped[str | None] = mapped_column(String(255), default=None)
```

**Domain Entities (Pydantic):**
```python
from pydantic import BaseModel, Field

class WorkflowExecution(BaseModel):
    id_: str = Field(...)
    workflow_id: str = Field(...)

    @property
    def elapsed_time(self) -> float:
        """Calculate elapsed time in seconds."""
        # Implementation

    @classmethod
    def new(cls, *, id_: str, workflow_id: str) -> "WorkflowExecution":
        return WorkflowExecution(id_=id_, workflow_id=workflow_id)
```

**Service Layer:**
```python
class AppService:
    def create_app(self, tenant_id: str, args: dict, account: Account) -> App:
        """Create app with validation."""
        # Business logic here
        app = App(**args)
        db.session.add(app)
        db.session.flush()
        return app
```

### Type Annotations

**Strict Type Checking Policy:**
- All function signatures MUST have type hints
- Use union types with `|` operator: `str | None` (not `Optional[str]`)
- Avoid `Any` - prefer explicit types
- Type hints on class attributes using `Mapped` for SQLAlchemy

**Example:**
```python
def get_paginate_apps(
    self,
    user_id: str,
    tenant_id: str,
    args: dict
) -> Pagination | None:
    """Get app list with pagination."""
    # Implementation
```

### Error Handling

**Domain-Specific Exceptions:**
```python
class WorkflowNodeRunFailedError(Exception):
    def __init__(self, node: Node, err_msg: str):
        self._node = node
        self._error = err_msg
        super().__init__(f"Node {node.title} run failed: {err_msg}")

    @property
    def node(self) -> Node:
        return self._node
```

- Raise domain-specific exceptions at the correct layer
- Store context in exception properties
- Use clear, descriptive exception names

### Testing Patterns

**Pytest with Arrange-Act-Assert:**
```python
def test_create_account():
    # Arrange
    email = "test@example.com"
    name = "Test User"

    # Act
    account = Account(email=email, name=name)

    # Assert
    assert account.email == email
    assert account.name == name
```

**Factory Pattern for Test Data:**
```python
class TestAccountFactory:
    @staticmethod
    def create_account_mock(
        account_id: str = "user-123",
        email: str = "test@example.com",
        **kwargs,
    ) -> MagicMock:
        account = MagicMock(spec=Account)
        account.id = account_id
        account.email = email
        return account
```

### Python Code Style

**Key Conventions:**
- Line length: 120 characters
- Quote style: double quotes
- Implement `__repr__` and `__str__` for classes
- Use dataclasses or Pydantic for data structures
- Prefer comprehensions over map/filter
- Use f-strings for formatting

**Ruff Configuration:**
- Auto-fix available: `ruff check --fix`
- Format code: `ruff format`
- Rules enforced: flake8-bugbear, pycodestyle, pyflakes, isort, pyupgrade, security

## Frontend Development Workflow

### Code Quality Commands

**IMPORTANT:** Before submission, frontend modifications must pass:

```bash
cd web
pnpm lint       # Run ESLint
pnpm type-check # TypeScript type checking
pnpm test       # Run Jest tests
```

**Auto-fix Issues:**
```bash
pnpm lint:fix   # Auto-fix ESLint issues
```

### Internationalization (i18n)

**CRITICAL RULE:** Frontend user-facing strings MUST use i18n files in `web/i18n/en-US/`

**Never hardcode user-facing text:**
```typescript
// ❌ BAD
<Button>Create App</Button>

// ✅ GOOD
const { t } = useTranslation()
<Button>{t('app.createApp')}</Button>
```

**Adding New Translations:**
1. Add to relevant file in `/web/i18n/en-US/` (e.g., `app.ts`, `workflow.ts`)
2. Run `pnpm check:i18n-types` to verify synchronization
3. Run `pnpm auto-gen-i18n` to generate for other languages (if needed)

### Component Patterns

**Functional Components with TypeScript:**
```typescript
'use client'
import { useTranslation } from 'react-i18next'

export type AppCardProps = {
  app: App
  onRefresh?: () => void
}

const AppCard = ({ app, onRefresh }: AppCardProps) => {
  const { t } = useTranslation()

  return (
    <div>
      <h2>{app.name}</h2>
    </div>
  )
}

export default AppCard
```

**Key Conventions:**
- Use `'use client'` directive for client components
- Define explicit prop types
- Use named exports for components
- Functional components with hooks (no class components)

### State Management

**Local State:**
```typescript
const [isOpen, setIsOpen] = useState(false)
```

**Context with Selectors:**
```typescript
import { useContextSelector } from 'use-context-selector'

const userProfile = useContextSelector(AppContext, v => v.userProfile)
```

**Data Fetching (SWR):**
```typescript
const { data, error, mutate } = useSWR('/api/apps', fetchAppList)
```

### API Integration

**Service Layer Pattern:**
```typescript
import { get, post } from '@/service/base'
import type { Fetcher } from 'swr'

export const fetchAppList: Fetcher<AppListResponse, { url: string }> = ({ url }) => {
  return get<AppListResponse>(url)
}

export const createApp = (data: CreateAppRequest) => {
  return post<AppDetailResponse>('apps', { body: data })
}
```

### TypeScript Conventions

**Strict Mode Configuration:**
- No `any` types - use explicit typing
- Prefer `type` over `interface` for consistency
- Use proper type imports: `import type { ... }`
- Path aliases: `@/*` maps to project root

**Type Definitions:**
```typescript
// Define explicit types
export type App = {
  id: string
  name: string
  mode: AppModeEnum
  created_at: string
}

// Use enums for constants
export enum AppModeEnum {
  CHAT = 'chat',
  AGENT_CHAT = 'agent-chat',
  COMPLETION = 'completion',
  WORKFLOW = 'workflow',
}
```

### Testing Patterns

**Jest + React Testing Library:**
```typescript
import { cleanup, fireEvent, render } from '@testing-library/react'

afterEach(cleanup)

describe('Button', () => {
  test('should render text from children', () => {
    const { getByRole } = render(<Button>Click me</Button>)
    expect(getByRole('button').textContent).toBe('Click me')
  })

  test('should call onClick when clicked', () => {
    const onClick = jest.fn()
    const { getByRole } = render(<Button onClick={onClick}>Click</Button>)

    fireEvent.click(getByRole('button'))
    expect(onClick).toHaveBeenCalledTimes(1)
  })
})
```

### React/TypeScript Code Style

**Key Conventions:**
- Indentation: 2 spaces
- Quotes: single
- No semicolons
- Use arrow functions for components
- Destructure props in function parameters
- Use `const` for all declarations (no `let` unless necessary)

**ESLint Configuration:**
- Auto-fix available: `eslint --fix`
- Plugins: Next.js, React Hooks, SonarJS, Tailwind CSS
- Complexity checking available

## Testing & Quality Practices

### Test-Driven Development (TDD)

Follow the red → green → refactor cycle:
1. Write failing test (red)
2. Write minimum code to pass (green)
3. Refactor while keeping tests green

### Backend Testing

**Unit Tests (Pytest):**
- Location: `/api/tests/unit_tests/`
- Structure: Arrange-Act-Assert pattern
- Use fixtures for shared setup
- Mock external dependencies
- Timeout: 20 seconds per test

**Running Tests:**
```bash
uv run --project api bash dev/pytest/pytest_unit_tests.sh
```

### Frontend Testing

**Unit Tests (Jest):**
- Co-located with components: `*.spec.tsx`
- Use `@testing-library/react` utilities
- Test user interactions, not implementation details

**Running Tests:**
```bash
cd web
pnpm test         # Run all tests
pnpm test:watch   # Watch mode
```

### Code Coverage

Backend coverage reports generated in:
- `api/coverage/coverage.json`
- `api/coverage/coverage.xml`

## Architecture Enforcement

### Import Linting

The project uses `import-linter` to enforce architectural boundaries:

```bash
make lint  # Includes import linter checks
```

**Enforced Contracts:**
- Workflow layers (prevent layer violations)
- Domain isolation (domain cannot import infrastructure)
- Graph engine architecture
- Worker management boundaries

## Asynchronous Work

### Celery Background Tasks

**Queue System:**
- Broker: Redis
- Main Queues:
  - `dataset` - RAG indexing and document processing
  - `workflow` - Workflow triggers (community edition)
  - `mail` - Email notifications
  - `ops_trace` - Operations tracing
  - `app_deletion` - Application cleanup
  - `plugin` - Plugin operations
  - `workflow_storage` - Workflow storage tasks
  - `conversation` - Conversation tasks
  - `priority_pipeline`, `pipeline` - Pipeline task processing
  - `schedule_poller`, `schedule_executor` - Scheduled task management
  - `triggered_workflow_dispatcher`, `trigger_refresh_executor` - Trigger management

**Starting Worker:**
```bash
dev/start-worker --queues dataset,workflow
# Or with concurrency:
dev/start-worker --queues dataset,workflow --concurrency 4
# Or let it auto-configure based on edition:
dev/start-worker
```

**Task Organization:**
- Location: `/api/tasks/`
- Examples: `annotation/`, `rag_pipeline/`, `workflow_cfs_scheduler/`

**Task Pattern:**
```python
from celery import shared_task

@shared_task(queue='dataset')
def process_dataset(dataset_id: str):
    """Process dataset in background."""
    # Implementation
```

## Configuration Management

### Environment Variables

**Backend (.env):**
- Template: `/api/.env.example`
- Database: PostgreSQL connection string
- Redis: Connection and Sentinel configuration
- Storage: S3/Azure/GCS credentials
- Model Providers: API keys

**Frontend (.env):**
- Template: `/web/.env.example`
- API endpoints
- Public paths
- Feature flags

### Feature Flags

Located in `/api/configs/feature/`:
- Use Pydantic for validation
- Environment-based configuration
- Type-safe access

## Docker Development

### Middleware Services

Start development dependencies:
```bash
cd docker
docker compose -f docker-compose.middleware.yaml up -d
```

**Services:**
- PostgreSQL with pgvector
- Redis
- Sandbox (code execution)
- SSRF Proxy
- Weaviate (vector database)

### Full Stack

```bash
cd docker
docker compose up -d
```

## CI/CD Pipeline

### GitHub Actions Workflows

**Main CI (`.github/workflows/main-ci.yml`):**
- Runs on PR and push to main
- Change detection determines which tests run
- Parallel execution: API tests, Web tests, Style checks

**Pre-commit Checks:**
- Husky hooks on frontend
- Automatic linting and type checking
- Unit tests for modified utils

### Code Quality Gates

**Backend:**
1. Import linter (architectural boundaries)
2. Basedpyright (type checking)
3. Mypy (additional type checking)
4. Ruff (linting and formatting)
5. Unit tests (pytest)

**Frontend:**
1. ESLint (linting)
2. TypeScript (type checking)
3. i18n type sync
4. Jest (unit tests)

## General Best Practices

### Code Documentation

- **Self-documenting code preferred** - use clear variable and function names
- **Comments explain "why", not "what"** - code should be obvious
- **Type hints as documentation** - types should make intent clear
- **Docstrings for public APIs** - document module/class/function purpose

### Dependency Injection

Inject dependencies through constructors:
```python
class WorkflowService:
    def __init__(self, workflow_repository: WorkflowRepository):
        self._repository = workflow_repository
```

### Clean Architecture Boundaries

- Domain layer cannot depend on infrastructure
- Services orchestrate domain logic
- Controllers handle HTTP concerns only
- Repositories abstract data access

### File Organization

**IMPORTANT:** Prefer editing existing files over creating new ones:
- Extend existing components/services/utilities
- Only create new files when adding distinct features
- Do not create documentation files unless explicitly requested

### Security Considerations

Prevent common vulnerabilities:
- SQL injection: Use SQLAlchemy parameterized queries
- XSS: Sanitize user input, use React's built-in escaping
- Command injection: Validate and sanitize shell inputs
- SSRF: Use SSRF proxy for external requests
- Secrets: Never commit to `.env` files

## Version Control

### Git Workflow

**Branch Naming:**
- Features: `feature/description`
- Fixes: `fix/description`
- CI: Special branch patterns for Claude Code

**Commit Messages:**
- Use conventional commits format
- Examples: `feat:`, `fix:`, `refactor:`, `test:`, `docs:`
- Be descriptive but concise

### Pre-commit Hooks

Frontend has Husky hooks that:
1. Run ESLint auto-fix on staged files
2. Run TypeScript type-check if `.ts`/`.tsx` changed
3. Run unit tests for modified utils

## Key Files Reference

### Configuration Files

**Backend:**
- `/api/pyproject.toml` - UV project configuration, dependencies
- `/api/.env.example` - Environment variable template
- `/api/.ruff.toml` - Linting configuration
- `/api/pytest.ini` - Test configuration
- `/api/.importlinter` - Architecture enforcement

**Frontend:**
- `/web/package.json` - npm scripts and dependencies
- `/web/.env.example` - Environment variables
- `/web/next.config.js` - Next.js configuration
- `/web/tsconfig.json` - TypeScript configuration
- `/web/tailwind.config.js` - Tailwind CSS configuration
- `/web/eslint.config.mjs` - ESLint configuration
- `/web/jest.config.ts` - Jest configuration

**Docker:**
- `/docker/docker-compose.yaml` - Production deployment
- `/docker/docker-compose.middleware.yaml` - Development middleware
- `/docker/.env.example` - Docker environment variables

**Repository:**
- `/Makefile` - Development workflow automation
- `/.github/workflows/` - CI/CD pipelines

### Entry Points

**Backend:**
- `/api/app.py` - Flask application entry
- `/api/app_factory.py` - Application factory
- `/api/celery_entrypoint.py` - Celery worker entry
- `/api/commands.py` - CLI commands

**Frontend:**
- `/web/app/layout.tsx` - Root layout
- `/web/app/page.tsx` - Root page

## Quick Reference Commands

### Development Setup
```bash
make dev-setup              # Complete setup
make prepare-docker         # Start middleware
make prepare-web           # Setup web
make prepare-api           # Setup API
```

### Backend Development
```bash
dev/start-api              # Start Flask server
dev/start-worker           # Start Celery worker
make lint                  # Lint and format
make type-check            # Type checking
uv run --project api bash dev/pytest/pytest_unit_tests.sh  # Tests
```

### Frontend Development
```bash
cd web
pnpm dev                   # Dev server
pnpm lint:fix              # Lint and fix
pnpm type-check            # Type check
pnpm test                  # Run tests
pnpm check:i18n-types      # Check i18n
```

### Docker
```bash
cd docker
docker compose -f docker-compose.middleware.yaml up -d  # Middleware
docker compose up -d                                     # Full stack
```

## Summary

Dify follows modern software engineering practices:

- **Clean Architecture** with DDD in backend
- **Strict typing** in both Python and TypeScript
- **Test-Driven Development** with comprehensive test coverage
- **Automated quality gates** via CI/CD
- **Internationalization-first** approach for user-facing text
- **Security by design** with OWASP awareness
- **Performance optimization** through caching and async processing

When developing for Dify:
1. Always use type hints/annotations
2. Follow the layer architecture strictly
3. Write tests before or alongside code
4. Use i18n for all user-facing strings
5. Run linting and type-checking before submission
6. Prefer editing existing files over creating new ones
