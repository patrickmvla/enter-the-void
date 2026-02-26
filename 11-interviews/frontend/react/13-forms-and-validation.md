# Forms & Validation — Production Patterns That Win Interviews

## The Form Landscape in React

```
Controlled (React owns value)     Uncontrolled (DOM owns value)
  useState + onChange               useRef + defaultValue
  re-renders on every keystroke     zero re-renders until submit
  real-time validation              submit-time validation
  full control                      minimal boilerplate

      │                                   │
      ▼                                   ▼
  Manual approach:                  React Hook Form:
  verbose, re-render heavy          uncontrolled internally,
  fine for 3-field forms            controlled API externally
                                    0 re-renders per keystroke
```

---

## React Hook Form — The Production Standard

### Why it exists

The naive approach re-renders the entire form on every keystroke:

```tsx
// BAD: typing in "name" re-renders email, address, phone, submit button...
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [address, setAddress] = useState('');
  // 20 more fields...

  return (
    <form>
      <input value={name} onChange={e => setName(e.target.value)} />
      {/* Every keystroke re-renders ALL 20+ inputs */}
    </form>
  );
}
```

React Hook Form uses **uncontrolled inputs internally** (refs, not state)
so the form doesn't re-render on input changes. It only re-renders when
validation state changes or on submit.

### Core API

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  email: z.string().email('Invalid email'),
  age: z.coerce.number().min(18, 'Must be 18+').max(120),
  role: z.enum(['admin', 'editor', 'viewer']),
});

type FormData = z.infer<typeof schema>;  // type derived from schema

