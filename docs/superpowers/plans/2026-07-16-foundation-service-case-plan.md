# Platform Foundation and ServiceCase Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first production-shaped vertical slice: a Python 3.11/FastAPI service that validates real OIDC identities and creates, submits, retrieves, audits, and safely retries PostgreSQL-backed student `ServiceCase` records.

**Architecture:** Use one installable backend package with strict `domain -> application -> infrastructure/API` dependency direction. Domain state changes are synchronous and deterministic; asynchronous application handlers perform authorization and persistence through ports, while PostgreSQL stores the aggregate, append-only events, idempotency results, and transactional Outbox rows. Keycloak supplies local OIDC tokens, but JWT validation and authorization contracts remain compatible with a university IdP.

**Tech Stack:** Python 3.11, Conda, FastAPI, Pydantic v2, SQLAlchemy 2 async, psycopg 3, Alembic, PostgreSQL 16, Keycloak, PyJWT, pytest, pytest-asyncio, Testcontainers, Ruff, mypy, Docker Compose.

## Global Constraints

- Local repository: `D:\agent\student-affairs-agent`; Conda prefix: `D:\conda_envs\student-affairs-agent`.
- One independently deployed instance serves one university; local single-node infrastructure must preserve production interfaces and state semantics.
- `ServiceCase` is the business source of truth; conversation, cache, queue, and checkpoint state cannot replace it.
- LLM output is untrusted and cannot directly mutate business tables. This milestone does not add an LLM dependency.
- Student and staff identities come from OIDC; business endpoints never accept caller identity from request bodies.
- Authorization is enforced before repository results are disclosed. Cross-student lookup returns `404`.
- Business writes use optimistic locking, idempotency records, append-only events, and transactional Outbox rows.
- Real credentials, student data, tokens, and unredacted traces must never be committed.
- Python code uses UTC-aware datetimes, UUID identifiers, explicit types, and ASCII identifiers.
- All implementation uses tests first and ends each task with a focused verification and commit.

---

## Target File Map

```text
student-affairs-agent/
├── apps/
│   └── api/
│       └── __main__.py                      # Local API process entry point
├── packages/
│   └── backend/
│       └── src/student_affairs/
│           ├── __init__.py
│           ├── api/
│           │   ├── app.py                   # FastAPI application factory
│           │   ├── dependencies.py          # Settings, UoW and identity dependencies
│           │   ├── errors.py                # Domain/application error to HTTP mapping
│           │   ├── health.py                # Liveness and readiness routes
│           │   └── v1/cases.py              # Case command/query endpoints
│           ├── application/
│           │   ├── authorization.py          # RBAC/ownership policies
│           │   ├── case_handlers.py          # Create/submit/get use cases
│           │   ├── case_ports.py             # Repository and UoW protocols
│           │   ├── case_schemas.py           # Command/result dataclasses
│           │   └── errors.py                 # Stable application errors
│           ├── domain/
│           │   ├── case.py                   # ServiceCase aggregate and events
│           │   └── errors.py                 # Domain invariant errors
│           ├── identity/
│           │   ├── context.py                # Trusted IdentityContext
│           │   └── oidc.py                   # OIDC/JWKS token verifier
│           ├── infrastructure/
│           │   └── postgres/
│           │       ├── models.py             # SQLAlchemy persistence models
│           │       ├── repositories.py       # Port implementations and mappings
│           │       ├── session.py            # Async engine/session factory
│           │       └── uow.py                # Transactional Unit of Work
│           └── settings.py                    # Environment configuration
├── deploy/
│   └── compose/
│       ├── keycloak/realm-student-affairs.json
│       └── postgres/init/001-create-app-role.sql
├── migrations/
│   ├── env.py
│   └── versions/0001_service_case_foundation.py
├── tests/
│   ├── conftest.py
│   ├── unit/domain/test_service_case.py
│   ├── unit/application/test_case_handlers.py
│   ├── unit/identity/test_oidc.py
│   ├── integration/postgres/test_case_uow.py
│   └── system/test_case_api.py
├── .env.example
├── alembic.ini
├── compose.yml
├── environment.yml
└── pyproject.toml
```

Dependency direction is enforced by tests: `domain` imports no project module outside `domain`; `application` may import `domain` and `identity.context`; adapters may import application ports; API composes adapters and handlers.

### Task 1: Reproducible Python Workspace and Quality Gate

**Files:**
- Create: `environment.yml`
- Create: `pyproject.toml`
- Create: `.env.example`
- Create: `packages/backend/src/student_affairs/__init__.py`
- Create: `tests/conftest.py`
- Create: `tests/unit/test_package_contract.py`

**Interfaces:**
- Consumes: Python 3.11 and the confirmed Conda prefix.
- Produces: importable package `student_affairs`, root test configuration, and commands used by every later task.

- [ ] **Step 1: Write the failing package contract test**

```python
# tests/unit/test_package_contract.py
from importlib import import_module
from pathlib import Path


def test_backend_package_imports_from_declared_source_root() -> None:
    package = import_module("student_affairs")
    assert package.__name__ == "student_affairs"


def test_repository_does_not_track_runtime_env_file() -> None:
    root = Path(__file__).parents[2]
    assert not (root / ".env").is_file()
    assert (root / ".env.example").is_file()
```

- [ ] **Step 2: Create the Conda environment and verify the test initially fails**

Create `environment.yml`:

```yaml
name: student-affairs-agent
channels:
  - conda-forge
dependencies:
  - python=3.11
  - pip
  - pip:
      - -e .[dev]
```

Run from `D:\agent\student-affairs-agent`:

```powershell
conda env create --prefix D:\conda_envs\student-affairs-agent --file environment.yml
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/test_package_contract.py -v
```

Expected before creating the package metadata: collection fails because `student_affairs` is not importable or the editable project cannot be installed.

- [ ] **Step 3: Add the installable backend package and tool configuration**

Create `pyproject.toml` with this complete baseline:

