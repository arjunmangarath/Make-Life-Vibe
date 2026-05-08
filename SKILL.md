---
name: Make-Life-Vibe
description: "Full-stack hackathon execution methodology: from problem statement to winning pitch. Covers scoring-first planning, tech stack selection, security hardening, testing strategy, deployment pipeline, and presentation. Battle-tested at PromptWars Bengaluru 2026 — scored 96.84%, won pitching round."
risk: low
source: personal
author: arjunmangarath
date_added: "2026-05-08"
---

# Hackathon Win Methodology

## Origin

Built and validated at **PromptWars Bengaluru 2026** (Build with AI, Google for Developers).
Result: 96.84% automated score + Pitching Arena Winner trophy.

---

## Phase 0 — Read the Scoring Rubric Before Writing One Line of Code

This is the single most important step. Every hackathon has a rubric. Find it.

Map every criterion to a concrete implementation decision:

| Criterion | Implementation |
|-----------|---------------|
| Code Quality | TypeScript strict, Zod validation, custom error hierarchy |
| Security | Rate limiting, CSP headers, prompt injection sanitization, CVE audit |
| Testing | Jest multi-env (node + jsdom), 100+ tests across all layers |
| Accessibility | ARIA, semantic HTML, focus management, skip link |
| Google Services | Use ALL available services, not just the obvious one |
| Performance | Server-side caching, standalone build, lazy loading |
| Problem Statement | Name the real user pain in the hero text |

**Rule:** If a feature doesn't map to a scoring criterion, build it last or not at all.

---

## Phase 1 — Tech Stack Selection (30 minutes max)

### Defaults that win

| Layer | Choice | Why |
|-------|--------|-----|
| Framework | Next.js 15 (App Router) | SSR + API routes + standalone build in one |
| Language | TypeScript strict mode | Scores points on code quality rubrics |
| AI | Provider's preferred model (Gemini for Google hackathons) | Judges know it, integrations work |
| Auth | Whoever sponsors the hackathon | Firebase if Google, Supabase if otherwise |
| DB / Cache | Same vendor | Firebase Firestore, Supabase Postgres |
| Deploy | Sponsor's cloud | Cloud Run for Google, Vercel for others |
| Validation | Zod | Runtime + compile-time, used at every boundary |
| Testing | Jest + Testing Library | Fast setup, covers node + browser |

### Google hackathon checklist (use all 6 services)
- [ ] Gemini AI via Vertex AI (ADC — no API key needed, no quota limits)
- [ ] Firebase Firestore (caching layer)
- [ ] Firebase Analytics (event tracking)
- [ ] Google Maps Embed (iframe, no key required)
- [ ] Google Calendar URL scheme (deep links, no OAuth)
- [ ] Google Cloud Run (deployment)

---

## Phase 2 — Project Bootstrap (1 hour)

```bash
npx create-next-app@latest my-app --typescript --tailwind --app --src-dir
cd my-app
npm install zod firebase firebase-admin @google-cloud/vertexai
npm install -D jest @testing-library/react @testing-library/jest-dom ts-jest jest-environment-jsdom
```

### Files to create immediately (skeleton, fill later)

```
src/
  types/index.ts          # All interfaces upfront — forces you to think about data shape
  lib/errors.ts           # Custom error hierarchy (AppError, ValidationError, etc.)
  lib/env.ts              # Centralised env access with startup warnings
  lib/logger.ts           # Structured JSON logging (Cloud Run auto-ingests)
  app/api/plan/route.ts   # Main API route
  app/api/health/route.ts # Health check listing all services
  middleware.ts           # Rate limiter
```

### security headers — paste into next.config.ts immediately

```typescript
const securityHeaders = [
  { key: "X-DNS-Prefetch-Control", value: "on" },
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
  { key: "X-Frame-Options", value: "SAMEORIGIN" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "X-XSS-Protection", value: "1; mode=block" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=(self)" },
  {
    key: "Content-Security-Policy",
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-eval' 'unsafe-inline' *.googleapis.com *.gstatic.com",
      "style-src 'self' 'unsafe-inline' *.googleapis.com",
      "img-src 'self' data: blob: *.googleapis.com *.gstatic.com *.google.com",
      "frame-src maps.google.com *.google.com",
      "connect-src 'self' *.googleapis.com *.google.com *.firebaseapp.com *.firebase.com",
      "font-src 'self' *.gstatic.com",
      "object-src 'none'",
      "base-uri 'self'",
      "upgrade-insecure-requests",
    ].join("; "),
  },
];
```

---

## Phase 3 — The API Route Pattern (the engine)

Every API route follows this exact pattern:

