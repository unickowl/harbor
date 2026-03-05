# Harbor Project Context

## Purpose

Harbor is a multi-project ecosystem for managing cryptocurrency payment operations. It consists of three independent projects unified under a single development environment:

- **Portal** - User-facing platform for customers to initiate transfers (deposits/withdrawals)
- **Admin** - Internal management platform for operators to monitor and manage operations
- **API** - Backend service providing business logic and data persistence

**Critical Architecture Principle**: These three projects are **completely independent** with separate git repositories, codebases, and development standards. They communicate exclusively through well-defined API contracts.

---

## Project Structure

```
harbor/                           # Unified development environment
├── .claude/                      # Shared Claude Code configuration
│   ├── agents/                   # Custom AI subagents for cross-project tasks
│   └── CLAUDE.md                 # Development guidelines
├── portal/                       # → Symbolic link to owlpay_harbor_portal
├── admin/                        # → Symbolic link to owlpay_harbor_admin
├── api/                          # → Symbolic link to owlpay_harbor_api
└── openspec/                     # Project specifications
```

**Important**: `portal/`, `admin/`, and `api/` are **symbolic links** to independent git repositories. Each maintains its own:
- Git history and branching strategy
- Dependencies and package versions
- Development workflows and CI/CD pipelines
- Coding standards and conventions

---

## Tech Stack

### Portal (owlpay_harbor_portal)

**Frontend Stack**:
- **Framework**: Vue 3.5 (Composition API with `<script setup>`)
- **Language**: TypeScript (strict mode)
- **Build Tool**: Vite
- **UI Library**: PrimeVue 4.3.x
- **State Management**: Pinia
- **Data Fetching**: TanStack Query (Vue Query)
- **HTTP Client**: Axios
- **Routing**: Vue Router 4
- **Styling**: Tailwind CSS (via `@apply` directives)
- **Testing**: Vitest

**Key Dependencies**:
- `@vueuse/core` - Composable utilities
- `zod` - Runtime type validation
- `dayjs` - Date manipulation
- `big.js` - Precise decimal calculations

**Node Version**: >= 22.0.0

---

### Admin (owlpay_harbor_admin)

**Frontend Stack**:
- **Framework**: Vue 3.5 (Composition API with `<script setup>`)
- **Language**: TypeScript (strict mode)
- **Build Tool**: Vite
- **UI Library**: PrimeVue 4.4.x (⚠️ Different version from Portal)
- **State Management**: Pinia
- **HTTP Client**: Axios
- **Routing**: Vue Router 4
- **Testing**: Vitest

**Note**: Admin uses similar stack to Portal but maintains **independent versions** and may have different implementation patterns.

---

### API (owlpay_harbor_api)

**Backend Stack**:
- **Language**: PHP 8.2+
- **Framework**: Laravel 12.0
- **Authentication**: Laravel Sanctum
- **Database**: MySQL/PostgreSQL
- **Cache**: Redis (via Predis)
- **Testing**: PHPUnit with Paratest

**Key Packages**:
- `spatie/laravel-permission` - Role and permission management
- `spatie/laravel-activitylog` - Activity logging
- `spatie/laravel-data` - Data transfer objects
- `spatie/laravel-pdf` - PDF generation
- `sentry/sentry-laravel` - Error tracking
- `maatwebsite/excel` - Excel operations
- Custom OwlTing packages (SSO, Mail templates)

---

## Project Conventions

### Portal Conventions

**Code Style**:
- **Function Spacing**: Always space between function name and brackets
  ```typescript
  // ✓ Correct
  function fn () { ... }
  fn () { ... }

  // ✗ Wrong
  function fn() { ... }
  fn() { ... }
  ```
- **Quotes**: Single quotes for strings (unless interpolation needed)
- **Styling**: Use Tailwind CSS via `@apply` in `<style scoped>`
- **Composition API**: Prefer `<script setup>` with TypeScript

**Naming Conventions**:
- **Components**: PascalCase (e.g., `DepositFlow.vue`)
- **Composables**: `use` prefix (e.g., `useCreateTransfer.ts`)
- **Types**: PascalCase interfaces (e.g., `CreateTransferRequest`)
- **Files**: kebab-case for utilities (e.g., `date-validator.ts`)

**Architecture Patterns**:
- **Data Fetching**: TanStack Query hooks in `src/composables/queries/`
- **State**: Pinia stores in `src/stores/`
- **Types**: Centralized in `src/types/`
- **Transformers**: API data transformation in `src/utils/transformers/`
- **Validators**: Pure validation functions in `src/utils/validators/`

