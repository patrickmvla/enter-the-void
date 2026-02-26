# Accessibility & Forms — What Senior Devs Must Know

## Why a11y Matters in Interviews

Accessibility isn't a "nice to have" — it's a legal requirement (ADA,
WCAG) and a signal that you build for all users. Senior candidates are
expected to know this.

---

## Semantic HTML First

```tsx
// BAD: div soup
<div onClick={handleClick} className="button">Click me</div>
<div className="nav">
  <div className="nav-item">Home</div>
</div>

// GOOD: semantic elements
<button onClick={handleClick}>Click me</button>
<nav>
  <a href="/">Home</a>
</nav>
```

Semantic elements give you **for free**:
- Keyboard navigation (button is focusable, responds to Enter/Space)
- Screen reader announcements ("button, Click me")
- Proper role in accessibility tree

---

## ARIA Roles & Attributes

Use ARIA only when semantic HTML isn't enough:

```tsx
// Custom dropdown (no native <select>)
<div
  role="listbox"
  aria-label="Choose a color"
  aria-activedescendant={selectedId}
>
  <div role="option" id="red" aria-selected={selected === 'red'}>
    Red
  </div>
  <div role="option" id="blue" aria-selected={selected === 'blue'}>
    Blue
  </div>
</div>
```

### Key ARIA attributes:

```
aria-label         — accessible name (when visible text isn't enough)
aria-labelledby    — points to another element's ID for naming
aria-describedby   — additional description
aria-expanded      — dropdown/accordion open state
aria-hidden        — hide from screen readers
aria-live          — announce dynamic changes ("polite" or "assertive")
aria-required      — form field is required
aria-invalid       — form field has validation error
role               — override semantic role
```

---

## Keyboard Navigation

All interactive elements must be keyboard accessible:

```tsx
function Dropdown({ items, onSelect }) {
  const [isOpen, setIsOpen] = useState(false);
  const [focusIndex, setFocusIndex] = useState(-1);

  function handleKeyDown(e: React.KeyboardEvent) {
    switch (e.key) {
      case 'Enter':
      case ' ':
        if (isOpen && focusIndex >= 0) {
          onSelect(items[focusIndex]);
          setIsOpen(false);
        } else {
          setIsOpen(true);
        }
        break;
      case 'ArrowDown':
        e.preventDefault();
        setFocusIndex(i => Math.min(i + 1, items.length - 1));
        break;
      case 'ArrowUp':
        e.preventDefault();
        setFocusIndex(i => Math.max(i - 1, 0));
        break;
      case 'Escape':
        setIsOpen(false);
        break;
    }
  }

  return (
    <div onKeyDown={handleKeyDown}>
      <button
        aria-expanded={isOpen}
        aria-haspopup="listbox"
        onClick={() => setIsOpen(!isOpen)}
      >
        Select item
      </button>
      {isOpen && (
        <ul role="listbox">
          {items.map((item, i) => (
            <li
              key={item.id}
              role="option"
              aria-selected={i === focusIndex}
              tabIndex={i === focusIndex ? 0 : -1}
            >
              {item.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## Focus Management

### Focus trapping (modals)

```tsx
function Modal({ isOpen, onClose, children }) {
  const modalRef = useRef(null);

  useEffect(() => {
    if (!isOpen) return;

    const modal = modalRef.current;
    const focusable = modal.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    first?.focus();

    function trapFocus(e) {
      if (e.key === 'Tab') {
        if (e.shiftKey && document.activeElement === first) {
          e.preventDefault();
          last.focus();
        } else if (!e.shiftKey && document.activeElement === last) {
          e.preventDefault();
          first.focus();
        }
      }
      if (e.key === 'Escape') onClose();
    }

    modal.addEventListener('keydown', trapFocus);
    return () => modal.removeEventListener('keydown', trapFocus);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div role="dialog" aria-modal="true" ref={modalRef}>
      {children}
      <button onClick={onClose}>Close</button>
    </div>,
    document.getElementById('modal-root')
  );
}
```

### Restoring focus

When a modal closes, focus should return to the element that opened it:

```tsx
const triggerRef = useRef(null);

function openModal() {
  triggerRef.current = document.activeElement;
  setIsOpen(true);
}

function closeModal() {
  setIsOpen(false);
  triggerRef.current?.focus();  // restore focus
}
```

---

## Live Regions (Dynamic Content)

```tsx
// Screen reader announces when status changes
<div aria-live="polite" aria-atomic="true">
  {status === 'success' && 'Form submitted successfully!'}
  {status === 'error' && 'Something went wrong. Please try again.'}
</div>
```

- `aria-live="polite"`: waits for user to finish current task
- `aria-live="assertive"`: interrupts immediately (use sparingly)

---

## Form Patterns

### Labels

```tsx
// Explicit label association
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// Implicit (wrapping)
<label>
  Email
  <input type="email" />
</label>
```

### Validation errors

```tsx
function FormField({ label, error, ...inputProps }) {
  const id = useId();
  const errorId = `${id}-error`;

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        aria-invalid={!!error}
        aria-describedby={error ? errorId : undefined}
        {...inputProps}
      />
      {error && (
        <span id={errorId} role="alert">
          {error}
        </span>
      )}
    </div>
  );
}
```

### Form with React Hook Form + accessibility

```tsx
function SignupForm() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
          {...register('email', { required: 'Email is required' })}
        />
        {errors.email && (
          <span id="email-error" role="alert">
            {errors.email.message}
          </span>
        )}
      </div>
      <button type="submit">Sign Up</button>
    </form>
  );
}
```

---

## Interview Questions

---

**Q: What makes a React component accessible?**

**How to answer:** Walk through the layers of accessibility, from
markup to interaction to testing. Show it's not just adding aria-*.

**Answer:** Accessibility is built in layers, from most fundamental to
most advanced:

**1. Semantic HTML (80% of the work):** Use the right element for the
job. A `<button>` gives you keyboard focus, Enter/Space activation, and
screen reader announcements for free. A `<div onClick>` gives you none
of that.

```jsx
// Accessible by default:
<button>, <a href>, <input>, <select>, <nav>, <main>, <header>, <form>
```

**2. ARIA attributes (when semantics aren't enough):** For custom widgets
(dropdowns, tabs, modals) where no native element exists:
```jsx
<div role="tablist">
  <button role="tab" aria-selected={true}>Tab 1</button>
