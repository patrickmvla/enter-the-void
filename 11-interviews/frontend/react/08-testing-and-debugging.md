# React Testing & Debugging — What Interviewers Expect

## Testing Philosophy

React Testing Library's guiding principle:

> "The more your tests resemble the way your software is used,
> the more confidence they give you."

Test **behavior**, not implementation. Don't test state values
or internal methods — test what the user sees and does.

---

## Testing Pyramid for React

```
        ╱╲
       ╱ E2E ╲        Few, slow, high confidence
      ╱────────╲       (Playwright / Cypress)
     ╱Integration╲    Medium amount, test flows
    ╱──────────────╲   (RTL + MSW)
   ╱   Unit Tests   ╲ Many, fast, test logic
  ╱──────────────────╲ (Vitest / Jest)
```

---

## Unit Testing Components (React Testing Library)

```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Counter } from './Counter';

test('increments count on button click', () => {
  render(<Counter />);

  const button = screen.getByRole('button', { name: /increment/i });
  expect(screen.getByText('Count: 0')).toBeInTheDocument();

  fireEvent.click(button);
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

### Query priority (use in this order):

```
1. getByRole        — accessible role (button, heading, textbox)
2. getByLabelText   — form fields by their label
3. getByPlaceholder — fallback for form fields
4. getByText        — non-interactive elements
5. getByDisplayValue — current value of form elements
6. getByAltText     — images
7. getByTitle       — title attribute
8. getByTestId      — last resort (data-testid)
```

**getBy** throws if not found. **queryBy** returns null. **findBy** waits (async).

---

## Testing Async Components

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import { UserProfile } from './UserProfile';

test('displays user data after loading', async () => {
  render(<UserProfile userId="1" />);

  // Initially shows loading state
  expect(screen.getByText(/loading/i)).toBeInTheDocument();

  // Wait for data to appear
  const userName = await screen.findByText('Alice');
  expect(userName).toBeInTheDocument();

  // Loading state should be gone
  expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
});
```

---

## Mocking API Calls (MSW)

Mock Service Worker intercepts at the network level:

```tsx
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('/api/users/1', () => {
    return HttpResponse.json({ id: 1, name: 'Alice' });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('handles server error', async () => {
  // Override for this test
  server.use(
    http.get('/api/users/1', () => {
      return new HttpResponse(null, { status: 500 });
    })
  );

  render(<UserProfile userId="1" />);
  expect(await screen.findByText(/error/i)).toBeInTheDocument();
});
```

MSW is preferred over jest.mock/vi.mock because:
- Tests your actual fetch/axios code
- Works in both browser and Node
- Tests the full request/response cycle

---

## Testing Custom Hooks

```tsx
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

test('increments and decrements', () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);

  act(() => result.current.decrement());
  expect(result.current.count).toBe(0);
});
```

---

## Testing Components with Context/Providers

```tsx
function renderWithProviders(ui, { theme = 'light', ...options } = {}) {
  function Wrapper({ children }) {
    return (
      <ThemeContext.Provider value={theme}>
        <QueryClientProvider client={new QueryClient()}>
          {children}
        </QueryClientProvider>
      </ThemeContext.Provider>
    );
  }

  return render(ui, { wrapper: Wrapper, ...options });
}

test('renders with dark theme', () => {
  renderWithProviders(<ThemeToggle />, { theme: 'dark' });
  expect(screen.getByText('Dark mode')).toBeInTheDocument();
});
```

---

## Snapshot Testing

```tsx
test('renders correctly', () => {
  const { container } = render(<Button label="Click me" />);
  expect(container).toMatchSnapshot();
});
```

**Pros:** Catches unintended UI changes.
**Cons:** Large snapshots become noise. Updated blindly with `-u`.

Use sparingly — for small, stable components. Prefer explicit assertions.

---

## Testing with userEvent (over fireEvent)