function UserForm() {
  const {
    register,     // connects input to form (returns ref + onChange + onBlur)
    handleSubmit,  // wraps your submit handler with validation
    formState: { errors, isSubmitting, isDirty, isValid },
    reset,         // reset to default values
    watch,         // subscribe to field changes (causes re-render)
    setValue,      // programmatically set a field
    trigger,       // manually trigger validation
  } = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: { name: '', email: '', age: 18, role: 'viewer' },
    mode: 'onBlur',  // validate on blur (not on every keystroke)
  });

  const onSubmit = async (data: FormData) => {
    // data is fully typed and validated
    await createUser(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" {...register('name')} />
        {errors.name && <span role="alert">{errors.name.message}</span>}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} />
        {errors.email && <span role="alert">{errors.email.message}</span>}
      </div>

      <div>
        <label htmlFor="age">Age</label>
        <input id="age" type="number" {...register('age')} />
        {errors.age && <span role="alert">{errors.age.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

### Validation Modes

```
mode: 'onSubmit'    — validate only on submit (default)
mode: 'onBlur'      — validate when field loses focus
mode: 'onChange'     — validate on every change (re-renders per keystroke!)
mode: 'onTouched'   — validate on first blur, then on change
mode: 'all'         — validate on blur AND change
```

**Production recommendation:** Use `mode: 'onBlur'` — validates when the
user moves to the next field. Fast enough to catch errors early, doesn't
re-render on every keystroke. Use `mode: 'onChange'` only for fields that
need real-time feedback (password strength meter).

---

## Zod — Schema Validation

### Why Zod over manual validation

```tsx
// BAD: manual validation — incomplete, inconsistent, hard to maintain
function validate(data) {
  const errors = {};
  if (!data.email) errors.email = 'Required';
  else if (!data.email.includes('@')) errors.email = 'Invalid';
  if (data.age < 18) errors.age = 'Must be 18+';
  return errors;
}

// GOOD: Zod schema — single source of truth for types AND validation
const schema = z.object({
  email: z.string().email(),
  age: z.number().min(18),
});
type FormData = z.infer<typeof schema>;  // TypeScript type derived automatically
```

### Advanced patterns

```tsx
// Conditional validation
const schema = z.discriminatedUnion('accountType', [
  z.object({
    accountType: z.literal('personal'),
    name: z.string().min(1),
  }),
  z.object({
    accountType: z.literal('business'),
    companyName: z.string().min(1),
    taxId: z.string().regex(/^\d{9}$/),
  }),
]);

// Transform on parse
const schema = z.object({
  email: z.string().email().transform(s => s.toLowerCase().trim()),
  tags: z.string().transform(s => s.split(',').map(t => t.trim())),
});

// Async validation (e.g., check if username is taken)
const schema = z.object({
  username: z.string().min(3).refine(
    async (val) => {
      const exists = await checkUsername(val);
      return !exists;
    },
    { message: 'Username already taken' }
  ),
});

// Reusable schemas
const addressSchema = z.object({
  street: z.string().min(1),
  city: z.string().min(1),
  zip: z.string().regex(/^\d{5}(-\d{4})?$/),
});

const userSchema = z.object({
  name: z.string(),
  billing: addressSchema,
  shipping: addressSchema.optional(),
});
```

---

## Server Actions + Forms (Next.js)

### Progressive Enhancement Pattern

```tsx
// actions.ts
'use server';
import { z } from 'zod';

const ContactSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email'),
  message: z.string().min(10, 'Message must be at least 10 characters'),
});

type State = {
  errors?: Record<string, string[]>;
  success?: boolean;
};

export async function submitContact(
  prevState: State,
  formData: FormData
): Promise<State> {
  const parsed = ContactSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    message: formData.get('message'),
  });

  if (!parsed.success) {
    return { errors: parsed.error.flatten().fieldErrors };
  }

  await db.contact.create({ data: parsed.data });
  return { success: true };
}
```

```tsx
// ContactForm.tsx
'use client';
import { useActionState } from 'react';
import { submitContact } from './actions';

function ContactForm() {
  const [state, formAction, isPending] = useActionState(submitContact, {});

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" required />
        {state.errors?.name && (
          <span role="alert">{state.errors.name[0]}</span>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" required />
        {state.errors?.email && (
          <span role="alert">{state.errors.email[0]}</span>
        )}
      </div>

      <div>
        <label htmlFor="message">Message</label>
        <textarea id="message" name="message" required />
        {state.errors?.message && (
          <span role="alert">{state.errors.message[0]}</span>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Sending...' : 'Send'}
      </button>

      {state.success && (
        <p role="status" aria-live="polite">Message sent successfully!</p>
      )}
    </form>
  );
}
```

**Why this pattern matters:** The form works **without JavaScript**. The
`action` attribute submits a regular POST request. When JS loads,
`useActionState` enhances it with pending state and error handling without
a full page reload. This is progressive enhancement.

---

## Multi-Step Forms

```tsx
type Step = 'personal' | 'address' | 'payment' | 'review';

function MultiStepForm() {
  const [step, setStep] = useState<Step>('personal');
  const [formData, setFormData] = useState<Partial<FullFormData>>({});

  function updateAndAdvance(stepData: Partial<FullFormData>, next: Step) {
    setFormData(prev => ({ ...prev, ...stepData }));
    setStep(next);
  }

  switch (step) {
    case 'personal':
      return (
        <PersonalStep
          defaults={formData}
          onNext={(data) => updateAndAdvance(data, 'address')}
        />
      );
    case 'address':
      return (
        <AddressStep
          defaults={formData}
          onBack={() => setStep('personal')}
          onNext={(data) => updateAndAdvance(data, 'payment')}
        />
      );
    // ...
  }
}

// Each step validates only its own fields
function PersonalStep({ defaults, onNext }) {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(personalSchema),
    defaultValues: defaults,
  });

  return (
    <form onSubmit={handleSubmit(onNext)}>
      <input {...register('name')} />
      <input {...register('email')} />
      <button type="submit">Next</button>
    </form>
  );
}
```

**Key pattern:** Each step has its own Zod schema and validates
independently. The parent holds accumulated data. Users can go back
without losing data. Only the final step submits to the server.

---

## Form Accessibility Patterns

### Error announcements

```tsx
// Errors must be associated with inputs AND announced to screen readers
<div>
  <label htmlFor="email">Email</label>
  <input
    id="email"
    type="email"
    aria-invalid={!!errors.email}
    aria-describedby={errors.email ? 'email-error' : undefined}
    {...register('email')}
  />
  {errors.email && (
    <span id="email-error" role="alert">
      {errors.email.message}
    </span>
  )}
</div>
```

**Critical details:**
- `aria-invalid` tells screen readers the field has an error
- `aria-describedby` associates the error message with the input
- `role="alert"` announces the error when it appears
- `id` on the error links it to `aria-describedby`

### Focus management on error

```tsx
const onSubmit = handleSubmit(async (data) => {
  await submitForm(data);
}, (errors) => {
  // On validation failure, focus the first errored field
  const firstError = Object.keys(errors)[0];
  if (firstError) {
    document.getElementById(firstError)?.focus();
  }
});
```

---

## Interview Questions

---

**Q: Why would you use React Hook Form over controlled components?**

**How to answer:** Explain the performance problem with controlled forms
at scale, then show how RHF solves it architecturally.

**Answer:**

**What most candidates say:** "React Hook Form is easier and has less
boilerplate." This is true but misses the real reason.

**What a senior candidate says:** The fundamental problem with controlled
components is **re-render cost at scale**. In a controlled form with 20
fields, typing one character in any input re-renders all 20 fields plus
the form container. On a fast machine you won't notice. On a low-end
mobile device or a form with complex validation, you'll see input lag.

React Hook Form solves this architecturally: inputs are **uncontrolled
internally** (using refs, not state). When you type, the DOM updates
directly — React doesn't know and doesn't re-render. RHF only triggers
re-renders when:
- Validation state changes (error appears/disappears)
- You call `watch()` (subscribes to changes, opt-in)
- Form is submitted

```
Controlled (20 fields):         React Hook Form:
  Type 'a' → 20 re-renders       Type 'a' → 0 re-renders
  Type 'b' → 20 re-renders       Submit → 1 re-render (errors)
  Submit → 1 re-render
```

**The production insight:** RHF isn't just about performance. It's about
**separation of concerns**. The Zod schema is your validation contract.
The form component is your UI. The `resolver` connects them. You can
test the schema independently, reuse it between client and server
(Server Actions validate with the same Zod schema), and change the UI
without touching validation logic.

**When NOT to use RHF:** Simple 2-3 field forms where the overhead of a
library isn't worth it. Login forms, search bars, simple modals — just
use `useState`. RHF pays for itself at 5+ fields or when you need
complex validation.

---

**Q: How do you handle form validation on both client and server?**

**How to answer:** Show the shared schema pattern and explain why
server validation is non-negotiable.

**Answer:**

**What most candidates say:** "Validate on the client for UX, validate
on the server for security." Correct principle, but how?

**What a senior candidate says:** The key is a **shared validation
schema** — one Zod schema used by both the client form and the Server
Action. This guarantees the validation rules are identical:

```tsx
// shared/schemas.ts — used by BOTH client and server
export const CreatePostSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  content: z.string().min(10).max(50000),
  tags: z.array(z.string()).max(10),
});

// Client: React Hook Form uses it for instant feedback
const { register } = useForm({
  resolver: zodResolver(CreatePostSchema),
});

// Server: Server Action uses the SAME schema
'use server';
export async function createPost(formData: FormData) {
  const parsed = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    tags: formData.getAll('tags'),
  });
  if (!parsed.success) return { errors: parsed.error.flatten().fieldErrors };
  // parsed.data is validated AND typed
}
```

**Why server validation is non-negotiable:** Client validation can be
bypassed entirely. A malicious user can:
- Disable JavaScript and submit the raw form
- Use `curl` to POST directly to the Server Action endpoint
- Modify the DOM to remove `required` attributes
- Use browser devtools to call the action with arbitrary arguments

Server Actions are **public HTTP endpoints**. Treat them like API routes.
Validate every input. Check authorization. Never trust the client.

**The production pattern:** Validate on the client for UX (instant
feedback), validate on the server for security (never trust the client),
use the same schema for both (prevents drift).

---

**Q: How do Server Action forms provide progressive enhancement?**

**How to answer:** Explain what happens with and without JavaScript,
and why this matters.

**Answer:**

**What most candidates say:** "The form works without JavaScript."
Correct but shallow.

**What a senior candidate says:** Progressive enhancement means the form
has a **baseline experience** that works in any browser, then JavaScript
**enhances** it with better UX.

**Without JavaScript (baseline):**
```
User clicks Submit → browser sends POST request → server processes →
server returns new HTML → full page reload with success/error state
```
The `action` attribute on the form points to the Server Action, which
Next.js exposes as a POST endpoint. The browser handles this natively.

**With JavaScript (enhanced):**
```
User clicks Submit → React serializes form data → sends POST via fetch →
server processes → React updates UI without page reload →
pending state shown during processing → errors shown inline
```

`useActionState` provides `isPending` for loading UI. `useFormStatus`
lets child components (like a generic SubmitButton) read the form's
submission state. Errors are displayed inline without losing form state.

**Why this matters in practice:**
1. **Resilience** — if JS fails to load (CDN issue, bundle error, slow
   connection), the form still works
2. **SEO crawlers** — forms that work without JS are more crawlable
3. **Accessibility** — screen readers and assistive tech handle native
   forms better than JS-driven forms
4. **Performance** — the form is interactive before JS loads

**The trade-off:** Progressive enhancement limits what you can do. You
can't do client-side validation before submit (no JS = no validation
feedback). You can't do optimistic updates. You can't prevent double
submission. All of these require JavaScript. The baseline is functional
but basic.

---

**Q: How do you handle complex form state like dependent fields?**

**How to answer:** Show the pattern where one field's value affects
another field's options, validation, or visibility.

**Answer:**

**What most candidates say:** "Use `watch()` to observe the field."

**What a senior candidate says:** Dependent fields are one of the
hardest form patterns because they introduce **cascading updates** —
changing one field invalidates or resets other fields.

```tsx
function AddressForm() {
  const { register, watch, setValue, formState: { errors } } = useForm({
    resolver: zodResolver(addressSchema),
  });

  const country = watch('country');  // re-renders when country changes

  // Reset state/city when country changes
  useEffect(() => {
    setValue('state', '');
    setValue('city', '');
  }, [country, setValue]);

  const states = useMemo(
    () => getStatesForCountry(country),
    [country]
  );

  return (
    <form>
      <select {...register('country')}>
        {countries.map(c => <option key={c.code} value={c.code}>{c.name}</option>)}
      </select>

      <select {...register('state')} disabled={!country}>
        {states.map(s => <option key={s.code} value={s.code}>{s.name}</option>)}
      </select>

      <input {...register('city')} disabled={!watch('state')} />
    </form>
  );
}
```

**The gotchas:**

1. **watch() causes re-renders** — use it sparingly. For fields that don't
   affect the UI, use `getValues()` instead (reads without subscribing).

2. **Race conditions with async options** — if fetching cities based on
   the selected state, the user can change the state before cities load.
   Use AbortController to cancel stale fetches.

3. **Validation drift** — if the country changes, the state field's
   allowed values change. Your Zod schema needs to handle this:
   ```tsx
   const schema = z.object({
     country: z.string(),
     state: z.string(),
   }).refine(data => {
     const validStates = getStatesForCountry(data.country);
     return validStates.some(s => s.code === data.state);
   }, { message: 'Invalid state for selected country', path: ['state'] });
   ```

4. **Reset vs clear** — when country changes, should state be empty or
   default to the first option? Business decision, but be explicit.

---

**Q: What are the most common form bugs in production?**

**How to answer:** List real bugs from production experience, not
theoretical issues.

**Answer:**

**What most candidates say:** "Forgetting to validate on the server."

**What a senior candidate says:** These are the bugs I've seen ship to
production:

1. **Double submission** — user clicks Submit, sees no feedback, clicks
   again. Two records created. Fix: disable the button during submission
   AND debounce the action. Use `isPending` from `useActionState`.

2. **Stale form data after navigation** — user fills a form, navigates
   away, comes back, form is empty. Fix: persist form state to
   `sessionStorage` or URL params. React Hook Form has no built-in
   persistence — you need a `useEffect` that syncs to storage.

3. **Uncontrolled to controlled warning** — starting with
   `defaultValue={undefined}` then setting a value from an API response.
   React treats this as switching from uncontrolled to controlled.
   Fix: always provide a default value, even if it's an empty string.

4. **File upload state mismatch** — file inputs are always uncontrolled
   (`<input type="file" />` can't have a `value`). If the form has a
   mix of controlled and file inputs, the mental model breaks. Fix: use
   `FormData` for the entire form when files are involved.

5. **Timezone-dependent validation** — a date field validates as "today"
   on the server but the client is in a different timezone. The form
   passes client validation and fails server validation. Fix: always
   validate dates in UTC.

6. **Multi-tab conflict** — user opens the same form in two tabs, edits
   in both, submits both. The second submission overwrites the first.
   Fix: optimistic locking (send a version/updatedAt field, reject if
   it doesn't match the current value).