```toml
[build-system]
requires = ["setuptools>=75,<82", "wheel>=0.45,<1"]
build-backend = "setuptools.build_meta"

[project]
name = "student-affairs-agent"
version = "0.1.0"
description = "Production-oriented university student affairs AI Agent platform"
requires-python = ">=3.11,<3.12"
dependencies = [
  "alembic>=1.14,<2",
  "fastapi>=0.115,<1",
  "httpx>=0.28,<1",
  "pydantic>=2.10,<3",
  "pydantic-settings>=2.7,<3",
  "psycopg[binary]>=3.2,<4",
  "PyJWT[crypto]>=2.10,<3",
  "sqlalchemy>=2.0.36,<3",
  "structlog>=24.4,<26",
  "uvicorn[standard]>=0.34,<1",
]

[project.optional-dependencies]
dev = [
  "mypy>=1.14,<2",
  "pytest>=8.3,<9",
  "pytest-asyncio>=0.25,<1",
  "pytest-cov>=6,<8",
  "ruff>=0.9,<1",
  "testcontainers[postgres]>=4.9,<5",
]

[tool.setuptools]
package-dir = {"" = "packages/backend/src"}

[tool.setuptools.packages.find]
where = ["packages/backend/src"]

[tool.pytest.ini_options]
addopts = "-ra --strict-markers --strict-config"
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "ASYNC", "S"]
ignore = ["S101"]

[tool.mypy]
python_version = "3.11"
strict = true
packages = ["student_affairs"]
mypy_path = ["packages/backend/src"]
```

Create `packages/backend/src/student_affairs/__init__.py`:

```python
"""University student affairs Agent platform."""

__all__ = ["__version__"]
__version__ = "0.1.0"
```

Create `.env.example`:

```dotenv
APP_ENV=local
LOG_LEVEL=INFO
DATABASE_URL=postgresql+psycopg://student_affairs:@127.0.0.1:5432/student_affairs
OIDC_ISSUER=http://127.0.0.1:18080/realms/student-affairs
OIDC_AUDIENCE=student-affairs-api
OIDC_JWKS_URL=http://127.0.0.1:18080/realms/student-affairs/protocol/openid-connect/certs
POSTGRES_DB=student_affairs
POSTGRES_USER=student_affairs
POSTGRES_PASSWORD=
KEYCLOAK_ADMIN=
KEYCLOAK_ADMIN_PASSWORD=
DOCKER_DATA_ROOT=D:/Programs/DockerData/student-affairs-agent
```

Create `tests/conftest.py` with only shared deterministic fixtures; do not initialize Docker, databases, or clients at import time.

- [ ] **Step 4: Reinstall and run the baseline quality gate**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pip install -e ".[dev]"
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/test_package_contract.py -v
D:\conda_envs\student-affairs-agent\python.exe -m ruff check .
D:\conda_envs\student-affairs-agent\python.exe -m mypy packages/backend/src
```

Expected: package tests pass; Ruff and mypy report no errors.

- [ ] **Step 5: Commit**

```powershell
git add environment.yml pyproject.toml .env.example packages/backend/src/student_affairs tests
git commit -m "build: establish backend workspace"
```

### Task 2: Typed Settings and Observable Application Factory

**Files:**
- Create: `packages/backend/src/student_affairs/settings.py`
- Create: `packages/backend/src/student_affairs/api/app.py`
- Create: `packages/backend/src/student_affairs/api/health.py`
- Create: `packages/backend/src/student_affairs/api/dependencies.py`
- Create: `apps/api/__main__.py`
- Test: `tests/unit/api/test_app.py`

**Interfaces:**
- Consumes: `student_affairs` package from Task 1.
- Produces: `Settings`, `create_app(settings, readiness_probe) -> FastAPI`, `/health/live`, and `/health/ready`.

- [ ] **Step 1: Write failing app-factory tests**

```python
# tests/unit/api/test_app.py
from fastapi.testclient import TestClient

from student_affairs.api.app import create_app
from student_affairs.settings import Settings


class ReadyProbe:
    async def check(self) -> dict[str, str]:
        return {"postgres": "up"}


class FailedProbe:
    async def check(self) -> dict[str, str]:
        raise RuntimeError("postgres unavailable")


def settings() -> Settings:
    return Settings(
        app_env="test",
        database_url="postgresql+psycopg://unused:unused@localhost/unused",
        oidc_issuer="https://idp.example.test/realms/student-affairs",
        oidc_audience="student-affairs-api",
        oidc_jwks_url="https://idp.example.test/certs",
    )


def test_liveness_does_not_depend_on_external_services() -> None:
    response = TestClient(create_app(settings(), ReadyProbe())).get("/health/live")
    assert response.status_code == 200
    assert response.json() == {"status": "alive"}


def test_readiness_reports_dependency_failure_without_stack_trace() -> None:
    response = TestClient(create_app(settings(), FailedProbe())).get("/health/ready")
    assert response.status_code == 503
    assert response.json() == {"detail": "service_not_ready"}
```

- [ ] **Step 2: Run tests and verify import failure**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/api/test_app.py -v
```

Expected: FAIL because `settings.py` and the API factory do not exist.

- [ ] **Step 3: Implement settings, health protocol, and app factory**

```python
# packages/backend/src/student_affairs/settings.py
from functools import lru_cache
from typing import Literal

from pydantic import AnyHttpUrl, Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
        case_sensitive=False,
    )

    app_env: Literal["local", "test", "staging", "production"] = "local"
    log_level: str = "INFO"
    database_url: str
    oidc_issuer: AnyHttpUrl
    oidc_audience: str = Field(min_length=1)
    oidc_jwks_url: AnyHttpUrl


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()  # type: ignore[call-arg]
```

```python
# packages/backend/src/student_affairs/api/health.py
from typing import Protocol

from fastapi import APIRouter, HTTPException


class ReadinessProbe(Protocol):
    async def check(self) -> dict[str, str]: ...


def build_health_router(probe: ReadinessProbe) -> APIRouter:
    router = APIRouter(tags=["health"])

    @router.get("/health/live")
    async def live() -> dict[str, str]:
        return {"status": "alive"}

    @router.get("/health/ready")
    async def ready() -> dict[str, object]:
        try:
            dependencies = await probe.check()
        except Exception as exc:
            raise HTTPException(status_code=503, detail="service_not_ready") from exc
        return {"status": "ready", "dependencies": dependencies}

    return router
```