</div>
```

**3. Keyboard navigation:** Every interactive element must be operable
with keyboard. This means managing focus order (tabIndex), arrow key
navigation in custom widgets, and Escape to close overlays.

**4. Focus management:** When modals open, focus must move into them.
When they close, focus must return to the trigger. Focus must be trapped
inside the modal (Tab shouldn't escape to the page behind).

**5. Visual:** Sufficient color contrast (WCAG AA: 4.5:1 for text),
visible focus indicators (never `outline: none` without a replacement),
and text that can be resized to 200%.

**6. Testing:** Use the axe accessibility checker in tests:
```jsx
import { axe } from 'jest-axe';
test('has no a11y violations', async () => {
  const { container } = render(<MyForm />);
  expect(await axe(container)).toHaveNoViolations();
});
```

---

**Q: How do you handle focus management in a modal?**

**How to answer:** Walk through the three requirements (trap, initial
focus, restore) with code.

**Answer:** Modal focus management has three requirements:

**1. Trap focus inside the modal.** When the user presses Tab, focus
should cycle through the modal's focusable elements and never escape to
the page behind it:

```jsx
function trapFocus(e) {
  if (e.key !== 'Tab') return;
  const focusable = modalRef.current.querySelectorAll(
    'button, input, select, textarea, a[href], [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0];
  const last = focusable[focusable.length - 1];

  if (e.shiftKey && document.activeElement === first) {
    e.preventDefault();
    last.focus();  // Shift+Tab from first → wrap to last
  } else if (!e.shiftKey && document.activeElement === last) {
    e.preventDefault();
    first.focus();  // Tab from last → wrap to first
  }
}
```

**2. Move focus into the modal on open.** Focus the first focusable
element (or the modal itself if it has `tabIndex={-1}`). Without this,
focus stays on the trigger behind the modal — screen reader users don't
know the modal opened.

**3. Restore focus on close.** When the modal closes, focus must return
to the element that triggered it. Store a reference to `document.activeElement`
before opening:

```jsx
const triggerRef = useRef(null);
function openModal() {
  triggerRef.current = document.activeElement;
  setIsOpen(true);
}
function closeModal() {
  setIsOpen(false);
  triggerRef.current?.focus();
}
```

**4. Close on Escape.** Standard expectation — pressing Escape should
close the modal. Also add `role="dialog"` and `aria-modal="true"` to
tell assistive technology this is a modal overlay.

**What most people get wrong:**

1. **Initial focus on the close button** — wrong. Focus the modal title
   or first meaningful content. The close button is a fallback action,
   not the primary intent.

2. **Building it yourself** — in production, use Radix UI, Headless UI, or
   React Aria. Custom modal focus management has dozens of edge cases:
   nested modals, scroll restoration, iOS Safari focus quirks, screen
   reader announcement timing. Libraries handle years of accumulated
   bug fixes. Your hand-rolled modal will break on mobile Safari.

3. **Forgetting `inert`** — the modern approach is adding `inert` to the
   content behind the modal instead of trapping focus. `inert` makes
   everything behind the modal non-interactive AND invisible to screen
   readers. Simpler than manually trapping Tab.

**But you should be able to explain the mechanics in an interview** — it
shows you understand the accessibility principles, not just the library API.

---

**Q: What's the difference between aria-label and aria-labelledby?**

**How to answer:** Define each, show when to use which, and mention the
priority order.

**Answer:** Both provide an **accessible name** for an element, but
they work differently:

**`aria-label`** — provides the name as a **string directly** on the
element. Use when there's no visible text that serves as a label:
```jsx
<button aria-label="Close dialog">×</button>
// Screen reader says: "Close dialog, button"
// The × symbol alone isn't descriptive enough
```

**`aria-labelledby`** — points to the **ID of another element** whose
visible text serves as the label. Use when visible text already exists:
```jsx
<h2 id="modal-title">Delete Account</h2>
<div role="dialog" aria-labelledby="modal-title">
  {/* Screen reader announces the dialog as "Delete Account" */}
</div>
```

**Priority rule:** If both are present, `aria-labelledby` wins. The
full naming priority is:
1. `aria-labelledby` (highest priority)
2. `aria-label`
3. `<label>` element (for form controls)
4. Visible text content (for buttons, links)
5. `title` attribute (lowest priority)

**Best practice:** Prefer `aria-labelledby` when visible text exists —
it keeps the label in sync with what sighted users see. Use `aria-label`
only when you need to provide a name that's different from or not present
in the visible UI.

---

**Q: How do you announce dynamic content to screen readers?**

**How to answer:** Explain aria-live regions, the polite vs assertive
distinction, and show a practical example.

**Answer:** When content updates dynamically (form submission result,
notification, error message), sighted users see the change, but screen
reader users don't — unless you use an **aria-live region**.

An aria-live region is a DOM element that, when its content changes,
automatically announces the new content to screen readers:

```jsx
<div aria-live="polite" aria-atomic="true">
  {message}  {/* when this changes, screen reader announces it */}
</div>
```

**`aria-live="polite"`** — waits until the screen reader finishes its
current announcement, then reads the new content. Use for:
- Form success messages ("Profile saved successfully")
- Status updates ("3 results found")
- Non-critical notifications

**`aria-live="assertive"`** — **interrupts** the current announcement
immediately. Use sparingly, only for:
- Critical errors ("Session expired, please log in again")
- Urgent alerts that need immediate attention

**`aria-atomic="true"`** — when content changes, read the **entire** region,
not just the part that changed. Important when the full message is
needed for context.

**Common pattern in React:**
```jsx
function FormWithFeedback() {
  const [status, setStatus] = useState('');

  async function handleSubmit(data) {
    try {
      await submitForm(data);
      setStatus('Form submitted successfully!');
    } catch {
      setStatus('Submission failed. Please try again.');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
      <div aria-live="polite" role="status">
        {status}
      </div>
    </form>
  );
}
```

**Important:** The aria-live element must be in the DOM **before** the
content changes. Don't conditionally render it — always render it, and
change its content.

---

**Q: What is the role of htmlFor in React?**

**How to answer:** Explain the HTML relationship it creates and why it
matters for both UX and accessibility.

**Answer:** `htmlFor` is React's version of HTML's `for` attribute (since
`for` is a reserved word in JavaScript). It creates an **explicit
association** between a `<label>` and a form control by matching the
label's `htmlFor` to the input's `id`:

```jsx
<label htmlFor="email">Email address</label>
<input id="email" type="email" />
```

This association does two things:

**1. UX improvement:** Clicking the label focuses/activates the input.
For checkboxes and radios, this makes the clickable target much larger —
the entire label text becomes clickable, not just the tiny checkbox.

**2. Accessibility:** Screen readers announce the label text when the
user focuses the input. Without this association, a screen reader user
focusing the email input hears "edit text" — they don't know what it's
for. With the association, they hear "Email address, edit text."

**Alternative: implicit association** by wrapping:
```jsx
<label>
  Email address
  <input type="email" />
</label>
```

This works for simple cases but explicit `htmlFor`/`id` is more robust
(works when label and input aren't adjacent) and is preferred in React
because it works with `useId()` for guaranteed unique IDs:

```jsx
function EmailField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" />
    </>
  );
}
```
