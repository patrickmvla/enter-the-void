# TypeScript with React — Type Safety Interview Questions

## Component Props

```tsx
// Basic props typing
interface ButtonProps {
  label: string;
  variant?: 'primary' | 'secondary' | 'danger';
  disabled?: boolean;
  onClick: () => void;
}

function Button({ label, variant = 'primary', disabled, onClick }: ButtonProps) {
  return (
    <button className={variant} disabled={disabled} onClick={onClick}>
      {label}
    </button>
  );
}
```

### Children typing

```tsx
// Accepts any renderable React content
interface CardProps {
  children: React.ReactNode;
}

// Accepts only a single React element
interface WrapperProps {
  children: React.ReactElement;
}

// Accepts a render function
interface RenderProps {
  children: (data: User) => React.ReactNode;
}
```

### Extending HTML elements

```tsx
// Button that accepts all native button props + custom ones
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant: 'primary' | 'secondary';
}

function Button({ variant, children, ...rest }: ButtonProps) {
  return <button className={variant} {...rest}>{children}</button>;
}

// ComponentPropsWithRef for forwarded refs
type InputProps = React.ComponentPropsWithRef<'input'> & {
  label: string;
};
```

---

## Hooks Typing

```tsx
// useState infers from initial value
const [count, setCount] = useState(0);           // number
const [name, setName] = useState('');             // string

// Explicit type when initial value doesn't tell the whole story
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);

// useRef
const inputRef = useRef<HTMLInputElement>(null);
const timerRef = useRef<number | null>(null);   // mutable ref

// useReducer
type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'set'; payload: number };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case 'increment': return state + 1;
    case 'decrement': return state - 1;
    case 'set': return action.payload;
  }
}

const [count, dispatch] = useReducer(reducer, 0);
// dispatch({ type: 'set', payload: 42 }) ← type-safe
```

---

## Generic Components

```tsx
// A list that works with any item type
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage — T inferred as User
<List
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>
```

### Generic with constraints

```tsx
interface HasId {
  id: string;
}

function SelectableList<T extends HasId>({ items }: { items: T[] }) {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => setSelectedId(item.id)}>
          {/* item.id is guaranteed to exist */}
        </li>
      ))}
    </ul>
  );
}
```

---

## Discriminated Unions

Model component variants that have different props:

```tsx
type AlertProps =
  | { severity: 'error'; onRetry: () => void }
  | { severity: 'warning'; onDismiss: () => void }
  | { severity: 'info' };

function Alert(props: AlertProps) {
  switch (props.severity) {
    case 'error':
      return <div className="error">
        <button onClick={props.onRetry}>Retry</button>
      </div>;
    case 'warning':
      return <div className="warning">
        <button onClick={props.onDismiss}>Dismiss</button>
      </div>;
    case 'info':
      return <div className="info">Info message</div>;
  }
}

// Type-safe: can't pass onRetry to severity="info"
<Alert severity="error" onRetry={() => {}} />   // ✓
<Alert severity="info" />                         // ✓
<Alert severity="info" onRetry={() => {}} />      // ✗ type error
```

---

## Event Handlers

```tsx
function Form() {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
  };

  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log(e.clientX, e.clientY);
  };

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') submit();
  };

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} onKeyDown={handleKeyDown} />
      <button onClick={handleClick}>Submit</button>
    </form>
  );
}
```

---

## Context Typing

```tsx
interface AuthContext {
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContext | null>(null);

// Custom hook with null check
function useAuth(): AuthContext {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

---

## Utility Types for React

```tsx
// Extract props from a component
type ButtonProps = React.ComponentProps<typeof Button>;

// Make all props optional (for defaultProps or partial updates)
type PartialUser = Partial<User>;

// Make specific props required
type RequiredUser = Required<Pick<User, 'name' | 'email'>> & Partial<User>;

// Omit props (extending but removing some)
type CardProps = Omit<React.HTMLAttributes<HTMLDivElement>, 'className'> & {
  variant: 'elevated' | 'flat';
};

// Record for maps
type ThemeColors = Record<'primary' | 'secondary' | 'background', string>;
```

---

## Type-Safe API Responses

```tsx
interface ApiResponse<T> {
  data: T;
  error: string | null;
  status: number;
}