```python
# packages/backend/src/student_affairs/api/app.py
from fastapi import FastAPI

from student_affairs.api.health import ReadinessProbe, build_health_router
from student_affairs.settings import Settings


def create_app(settings: Settings, readiness_probe: ReadinessProbe) -> FastAPI:
    app = FastAPI(title="Student Affairs Agent API", version="0.1.0")
    app.state.settings = settings
    app.include_router(build_health_router(readiness_probe))
    return app
```

`apps/api/__main__.py` must create the production composition root inside `main()`, then call `uvicorn.run`; imports must not open sockets or database connections.

- [ ] **Step 4: Verify app tests and static checks**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/api/test_app.py -v
D:\conda_envs\student-affairs-agent\python.exe -m ruff check packages/backend/src tests/unit/api
D:\conda_envs\student-affairs-agent\python.exe -m mypy packages/backend/src
```

Expected: both health tests pass; no lint or type errors.

- [ ] **Step 5: Commit**

```powershell
git add apps/api packages/backend/src/student_affairs/api packages/backend/src/student_affairs/settings.py tests/unit/api
git commit -m "feat: add typed API application factory"
```

### Task 3: ServiceCase Aggregate and State Invariants

**Files:**
- Create: `packages/backend/src/student_affairs/domain/errors.py`
- Create: `packages/backend/src/student_affairs/domain/case.py`
- Test: `tests/unit/domain/test_service_case.py`

**Interfaces:**
- Consumes: only Python standard library.
- Produces: `ServiceCase.create_draft(...)`, `ServiceCase.submit(...)`, `CaseEvent`, `CaseStatus`, `CaseStage`, `WaitingOn`, and stable domain errors.

- [ ] **Step 1: Write failing aggregate tests**

```python
# tests/unit/domain/test_service_case.py
from datetime import UTC, datetime
from uuid import UUID

import pytest

from student_affairs.domain.case import CaseStatus, ServiceCase, WaitingOn
from student_affairs.domain.errors import InvalidCaseTransition, VersionConflict

NOW = datetime(2026, 7, 16, 8, 0, tzinfo=UTC)


def test_create_draft_emits_first_event() -> None:
    case, event = ServiceCase.create_draft(
        student_id="student-001",
        service_definition_code="temporary-hardship-grant",
        service_definition_version=1,
        actor_id="student-001",
        now=NOW,
    )
    assert isinstance(case.id, UUID)
    assert case.status is CaseStatus.DRAFT
    assert case.version == 1
    assert event.case_id == case.id
    assert event.sequence == 1
    assert event.event_type == "case.created"


def test_submit_requires_matching_version() -> None:
    case, _ = ServiceCase.create_draft(
        "student-001", "temporary-hardship-grant", 1, "student-001", NOW
    )
    with pytest.raises(VersionConflict):
        case.submit(expected_version=0, actor_id="student-001", now=NOW)


def test_submit_changes_state_once_and_emits_event() -> None:
    case, _ = ServiceCase.create_draft(
        "student-001", "temporary-hardship-grant", 1, "student-001", NOW
    )
    event = case.submit(expected_version=1, actor_id="student-001", now=NOW)
    assert case.status is CaseStatus.SUBMITTED
    assert case.waiting_on is WaitingOn.SYSTEM
    assert case.version == 2
    assert event.sequence == 2
    with pytest.raises(InvalidCaseTransition):
        case.submit(expected_version=2, actor_id="student-001", now=NOW)
```

- [ ] **Step 2: Run the tests and verify they fail**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/domain/test_service_case.py -v
```

Expected: FAIL because the domain module does not exist.

- [ ] **Step 3: Implement explicit domain types and transitions**

```python
# packages/backend/src/student_affairs/domain/errors.py
class DomainError(Exception):
    """Base class for deterministic business-rule failures."""


class InvalidCaseTransition(DomainError):
    pass


class VersionConflict(DomainError):
    pass
```

```python
# packages/backend/src/student_affairs/domain/case.py
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime
from enum import StrEnum
from typing import Any
from uuid import UUID, uuid4

from student_affairs.domain.errors import InvalidCaseTransition, VersionConflict


class CaseStatus(StrEnum):
    DRAFT = "DRAFT"
    SUBMITTED = "SUBMITTED"
    ACTIVE = "ACTIVE"
    RESOLVED = "RESOLVED"
    CLOSED = "CLOSED"
    CANCELLED = "CANCELLED"


class CaseStage(StrEnum):
    INTAKE = "INTAKE"
    PRECHECK = "PRECHECK"
    TRIAGE = "TRIAGE"
    HANDLING = "HANDLING"
    APPROVAL = "APPROVAL"
    FOLLOW_UP = "FOLLOW_UP"


class WaitingOn(StrEnum):
    NONE = "NONE"
    STUDENT = "STUDENT"
    STAFF = "STAFF"
    SYSTEM = "SYSTEM"
    EXTERNAL_SYSTEM = "EXTERNAL_SYSTEM"


@dataclass(frozen=True, slots=True)
class CaseEvent:
    id: UUID
    case_id: UUID
    sequence: int
    event_type: str
    actor_id: str
    payload: dict[str, Any]
    occurred_at: datetime


@dataclass(slots=True)
class ServiceCase:
    id: UUID
    student_id: str
    service_definition_code: str
    service_definition_version: int
    status: CaseStatus
    current_stage: CaseStage
    waiting_on: WaitingOn
    resolution_code: str | None
    version: int
    created_at: datetime
    updated_at: datetime

    @classmethod
    def create_draft(
        cls,
        student_id: str,
        service_definition_code: str,
        service_definition_version: int,
        actor_id: str,
        now: datetime,
    ) -> tuple[ServiceCase, CaseEvent]:
        case = cls(
            id=uuid4(),
            student_id=student_id,
            service_definition_code=service_definition_code,
            service_definition_version=service_definition_version,
            status=CaseStatus.DRAFT,
            current_stage=CaseStage.INTAKE,
            waiting_on=WaitingOn.STUDENT,
            resolution_code=None,
            version=1,
            created_at=now,
            updated_at=now,
        )
        return case, case._event("case.created", actor_id, {}, now)

    def submit(self, expected_version: int, actor_id: str, now: datetime) -> CaseEvent:
        if expected_version != self.version:
            raise VersionConflict(f"expected {expected_version}, actual {self.version}")
        if self.status is not CaseStatus.DRAFT:
            raise InvalidCaseTransition(f"cannot submit case in {self.status}")
        self.status = CaseStatus.SUBMITTED
        self.current_stage = CaseStage.PRECHECK
        self.waiting_on = WaitingOn.SYSTEM
        self.version += 1
        self.updated_at = now
        return self._event("case.submitted", actor_id, {}, now)

    def _event(
        self, event_type: str, actor_id: str, payload: dict[str, Any], now: datetime
    ) -> CaseEvent:
        return CaseEvent(
            id=uuid4(),
            case_id=self.id,
            sequence=self.version,
            event_type=event_type,
            actor_id=actor_id,
            payload=payload,
            occurred_at=now,
        )
```

