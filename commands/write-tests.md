---
description: Write unit + functional tests for a ticket — targets 90% AC coverage
argument-hint: <repo-path> <ticket-id>  e.g. /Users/sudheer7781/Documents/pw-server-v3 PW-123
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, mcp__claude_ai_Atlassian__getJiraIssue
---

You are the test writer for PossibleWorks engineering.

Input: $ARGUMENTS
Parse: REPO_PATH = first arg, TICKET_ID = second arg

**Goal:** Write both unit tests and functional tests for the ticket. Target ≥ 90% of ACs covered by automated tests.

---

## STEP 1 — Fetch ticket ACs + detect framework

Run both in parallel:

**1a — Fetch ACs:**
```
Tool: mcp__claude_ai_Atlassian__getJiraIssue
issueIdOrKey: "<TICKET-ID>"
```
Extract every AC item. Number them AC-1, AC-2 ... AC-N.

**1b — Detect framework:**
```bash
cd <REPO_PATH>
cat package.json | grep -E '"test"|jest|vitest|mocha|@testing-library|supertest'
find src -name "*.spec.ts" -o -name "*.spec.tsx" -o -name "*.test.ts" -o -name "*.test.tsx" 2>/dev/null | head -5
ls node_modules/socket.io-client 2>/dev/null && echo "socket.io-client: available" || echo "socket.io-client: not found"
```

Display:
```
─── Setup ───────────────────────────────────────────
  Repo:      <REPO_PATH>
  Framework: <jest | vitest | none>
  Existing specs: <count>
  socket.io-client: <available | not found>
  ACs: N total → target ≥ ceil(N × 0.9) automated

  AC-1: <text>
  AC-2: <text>
  ...
─────────────────────────────────────────────────────
```

---

## STEP 2 — Classify changed files (this feature only)

Tests are written **only for files changed in this feature branch** — not the whole repo.

```bash
cd <REPO_PATH>
git diff --name-only HEAD   # files changed vs base branch
git status --short          # untracked new files added in this branch
```

This gives the exact set of files touched by this ticket. Classify each changed file into what kind of test it needs:

| File | Unit test | Functional test |
|------|-----------|----------------|
| `src/services/*.ts` | ✅ mock Prisma, test methods | ✅ `node:test` + `fetch` against real server |
| `src/controllers/*.ts` | ✅ if supertest available | ✅ `node:test` + `fetch` |
| `src/routes/*.ts` | ⬜ skip | ✅ `node:test` + `fetch` |
| `src/components/**/*.tsx` | ✅ if @testing-library available | ✅ interaction test |
| `src/hooks/*.ts` | ✅ if testing-library available | ✅ socket event flow |
| `src/constants/**` or `src/models/enums/**` | ⬜ skip | ⬜ skip |
| `prisma/schema.prisma` | ⬜ skip | ⬜ skip |

Display:
```
─── File Classification ─────────────────────────────
  Unit tests needed:
  - <file> → <what to test>

  Functional tests needed:
  - <file> → <endpoint or event to test>

  Skip:
  - <file> → constants/schema only
─────────────────────────────────────────────────────
```

---

## STEP 3 — Unit tests

### If framework EXISTS (jest / vitest / mocha):

For each file needing unit tests:

1. Read the changed file fully — understand every new public method/component
2. Find the nearest existing spec:
   ```bash
   find src/services -name "*.spec.ts" | head -1    # for services
   find src/hooks    -name "*.spec.ts" | head -1    # for hooks
   find src          -name "*.spec.tsx" | head -1   # for components
   ```
3. Read that spec — mirror its exact import style, describe/it structure, and mock patterns

**Write rules per file type:**

*Backend services* → `src/services/<Name>Service.spec.ts`:
- Mock Prisma exactly as the nearest existing spec does
- For each NEW public method, write 3 tests:
  1. Happy path — valid input → expected return value
  2. Edge case — null/undefined → safe return or expected error
  3. Error path — service throws → error propagates
- Label each test: `// AC-N: <AC text>`
- Never test private methods