async function fetchApi<T>(url: string): Promise<ApiResponse<T>> {
  const res = await fetch(url);
  const data = await res.json();
  return { data, error: null, status: res.status };
}

// Usage
const { data } = await fetchApi<User[]>('/api/users');
// data is User[]
```

---

## Interview Questions

---

**Q: interface vs type — when to use each?**

**How to answer:** Show you know the technical differences, then give
a pragmatic recommendation.

**Answer:** Technically they have a few differences:

**Interface can:**
- Be **extended** by other interfaces or classes (`extends`)
- Be **declaration merged** (same name in two files merges automatically)
- Be **augmented** by third-party libraries (module augmentation)

**Type can:**
- Create **unions** (`type Status = 'loading' | 'error' | 'success'`)
- Create **intersections** (`type Combined = TypeA & TypeB`)
- Use **mapped types** (`type ReadOnly<T> = { readonly [K in keyof T]: T[K] }`)
- Alias **primitives** (`type ID = string`)

**The common answer "they're interchangeable" is dangerously incomplete.**
Here are the real differences that matter in production:

**Interfaces** support **declaration merging** — multiple declarations
with the same name merge into one:
```tsx
// In your library:
interface Window { analytics: AnalyticsSDK; }
// The consumer's TypeScript now knows window.analytics exists.
// This is HOW libraries augment global types.
```

This is critical for library authors. If you export `type Props` instead
of `interface Props`, consumers can't extend your types without
wrapping/intersecting them.

**Types** support **unions and mapped types** — interfaces can't:
```tsx
type Result = Success | Failure;           // Union — interface can't do this
type Partial<T> = { [K in keyof T]?: T[K] }; // Mapped — interface can't
type Status = 'idle' | 'loading' | 'error';  // Literal union
```

**My production rule:**
- Use **interface** for anything **consumers might extend** — component
  props in a shared library, module augmentations, object shapes that
  are part of a public API
- Use **type** for everything else — unions, utility types, internal
  implementation types, function signatures
- In an **application** (not a library), the distinction barely matters.
  Pick one convention and enforce it with ESLint.

```tsx
// Library: interface (consumers can augment)
interface ButtonProps { label: string; variant: 'primary' | 'secondary'; }

// App: type is fine
type Status = 'idle' | 'loading' | 'success' | 'error';
type ButtonVariant = ButtonProps['variant'];
```

---

**Q: How do you type a component that accepts any HTML element's props?**

**How to answer:** Show the pattern with ComponentPropsWithRef, explain
why you'd use it, and demonstrate rest spreading.

**Answer:** Use `React.ComponentPropsWithRef<'element'>` to inherit all
native HTML props, then intersect with your custom props:

```tsx
type ButtonProps = React.ComponentPropsWithRef<'button'> & {
  variant: 'primary' | 'secondary';
  isLoading?: boolean;
};

function Button({ variant, isLoading, children, ref, ...rest }: ButtonProps) {
  return (
    <button
      ref={ref}
      className={`btn-${variant}`}
      disabled={isLoading || rest.disabled}
      {...rest}   // onClick, aria-label, type, etc. — all passed through
    >
      {isLoading ? <Spinner /> : children}
    </button>
  );
}
```

This gives consumers **full autocomplete** for native button attributes
(onClick, disabled, type, aria-*, etc.) while adding your custom props
on top. Use `ComponentPropsWithRef` (not `ComponentProps`) to include
ref support.

If you need to **exclude** certain native props (because you override them):
```tsx
type InputProps = Omit<React.ComponentPropsWithRef<'input'>, 'size'> & {
  size: 'sm' | 'md' | 'lg';  // our own size, not HTML's
};
```

**Production gotcha:** Over-extending HTML props can **expose attributes
you don't want**. A `<Button>` component that spreads `...rest` onto a
`<button>` now accepts `formAction`, `formEncType`, `formMethod` — none
of which make sense for your component. In a large team, someone WILL
pass `type="submit"` to a button that should always be `type="button"`,
and forms will submit unexpectedly.

**The defensive pattern:**
```tsx
// Instead of extending ALL button props, pick what you need:
type ButtonProps = Pick<
  React.ComponentPropsWithRef<'button'>,
  'onClick' | 'disabled' | 'aria-label' | 'className' | 'ref'