```tsx
import userEvent from '@testing-library/user-event';

test('types in search input', async () => {
  const user = userEvent.setup();
  render(<SearchBar onSearch={mockFn} />);

  const input = screen.getByRole('searchbox');
  await user.type(input, 'react hooks');

  expect(input).toHaveValue('react hooks');
});
```

`userEvent` simulates real browser events (focus, keydown, keypress,
keyup, input, change). `fireEvent` dispatches a single synthetic event.
`userEvent` is more realistic.

---

## Debugging React

### React DevTools

- **Components tab:** inspect component tree, props, state, hooks
- **Profiler tab:** record renders, see what re-rendered and why
- **Highlight updates:** visual flash when components re-render

### why-did-you-render

```tsx
// Logs unnecessary re-renders to console
import whyDidYouRender from '@welldone-software/why-did-you-render';
whyDidYouRender(React);

MyComponent.whyDidYouRender = true;
```

### Common debugging strategies

```
"Component re-renders too often"
  → Profiler → check "Why did this render?"
  → Usually: new object/array/function props from parent

"State update not working"
  → Check if mutating state directly (objects/arrays)
  → Check if using stale closure in useEffect

"useEffect runs in infinite loop"
  → Dependency creates new reference each render
  → Object/array in deps without useMemo

"Hydration mismatch"
  → Client renders different content than server
  → Usually: Date, window, localStorage in render
```

---

## E2E Testing (Playwright)

```tsx
import { test, expect } from '@playwright/test';

test('user can create a post', async ({ page }) => {
  await page.goto('/dashboard');

  await page.getByRole('link', { name: 'New Post' }).click();
  await page.getByLabel('Title').fill('My First Post');
  await page.getByLabel('Content').fill('Hello world');
  await page.getByRole('button', { name: 'Publish' }).click();

  await expect(page.getByText('Post published')).toBeVisible();
  await expect(page).toHaveURL(/\/posts\/.+/);
});
```

---

## Interview Questions

---

**Q: What's the difference between getBy, queryBy, and findBy?**

**How to answer:** Define each variant, its return behavior, and when
to use it. Show that you know the async one is for loading states.

**Answer:** React Testing Library provides three variants for every query:

**`getBy`** — **synchronous, throws on failure.** Use when the element
should be in the DOM right now. If it's not found, the test fails
immediately with a helpful error showing the current DOM:
```jsx
const button = screen.getByRole('button', { name: /submit/i });
// Element MUST exist. Throws if not found.
```

**`queryBy`** — **synchronous, returns null on failure.** Use when
asserting an element is **NOT** in the DOM:
```jsx
expect(screen.queryByText('Error')).not.toBeInTheDocument();
// Returns null instead of throwing — perfect for asserting absence
```

**`findBy`** — **asynchronous, waits and retries.** Use for elements
that appear after an async operation (data fetch, timer, animation):
```jsx
const userName = await screen.findByText('Alice');
// Waits up to 1000ms (configurable), retrying until found or timeout
// Uses MutationObserver under the hood
```

**The pattern in practice:**
```jsx
test('loads and displays user', async () => {
  render(<UserProfile />);

  // 1. getBy: assert loading state exists immediately
  expect(screen.getByText('Loading...')).toBeInTheDocument();

  // 2. findBy: wait for async data to appear
  expect(await screen.findByText('Alice')).toBeInTheDocument();

  // 3. queryBy: assert loading state is gone
  expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
});
```

---

**Q: How do you test a component that fetches data?**

**How to answer:** Walk through the full approach — mock setup, render,
assert loading, assert data, test error case.

**Answer:** I use **Mock Service Worker (MSW)** to intercept network
requests at the network level. This tests the actual fetch code (not
mocked modules) and works with any HTTP client (fetch, axios, etc.).

**Step 1: Set up the mock server:**
```jsx
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('/api/users/1', () => {
    return HttpResponse.json({ id: 1, name: 'Alice', role: 'admin' });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());  // reset any per-test overrides
afterAll(() => server.close());
```