Do not add `activate`, `resolve`, `close`, or `cancel` until a later journey requires and tests those transitions.

- [ ] **Step 4: Run domain tests and coverage**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/domain/test_service_case.py --cov=student_affairs.domain --cov-report=term-missing -v
```

Expected: all tests pass and every branch in `submit` is covered.

- [ ] **Step 5: Commit**

```powershell
git add packages/backend/src/student_affairs/domain tests/unit/domain
git commit -m "feat: model service case lifecycle"
```

### Task 4: Authorization, Commands, Ports, and Application Handlers

**Files:**
- Create: `packages/backend/src/student_affairs/identity/context.py`
- Create: `packages/backend/src/student_affairs/application/errors.py`
- Create: `packages/backend/src/student_affairs/application/authorization.py`
- Create: `packages/backend/src/student_affairs/application/case_schemas.py`
- Create: `packages/backend/src/student_affairs/application/case_ports.py`
- Create: `packages/backend/src/student_affairs/application/case_handlers.py`
- Test: `tests/unit/application/test_case_handlers.py`

**Interfaces:**
- Consumes: `ServiceCase` aggregate and `CaseEvent`.
- Produces: `CaseService.create_case`, `CaseService.submit_case`, `CaseService.get_case`; `CaseUnitOfWork` protocol; trusted `IdentityContext`.

- [ ] **Step 1: Write failing handler tests with in-memory ports**

The test module must define `FakeCaseUnitOfWork` that snapshots dictionaries on entry and restores them on rollback. Cover these behaviors:

```python
async def test_student_creates_case_for_self_and_replay_is_idempotent() -> None:
    identity = IdentityContext("student-001", frozenset({"student"}), frozenset())
    command = CreateCaseCommand(
        request_id="req-001",
        idempotency_key="create-001",
        service_definition_code="temporary-hardship-grant",
        service_definition_version=1,
    )
    first = await service.create_case(identity, command)
    second = await service.create_case(identity, command)
    assert first == second
    assert len(uow.cases) == 1
    assert [event.event_type for event in uow.events] == ["case.created"]
    assert [event.topic for event in uow.outbox] == ["case.created"]


async def test_student_cannot_read_another_students_case() -> None:
    with pytest.raises(ResourceNotFound):
        await service.get_case(student_b, case_owned_by_student_a)


async def test_submit_uses_expected_version_and_records_outbox() -> None:
    result = await service.submit_case(
        student_a,
        SubmitCaseCommand("req-002", "submit-001", case_id, expected_version=1),
    )
    assert result.status == "SUBMITTED"
    assert result.version == 2
    assert uow.outbox[-1].topic == "case.submitted"
```

Also assert a caller without role `student` receives `Forbidden`, and reusing an idempotency key with a different command fingerprint receives `IdempotencyConflict`.

- [ ] **Step 2: Run tests and verify missing interfaces**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/application/test_case_handlers.py -v
```

Expected: FAIL during import because application interfaces do not exist.

- [ ] **Step 3: Define stable identity, command, result, and port contracts**

```python
# identity/context.py
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class IdentityContext:
    subject: str
    roles: frozenset[str]
    department_ids: frozenset[str]
```

```python
# application/case_schemas.py
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID


@dataclass(frozen=True, slots=True)
class CreateCaseCommand:
    request_id: str
    idempotency_key: str
    service_definition_code: str
    service_definition_version: int


@dataclass(frozen=True, slots=True)
class SubmitCaseCommand:
    request_id: str
    idempotency_key: str
    case_id: UUID
    expected_version: int


@dataclass(frozen=True, slots=True)
class CaseResult:
    id: UUID
    student_id: str
    service_definition_code: str
    service_definition_version: int
    status: str
    current_stage: str
    waiting_on: str
    resolution_code: str | None
    version: int


@dataclass(frozen=True, slots=True)
class PendingOutboxEvent:
    id: UUID
    topic: str
    aggregate_id: UUID
    aggregate_version: int
    payload: dict[str, object]
    occurred_at: datetime


@dataclass(frozen=True, slots=True)
class StoredIdempotencyResult:
    subject: str
    key: str
    command_fingerprint: str
    response: dict[str, object]
    created_at: datetime
    expires_at: datetime
```

`case_ports.py` must expose these exact async protocols:

```python
class CaseRepository(Protocol):
    async def add(self, case: ServiceCase) -> None: ...
    async def save(self, case: ServiceCase) -> None: ...
    async def get_for_update(self, case_id: UUID) -> ServiceCase | None: ...
    async def get_visible(self, case_id: UUID, identity: IdentityContext) -> ServiceCase | None: ...


class CaseUnitOfWork(Protocol):
    cases: CaseRepository
    events: CaseEventRepository
    outbox: OutboxRepository
    idempotency: IdempotencyRepository

    async def __aenter__(self) -> "CaseUnitOfWork": ...
    async def __aexit__(self, exc_type: object, exc: object, tb: object) -> None: ...
    async def commit(self) -> None: ...
```

The remaining repository protocols use these methods:

- `CaseEventRepository.add(event: CaseEvent) -> None`
- `OutboxRepository.add(event: PendingOutboxEvent) -> None`
- `IdempotencyRepository.get(subject, key) -> StoredIdempotencyResult | None`
- `IdempotencyRepository.add(record) -> None`

`PendingOutboxEvent` contains `id`, `topic`, `aggregate_id`, `aggregate_version`, `payload`, `occurred_at`; `StoredIdempotencyResult` contains `subject`, `key`, `command_fingerprint`, and serialized `CaseResult`.

- [ ] **Step 4: Implement deterministic authorization and `CaseService`**

`AuthorizationPolicy` must implement:

```python
def require_student(identity: IdentityContext) -> None:
    if "student" not in identity.roles:
        raise Forbidden("student role required")


def require_case_owner(identity: IdentityContext, case: ServiceCase) -> None:
    require_student(identity)
    if identity.subject != case.student_id:
        raise ResourceNotFound("case not found")
```

`CaseService` receives `uow_factory: Callable[[], CaseUnitOfWork]` and `clock: Callable[[], datetime]`. For each command it must:

1. Require the student role.
2. Compute SHA-256 over canonical JSON containing command type and immutable command fields.
3. Load `(identity.subject, idempotency_key)` inside the transaction.
4. Return the stored result when fingerprints match; raise `IdempotencyConflict` otherwise.
5. Create/load and authorize the aggregate.
6. Perform the domain transition with `expected_version`.
7. Persist aggregate, `CaseEvent`, `PendingOutboxEvent`, and idempotency result.
8. Commit once, then return `CaseResult`.

Map `VersionConflict` to application `ConcurrencyConflict`; do not retry a stale user command automatically.

- [ ] **Step 5: Run application tests and architecture import checks**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/application/test_case_handlers.py -v
D:\conda_envs\student-affairs-agent\python.exe -m ruff check packages/backend/src/student_affairs tests/unit/application
D:\conda_envs\student-affairs-agent\python.exe -m mypy packages/backend/src
```

Expected: all authorization, idempotency, event, Outbox, and concurrency tests pass.

- [ ] **Step 6: Commit**

```powershell
git add packages/backend/src/student_affairs/application packages/backend/src/student_affairs/identity tests/unit/application
git commit -m "feat: add authorized case command handlers"
```

### Task 5: PostgreSQL Schema, Repository, and Transactional Unit of Work

**Files:**
- Create: `alembic.ini`
- Create: `migrations/env.py`
- Create: `migrations/versions/0001_service_case_foundation.py`
- Create: `packages/backend/src/student_affairs/infrastructure/postgres/models.py`
- Create: `packages/backend/src/student_affairs/infrastructure/postgres/session.py`
- Create: `packages/backend/src/student_affairs/infrastructure/postgres/repositories.py`
- Create: `packages/backend/src/student_affairs/infrastructure/postgres/uow.py`
- Test: `tests/integration/postgres/test_case_uow.py`

**Interfaces:**
- Consumes: all application repository/UoW protocols from Task 4.
- Produces: `create_engine_and_sessionmaker(database_url)`, `SqlAlchemyCaseUnitOfWork`, and migration revision `0001`.

- [ ] **Step 1: Write failing PostgreSQL integration tests**

Use `testcontainers.postgres.PostgresContainer("postgres:16.9-bookworm")` in a session-scoped fixture, convert its URL to `postgresql+psycopg://`, run `alembic upgrade head`, and test:

```python
async def test_transaction_persists_case_event_outbox_and_idempotency_atomically() -> None:
    result = await service.create_case(student, create_command)
    async with session_factory() as session:
        assert await session.scalar(select(func.count(CaseRow.id))) == 1
        assert await session.scalar(select(func.count(CaseEventRow.id))) == 1
        assert await session.scalar(select(func.count(OutboxRow.id))) == 1
        assert await session.scalar(select(func.count(IdempotencyRow.id))) == 1
    assert result.version == 1


async def test_rollback_leaves_no_partial_business_or_outbox_rows() -> None:
    failing_uow = SqlAlchemyCaseUnitOfWork(session_factory, fail_before_commit=True)
    with pytest.raises(RuntimeError, match="injected commit failure"):
        await service_using(failing_uow).create_case(student, create_command)
    await assert_all_foundation_tables_empty(session_factory)


async def test_concurrent_submit_allows_one_winner() -> None:
    outcomes = await asyncio.gather(submit_once(), submit_once(), return_exceptions=True)
    assert sum(isinstance(item, CaseResult) for item in outcomes) == 1
    assert sum(isinstance(item, ConcurrencyConflict) for item in outcomes) == 1
```