*Frontend hooks* → `src/hooks/use<Name>.spec.ts`:
- Mock socket using nearest hook spec pattern
- Test: initial state correct
- Test: socket event received → state updates
- Test: cleanup — socket.off() called on unmount

*Frontend components* → `src/components/molecules/<Name>/index.spec.tsx`:
- Use `@testing-library/react`
- Test: renders without crashing
- Test: correct text/labels from props
- Test: key interaction triggers expected callback

Run unit tests:
```bash
cd <REPO_PATH>
yarn test --testPathPattern="<feature-name>" 2>&1 | tail -20
```
Fix ALL failures before moving on. Never skip or comment out.

### If NO framework:
Skip unit tests. Move directly to STEP 4 (functional tests cover the gap).

---

## STEP 4 — Functional tests (always — regardless of unit test framework)

Functional tests call the **real running server** and verify the feature works end-to-end. Uses only built-in tools — no new packages needed (Node 25 has native `fetch` + `node:test`).

### 4A — REST API functional tests

For each new/changed REST endpoint, write:
`src/tests/<ticket-id>-<feature>.functional.ts`

```typescript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

const BASE   = process.env.TEST_BASE_URL  ?? 'http://localhost:3000/api/v1';
const TOKEN  = process.env.TEST_AUTH_TOKEN ?? '';
const TENANT = process.env.TEST_TENANT_ID  ?? '';

const h = {
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${TOKEN}`,
  'Tenantid': TENANT,
};

// AC-1: <AC text>
describe('<Feature> REST — functional', () => {

  it('POST /<endpoint> — happy path returns 201 with correct shape', async () => {
    const res = await fetch(`${BASE}/<endpoint>`, {
      method: 'POST', headers: h,
      body: JSON.stringify({ /* minimum valid payload — read from controller */ }),
    });
    assert.equal(res.status, 201);
    const body = await res.json();
    assert.ok(body.data?.id, 'created record must have an id');
  });

  // AC-2: <AC text>
  it('GET /<endpoint>/:id — returns the created record', async () => {
    const res = await fetch(`${BASE}/<endpoint>/<test-id>`, { headers: h });
    assert.equal(res.status, 200);
    const body = await res.json();
    assert.equal(body.data.<field>, <expectedValue>);
  });

  it('POST /<endpoint> — missing required field returns 400', async () => {
    const res = await fetch(`${BASE}/<endpoint>`, {
      method: 'POST', headers: h,
      body: JSON.stringify({ /* empty or missing required field */ }),
    });
    assert.equal(res.status, 400);
  });

  it('POST /<endpoint> — no auth returns 401', async () => {
    const res = await fetch(`${BASE}/<endpoint>`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ /* any */ }),
    });
    assert.equal(res.status, 401);
  });

});
```

**Before writing** — read the controller to confirm: route path, required body fields, response shape.
**Label every test** with `// AC-N:` comment.

### 4B — Socket event functional tests (if socket.io-client available)

For each new socket event, write:
`src/tests/<ticket-id>-<feature>.socket.ts`

```typescript
import { describe, it, before, after } from 'node:test';
import assert from 'node:assert/strict';
import { io, Socket } from 'socket.io-client';

const SERVER = process.env.TEST_SERVER_URL  ?? 'http://localhost:3000';
const TOKEN  = process.env.TEST_AUTH_TOKEN  ?? '';
const TENANT = process.env.TEST_TENANT_ID   ?? '';

// AC-N: <AC text>
describe('<EventName> socket — functional', () => {
  let socket: Socket;

  before(() => new Promise<void>((resolve) => {
    socket = io(SERVER, {
      auth: { token: TOKEN },
      extraHeaders: { tenantid: TENANT },
    });
    socket.on('connect', resolve);
  }));

  after(() => { socket.disconnect(); });

  it('emitting <event_name> with valid payload → success callback', () =>
    new Promise<void>((resolve, reject) => {
      const t = setTimeout(() => reject(new Error('No response in 3s')), 3000);
      socket.emit('<event_name>',
        { /* exact payload per cross-repo contract */ },
        (res: { status: string }) => {
          clearTimeout(t);
          assert.equal(res.status, 'success');
          resolve();
        }
      );
    })
  );

  // AC-N: server broadcasts to client
  it('after <event_name>, server broadcasts <response_event>', () =>
    new Promise<void>((resolve, reject) => {
      const t = setTimeout(() => reject(new Error('No broadcast in 3s')), 3000);
      socket.once('<response_event>', (data) => {
        clearTimeout(t);
        assert.ok(data.<expectedField>);
        resolve();
      });
      socket.emit('<event_name>', { /* payload */ }, () => {});
    })
  );

});
```