```typescript
export async function POST(req: NextRequest) {
  const requestId = crypto.randomUUID();
  const startTime = Date.now();

  try {
    // 1. Parse body
    const body = await req.json();

    // 2. Validate with Zod (throw ValidationError on failure)
    const parsed = InputSchema.safeParse(body);
    if (!parsed.success) throw new ValidationError("Invalid input", parsed.error.message);

    // 3. Check cache
    const cached = await getCache(parsed.data);
    if (cached) return jsonResponse({ data: cached, cached: true }, { "X-Cache": "HIT" });

    // 4. Call AI / external service
    const result = await callAI(parsed.data);

    // 5. Validate AI output with Zod (AI can return garbage)
    const validated = OutputSchema.safeParse(result);
    if (!validated.success) throw new GenerationError("Invalid AI response");

    // 6. Store in cache
    await setCache(parsed.data, validated.data);

    // 7. Return with diagnostic headers
    return jsonResponse(
      { data: validated.data, cached: false },
      {
        "X-Cache": "MISS",
        "X-Request-ID": requestId,
        "X-Response-Time": `${Date.now() - startTime}ms`,
      }
    );
  } catch (err) {
    if (isAppError(err)) return errorResponse(err.message, err.statusCode);
    return errorResponse("Internal server error", 500);
  }
}
```

---

## Phase 4 — AI Prompt Engineering

### The prompt template

```typescript
function buildPrompt(input: ValidatedInput, today: string): string {
  return `You are an expert [domain] assistant. Today's date is ${today}.

User request: ${sanitizeForPrompt(input.mainField)}
Context: ${sanitizeForPrompt(input.contextField)}

Return ONLY valid JSON matching this exact schema:
${JSON.stringify(outputSchema, null, 2)}