**Step 2: Test the happy path:**
```jsx
test('displays user data after loading', async () => {
  render(<UserProfile userId="1" />);

  // Loading state
  expect(screen.getByText(/loading/i)).toBeInTheDocument();

  // Data appears (async)
  expect(await screen.findByText('Alice')).toBeInTheDocument();
  expect(screen.getByText('admin')).toBeInTheDocument();

  // Loading gone
  expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
});
```

**Step 3: Test the error path:**
```jsx
test('displays error on server failure', async () => {
  server.use(
    http.get('/api/users/1', () => {
      return new HttpResponse(null, { status: 500 });
    })
  );

  render(<UserProfile userId="1" />);
  expect(await screen.findByText(/error/i)).toBeInTheDocument();
});
```

**Why MSW over jest.mock:** MSW intercepts at the network layer, so your
actual fetch/axios code runs. If you switch from fetch to axios, tests
still work. jest.mock replaces the module, so you're not testing the
real code path.

---

**Q: What's the testing query priority?**

**How to answer:** List the priority order and explain the reasoning
(accessibility first).

**Answer:** React Testing Library recommends this query priority,
designed to mirror how users and assistive technology interact with the UI:

```
1. getByRole       — "button", "heading", "textbox"
2. getByLabelText  — form fields associated with a <label>
3. getByPlaceholderText — fallback for unlabeled inputs
4. getByText       — non-interactive visible text
5. getByDisplayValue — current value of input/select
6. getByAltText    — images
7. getByTitle      — title attribute (less accessible)
8. getByTestId     — data-testid attribute (LAST RESORT)
```

**The philosophy:** If you can't find an element with getByRole, your
component probably has an accessibility problem. A button without a
role is just a div — it won't work with keyboards or screen readers.

```jsx
// GOOD: accessible and testable
<button aria-label="Close dialog">×</button>
screen.getByRole('button', { name: /close dialog/i });

// BAD: only testable via testId, not accessible
<div data-testid="close-btn" onClick={onClose}>×</div>
screen.getByTestId('close-btn');
```

Using getByRole as the default means your **tests enforce accessibility**.
If you find yourself reaching for getByTestId, ask: "Why can't I find
this by role? Is the component accessible?"

**The killer insight:** The query priority isn't just a testing convention
— it's an **accessibility audit**. If you can't find an element by role,
your component fails WCAG. A `<div onClick={...}>` can't be found by
getByRole('button') — and it can't be used by keyboard users or screen
readers either. The test failure IS the a11y bug.

**Gotcha with getBy vs queryBy:** Use `getBy` when asserting presence
(throws if missing — clear error). Use `queryBy` when asserting
**absence** (`queryByText('loading')` returns null; `getByText('loading')`
would throw and fail the test). This is the most common RTL mistake — using
getBy to check something doesn't exist.

---

**Q: Why prefer userEvent over fireEvent?**

**How to answer:** Explain the difference in how they simulate events,
and give a concrete example where fireEvent misses a bug.

**Answer:** `fireEvent` dispatches a single DOM event directly.
`userEvent` simulates the **full sequence of events** a real user action
would produce.

When a real user types "a" into an input, the browser fires:
```
focus → keydown → keypress → beforeinput → input → keyup
```

`fireEvent.change(input, { target: { value: 'a' } })` skips all of that
and directly triggers a change event. `userEvent.type(input, 'a')` fires
the entire sequence.

**Where this matters:**

```jsx
function PhoneInput({ onChange }) {
  function handleKeyDown(e) {
    // Only allow digits
    if (!/\d/.test(e.key) && e.key !== 'Backspace') {
      e.preventDefault();
    }
  }
  return <input onKeyDown={handleKeyDown} onChange={onChange} />;
}
```

- `fireEvent.change(input, { target: { value: 'abc' } })` — **passes**
  the test because it skips keyDown entirely. The validation never runs.
- `userEvent.type(input, 'abc')` — **correctly blocks** the input because
  it fires keyDown for each character, and preventDefault stops non-digits.