**Testing Strategy**:
- Unit tests with Vitest
- Focus on utilities, transformers, and composables
- Test file naming: `*.spec.ts`

---

### Admin Conventions

**Code Style**:
- Similar to Portal but **may have variations**
- Refer to Admin's own codebase for specific patterns
- **Do not assume** Portal conventions apply to Admin

**Important**: Admin is an **independent project**. When working on Admin:
1. Check Admin's actual implementation patterns
2. Do not copy Portal's conventions blindly
3. Respect Admin's version choices (e.g., PrimeVue 4.4 vs Portal's 4.3)

---

### API Conventions

**Code Style**:
- Follow Laravel conventions and PSR-12 standards
- Use Laravel's built-in features (Eloquent, Service Container, etc.)

**Architecture Patterns**:
- **Controllers**: Handle HTTP requests (`app/Http/Controllers/`)
- **Resources**: API response transformers (`app/Http/Resources/`)
- **Models**: Eloquent ORM (`app/Models/`)
- **Services**: Business logic (if used)
- **Migrations**: Database schema changes (`database/migrations/`)

**API Response Format**:
- Use Laravel Resources for consistent response structure
- Follow RESTful conventions
- Versioned endpoints: `/api/v2/[resource]`

**Testing Strategy**:
- Feature tests for API endpoints
- Unit tests for business logic
- Test file location: `tests/Feature/`, `tests/Unit/`

---

## Cross-Project Integration

### API Contracts

Projects communicate **exclusively** through HTTP APIs with well-defined contracts.

**Contract Ownership**:
- **API** defines the source of truth (schema, validation, responses)
- **Portal** and **Admin** consume the API as clients

**Type Safety**:
- Frontend TypeScript types should **mirror** backend response structures
- Use transformers if naming conventions differ (e.g., snake_case → camelCase)
- Validate contracts with `contract-validator` subagent

**Version Management**:
- API uses versioned endpoints (`/api/v2/`)
- Breaking changes require new API versions
- Frontends can support multiple API versions during migration

---

### Data Flow

```
User Action (Portal/Admin)
  ↓
Frontend Component
  ↓
TanStack Query Composable
  ↓
Axios HTTP Request
  ↓
API Endpoint (Laravel Route)
  ↓
Controller → Service → Model
  ↓
Database
  ↓
Laravel Resource (Response Transformer)
  ↓
JSON Response
  ↓
Frontend Transformer (if needed)
  ↓
TypeScript Type
  ↓
Component Updates UI
```

---

## Domain Context

### Business Domain

Harbor manages **cryptocurrency payment operations**:

**Core Concepts**:
- **Transfers**: Financial transactions (deposits/withdrawals)
- **Quotes**: Exchange rate and fee calculations
- **Customers**: End users making transactions
- **Requirements**: Dynamic form fields based on transfer context
- **Travel Rule**: Compliance requirements for large transactions

**Transfer Types**:
- **Deposit (On-Ramp)**: Fiat → Crypto
- **Withdrawal (Off-Ramp)**: Crypto → Fiat

**Key Workflows**:
1. Customer selection
2. Path configuration (source → destination)
3. Amount and fee calculation
4. Quote generation
5. Requirements collection (KYC/Travel Rule)
6. Review and submission
7. Processing and settlement

---

## Important Constraints

### Architectural Constraints

1. **Project Independence**
   - Portal, Admin, and API are **separate codebases**
   - Each has independent git history, dependencies, and versions
   - **Do NOT** copy code between projects without justification
   - **Do NOT** assume conventions from one apply to another

2. **API as Source of Truth**
   - API defines data schemas and business rules
   - Frontends are **consumers**, not owners of data contracts
   - Schema changes must originate from API

3. **No Shared Code**
   - Currently no shared npm packages or composer packages
   - Duplication is acceptable to maintain independence
   - Future consideration: Create `@harbor/*` shared packages if duplication becomes problematic

### Technical Constraints

4. **Version Constraints**
   - Portal: Node >= 22.0.0
   - API: PHP >= 8.2
   - Each project may use different versions of shared libraries (e.g., PrimeVue)

5. **Build Independence**
   - Each project builds independently
   - No cross-project build dependencies
   - Deployment pipelines are separate

6. **Git Workflow**
   - Harbor directory is **NOT** a git repository
   - Changes must be committed in individual project repositories
   - Symbolic links are for development convenience only

### Security Constraints

7. **Authentication**
   - API uses Laravel Sanctum for authentication
   - Frontend stores auth tokens in cookies
   - No sensitive data in frontend code

8. **Data Validation**
   - Backend **always** validates all input
   - Frontend validation is for UX, not security
   - Never trust frontend data