> & {
  variant: 'primary' | 'secondary';
};
```

Use `Omit` for "everything except X" and `Pick` for "only these specific
attributes." Pick is safer for design system components.

---

**Q: How do you type a generic React component?**

**How to answer:** Show the syntax and explain how TypeScript infers
the generic parameter from usage.

**Answer:** Use a generic function with a type parameter on the component:

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}
```

At the call site, TypeScript **infers T** from the `items` prop:

```tsx
// T is inferred as User — renderItem and keyExtractor are typed automatically
<List
  items={users}                           // User[]
  renderItem={(user) => <span>{user.name}</span>}   // user: User ✓
  keyExtractor={(user) => user.id}         // user: User ✓
/>
```

You can add **constraints** if the generic must have certain properties:

```tsx
function SelectableList<T extends { id: string }>({ items }: { items: T[] }) {
  // T is guaranteed to have an id field
  return items.map(item => <div key={item.id}>{/* ... */}</div>);
}
```

**Note:** Generic components can't use React.memo directly (it erases the
generic). Use a named function and assign memo separately, or just don't
memo generic components.

---

**Q: How do you handle the "context might be undefined" problem?**

**How to answer:** Show the problem, the solution pattern, and why
throwing is better than a default value.

**Answer:** When you create a context, you need an initial value. If the
context represents something that must come from a provider (like auth),
there's no meaningful default:

```tsx
// Option 1: default to undefined — consumers get T | undefined everywhere
const AuthContext = createContext<AuthContextType | undefined>(undefined);
// Every useContext(AuthContext) returns AuthContextType | undefined
// You'd need if-checks or non-null assertions everywhere — annoying
```

**The solution:** Create a custom hook that throws if the context is null:

```tsx
const AuthContext = createContext<AuthContextType | null>(null);

function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (context === null) {
    throw new Error('useAuth must be used within an <AuthProvider>');
  }
  return context;  // TypeScript narrows this to AuthContextType (non-null)
}
```

Now every component that calls `useAuth()` gets a **guaranteed non-null**
type. No optional chaining, no if-checks, no `as` casts. And if someone
forgets the provider, they get a clear error message instead of a cryptic
"cannot read property of null."

**Why throwing is better than defaulting:** You might be tempted to use
`createContext<AuthContextType>({ user: null, login: () => {} })` with
a default value. This silently works without a provider — which sounds
convenient but is actually dangerous. If someone renders a component
outside the provider tree, it silently uses the stub default. No error,
no crash, just **wrong behavior** (login does nothing, user is always null).
The throw pattern turns a silent bug into a loud, obvious error.

**When you DO want a default value:** Optional contexts where the
component works with or without the provider. Example: a theme context
that defaults to 'light'. The component renders correctly either way.
But authentication, permissions, and data contexts should always throw.

This pattern is used by virtually every well-typed React context in
production codebases.

---

**Q: What are discriminated unions useful for in React?**

**How to answer:** Define the pattern, show a practical React example,
and explain how TypeScript narrows the type.

**Answer:** Discriminated unions let you model component variants where
**different variants require different props**. A shared field (the
"discriminant") tells TypeScript which variant you're dealing with.

```tsx
type NotificationProps =
  | { type: 'success'; message: string }
  | { type: 'error'; message: string; onRetry: () => void }
  | { type: 'loading'; progress: number };

function Notification(props: NotificationProps) {
  switch (props.type) {
    case 'success':
      return <p className="success">{props.message}</p>;
    case 'error':
      // TypeScript KNOWS onRetry exists here
      return (
        <div className="error">
          <p>{props.message}</p>
          <button onClick={props.onRetry}>Retry</button>
        </div>
      );
    case 'loading':
      // TypeScript KNOWS progress exists here
      return <progress value={props.progress} max={100} />;
  }
}
```

At the call site, TypeScript enforces the correct props per variant:
```tsx
<Notification type="success" message="Saved!" />          // ✓
<Notification type="error" message="Failed" onRetry={fn} /> // ✓
<Notification type="error" message="Failed" />              // ✗ missing onRetry
<Notification type="loading" progress={50} />               // ✓
<Notification type="loading" message="..." />               // ✗ message doesn't exist on loading
```

This is much safer than optional props (`onRetry?: () => void`) because
TypeScript catches missing or extra props at compile time. It's used
extensively for: alert components, form field variants, API response
types, state machines, and action types in reducers.
