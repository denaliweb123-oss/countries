# GraphQL API — Test Strategy

**Role:** Senior QA Architect | **Audience:** Engineering Leadership
**Scope:** `https://countries.trevorblades.com` — public read-only GraphQL API
**Stack:** Cloudflare Workers (128 MB / 10–30 ms CPU), Pothos code-first, Sift.js filters, in-memory npm data, Bun test

---

## 0. Discovery

### 0.1 Questions Asked Before Testing (Prioritized & Graded)

- **[P0 - Critical] Catastrophic Backtracking (ReDoS)**: Are we protected against "Catastrophic Backtracking"? (The `regex` filter in `StringQueryOperatorInput` is a major risk).
  *Answer: Partially addressed with `maxLength` validation, but complex patterns within the limit could still hang the JS isolate. **High Risk.***

- **[P0 - Critical] GraphQL Batching Limits**: Is there a hard limit on GraphQL Batching? (`batching: true` without a `limit`).
  *Answer: No limit. An attacker could send hundreds of queries in one request, bypassing depth limits and exhausting Worker memory. **Gap.***

- **[P1 - High] Resource constraints**: At what query complexity does the Worker hit its CPU ceiling? 
  *Answer: `Language.countries` × `Continent.countries` produce O(n³) iterations at depth 4. No depth/complexity limit enforced yet. **Gap.***

- **[P1 - High] Consumer Safety**: How many typed clients break silently if a field is removed or nullability changes?
  *Answer: Undetectable pre-production — no schema diffing in CI, no snapshot committed. **Gap.***

- **[P1 - High] Data Integrity**: How do we know an upstream package update silently breaks a resolver transformation?
  *Answer: We don't — no regression fixtures exist for these transformations. **Gap.***

- **[P2 - Medium] Query Amplification**: How do we prevent "Query Amplification" via Aliases?
  *Answer: Unaddressed. A user could query the same expensive field multiple times using aliases in a single request. **Gap.***

- **[P2 - Medium] Nullability Failure Mode**: What is our failure mode with `defaultFieldNullability: false`?
  *Answer: If a source returns `undefined`, the entire object returns `null`. This is an "all-or-nothing" design risk.

- **[P3 - Low] Developer Recommendations**: What do developers recommend paying more attention to?
  *Answer: Recursive relationship performance and localization fallback logic were flagged by the team as high-complexity areas.*

### 0.2 AI Methodology (Audit Compliance)
*   **Prompting Strategy:** Utilized Claude 3.5 Sonnet to perform adversarial analysis on the Pothos schema, specifically looking for N+1 vulnerabilities and ReDoS vectors.
*   **Refinement:** AI suggested a generic complexity plugin; we refined this to a hard **Depth Limit of 7** to specifically fit the Cloudflare Worker 128MB ceiling.

### 0.3 Out of Scope & Trade-offs
*   **Load Testing:** Skipped this cycle. Reason: Focusing on logical correctness and resource safety. Cloudflare's Edge provides horizontal scale; we are protecting the single-isolate execution limit instead.

### Priority Scenarios

| # | Scenario | Severity | Why |
|---|---|---|---|
| 1 | ReDoS via `regex` filter (`schema.ts:216`) | Critical | Direct user string to `$regex` bypasses CPU limits via catastrophic backtracking. |
| 2 | Unbounded nested traversal | High | No depth limit. O(n³) at depth 4. CPU ceiling hit for entire isolate. |
| 3 | Silent data regression on package update | High | 5 tests exist; none cover split/coerce/filter resolver logic. |
| 4 | Breaking schema contract reaches production | High | No `graphql-inspector` diff in CI. No snapshot on `main`. |
| 5 | Batching abuse (`graphql.ts:7`) | Medium | `batching: true`, no size limit. N operations per HTTP request. |

### Out of Scope

| Item | Reason | Proxy Signal |
|---|---|---|
| Load/stress testing | Infrastructure setup time not available this cycle | Cloudflare Analytics CPU histograms |
| External Connectivity | Using Countries API as sole target | **Backup API:** `https://rickandmortyapi.com/graphql` (used as a heartbeat to verify CI network state) |
| Full i18n coverage | Would duplicate `i18n-iso-countries` test suite | Spot-check 3 locales × 5 countries |
| Mutation/subscription testing | Neither exists in the schema | N/A until added |
| E2E against live URL | `yoga.fetch` integration tests cover the full stack | Staging smoke tests pre-release |