Rules:
- No markdown, no explanation, no code fences
- All fields required unless marked optional
- Use today's date (${today}) for any date-sensitive context`;
}
```

### sanitizeForPrompt — always use this on user input

```typescript
function sanitizeForPrompt(text: string, maxLength = 200): string {
  return text
    .replace(/[\x00-\x08\x0B-\x1F\x7F]/g, "") // control characters
    .replace(/`{3,}/g, "")                       // triple-backtick injection
    .replace(/<script[^>]*>.*?<\/script>/gi, "") // script tags
    .trim()
    .slice(0, maxLength);
}
```

---

## Phase 5 — Caching Pattern (makes demos instant)

```typescript
// SHA-256 hash of normalised inputs = cache key
async function hashInput(input: Record<string, unknown>): Promise<string> {
  const normalised = JSON.stringify(input, Object.keys(input).sort())
    .toLowerCase()
    .replace(/\s+/g, " ");
  const buf = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(normalised));
  return Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2, "0")).join("").slice(0, 20);
}
```

- Store in Firestore with `cachedAt` timestamp
- TTL check on read: reject if `now - cachedAt > 7 days`
- Return `null` on any Firestore error (never crash the main request)
- Identical requests from any user → instant response + "⚡ Cached" badge

---

## Phase 6 — Testing Strategy (100+ tests in 2 hours)

### Jest multi-environment config

```typescript
// jest.config.ts
export default {
  projects: [
    {
      displayName: "node",
      testEnvironment: "node",
      testMatch: ["<rootDir>/src/__tests__/*.test.ts"],
      transform: { "^.+\\.ts$": ["ts-jest", {}] },
    },
    {
      displayName: "jsdom",
      testEnvironment: "jsdom",
      testMatch: ["<rootDir>/src/__tests__/*.test.tsx"],
      setupFilesAfterFramework: ["@testing-library/jest-dom"],
      transform: { "^.+\\.tsx?$": ["ts-jest", { tsconfig: { jsx: "react-jsx" } }] },
    },
  ],
};
```

### Test suites to write (in order of impact)

1. **Cache hashing** — determinism, normalisation (casing, array order), different inputs differ
2. **Sanitize function** — control chars, injection markers, truncation, unicode preserved
3. **Error hierarchy** — status codes, subclass instanceof, type guard
4. **Rate limiter algorithm** — allow up to limit, block 11th, reset after window, IP isolation
5. **URL builders** — calendar links, maps links, encoding special chars
6. **React components** — render, interaction, conditional UI, ARIA attributes

---

## Phase 7 — Accessibility Checklist (free points)

- [ ] `<a href="#main-content">Skip to main content</a>` as first element
- [ ] Every input has a `<label htmlFor>` or `aria-label`
- [ ] Toggle buttons use `aria-pressed`
- [ ] Results area uses `aria-live="polite"` + `aria-atomic="true"`
- [ ] Focus moves to results after async operation (`useRef` + `useEffect`)
- [ ] Semantic elements: `<article>`, `<section>`, `<header>`, `<nav>`, `<time>`, `<dl>`
- [ ] Loading state has `role="status"`, error has `role="alert"`
- [ ] External links have `aria-label` describing destination

---

## Phase 8 — Deployment (Cloud Run)

```dockerfile
# Dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3080
ENV PORT=3080
CMD ["node", "server.js"]
```

```bash
# Deploy
gcloud run deploy my-service \
  --source . \
  --region asia-south1 \
  --memory 512Mi \
  --timeout 300 \
  --allow-unauthenticated \
  --project YOUR_PROJECT_ID
```

**Critical:** Set timeout to 300s. Default 60s will 504 on AI calls.

---

## Phase 9 — The Pitch Structure (5 minutes)

### Slide order that wins

1. **The pain** (10 sec) — one sentence, one number. "Planning a trip takes 4 hours. This does it in 10 seconds."
2. **The demo** (2 min) — live, not slides. Show the golden path. Show one edge case.
3. **The tech** (1 min) — name every sponsor technology. Show the architecture diagram. One sentence per service.
4. **The numbers** (30 sec) — test count, cache hit time, security headers count. Judges love specifics.
5. **The ask / next step** (30 sec) — what would you build next? Shows thinking beyond the hackathon.

### What judges actually look for
- **Does it work live?** Never demo from a recording.
- **Do you understand what you built?** Be able to answer "how does the caching work?" in 2 sentences.
- **Did you use our platform?** Name every sponsor service by its real product name.
- **Is it real code?** Have the GitHub repo open on a second tab.

---

## Phase 10 — The 2-Hour Sprint Checklist

When time is short, do these in order and stop when time runs out:

```
[ ] npm audit fix — patch known CVEs (takes 2 minutes, massive security score boost)
[ ] Add X-Cache, X-Request-ID, X-Response-Time headers to every API response
[ ] Add /api/health endpoint listing all integrated services
[ ] Add "Add to Calendar" links (Google Calendar URL scheme, no OAuth)
[ ] Add location links (Google Maps search URL, no API key)
[ ] Add weather/seasonal context field to AI output schema
[ ] Add aria-live to results container
[ ] Write 5 cache tests + 5 sanitize tests (fast to write, high test count impact)
[ ] Add "⚡ Cached" badge to UI
[ ] Check `npm run build` passes clean
```

---

## Reusable Patterns

### Custom error hierarchy (copy-paste)

```typescript
export class AppError extends Error {
  constructor(message: string, public statusCode = 500) {
    super(message); this.name = this.constructor.name;
  }
}
export class ValidationError extends AppError {
  constructor(message: string, public details: string) { super(message, 400); }
}
export class GenerationError extends AppError {
  constructor(message: string, public cause?: Error) { super(message, 502); }
}
export class RateLimitError extends AppError {
  constructor(public retryAfterSeconds: number) { super("Too many requests", 429); }
}
export const isAppError = (e: unknown): e is AppError => e instanceof AppError;
```

### Sliding-window rate limiter (Edge-compatible, copy-paste)

```typescript
const ipMap = new Map<string, { count: number; resetAt: number }>();
const LIMIT = 10, WINDOW = 60_000;

export function middleware(req: NextRequest) {
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0] ?? "unknown";
  const now = Date.now();
  const entry = ipMap.get(ip);

  if (!entry || now > entry.resetAt) {
    ipMap.set(ip, { count: 1, resetAt: now + WINDOW });
  } else if (entry.count >= LIMIT) {
    const retryAfter = Math.ceil((entry.resetAt - now) / 1000);
    return new NextResponse("Rate limit exceeded", {
      status: 429,
      headers: { "Retry-After": String(retryAfter) },
    });
  } else {
    entry.count++;
  }
  return NextResponse.next();
}
```

### Structured logger (Cloud Run auto-ingests, copy-paste)

```typescript
type Level = "INFO" | "WARN" | "ERROR";
export function log(level: Level, message: string, data?: Record<string, unknown>) {
  console.log(JSON.stringify({ severity: level, message, timestamp: new Date().toISOString(), ...data }));
}
export const logInfo  = (msg: string, d?: Record<string, unknown>) => log("INFO",  msg, d);
export const logWarn  = (msg: string, d?: Record<string, unknown>) => log("WARN",  msg, d);
export const logError = (msg: string, d?: Record<string, unknown>) => log("ERROR", msg, d);
```

---

## Anti-Patterns (what loses hackathons)

- Building features before reading the rubric
- Using API keys when ADC (Application Default Credentials) is available
- Skipping tests because "there's no time" — 30 tests take 45 minutes
- Single-environment Jest (missing jsdom component tests)
- Demo from screenshots instead of live app
- Forgetting `npm audit` — CVEs on submission look bad
- No cache — AI calls during live demo can time out
- Hardcoded secrets in repo — instant disqualification at serious hackathons
- Generic pitch ("we built an AI app") vs specific ("10 seconds vs 4 hours")

---

## Time Budget for a 6-Hour Hackathon

| Time | Activity |
|------|----------|
| 0:00 – 0:30 | Read rubric, map criteria to features, pick stack |
| 0:30 – 1:30 | Bootstrap: project, types, error hierarchy, env, logger, API skeleton |
| 1:30 – 3:30 | Core feature: AI integration, caching, main API route |
| 3:30 – 4:30 | UI: form + results view, accessibility pass |
| 4:30 – 5:00 | Tests: cache, sanitize, errors, rate limit, 1 component |
| 5:00 – 5:20 | Deploy, `npm audit fix`, health endpoint |
| 5:20 – 5:40 | Polish: loading states, error states, "⚡ Cached" badge |
| 5:40 – 6:00 | Rehearse pitch: 3 run-throughs out loud |
