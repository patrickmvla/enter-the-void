# React Coding Challenges — Common Interview Problems

## What They're Testing

Coding challenges test whether you can **build working UI under pressure**.
They're looking for:

- Clean component structure
- Proper state management
- Edge case handling
- TypeScript comfort
- Performance awareness

Give yourself 15-20 minutes per problem. Real interviews have time pressure.

---

## Challenge 1: Debounced Search Input

**Build a search input that debounces API calls.**

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

function Search() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Result[]>([]);
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (!debouncedQuery) {
      setResults([]);
      return;
    }

    const controller = new AbortController();
    fetch(`/api/search?q=${debouncedQuery}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setResults)
      .catch(e => { if (e.name !== 'AbortError') throw e; });

    return () => controller.abort();
  }, [debouncedQuery]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <ul>
        {results.map(r => <li key={r.id}>{r.name}</li>)}
      </ul>
    </div>
  );
}
```

**Key points:** AbortController for cleanup, debounce as custom hook,
empty query handling.

---

## Challenge 2: Todo List with CRUD

**Build a todo app with add, toggle, and delete.**

```tsx
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [input, setInput] = useState('');

  const addTodo = () => {
    if (!input.trim()) return;
    setTodos(prev => [
      ...prev,
      { id: crypto.randomUUID(), text: input.trim(), completed: false }
    ]);
    setInput('');
  };

  const toggleTodo = (id: string) => {
    setTodos(prev =>
      prev.map(t => t.id === id ? { ...t, completed: !t.completed } : t)
    );
  };

  const deleteTodo = (id: string) => {
    setTodos(prev => prev.filter(t => t.id !== id));
  };

  return (
    <div>
      <form onSubmit={e => { e.preventDefault(); addTodo(); }}>
        <input value={input} onChange={e => setInput(e.target.value)} />
        <button type="submit">Add</button>
      </form>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{
              textDecoration: todo.completed ? 'line-through' : 'none'
            }}>
              {todo.text}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**Key points:** Immutable state updates, form submission with Enter,
unique IDs, empty input guard.

---

## Challenge 3: Infinite Scroll

**Load more items as user scrolls to the bottom.**

```tsx
function InfiniteList() {
  const [items, setItems] = useState<Item[]>([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const observerRef = useRef<IntersectionObserver | null>(null);

  const lastItemRef = useCallback((node: HTMLLIElement | null) => {
    if (loading) return;
    if (observerRef.current) observerRef.current.disconnect();

    observerRef.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && hasMore) {
        setPage(p => p + 1);
      }
    });

    if (node) observerRef.current.observe(node);
  }, [loading, hasMore]);

  useEffect(() => {
    setLoading(true);
    fetch(`/api/items?page=${page}`)
      .then(r => r.json())
      .then(data => {
        setItems(prev => [...prev, ...data.items]);
        setHasMore(data.hasMore);
        setLoading(false);
      });
  }, [page]);

  return (
    <ul>
      {items.map((item, i) => (
        <li
          key={item.id}
          ref={i === items.length - 1 ? lastItemRef : undefined}
        >
          {item.name}
        </li>
      ))}
      {loading && <li>Loading...</li>}
    </ul>
  );
}
```

**Key points:** IntersectionObserver for scroll detection, callback ref
pattern, prevent duplicate fetches while loading.

**Production gotchas to mention in an interview:**
- **Stale closure on `hasMore`**: The observer callback captures `hasMore`
  from the render when it was created. If the API returns `hasMore: false`,
  the observer may still fire with the old `true` value. The fix: include
  `hasMore` in the callback ref's dependency array (which this code does).
- **Race condition**: Fast scrolling can trigger multiple page increments
  before the first fetch completes. The `if (loading) return` guard
  prevents creating a new observer, but the page state has already
  incremented. Consider using a ref for `loading` instead of state.
- **Memory**: Disconnect the observer on unmount. This code handles it
  via the callback ref pattern — when the ref receives `null` (unmount),
  the observer is disconnected.
- **Sentinel element**: A more robust pattern uses a dedicated sentinel
  `<div ref={sentinelRef} />` after the list instead of attaching the
  ref to the last item. This avoids recreating the observer on every
  new page of items.

---

## Challenge 4: Modal Component

**Build a reusable modal with portal, focus trap, and escape to close.**

```tsx
function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!isOpen) return;

    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };

    document.addEventListener('keydown', handleEscape);
    document.body.style.overflow = 'hidden';

    // Focus first focusable element
    const focusable = modalRef.current?.querySelector<HTMLElement>(
      'button, input, [tabindex]'
    );
    focusable?.focus();

    return () => {
      document.removeEventListener('keydown', handleEscape);
      document.body.style.overflow = '';
    };
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div className="overlay" onClick={onClose}>
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-label={title}
        onClick={e => e.stopPropagation()}
      >
        <h2>{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body
  );
}
```

**Key points:** Portal, escape handler, scroll lock, overlay click to close,
stop propagation on modal content, ARIA attributes.

**What's missing (mention in interview to show depth):**

1. **Focus trap** — Tab/Shift+Tab should cycle within the modal:
```tsx
function trapFocus(e: KeyboardEvent) {
  if (e.key !== 'Tab') return;
  const focusable = modalRef.current!.querySelectorAll<HTMLElement>(
    'button, input, select, textarea, a[href], [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0], last = focusable[focusable.length - 1];
  if (e.shiftKey && document.activeElement === first) {
    e.preventDefault(); last.focus();
  } else if (!e.shiftKey && document.activeElement === last) {
    e.preventDefault(); first.focus();
  }
}
```

2. **Focus restoration** — save `document.activeElement` before opening,
   restore on close. Without this, focus jumps to `<body>` after close.

3. **Nested modals** — if a modal opens another modal, Escape should
   close only the topmost. Use a modal stack or check if the event
   has already been handled.

4. **In production:** Use Radix Dialog, Headless UI Dialog, or React
   Aria's useDialog. A complete modal has 20+ edge cases (iOS scroll
   lock, screen reader announcements, animation exit, nested modals).
   Hand-rolling is an interview exercise, not a production strategy.

---

## Challenge 5: useLocalStorage Hook

```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const stored = window.localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch {
      console.warn(`Failed to save ${key} to localStorage`);
    }
  }, [key, value]);

  return [value, setValue] as const;
}
```

**Key points:** Lazy initialization, JSON parse/stringify, error handling
for SSR/private browsing, `as const` for tuple return type.

---

## Challenge 6: Throttled Window Resize Hook

```tsx
function useWindowSize(throttleMs = 200) {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    let timeoutId: number | null = null;

    function handleResize() {
      if (timeoutId) return;
      timeoutId = window.setTimeout(() => {
        setSize({ width: window.innerWidth, height: window.innerHeight });
        timeoutId = null;
      }, throttleMs);
    }

    window.addEventListener('resize', handleResize);
    return () => {
      window.removeEventListener('resize', handleResize);
      if (timeoutId) clearTimeout(timeoutId);
    };
  }, [throttleMs]);

  return size;
}
```

---

## Challenge 7: Data Table with Sort

```tsx
type SortDir = 'asc' | 'desc' | null;
type SortConfig = { key: string; dir: SortDir };