### AI Methodology

**Model:** Claude claude-sonnet-4-6

**Prompt:** *"Act as a Senior QA Architect. Generate a production-grade GraphQL test strategy covering testing levels, performance, security, automation, and tooling for engineering leadership review."*

**AI produced:** Complete draft with file:line citations, O(n³) complexity framing, ReDoS analysis, `graphql-inspector` workflow, DataLoader gap analysis.

**Human changes:** Elevated ReDoS to Priority 1 (Critical); added batching abuse scenario (AI missed it); added discovery question answers; expanded scope exclusions with proxy signals; verified all `[x]` checklist items against source before marking.

---

## 1. Schema Governance

- **Approach:** Code-first Pothos (`schema.ts:16`). SDL generated at runtime — no `.graphql` file committed.
- **Nullability:** `defaultFieldNullability: false` (`schema.ts:17-19`). Nullable exceptions: `capital`, `currency`, `State.code`, `Subdivision.emoji`, single-item queries.
- **Breaking change detection:** Not implemented. Recommended: commit SDL as `schema.graphql`, run `graphql-inspector diff` in CI before merge. Exit criteria: zero breaking changes without approval.
- **Snapshot testing:** Not implemented. Recommended: JSON fixtures for top-level queries, assert `deepEqual` in CI.
- **Linting:** `@graphql-eslint/eslint-plugin` installed but not in CI.

---

## 2. Testing Levels

### Unit — Resolver Logic (none exist today)

| Function | Location | Risk |
|---|---|---|
| `currencies` / `phones` split on `,` | `schema.ts:118,107` | Empty string → `[""]`; wrong count for multi-value |
| `capital` / `currency` empty-string coercion | `schema.ts:111,115` | `""` returned as non-null |
| `subdivisions` type filter | `schema.ts:163` | Other types silently dropped |
| `name(lang)` fallback | `schema.ts:98` | Unregistered locale silently falls back |
| `isValid*Code` guards | `schema.ts:249-259` | Invalid code must return `null`, not throw |

### Integration — Schema-Driven (5 tests today)

Covered: `countries` filtering (`eq`, `in`, `regex`), all-countries count, GB subdivisions.

Critical gaps: `continent`/`language` queries, `null` returns, localization, nullable fields (`capital` of AQ), `ne`/`nin` operators, split fields (`currencies`, `phones`), batched request.

**Snapshot testing:** No built-in support in Bun. Commit JSON fixtures, assert `deepEqual`.

### Contract Testing

Add `generate-sdl` script → commit `schema.graphql` → `graphql-inspector diff` in CI → block breaking changes without approval label.

### E2E

Not warranted now (read-only, no auth). Revisit when UI consumers or auth flows are introduced.

---

## 3. Security

| Concern | Status | Action |
|---|---|---|
| ReDoS (`schema.ts:216`) | **Critical gap** | Add `safe-regex` validation in Zod validator before Sift executes |
| Query depth | **Gap** | `useDepthLimit({ maxDepth: 7 })` in `graphql.ts` |
| Query complexity | **Gap** | `graphql-query-complexity`; list fields cost 10, budget 1000 |
| Rate limiting | **Gap** | Cloudflare WAF rule: 300 req/min per IP on `POST /graphql` |
| Batching | **Gap** | `batching: { limit: 10 }` in `graphql.ts` |
| Introspection | Intentional — public API with playground. Document in `graphql.ts`. | — |
| Field-level auth | N/A — public read-only API | Address if writes are added |

---

## 4. Performance

- **N+1:** `Language.countries` (180+ × 250 = 45,000+ iterations) and `Continent.countries` (8 × 250 = 2,000) scan all countries per call. Acceptable at current scale. Detection: canary test asserting `languages { countries { code } }` < 100 ms.
- **DataLoader:** Recommended for `Language.countries` / `Continent.countries` if nested patterns become common. Per-request cache scope only.
- **Load testing:** Deferred — see Out of Scope. When implemented: k6, 100 VUs, p95 < 200 ms.
- **Caching:** None configured. Verify via `CF-Cache-Status` header when/if added.
- **Observability:** Cloudflare Analytics (basic). Add Apollo GraphOS or Yoga tracing plugin for resolver-level visibility.

---

## 5. Operational Health

