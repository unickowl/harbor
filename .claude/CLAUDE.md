# Harbor Development Rules

**Harbor Ecosystem**: Multi-project cryptocurrency payment platform (Portal, Admin, API)

## Critical Architecture Awareness

### Project Independence Principle

Harbor consists of **three independent projects** with separate repositories:
- **Portal** (`owlpay_harbor_portal`) - User-facing frontend
- **Admin** (`owlpay_harbor_admin`) - Internal management frontend
- **API** (`owlpay_harbor_api`) - Backend service

**NEVER assume conventions from one project apply to another.**

Each project maintains:
- Independent git repository and history
- Separate dependencies and versions
- Own coding standards and patterns
- Distinct development workflows

**Integration**: Projects communicate **exclusively** through well-defined API contracts.

---

## Core Development Principles

### 0. Fundamental Design Principles (Always Apply)

#### YAGNI (You Aren't Gonna Need It)
- **Avoid premature features**: Only implement what is actually required right now
- **Resist over-engineering**: Don't build for hypothetical future requirements
- **Question every addition**: Ask "Do we actually need this now?" before adding complexity
- **Focus on current use cases**: Solve today's problems, not tomorrow's maybes
- **Remove unused code**: Regularly clean up dead code and unused dependencies

#### KISS (Keep It Simple, Stupid)
- **Choose simplicity**: When multiple solutions exist, prefer the simpler one
- **Minimize cognitive load**: Write code that's easy to understand and maintain
- **Avoid clever code**: Prioritize clarity over showing off technical prowess
- **Simple architectures**: Use the simplest architecture that meets current needs
- **Clear naming**: Use descriptive, unambiguous names for variables, functions, and classes

#### DRY (Don't Repeat Yourself)
- **Extract common logic**: Identify and consolidate duplicate code patterns
- **Create reusable components**: Build shared utilities and libraries within each project
- **Single source of truth**: Ensure each piece of knowledge exists in only one place
- **Abstract appropriately**: Create abstractions that eliminate meaningful duplication
- **But avoid over-DRYing**: Don't abstract too early or create unnecessary coupling
- **Respect project boundaries**: Don't force code sharing between independent projects

#### Self-Improvement Loop
- After ANY correction from the user: update `harbor/.claude/lessons.md` to reflect the new understanding
- Write rules for yourself that prevent the same mistake
- Ruthlessly iterate on these lessons until mistake rate drops

### 1. Architecture Analysis

Before making changes:
- **Identify which project(s)** are affected
- **Read project-specific conventions** from each project's codebase
- **Understand API contracts** if working across project boundaries
- **Use custom subagents** for cross-project analysis:
  - `cross-project-explorer` - Trace features across projects
  - `api-impact-analyzer` - Assess API changes
  - `contract-validator` - Validate type safety

**Standard analysis**:
- Read and analyze existing codebase structure, patterns, and conventions
- Document current patterns (MVC, microservices, layered architecture, etc.)
- Assess technical debt and improvement opportunities
- Consider scalability and growth implications
- Map component relationships and dependencies

### 2. Feature Development

#### Multi-Project Development Strategy

**When changes span multiple projects**:

1. **API-First Approach** (if backend changes needed):
   - Design API contract first (request/response schema)
   - Implement API changes
   - Update API tests
   - Deploy API
   - Then update frontend consumers (Portal and/or Admin)

2. **Frontend-Only Changes**:
   - Determine which frontend(s) need updates (Portal, Admin, or both)
   - Respect each project's independent conventions
   - **Do not copy code** between Portal and Admin without understanding differences
   - Check version differences (e.g., PrimeVue 4.3 vs 4.4)

3. **No Cross-Frontend Consistency Enforcement**:
   - Portal and Admin are independent projects with different needs
   - **Do not enforce** similar patterns between them
   - Each can evolve independently based on their own requirements

#### Planning Requirements