- [ ] **Step 2: Run the integration test and verify migration/import failure**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/integration/postgres/test_case_uow.py -v
```

Expected: FAIL because migration and PostgreSQL adapters do not exist. Docker Desktop must be running; otherwise record the environment blocker and do not mark the task complete.

- [ ] **Step 3: Create the exact foundation schema**

Migration `0001` creates:

```sql
CREATE TABLE service_cases (
  id UUID PRIMARY KEY,
  student_id VARCHAR(128) NOT NULL,
  service_definition_code VARCHAR(128) NOT NULL,
  service_definition_version INTEGER NOT NULL CHECK (service_definition_version > 0),
  status VARCHAR(32) NOT NULL,
  current_stage VARCHAR(32) NOT NULL,
  waiting_on VARCHAR(32) NOT NULL,
  resolution_code VARCHAR(64),
  version INTEGER NOT NULL CHECK (version > 0),
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX ix_service_cases_student_updated
  ON service_cases (student_id, updated_at DESC);

CREATE TABLE case_events (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES service_cases(id),
  sequence INTEGER NOT NULL,
  event_type VARCHAR(128) NOT NULL,
  actor_id VARCHAR(128) NOT NULL,
  payload JSONB NOT NULL,
  occurred_at TIMESTAMPTZ NOT NULL,
  UNIQUE (case_id, sequence)
);

CREATE TABLE outbox_events (
  id UUID PRIMARY KEY,
  topic VARCHAR(128) NOT NULL,
  aggregate_id UUID NOT NULL,
  aggregate_version INTEGER NOT NULL,
  payload JSONB NOT NULL,
  occurred_at TIMESTAMPTZ NOT NULL,
  published_at TIMESTAMPTZ,
  attempts INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  UNIQUE (topic, aggregate_id, aggregate_version)
);
CREATE INDEX ix_outbox_unpublished
  ON outbox_events (occurred_at) WHERE published_at IS NULL;

CREATE TABLE idempotency_records (
  subject VARCHAR(128) NOT NULL,
  idempotency_key VARCHAR(128) NOT NULL,
  command_fingerprint CHAR(64) NOT NULL,
  response JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (subject, idempotency_key)
);
```

The Alembic downgrade removes indexes/tables in reverse dependency order. SQLAlchemy models must mirror every nullability, uniqueness, index, and length constraint.

- [ ] **Step 4: Implement mapping repositories and optimistic updates**

`SqlAlchemyCaseRepository.get_for_update` loads by ID. `save` performs an explicit compare-and-swap update:

```python
statement = (
    update(CaseRow)
    .where(CaseRow.id == case.id, CaseRow.version == case.version - 1)
    .values(
        status=case.status.value,
        current_stage=case.current_stage.value,
        waiting_on=case.waiting_on.value,
        resolution_code=case.resolution_code,
        version=case.version,
        updated_at=case.updated_at,
    )
)
result = await self._session.execute(statement)
if result.rowcount != 1:
    raise ConcurrencyConflict("case version changed")
```

`get_visible` must include `WHERE id = :case_id AND student_id = :identity.subject` for student identities. It must not fetch first and authorize later. Repository mappers convert enum strings and JSON into domain values without exposing ORM rows to application code.

`SqlAlchemyCaseUnitOfWork.__aenter__` creates one `AsyncSession` and repositories; `commit` flushes and commits; `__aexit__` rolls back on exceptions and always closes the session.

- [ ] **Step 5: Run migration, integration tests, and schema drift check**

```powershell
D:\conda_envs\student-affairs-agent\Scripts\alembic.exe upgrade head
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/integration/postgres/test_case_uow.py -v
D:\conda_envs\student-affairs-agent\Scripts\alembic.exe check
```

Expected: migration succeeds, all atomicity/concurrency tests pass, and Alembic reports no new upgrade operations.

- [ ] **Step 6: Commit**

```powershell
git add alembic.ini migrations packages/backend/src/student_affairs/infrastructure tests/integration/postgres
git commit -m "feat: persist service cases transactionally"
```

### Task 6: OIDC Verification and Keycloak Development Realm

**Files:**
- Create: `packages/backend/src/student_affairs/identity/oidc.py`
- Create: `packages/backend/src/student_affairs/api/dependencies.py`
- Create: `deploy/compose/keycloak/realm-student-affairs.json`
- Create: `deploy/compose/postgres/init/001-create-app-role.sql`
- Create: `compose.yml`
- Test: `tests/unit/identity/test_oidc.py`

**Interfaces:**
- Consumes: `Settings` and `IdentityContext`.
- Produces: `OidcTokenVerifier.verify(token) -> IdentityContext` and FastAPI `current_identity` dependency.

- [ ] **Step 1: Write failing JWT verification tests**

Generate an RSA key pair once per test module. Serve a static JWKS through `httpx.MockTransport`, sign tokens with `kid=test-key`, and cover:

```python
def test_valid_student_token_creates_trusted_identity() -> None:
    identity = verifier.verify(
        token(sub="student-001", aud="student-affairs-api", roles=["student"])
    )
    assert identity == IdentityContext(
        subject="student-001",
        roles=frozenset({"student"}),
        department_ids=frozenset(),
    )


@pytest.mark.parametrize("mutation", ["expired", "wrong_issuer", "wrong_audience", "unknown_kid"])
def test_invalid_security_claim_is_rejected(mutation: str) -> None:
    with pytest.raises(InvalidIdentityToken):
        verifier.verify(mutated_token(mutation))
```

Add a staff token test where departments come only from claim `departments`; never infer department access from a request parameter.

- [ ] **Step 2: Run tests and verify verifier is missing**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/identity/test_oidc.py -v
```

Expected: FAIL because `OidcTokenVerifier` does not exist.

- [ ] **Step 3: Implement strict JWT validation with bounded JWKS caching**

`OidcTokenVerifier` constructor accepts issuer, audience, JWKS URL, `httpx.Client`, clock, and a 300-second cache TTL. `verify` must:

1. Reject an empty token.
2. Read unverified header only to select `kid`.
3. Refresh JWKS when cache expires or `kid` is absent; use a 3-second connect and 5-second total timeout.
4. Accept only `RS256`.
5. Validate signature, `exp`, `nbf`, exact issuer, and configured audience.
6. Require non-empty `sub`.
7. Normalize realm/client roles and department claims into immutable sets.
8. Raise one `InvalidIdentityToken` without exposing token content or crypto details.

The FastAPI dependency extracts `Authorization: Bearer <token>`, calls the verifier, and maps missing/invalid credentials to `401` with `WWW-Authenticate: Bearer`.

- [ ] **Step 4: Add local Compose services and importable Keycloak realm**

`compose.yml` defines two Profile `foundation` services:

- `postgres:16.9-bookworm`, port `15432:5432`, health check `pg_isready`, bind data `${DOCKER_DATA_ROOT}/postgres:/var/lib/postgresql/data`.
- `quay.io/keycloak/keycloak:26.2.5`, port `18080:8080`, `start-dev --import-realm`, depends on healthy PostgreSQL, bind-mounts the realm JSON read-only, and stores its tables in a separate `keycloak` database/schema.

Secrets are mandatory Compose substitutions such as `${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}`; they are not defaulted in YAML. The realm import contains:

- Realm `student-affairs`.
- API audience `student-affairs-api`.
- Public PKCE clients `student-web` and `staff-web` with localhost redirect URIs only.
- Realm roles `student`, `staff`, `system_admin`.
- Development users `student-001` and `staff-aid-001` only when passwords are injected by a documented post-start bootstrap command; no password is committed in realm JSON.

Before starting Compose, verify `D:\Programs\DockerData\student-affairs-agent` is the resolved data root and create only its `postgres` and `keycloak` subdirectories.

- [ ] **Step 5: Run identity tests and service health checks**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/identity/test_oidc.py -v
docker compose --profile foundation config
docker compose --profile foundation up -d postgres keycloak
docker compose ps
```

Expected: identity tests pass; rendered Compose contains no blank mandatory secret; PostgreSQL and Keycloak become healthy/running. Do not mark complete if Keycloak only starts but realm discovery fails.

- [ ] **Step 6: Commit**

```powershell
git add compose.yml deploy/compose packages/backend/src/student_affairs/identity packages/backend/src/student_affairs/api/dependencies.py tests/unit/identity
git commit -m "feat: enforce oidc identity boundary"
```

### Task 7: Versioned Case API and Stable Error Contract

**Files:**
- Create: `packages/backend/src/student_affairs/api/errors.py`
- Create: `packages/backend/src/student_affairs/api/v1/cases.py`
- Modify: `packages/backend/src/student_affairs/api/app.py`
- Modify: `packages/backend/src/student_affairs/api/dependencies.py`
- Test: `tests/system/test_case_api.py`

**Interfaces:**
- Consumes: OIDC identity dependency, `CaseService`, and PostgreSQL UoW.
- Produces: `POST /api/v1/cases`, `POST /api/v1/cases/{case_id}/submit`, and `GET /api/v1/cases/{case_id}`.

- [ ] **Step 1: Write failing HTTP contract tests**

Run the FastAPI app against the Testcontainers PostgreSQL fixture and override only the cryptographic verifier with one that returns prebuilt trusted identities. Cover:

```python
def test_create_submit_and_get_case() -> None:
    created = client.post(
        "/api/v1/cases",
        headers={"Authorization": "Bearer student-a", "Idempotency-Key": "create-001"},
        json={
            "service_definition_code": "temporary-hardship-grant",
            "service_definition_version": 1,
        },
    )
    assert created.status_code == 201
    case_id = created.json()["id"]

    submitted = client.post(
        f"/api/v1/cases/{case_id}/submit",
        headers={"Authorization": "Bearer student-a", "Idempotency-Key": "submit-001"},
        json={"expected_version": 1},
    )
    assert submitted.status_code == 200
    assert submitted.json()["status"] == "SUBMITTED"

    loaded = client.get(
        f"/api/v1/cases/{case_id}", headers={"Authorization": "Bearer student-a"}
    )
    assert loaded.status_code == 200
    assert loaded.json()["version"] == 2
```

Additional assertions:

- Missing token: `401` and stable code `invalid_identity`.
- Missing/blank `Idempotency-Key`: `400` and `idempotency_key_required`.
- Student B reads Student A case: `404` and `case_not_found`.
- Stale submit: `409` and `case_version_conflict`.
- Same key with different payload: `409` and `idempotency_conflict`.
- Validation failure: `422`; no database rows created.
- Response never includes `actor_id`, Outbox payload, internal exception, or stack trace.

- [ ] **Step 2: Run system tests and verify routes are absent**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/system/test_case_api.py -v
```

Expected: FAIL with `404` for case routes.

- [ ] **Step 3: Implement Pydantic request/response contracts and endpoints**

Use these API models:

```python
class CreateCaseRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")
    service_definition_code: str = Field(min_length=1, max_length=128)
    service_definition_version: int = Field(gt=0)


class SubmitCaseRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")
    expected_version: int = Field(gt=0)


class CaseResponse(BaseModel):
    id: UUID
    student_id: str
    service_definition_code: str
    service_definition_version: int
    status: str
    current_stage: str
    waiting_on: str
    resolution_code: str | None
    version: int
```

Each endpoint creates a request ID from trusted middleware state or a new UUID, reads identity only from `current_identity`, delegates to `CaseService`, and returns the mapped response. Do not place business rules in route functions.

- [ ] **Step 4: Register stable exception handlers**

Return this envelope for application and authentication failures:

```json
{
  "error": {
    "code": "case_version_conflict",
    "message": "The case changed. Refresh and try again.",
    "request_id": "..."
  }
}
```

Map `InvalidIdentityToken -> 401`, `Forbidden -> 403`, `ResourceNotFound -> 404`, `ConcurrencyConflict/IdempotencyConflict -> 409`, and unexpected exceptions to `500 internal_error`. Log exception details with request ID server-side; never return them to the caller.

- [ ] **Step 5: Run complete API and persistence verification**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/system/test_case_api.py tests/integration/postgres/test_case_uow.py -v
D:\conda_envs\student-affairs-agent\python.exe -m ruff check .
D:\conda_envs\student-affairs-agent\python.exe -m mypy packages/backend/src
```

Expected: all HTTP, authorization, persistence, idempotency, and concurrency assertions pass.

- [ ] **Step 6: Commit**

```powershell
git add packages/backend/src/student_affairs/api tests/system
git commit -m "feat: expose secure service case API"
```

### Task 8: Request Correlation, Audit-Safe Logging, and Readiness

**Files:**
- Create: `packages/backend/src/student_affairs/api/middleware.py`
- Create: `packages/backend/src/student_affairs/observability/logging.py`
- Create: `packages/backend/src/student_affairs/infrastructure/postgres/health.py`
- Modify: `packages/backend/src/student_affairs/api/app.py`
- Test: `tests/unit/api/test_observability.py`
- Test: `tests/integration/postgres/test_readiness.py`

**Interfaces:**
- Consumes: FastAPI app, SQLAlchemy engine, and trusted identity.
- Produces: validated `X-Request-ID`, structured logs, response correlation headers, and actual PostgreSQL readiness.

- [ ] **Step 1: Write failing correlation and redaction tests**

```python
def test_request_id_is_returned_and_invalid_value_is_replaced() -> None:
    response = client.get("/health/live", headers={"X-Request-ID": "bad value\nforged"})
    assert response.status_code == 200
    assert UUID(response.headers["X-Request-ID"])


def test_auth_header_and_token_never_appear_in_structured_logs(caplog) -> None:
    client.get("/api/v1/cases/missing", headers={"Authorization": "Bearer secret-token"})
    assert "secret-token" not in caplog.text
    assert "Authorization" not in caplog.text
```

The readiness integration test stops or points to an unavailable PostgreSQL endpoint and asserts `/health/ready` returns `503`, then restores the database and expects `200`.

- [ ] **Step 2: Run tests and verify missing middleware/probe**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/api/test_observability.py tests/integration/postgres/test_readiness.py -v
```

Expected: FAIL because correlation middleware and database readiness do not exist.

- [ ] **Step 3: Implement bounded request correlation and structured logging**

Accept caller `X-Request-ID` only when it matches `^[A-Za-z0-9._:-]{1,128}$`; otherwise generate `uuid4()`. Bind `request_id`, route, method, status, duration_ms, and authenticated subject hash to logging context. Never log headers, request bodies, raw student identifiers, tokens, or materials by default.

Add the same request ID to every response, including mapped errors. Configure JSON logs for staging/production and readable structured console logs for local development.

- [ ] **Step 4: Implement real readiness without leaking connection details**

`PostgresReadinessProbe.check` runs `SELECT 1` with a two-second timeout and returns `{"postgres": "up"}`. Timeout/authentication/network errors are logged with request-independent service context, then converted by the health router to `503 service_not_ready`.

- [ ] **Step 5: Verify observability behavior**

```powershell
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests/unit/api/test_observability.py tests/integration/postgres/test_readiness.py -v
D:\conda_envs\student-affairs-agent\python.exe -m pytest tests -v
```

Expected: request IDs are valid and stable per response, secrets are absent from captured logs, readiness follows PostgreSQL state, and the complete suite passes.

- [ ] **Step 6: Commit**

```powershell
git add packages/backend/src/student_affairs/api packages/backend/src/student_affairs/observability packages/backend/src/student_affairs/infrastructure/postgres/health.py tests
git commit -m "feat: add safe request observability"
```

### Task 9: Operational Documentation and Milestone Acceptance

**Files:**
- Modify: `README.md`
- Modify: `docs/superpowers/specs/2026-07-16-student-affairs-platform-design.md`
- Create: `docs/adr/0001-modular-monolith-and-service-case.md`
- Create: `docs/operations/local-development.md`
- Create: `docs/security/foundation-threat-model.md`
- Create: `scripts/verify-foundation.ps1`

**Interfaces:**
- Consumes: all M1 runtime, test, Compose, migration, and identity commands.
- Produces: one reproducible verification entry point and reviewed architecture/operations records.

- [ ] **Step 1: Write the acceptance script before the documentation claims success**

`scripts/verify-foundation.ps1` must stop on first error and run:

```powershell
$ErrorActionPreference = 'Stop'
$python = 'D:\conda_envs\student-affairs-agent\python.exe'

& $python -m ruff check .
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& $python -m mypy packages/backend/src
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& $python -m pytest tests/unit tests/integration tests/system -v
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& docker compose --profile foundation config --quiet
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& git diff --check
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
```

Add a final tracked-file secret scan using `git grep` patterns for private keys, common API-key forms, bearer tokens, and non-empty password assignments. The script must fail when a fixture intentionally containing `-----BEGIN PRIVATE KEY-----` is staged.

- [ ] **Step 2: Run the acceptance script and record real blockers**

```powershell
powershell -ExecutionPolicy Bypass -File scripts/verify-foundation.ps1
```

Expected: all checks exit `0`. If Docker or Keycloak is unavailable, the milestone remains incomplete; document the exact blocker rather than weakening the script.

- [ ] **Step 3: Write decisions and operator instructions**

`ADR-0001` records context, decision, alternatives, consequences, and revisit triggers for:

- modular monolith plus independent workers;
- `ServiceCase` as source of truth;
- PostgreSQL for business facts, Outbox, audit, and future Checkpoint;
- OIDC identity boundary;
- no Neo4j in the initial architecture.

`local-development.md` contains exact PowerShell commands to:

1. Create/update the Conda prefix.
2. Create a local `.env` from `.env.example` and set secrets without committing it.
3. Validate and start the `foundation` Compose Profile.
4. Apply/rollback migrations.
5. Bootstrap the two local Keycloak users.
6. Start/stop the API in the foreground so logs remain visible.
7. Run focused and full tests.
8. Back up and restore local PostgreSQL data without deleting volumes.

The threat model names assets, trust boundaries, attacker capabilities, and mitigations for token theft, broken object authorization, ID enumeration, replay/idempotency abuse, SQL injection, log leakage, malicious payloads, and compromised local secrets.

- [ ] **Step 4: Update status and README only after evidence passes**

Change the design spec status to `已批准，M1 实施中` when Task 1 begins, and to `已批准，M1 已验收` only after this task passes. README must report actual capability and commands; it must not claim Agent, RAG, student UI, staff UI, or production SLA are implemented in M1.

- [ ] **Step 5: Final milestone verification**

```powershell
powershell -ExecutionPolicy Bypass -File scripts/verify-foundation.ps1
git status --short
git log --oneline --decorate -10
```

Expected: verification exits `0`; worktree contains only intentional documentation updates before commit; commit history has one focused commit per task.

- [ ] **Step 6: Commit**

```powershell
git add README.md docs scripts/verify-foundation.ps1
git commit -m "docs: complete foundation operations guide"
```

## M1 Completion Evidence

M1 is complete only when all evidence exists:

- `scripts/verify-foundation.ps1` exits `0` on the target Windows machine.
- API obtains identity from a cryptographically verified OIDC token.
- Create, replay, submit, stale-submit, owner-read, and cross-student denial tests pass.
- PostgreSQL contains exactly one aggregate/event/Outbox/idempotency set for a replayed command.
- Service and database restart tests preserve facts and behavior.
- Keycloak discovery and JWKS endpoints are reachable from the API process.
- Alembic upgrade, check, and tested downgrade execute successfully.
- Structured logs correlate requests without exposing credentials or request bodies.
- No tracked file contains a real secret or student record.

Do not start M2 until these checks pass or an explicit ADR documents why a requirement changed.