**Before writing** — read the socket handler to confirm event name, payload shape, callback format, and broadcast event name.

### 4C — Always extend pw-api.rest

Regardless of automated tests, append new entries to `pw-api.rest` for every new endpoint:

```http
# @name <TicketID> — <AC text>
<METHOD> {{baseUrl}}/<path> HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>
Tenantid: <tenantId>

{
  "field": "value"
}
###
```

This enables one-click manual re-testing in VS Code for QA and future devs.

### 4D — Run functional tests

```bash
cd <REPO_PATH>
export TEST_BASE_URL=http://localhost:3000/api/v1
export TEST_AUTH_TOKEN=<token from pw-api.rest>
export TEST_TENANT_ID=<tenantId from pw-api.rest>
export TEST_SERVER_URL=http://localhost:3000

node --test src/tests/<ticket-id>-<feature>.functional.ts 2>&1
node --test src/tests/<ticket-id>-<feature>.socket.ts 2>&1
```

If server not running — report clearly:
```
⚠️  Local server not running. To execute:
    cd <REPO_PATH> && yarn dev
    Then: node --test src/tests/<ticket-id>-<feature>.functional.ts
```
Do NOT silently skip the run.

---

## STEP 5 — Coverage report + summary

Calculate:
- Total ACs: N
- ACs with automated test (unit or functional): X
- ACs manual-only (browser/Frappe-live required): Y
- **Coverage = X / N × 100%**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Test Coverage — <TICKET-ID>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Unit Tests:
  AC-1: ✅  src/services/<Name>Service.spec.ts:12
  AC-2: ✅  src/hooks/use<Name>.spec.ts:34

  Functional Tests:
  AC-3: ✅  src/tests/<ticket-id>-<feature>.functional.ts:18
  AC-4: ✅  src/tests/<ticket-id>-<feature>.socket.ts:22
  AC-5: 🖐  manual — UI browser-only (cannot automate headlessly)

  Coverage: X/N = XX%  [target ≥ 90%]

  Files written:
  - src/services/<Name>Service.spec.ts        (unit, X tests)
  - src/tests/<ticket-id>-<feature>.functional.ts  (functional, X tests)
  - src/tests/<ticket-id>-<feature>.socket.ts      (socket, X tests)
  - pw-api.rest                                (X new entries)

  Run results:
  Unit:       ✅ X passed  ❌ 0 failed
  Functional: ✅ X passed  ❌ 0 failed  (or ⚠️ server not running)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If coverage < 90% and remaining ACs CAN be automated — write the missing tests before finishing.
If coverage < 90% and remaining ACs CANNOT be automated — document exactly why (browser-only, live Frappe required, etc.) and add specific manual steps.

---

## HARD RULES

- **Target ≥ 90% AC coverage** — both unit + functional tests count toward the total
- **Functional tests always written** — even if unit framework exists; they serve a different purpose
- **Never install new packages** — use `node:test`, native `fetch`, `socket.io-client` from existing node_modules
- **Never mock in functional tests** — they call the real server; mocking defeats the purpose
- **Always extend pw-api.rest** — every new endpoint gets an entry, no exceptions
- **Read before writing** — always read the service/controller/handler before writing tests to get real field names
- **Label every test with its AC** — `// AC-N: <text>` above each `it()` block
- **Socket tests must have 3s timeout** — always reject after 3 seconds; no hanging tests
- **Never snapshot test** — brittle and useless for this codebase
- **Never test private methods** — test behaviour and output only
- **Never skip or comment out failing tests** — fix the root cause