---

## External Dependencies

### Portal Dependencies

**APIs Consumed**:
- Harbor API (`/api/v2/*`) - Primary backend
- Sentry - Error tracking

**Third-party Libraries**:
- PrimeVue - UI components
- TanStack Query - Server state management
- Axios - HTTP client
- Zod - Runtime validation

### Admin Dependencies

**APIs Consumed**:
- Harbor API (`/api/v2/*`) - Primary backend
- Sentry (if configured) - Error tracking

**Third-party Libraries**:
- Similar to Portal (but independent versions)

### API Dependencies

**External Services**:
- Database (MySQL/PostgreSQL)
- Redis - Caching and queues
- S3 (AWS) - File storage
- Sentry - Error tracking
- OwlTing SSO - Single sign-on
- OwlTing Mail Templates - Email templates

**Infrastructure**:
- Laravel framework
- Spatie packages ecosystem

---

## Development Workflow

### Working Across Projects

When working on features that span multiple projects:

1. **Plan API Changes First**
   - Design API contract (request/response schema)
   - Update API implementation
   - Deploy API changes

2. **Update Frontend Consumers**
   - Update TypeScript types
   - Update API composables/services
   - Update UI components
   - Test integration

3. **Validate Contracts**
   - Use `contract-validator` subagent
   - Ensure type consistency
   - Test backward compatibility if needed

### Using Custom Subagents

Harbor provides specialized AI subagents for cross-project tasks:

- **cross-project-explorer** - Trace features across all projects
- **api-impact-analyzer** - Assess API change impact on frontends
- **frontend-consistency-checker** - Compare Portal and Admin implementations
- **contract-validator** - Validate API contracts match frontend types

See `.claude/agents/README.md` for usage details.

---

## Git Workflow

### Per-Project Workflow

Each project maintains its own:

**Branching Strategy**:
- Main branch: `dev` (not `main`)
- Feature branches: `feat/TICKET-ID-description`
- Bugfix branches: `fix/TICKET-ID-description`

**Commit Conventions**:
- Portal: Check portal's own conventions
- Admin: Check admin's own conventions
- API: Check API's own conventions

**Pull Requests**:
- Create PRs in individual project repositories
- Each project has its own CI/CD pipeline
- Merge to `dev` branch (not `main`)

### Important Git Rules

- **NEVER** commit in Harbor directory (it's not a git repo)
- **ALWAYS** commit in individual project directories
- Symbolic links are for development convenience, not git management
- Changes to `.claude/` should be committed in relevant project

---

## Quality Standards

### Code Quality

- **Portal**: Follow Portal's ESLint rules and code style
- **Admin**: Follow Admin's ESLint rules and code style
- **API**: Follow Laravel conventions and PSR-12

### Testing Requirements

- **Unit tests** for utilities and business logic
- **Integration tests** for API endpoints
- **Type safety** enforced by TypeScript (Portal/Admin)
- **Runtime validation** where needed (Zod, backend validation)

### Performance

- API response times < 200ms for standard endpoints
- Frontend bundle size optimization (code splitting)
- Efficient database queries (avoid N+1)
- Redis caching for expensive operations

---

## OpenSpec Integration

This project uses **OpenSpec** for spec-driven development:

- **Proposals**: Document planned changes before implementation
- **Specs**: Living documentation of system behavior
- **Tasks**: Track implementation progress

See `openspec/AGENTS.md` for detailed workflow.

---

## Key Principles

1. **Independence First**: Respect project boundaries
2. **API as Contract**: All integration through well-defined APIs
3. **Type Safety**: Maintain type consistency across the stack
4. **Simplicity**: Follow YAGNI, KISS, DRY (in that order)
5. **No Assumptions**: Each project has its own conventions

---

## Getting Help

### Documentation Locations

- Portal docs: `portal/README.md`
- Admin docs: `admin/README.md`
- API docs: `api/README.md`
- Harbor overview: This file
- Subagents guide: `.claude/agents/README.md`

### When Working on Cross-Project Features

1. Use `cross-project-explorer` to understand current implementation
2. Use `api-impact-analyzer` before API changes
3. Use `contract-validator` after type changes
4. Use `frontend-consistency-checker` when refactoring

### When in Doubt

- **For API questions**: Check Laravel docs and API codebase
- **For Portal questions**: Check Portal's actual code and patterns
- **For Admin questions**: Check Admin's actual code and patterns
- **For cross-project questions**: Use custom subagents or ask for clarification

---

**Last Updated**: 2026-03-05
**Maintainer**: Harbor Development Team