function DataTable<T extends Record<string, any>>({
  data,
  columns,
}: {
  data: T[];
  columns: { key: keyof T & string; label: string }[];
}) {
  const [sort, setSort] = useState<SortConfig>({ key: '', dir: null });

  const sorted = useMemo(() => {
    if (!sort.dir) return data;
    return [...data].sort((a, b) => {
      const aVal = a[sort.key];
      const bVal = b[sort.key];
      const cmp = aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
      return sort.dir === 'asc' ? cmp : -cmp;
    });
  }, [data, sort]);

  function toggleSort(key: string) {
    setSort(prev => ({
      key,
      dir: prev.key !== key ? 'asc' : prev.dir === 'asc' ? 'desc' : null,
    }));
  }

  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => (
            <th key={col.key} onClick={() => toggleSort(col.key)}>
              {col.label}
              {sort.key === col.key && (sort.dir === 'asc' ? ' ▲' : ' ▼')}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {sorted.map((row, i) => (
          <tr key={i}>
            {columns.map(col => (
              <td key={col.key}>{String(row[col.key])}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Key points:** Generic typing, useMemo for sort, tri-state sort cycle,
immutable sort with spread.

**What a senior candidate would add:**

1. **Accessibility** — `<th onClick>` is not keyboard accessible. Use
   a `<button>` inside the `<th>`, add `aria-sort="ascending"` or
   `"descending"`, and use `scope="col"`:
```tsx
<th scope="col" aria-sort={sort.key === col.key ? sort.dir === 'asc' ? 'ascending' : 'descending' : 'none'}>
  <button onClick={() => toggleSort(col.key)}>
    {col.label} {sort.key === col.key && (sort.dir === 'asc' ? '▲' : '▼')}
  </button>
</th>
```

2. **Null handling** — the comparison `aVal < bVal` returns `false` for
   `null` or `undefined`. Add: `if (aVal == null) return 1; if (bVal == null) return -1;`

3. **Stable sort** — when values are equal, maintain original order.
   JavaScript's `Array.sort` is not guaranteed stable in all environments
   (though modern engines are). For safety, include the original index.

4. **In production** — use TanStack Table. Real data tables need column
   resizing, virtualization, pagination, filtering, column visibility,
   and sticky headers. This challenge is a 10-minute exercise; a
   production table is a 10-week project.

---

## Challenge 8: Undo/Redo

```tsx
function useUndoRedo<T>(initialState: T) {
  const [past, setPast] = useState<T[]>([]);
  const [present, setPresent] = useState(initialState);
  const [future, setFuture] = useState<T[]>([]);

  const set = useCallback((newState: T) => {
    setPast(prev => [...prev, present]);
    setPresent(newState);
    setFuture([]);
  }, [present]);

  const undo = useCallback(() => {
    if (past.length === 0) return;
    setFuture(prev => [present, ...prev]);
    setPresent(past[past.length - 1]);
    setPast(prev => prev.slice(0, -1));
  }, [past, present]);

  const redo = useCallback(() => {
    if (future.length === 0) return;
    setPast(prev => [...prev, present]);
    setPresent(future[0]);
    setFuture(prev => prev.slice(1));
  }, [future, present]);

  return {
    state: present,
    set,
    undo,
    redo,
    canUndo: past.length > 0,
    canRedo: future.length > 0,
  };
}
```

---

## Tips for Coding Interviews

1. **Talk through your approach** before coding
2. **Start with the data model** — what state do you need?
3. **Handle edge cases** — empty states, loading, errors
4. **Name things well** — interviewers read your code
5. **Don't over-optimize** — make it work first, optimize if asked
6. **Use TypeScript** — shows professionalism, catches errors live
7. **Clean up effects** — always return cleanup in useEffect
8. **Immutable updates** — never mutate state directly
