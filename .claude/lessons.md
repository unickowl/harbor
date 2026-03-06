# Lessons Learned

## 2026-03-05: OpenSpec Must Come First for New Features

**Mistake**: When asked to implement a new feature, spent 5 rounds of brainstorming Q&A before creating any OpenSpec files. User had to correct twice.

**Root cause**: Treated brainstorming skill and OpenSpec as sequential phases instead of integrated workflow.

**Rule**: When a request introduces new capabilities, breaking changes, or architecture shifts:
1. Read `openspec/AGENTS.md` FIRST
2. Ask clarifying questions if needed (keep them brief)
3. The FIRST artifact produced must be OpenSpec files (proposal.md, tasks.md, design.md, spec deltas)
4. Never present design conclusions as chat-only text — always write them into OpenSpec documents
5. Brainstorming output = OpenSpec files, not conversation summaries

**Trigger**: Any request that matches CLAUDE.md's OpenSpec triggers (new capabilities, breaking changes, architecture shifts, big performance/security work).

## 2026-03-05: Don't Propose Options That Contradict Domain Knowledge

**Mistake**: During brainstorming for stablecoin receive methods, proposed "one address + multi-chain selection" as an option. Different blockchain networks have incompatible address formats (EVM 0x... vs Solana base58 vs Stellar G...), so this option was fundamentally wrong.

**Rule**: When proposing design options:
1. Validate each option against domain constraints before presenting it
2. Don't invent shortcuts that sound convenient but violate technical reality
3. When the backend schema has per-item fields (each chain has its own address), respect that structure — it exists for a reason
4. If unsure about domain-specific constraints, state the assumption and ask rather than presenting it as a valid option

## 2026-03-05: Always Use Existing UI Components Before Custom Implementations

**Mistake**: Wrote a custom `<button>` with hand-crafted Tailwind classes instead of using PrimeVue `Button` with `text`, `size="small"` props. CLAUDE.md explicitly says "Look up primevue components for UI elements if applicable before doing custom implementations."

**Rule**: Before writing any custom HTML element with styling:
1. Check if PrimeVue already has a component for it (Button, Tag, Message, etc.)
2. Check if the component's existing props/variants can achieve the desired look
3. Only write custom elements when PrimeVue genuinely cannot express the requirement
4. This applies to ALL UI elements, not just complex ones

## 2026-03-06: Plans and Docs for API/Admin Go in harbor/, Not in the Project

**Mistake**: Created `api/docs/plans/` for an API implementation plan. User's rule clearly states plans for API or Admin should live in `harbor/` (the umbrella repo), never inside the individual project directories.

**Root cause**: Followed the writing-plans skill's default path (`docs/plans/`) without checking CLAUDE.md's OpenSpec Location Rules.

**Rule**:
1. **Portal-only** changes: plan can go in `portal/` or `harbor/`
2. **API or Admin** changes: plan MUST go in `harbor/` — never create docs/openspec inside `api/` or `admin/`
3. **Cross-project** changes: always in `harbor/`
4. User-defined rules in CLAUDE.md ALWAYS override skill template defaults
5. New API endpoints = new capabilities = MUST use OpenSpec, not freeform plans
6. ALWAYS check OpenSpec trigger conditions in CLAUDE.md BEFORE choosing a planning approach — skills like writing-plans do NOT replace OpenSpec when OpenSpec applies

## 2026-03-06: Write Tests That Protect Business Logic, Not Framework Behavior

**Mistake**: Wrote 8 test cases including auth checks (testing Laravel middleware), a list test that only verified JSON structure, and a beneficiary info test with 13 redundant field assertions that overlapped with another test.

**Root cause**: Writing tests mechanically to "cover" endpoints rather than thinking about what risks each test is actually protecting against.

**Rule — Every test must answer: "What real bug or security issue would this catch?"**

1. **Don't test framework guarantees** — auth middleware, CSRF, route model binding are Laravel's responsibility. Unless you manually moved a route outside middleware, testing `assertStatus(401)` is noise.
2. **Test security boundaries** — cross-application data access, sensitive data exclusion (AML), authorization scoping. These protect against real vulnerabilities.
3. **Test transformation logic** — null filtering (`array_filter`), conditional field inclusion, data restructuring. These are your custom Resource logic where bugs actually happen.
4. **Test data isolation** — "list only returns own application's transfers" is valuable; "list returns 200 with some fields" is not.
5. **Don't duplicate coverage** — if two tests both exercise the same function (`getBeneficiaryInfo`), keep the one testing the edge case (null omission), not the one asserting 13 happy-path field mappings.
6. **Fragile assertions are red flags** — `assertCount(2)` that depends on setUp state, exact field-by-field matching that breaks on any schema change. Prefer behavioral assertions over structural ones.