- **Error masking:** `wrangler.toml` has no `NODE_ENV` binding — stack traces may leak in production. Fix: add `NODE_ENV = "production"` to `[vars]`. Test: assert no `extensions.stacktrace` in error responses.
- **Audit logs:** No structured logging. Recommended: Cloudflare Workers Logpush with fields `operationName`, `responseTime`, `statusCode`, `cfRay`.

---

## 6. CI/CD

### Exit Criteria

| Gate | Target | Status |
|---|---|---|
| Integration tests | 0 failures | Enforced |
| Schema diff | 0 breaking changes | Not implemented |
| Schema snapshot | 0 structural drift | Not implemented |
| Type check (`tsc --noEmit`) | 0 errors | Not in CI |
| ESLint | 0 errors | Not in CI |
| ReDoS fuzz | 0 catastrophic patterns | Not implemented |

### Recommended Step Order

```yaml
- run: bunx tsc --noEmit       # type safety
- run: bunx eslint .            # lint
- run: bun run schema:diff      # breaking change detection
- run: bun test                 # unit + integration
- run: bun run test:snapshot    # snapshot validation
- run: bun run test:security    # ReDoS fuzz + depth tests
```

### Environment Matrix

| Environment | Target | Suite | Trigger |
|---|---|---|---|
| In-memory | `yoga.fetch` | All tests | Every PR |
| Staging Worker | `wrangler deploy --env staging` | Smoke tests + error-masking assertion | Pre-release |
| Production | Live URL | Read-only smoke tests | Post-deploy only |

**Parity enforcement rule:** If a test passes in-memory but fails against the staging Worker, the staging result is the source of truth — no release proceeds until both environments pass.

---

## 7. Tooling

| Concern | Tool | Status |
|---|---|---|
| Testing | Bun test | In use |
| Integration executor | `@graphql-tools/executor-http` | In use |
| Schema linting | `@graphql-eslint/eslint-plugin` | Installed, not in CI |
| Breaking change detection | `graphql-inspector` | Recommended |
| Monitoring | Cloudflare Analytics | In use (basic) |
| Resolver tracing | Apollo GraphOS | Recommended |
| Load testing | k6 | Recommended |
| Depth limiting | `graphql-yoga` `useDepthLimit` | Recommended |
| Complexity | `graphql-query-complexity` | Recommended |
| ReDoS prevention | `safe-regex` | Recommended |

---

## 8. Checklist

### Implemented
- [x] Integration tests via `yoga.fetch` (`graphql.test.ts:7-10`)
- [x] `countries` filtering — `eq`, `in`, `regex` (`graphql.test.ts:29-88`)
- [x] Subdivision resolver — GB (`graphql.test.ts:90-106`)
- [x] Zod validation + `VALIDATION_ERROR` extension (`schema.ts:22-31`, `93-94`)
- [x] `defaultFieldNullability: false` (`schema.ts:17-19`)
- [x] Null prototype fix before Sift (`schema.ts:276, 303, 334`)
- [x] Batching enabled (`graphql.ts:7`)
- [x] CI runs `bun test` on push (`.github/workflows/test.yml`)

### To Implement
- [ ] Unit tests: currency/phone split, empty-string coercion, subdivision filter, locale fallback
- [ ] Integration tests: `continent`, `language`, null returns, localization, nullable fields, `ne`/`nin`, batched request
- [ ] `safe-regex` validation on `StringQueryOperatorInput.regex`
- [ ] `useDepthLimit({ maxDepth: 7 })` in `src/graphql.ts`
- [ ] `graphql-query-complexity` (budget: 1000) in `src/graphql.ts`
- [ ] `batching: { limit: 10 }` in `src/graphql.ts`
- [ ] `NODE_ENV = "production"` in `wrangler.toml [vars]`
- [ ] Error-masking integration test (no `extensions.stacktrace`)
- [ ] Cloudflare WAF rate-limiting rule (300 req/min per IP)
- [ ] Cloudflare Workers Logpush structured logging
- [ ] Schema SDL snapshot + `graphql-inspector diff` in CI
- [ ] JSON response fixtures for snapshot regression
- [ ] `bunx tsc --noEmit` and `bunx eslint .` in CI
- [ ] N+1 canary test: `languages { countries { code } }` < 100 ms
- [ ] CI Heartbeat: Connectivity check against Rick and Morty API to isolate network vs. service failures
- [ ] Staging Worker environment in `wrangler.toml` + smoke test suite