Use `userEvent.setup()` at the start of the test for the best behavior:
```jsx
const user = userEvent.setup();
await user.click(button);
await user.type(input, 'hello');
```

---

**Q: How do you test components that use context?**

**How to answer:** Show the wrapper pattern and explain why it's
reusable across your test suite.

**Answer:** Create a custom render function that wraps the component in
the required providers. This is reusable across all tests:

```jsx
// test-utils.tsx
function AllProviders({ children, initialState = {} }) {
  return (
    <ThemeContext.Provider value={initialState.theme ?? 'light'}>
      <AuthContext.Provider value={initialState.user ?? null}>
        <QueryClientProvider client={new QueryClient()}>
          {children}
        </QueryClientProvider>
      </AuthContext.Provider>
    </ThemeContext.Provider>
  );
}

function renderWithProviders(ui, { initialState = {}, ...options } = {}) {
  return render(ui, {
    wrapper: (props) => <AllProviders {...props} initialState={initialState} />,
    ...options,
  });
}

export { renderWithProviders as render };
```

```jsx
// In tests:
import { render } from './test-utils';

test('shows admin panel for admin users', () => {
  render(<Dashboard />, {
    initialState: { user: { role: 'admin' } }
  });
  expect(screen.getByText('Admin Panel')).toBeInTheDocument();
});

test('hides admin panel for regular users', () => {
  render(<Dashboard />, {
    initialState: { user: { role: 'viewer' } }
  });
  expect(screen.queryByText('Admin Panel')).not.toBeInTheDocument();
});
```

This pattern lets you control context values per test and is the
recommended approach from the Testing Library docs.

---

**Q: When would you use snapshot testing?**

**How to answer:** Give the narrow use case, then explain why it's
overused and when to avoid it.

**Answer:** Snapshot testing captures the rendered output of a component
and compares it against a saved file. On subsequent runs, if the output
changes, the test fails.

**Good use cases:**
- **Small, stable UI components** — icons, badges, simple cards. Components
  that rarely change and where you want to catch unintended visual regressions.
- **Serializable data structures** — API response shapes, config objects.

**When to avoid (most of the time):**
- **Large components** — a 500-line snapshot is unreadable. Reviewers
  blindly approve `--updateSnapshot` without checking the diff.
- **Frequently changing components** — every feature update requires
  updating snapshots, which becomes noise in PRs.
- **Components with dynamic content** — dates, random IDs, or user data
  cause snapshots to break even though the behavior is correct.

**Better alternatives:**
```jsx
// Instead of a snapshot, make specific assertions:
test('renders user card', () => {
  render(<UserCard user={mockUser} />);
  expect(screen.getByText('Alice')).toBeInTheDocument();
  expect(screen.getByRole('img')).toHaveAttribute('src', '/alice.jpg');
  expect(screen.getByText('Admin')).toHaveClass('badge-admin');
});
```

Explicit assertions document **what matters** about the component.
Snapshots document **everything**, making it hard to tell what's
intentional vs incidental.

**The fundamental problem with snapshots:** They're **regression detection
without understanding**. When a snapshot fails, the developer reads a
500-line diff, can't tell if the change is intentional, and runs
`--updateSnapshot`. In code review, the reviewer sees "updated 47
snapshots" and approves without checking. This is not testing — it's
a rubber stamp that creates false confidence.

**When snapshots ARE good:**
- **Serializable data structures** — API response shapes, GraphQL query
  shapes, config objects. Small, stable, meaningful diffs.
- **Small, leaf components** — an icon SVG, a badge. The entire output
  is 3 lines and changes are obvious.

**When snapshots are harmful:**
- **Any component with dynamic content** — dates, user data, random IDs
- **Components that change frequently** — every feature adds snapshot noise
- **Large components** — 100+ line snapshots are unreadable diffs

Use explicit assertions by default. Reach for snapshots only when the
output IS the contract (data shapes, SVG icons).
