# Frontend Testing Rules

Tests use **Vitest** with **jsdom** environment and **@testing-library/react**. Config lives in `frontend/vite.config.ts` (under `test`); global setup is `frontend/src/test/setup.ts`.

## Running Tests

```bash
cd frontend && nvm use && pnpm test -- --run src/path/to/file.test.tsx
```

- Use `--run` to exit after a single pass (no watch mode).
- To filter by test name: `pnpm test -- --run -t "test name substring"`.
- Tests live in `__tests__/` directories beside the code they test.

## Fake Timers

**Any test that depends on real time (`setTimeout`, `setInterval`, `Date.now`) must use fake timers.** Never let timers run against wall-clock time — it makes tests slow and non-deterministic.

```ts
// Compute once at module scope — relative to wall clock, floored to a second boundary
const FAKE_NOW = new Date(Math.floor(Date.now() / 1000) * 1000);

// ✅ GOOD — pinned to a second boundary, relative to current date (won't go stale)
vi.useFakeTimers({ now: FAKE_NOW });

// ❌ BAD — hardcoded date goes stale as time passes
vi.useFakeTimers({ now: new Date('2025-06-01T00:00:00.000Z') });

// ❌ BAD — leaves Date.now() coupled to wall clock with sub-second drift
vi.useFakeTimers(); // without `now`
```

### Advancing Time

Use `vi.advanceTimersByTimeAsync()` wrapped in `act()` to flush React state updates triggered by timers:

```ts
await act(async () => { await vi.advanceTimersByTimeAsync(5000); });
```

**Do not use** `waitFor` when fake timers are active — `waitFor` polls on real-time intervals that never fire under fake timers, causing test timeouts.

### Dynamic Mocks for Time-Dependent Values

When the code under test reads `Date.now()` and a mock returns a value computed from time (e.g. a JWT `exp`), compute the mock's return value **at call time** so it stays relative to the fake clock:

```ts
// ✅ GOOD — token is always "10 min from fake-now" regardless of how far time advances
mockPost.mockImplementation(() => {
  const fakeSec = Math.floor(Date.now() / 1000);
  return Promise.resolve({
    access_token: createTestJwt({ exp: fakeSec + 600 }),
  });
});

// ❌ BAD — static value drifts as fake time advances, causing cascading side effects
const refreshedToken = createTestJwt({ exp: nowSec + 600 });
mockPost.mockResolvedValue({ access_token: refreshedToken });
```

### Timer Boundary Assertions

When asserting "timer hasn't fired yet" vs "timer has fired," use **wide margins** rather than exact millisecond edges. Off-by-one errors from microtask flushing make exact-edge tests flaky:

```ts
// ✅ GOOD — well before (8 min) and well after (10 min) a 9 min timer
await act(async () => { await vi.advanceTimersByTimeAsync(8 * 60 * 1000); });
expect(callback).not.toHaveBeenCalled();
await act(async () => { await vi.advanceTimersByTimeAsync(2 * 60 * 1000); });
expect(callback).toHaveBeenCalled();

// ❌ BAD — relies on exact ms boundary
await act(async () => { await vi.advanceTimersByTimeAsync(539_999); });
expect(callback).not.toHaveBeenCalled();
await act(async () => { await vi.advanceTimersByTimeAsync(2); });
expect(callback).toHaveBeenCalled();
```

### Cleanup

Always restore real timers in `afterEach`. If mock spies were installed, pair with `vi.restoreAllMocks()` (typically in an outer `afterEach`):

```ts
afterEach(() => {
  vi.clearAllTimers();
  vi.useRealTimers();
});

// In the outer describe's afterEach (if spies are used):
afterEach(() => {
  vi.restoreAllMocks();
});
```

## Mocking

### Module Mocks

Use `vi.mock()` at the top level. For partial mocks, spread `vi.importActual`. Capture mock functions in stable `const` references so tests can assert on them:

```ts
const mockNavigate = vi.fn();

vi.mock('react-router-dom', async () => {
  const actual = await vi.importActual('react-router-dom');
  return { ...actual, useNavigate: () => mockNavigate };
});
```

### API Mocks

Mock the axios instance from `@/api/api`. Include interceptors in the mock shape since `AuthContext` registers them on mount:

```ts
vi.mock('../../api/api', () => ({
  api: {
    interceptors: {
      request: { use: vi.fn(), eject: vi.fn() },
      response: { use: vi.fn(), eject: vi.fn() },
    },
    post: vi.fn(),
  },
}));
```

### localStorage

Call `localStorage.clear()` in `beforeEach` to prevent leakage between tests. Vitest's jsdom environment provides a working `localStorage` — no need to mock it.

### Reset Order

```ts
beforeEach(() => {
  vi.clearAllMocks();
  localStorage.clear();
});
```

## Async Rendering

- Use `act()` to wrap any operation that triggers React state updates (clicks, timer flushes, promise resolution).
- Use `waitFor()` only when waiting for **real async work** (e.g., a resolved promise to re-render). Never use `waitFor` with fake timers.
- Prefer `screen.getByTestId` / `screen.getByText` for assertions. Use `queryBy*` when asserting absence.

## Test Structure

- Test files go in a `__tests__/` directory beside the module they test (e.g., `contexts/__tests__/AuthContext.test.tsx`).
- Name files `<ModuleName>.test.tsx` (`.ts` for non-component logic).
- Group related tests with `describe` blocks. Use descriptive `it` strings that state the expected behavior, not the implementation.
- Each test should be independent — no shared mutable state between `it` blocks beyond what `beforeEach` resets.

## E2E Tests (Playwright)

E2e tests live in `frontend/e2e/` and use **Playwright**. They run against the full stack via Docker Compose.

### Running E2E Tests

1. **Start the e2e containers** (from the repo root):
   ```bash
   cd frontend && pnpm test:e2e:setup
   ```
   This runs `docker compose -f docker-compose.e2e.yml up -d` from the repo root.

2. **Run the tests** (from `frontend/`):
   ```bash
   cd frontend && nvm use && pnpm test:e2e
   ```

3. **Tear down** when done:
   ```bash
   cd frontend && pnpm test:e2e:teardown
   ```

### Workflow for Agents

After updating e2e test code, run the e2e suite **in the background** using the Agent tool (`run_in_background: true`) so the main context can continue with other work. The containers must be running before the tests start — use `pnpm test:e2e:setup` first, then `pnpm test:e2e`.

## Test Components

When testing context providers or hooks, create minimal test components that expose the values you need via `data-testid` attributes:

```ts
const TestComponent: React.FC = () => {
  const { token, isAuthenticated } = useAuth();
  return (
    <div>
      <div data-testid="token">{token || 'null'}</div>
      <div data-testid="isAuthenticated">{isAuthenticated.toString()}</div>
    </div>
  );
};
```

Keep test components as thin as possible — they exist only to bridge hooks/context to assertable DOM.