**Plan before implementing**:
- **Requirements analysis** (apply YAGNI - only what's needed now)
- **Implementation approach** (apply KISS - simplest solution)
- **Affected components** across projects
- **API contract changes** (if any)
- **Potential risks** and mitigation strategies
- **Project-specific considerations**:
  - Portal: Check PrimeVue components, use pnpm
  - Admin: Check actual patterns (don't assume Portal conventions)
  - API: Follow Laravel conventions, PSR-12

**CRITICAL Git Policy**:
- NEVER execute git commit, push, merge, or any git write operations unless explicitly requested
- Read-only git commands (status, log, diff) are allowed
- Each project has independent git history

**Apply core principles consistently**:
- **YAGNI**: Build only what's required for current user stories
- **KISS**: Choose the simplest solution that works reliably
- **DRY**: Reuse within project boundaries, be cautious across projects

**Implementation standards**:
- Follow project-specific patterns (don't mix!)
- Implement incrementally with testable components
- Consider edge cases and error conditions
- Maintain backward compatibility (especially for API)
- Resist feature creep

### 3. Debugging Methodology

**Systematic approach**:
1. **Identify the project scope** - Which project has the issue?
2. Reproduce the issue consistently
3. Isolate the problem scope (frontend, backend, integration?)
4. Analyze relevant code paths
5. Form hypotheses about root causes
6. Test hypotheses systematically
7. Implement and verify fixes

**Cross-project debugging**:
- Use browser DevTools Network tab for API contract issues
- Check API response against frontend TypeScript types
- Use `contract-validator` subagent for type mismatches
- Verify API versioning (`/api/v2/` endpoints)

**Standard debugging**:
- Use logging strategically at key decision points
- Preserve evidence (symptoms, errors, investigation steps)
- Consider multiple root causes

### 4. Test Writing Standards

**Test requirements per project**:
- **Portal**: Vitest for unit tests (utilities, composables, transformers)
- **Admin**: Follow admin's test patterns (if any)
- **API**: PHPUnit for feature and unit tests

**Test coverage**:
- Happy path scenarios
- Edge cases and boundary conditions
- Error conditions and exception handling
- Integration points (especially API contracts)

**Test structure**:
- Follow AAA pattern (Arrange, Act, Assert) or Given-When-Then
- Test behavior, not implementation details
- Write clear, readable tests as documentation

---

## Code Quality Standards

### Design Principle Enforcement
- **YAGNI Reviews**: Regularly question if features/code are actually needed
- **KISS Validation**: Ensure solutions are as simple as possible but no simpler
- **DRY Audits**: Identify duplication within projects (be cautious across projects)
- **Principle Balance**: When principles conflict, prioritize based on context:
  - Early development: YAGNI > KISS > DRY
  - Mature systems: DRY > KISS > YAGNI
  - Critical systems: KISS > DRY > YAGNI

### Security First
- **Validate all inputs**: Backend ALWAYS validates, frontend validation is UX only
- **Handle sensitive data carefully**: Use encryption, avoid logging secrets
- **Follow security best practices**: OWASP guidelines, principle of least privilege
- **API security**: Laravel Sanctum authentication, proper authorization
- **Never trust frontend data**: All validation must occur server-side

### Performance Considerations
- **Profile before optimizing**: Measure actual bottlenecks
- **Consider resource usage**: Memory, CPU, I/O, network
- **Database efficiency**: Optimize queries, indexing, avoid N+1 (API)
- **Frontend performance**: Code splitting, lazy loading, efficient re-renders
- **API response times**: Target < 200ms for standard endpoints
- **Caching strategies**: Redis for backend, TanStack Query for frontend

### Documentation Requirements
- **Code self-documentation**: Write clear, expressive code
- **API documentation**: Maintain up-to-date API contracts
- **Architecture decisions**: Document significant design choices in project.md
- **Update documentation**: Keep docs in sync with code changes
- **Cross-project context**: Use `harbor/openspec/project.md` for ecosystem overview

---

## Project-Specific Guidelines

### Portal (Primary Development Focus)

**Code Style**:
- Always leave space between function name and brackets: `function fn () {...}`, `fn () {...}`
- Use single quotes for strings unless interpolation needed
- Tailwind CSS via `@apply` in `<style scoped>`
- Follow Vue 3 Composition API with `<script setup>`

**Architecture**:
- TanStack Query composables in `src/composables/queries/`
- Pinia stores in `src/stores/`
- Types in `src/types/`
- Transformers in `src/utils/transformers/`
- Always check PrimeVue 4.3.x components before custom implementations

**Commands**: Use `pnpm` for all package operations

### Admin (Secondary Development)

**Important**: Admin is an **independent project**
- **Check Admin's actual implementation** - don't assume Portal patterns
- **Respect version differences** (e.g., PrimeVue 4.4.x vs Portal's 4.3.x)
- **Don't blindly copy** Portal code without understanding Admin's architecture
- **No consistency enforcement** - Admin can have completely different patterns from Portal
- When in doubt, read Admin's codebase first

### API (Backend Service)

**Guidelines**:
- Follow Laravel 12.0 conventions and PSR-12 standards
- Use Eloquent ORM, Service Container, built-in features
- API Resources for response transformation
- Versioned endpoints: `/api/v2/[resource]`
- Migrations for all schema changes
- Feature tests for endpoints, unit tests for business logic

**API Contract Ownership**:
- API defines the source of truth
- Frontend types must mirror backend schemas
- Breaking changes require new API versions
- Use transformers if naming differs (snake_case ↔ camelCase)

---

## Communication Guidelines

### Code Reviews
- Explain reasoning and design trade-offs
- Suggest alternatives when possible
- Focus on learning opportunities
- Be constructive with actionable feedback

### Problem-Solving Communication
- Clear problem statements with reproducible examples
- Solution rationale explaining the chosen approach
- Risk assessment with mitigation strategies
- Realistic time estimates with confidence levels

---

## Error Handling and Recovery

### Graceful Degradation
- Fail safely without cascading failures or data corruption
- Provide meaningful error messages and recovery options
- Implement proper logging and alerting
- Ensure changes can be safely reverted

### Continuous Improvement
- Conduct blameless post-mortems for significant issues
- Refactor regularly for code quality
- Keep dependencies current
- Document lessons learned in `harbor/.claude/lessons.md`

---

## Implementation Workflow

### Before Every Phase: Apply Core Principles
- **YAGNI Check**: "Do we actually need this right now?"
- **KISS Check**: "Is this the simplest solution that works?"
- **DRY Check**: "Are we duplicating existing functionality?"
- **Project Boundary Check**: "Which projects are affected?"

### Workflow Phases

1. **Analysis Phase**: Understand requirements, assess current state, identify constraints
   - Identify affected projects (Portal, Admin, API)
   - Focus on current, validated requirements (YAGNI)
   - Look for simple solutions (KISS)
   - Identify existing solutions to reuse (DRY)
   - Use subagents for cross-project analysis if needed

2. **Planning Phase**: Create detailed implementation plan with milestones and risks
   - Plan minimal viable implementation (YAGNI)
   - Design for simplicity and clarity (KISS)
   - Leverage existing patterns within each project (DRY)
   - **API-first if backend changes needed**
   - Document in OpenSpec (see below)

3. **Development Phase**: Implement with continuous testing and validation
   - Build only planned features, resist scope creep (YAGNI)
   - Write clear, straightforward code (KISS)
   - Extract common patterns as they emerge, not before (DRY)
   - Respect project-specific conventions

4. **Testing Phase**: Comprehensive testing including unit, integration, and end-to-end tests
   - Test within each project
   - Test API contracts between projects
   - Use `contract-validator` for type safety validation

5. **Review Phase**: Code review, security review, performance validation
   - Review for unnecessary complexity (YAGNI + KISS)
   - Check for duplication (DRY)
   - Validate API contracts if changed

6. **Deployment Phase**: Gradual rollout with monitoring and rollback capability
   - Deploy API changes first (if any)
   - Then deploy frontend changes
   - Monitor error rates and performance

7. **Monitoring Phase**: Post-deployment monitoring and issue resolution

---

## OpenSpec Integration

### Harbor's OpenSpec Strategy

Harbor uses **OpenSpec** for spec-driven development with a specific multi-project approach:

#### OpenSpec Location Rules

**CRITICAL**: Different projects have different OpenSpec usage:

1. **Cross-Project Changes** (Portal + Admin/API changes):
   - Write OpenSpec in `harbor/openspec/`
   - **Do NOT modify** `admin/openspec/` or `api/openspec/`
   - Reason: Other collaborators don't use OpenSpec

2. **Portal-Only Changes**:
   - **Option A**: Plan big picture in `harbor/openspec/`, detailed spec in `portal/openspec/`
   - **Option B**: Directly in `portal/openspec/` if change is isolated to Portal
   - Use your judgment based on change scope

3. **Admin/API Changes** (rare):
   - Plan in `harbor/openspec/` only
   - **Never create** OpenSpec in admin or api projects

### When to Use OpenSpec

**Create a proposal for**:
- New features spanning multiple projects
- API schema changes (breaking or non-breaking)
- Architecture changes in any project
- Performance optimizations that change behavior
- Security pattern updates
- Cross-project refactoring

**Skip proposal for**:
- Bug fixes (restoring intended behavior)
- Typos, formatting, comments
- Non-breaking dependency updates in a single project
- Configuration changes
- Small, isolated portal-only changes

### OpenSpec Workflow

**Three-stage workflow**:
1. **Creating Changes**: Proposal → Spec Deltas → Validation → Approval
2. **Implementing Changes**: Read tasks → Implement → Update checkboxes → Complete
3. **Archiving Changes**: Archive → Update specs → Validate

**For detailed workflow**, see [openspec/AGENTS.md](../openspec/AGENTS.md)

### Key Commands

```bash
# From harbor/ directory
openspec list                  # List active changes
openspec list --specs          # List specifications
openspec show [item]           # Display change or spec
openspec validate [item] --strict  # Validate changes
openspec archive <change-id>   # Archive after deployment
```

### Quality Assurance

Before marking implementation complete:
- All tasks in `openspec/changes/[change-id]/tasks.md` completed
- `openspec validate --strict` passes
- Code passes quality standards for each project
- Tests achieve >80% coverage for critical paths
- Performance metrics met (API < 200ms, frontend responsive)
- No critical security issues
- API contracts validated with `contract-validator` subagent (if applicable)

---

## Custom Subagents

Harbor provides specialized subagents in `harbor/.claude/agents/` for cross-project tasks:

### Available Subagents

1. **cross-project-explorer**
   - Trace features across Portal, Admin, and API
   - Understand end-to-end flows
   - Find all files related to a feature

2. **api-impact-analyzer**
   - Assess impact of API changes on frontends
   - Identify affected Portal and Admin code
   - Plan migration strategies

3. **contract-validator**
   - Validate API contracts match frontend types
   - Catch type mismatches before deployment
   - Ensure type safety across the stack

### When to Use Subagents

- **Before API changes**: Use `api-impact-analyzer`
- **Understanding features**: Use `cross-project-explorer`
- **Type safety concerns**: Use `contract-validator`

**See** [`harbor/.claude/agents/README.md`](./agents/README.md) for detailed usage.

---

## Emergency Protocols

### Critical Issues
- **Immediate response**: Prioritize system stability and user safety
- **Identify affected project(s)**: Portal, Admin, API, or multiple?
- **Communication**: Keep stakeholders informed with regular updates
- **Documentation**: Record all actions in `harbor/.claude/lessons.md`
- **Follow-up**: Conduct thorough post-incident analysis and implement preventive measures

---

## Key Principles Summary

**Golden Rules**:
1. **Project Independence**: Portal, Admin, and API are separate - respect boundaries
2. **API as Contract**: All integration through well-defined API contracts
3. **YAGNI**: Build only what you need right now
4. **KISS**: Keep solutions simple and understandable
5. **DRY**: Don't repeat yourself within projects, be cautious across projects
6. **OpenSpec in Harbor**: Cross-project changes planned in `harbor/openspec/`
7. **When in doubt**: Choose simplicity over cleverness, and ask before crossing project boundaries

---

Remember: Always prioritize code quality, security, and maintainability over speed of delivery. Take time to understand each project's patterns before making changes. Use subagents for cross-project analysis. Keep OpenSpec planning in harbor/ when changes span multiple projects.

**Last Updated**: 2026-03-05
**Scope**: Harbor Ecosystem (Portal, Admin, API)
