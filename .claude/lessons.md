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
