# TanStack Table & Data Display — Building Production Data UIs

## Why TanStack Table Exists

The naive approach to data tables (map over rows, add sort/filter) breaks
down the moment you need any of:

```
Sorting (multi-column, custom comparators)
Filtering (per-column, global, faceted)
Pagination (client-side, server-side, cursor-based)
Column resizing (drag to resize, auto-size)
Column reordering (drag to reorder)
Row selection (single, multi, range)
Row expansion (nested rows, detail panels)
Virtualization (10K+ rows without DOM bloat)
Column visibility (toggle columns on/off)
Pinned columns (sticky left/right)
Grouping & aggregation
```

TanStack Table is **headless** — it provides the logic (sorting,
filtering, pagination state) but renders nothing. You bring your own UI.

---

## Core Architecture

```
┌──────────────────────────────────────┐
│            Your Data                 │
│  [{ id: 1, name: 'Alice', ... }]    │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│        Column Definitions            │
│  Define accessors, headers, cells    │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│         useReactTable()              │
│  Takes data + columns + options      │
│  Returns table instance with:        │
│    - getHeaderGroups()               │
│    - getRowModel()                   │
│    - sorting/filtering/pagination    │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│         Your JSX                     │
│  <table>, <div>, whatever you want   │
│  Table instance drives the rendering │
└──────────────────────────────────────┘
```

---

## Basic Table with Sorting & Filtering

```tsx
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  getPaginationRowModel,
  flexRender,
  type ColumnDef,
  type SortingState,
  type ColumnFiltersState,
} from '@tanstack/react-table';

type User = {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'editor' | 'viewer';
  createdAt: string;
};

// Column definitions — the schema for your table
const columns: ColumnDef<User>[] = [
  {
    accessorKey: 'name',
    header: ({ column }) => (
      <button onClick={column.getToggleSortingHandler()}>
        Name
        {{ asc: ' ▲', desc: ' ▼' }[column.getIsSorted() as string] ?? ''}
      </button>
    ),
    cell: ({ row }) => (
      <span className="font-medium">{row.getValue('name')}</span>
    ),
  },
  {
    accessorKey: 'email',
    header: 'Email',
  },
  {
    accessorKey: 'role',
    header: 'Role',
    cell: ({ getValue }) => (
      <span className={`badge-${getValue()}`}>{getValue<string>()}</span>
    ),
    filterFn: 'equals',  // exact match for enum filtering
  },
  {
    accessorKey: 'createdAt',
    header: 'Joined',
    cell: ({ getValue }) => new Date(getValue<string>()).toLocaleDateString(),
    sortingFn: 'datetime',
  },
  {
    id: 'actions',
    header: () => <span className="sr-only">Actions</span>,
    cell: ({ row }) => (
      <div>
        <button aria-label={`Edit ${row.original.name}`}>Edit</button>
        <button aria-label={`Delete ${row.original.name}`}>Delete</button>
      </div>
    ),
    enableSorting: false,
  },
];

function UsersTable({ data }: { data: User[] }) {
  const [sorting, setSorting] = useState<SortingState>([]);
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
  const [globalFilter, setGlobalFilter] = useState('');

  const table = useReactTable({
    data,
    columns,
    state: { sorting, columnFilters, globalFilter },
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
  });

  return (
    <div>
      {/* Global search */}
      <input
        placeholder="Search all columns..."
        value={globalFilter}
        onChange={e => setGlobalFilter(e.target.value)}
        aria-label="Search table"
      />

      {/* Role filter */}
      <select
        value={(table.getColumn('role')?.getFilterValue() as string) ?? ''}
        onChange={e => table.getColumn('role')?.setFilterValue(e.target.value || undefined)}
        aria-label="Filter by role"
      >
        <option value="">All roles</option>
        <option value="admin">Admin</option>
        <option value="editor">Editor</option>
        <option value="viewer">Viewer</option>
      </select>

      {/* Table */}
      <table role="grid">
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th
                  key={header.id}
                  scope="col"
                  aria-sort={
                    header.column.getIsSorted()
                      ? header.column.getIsSorted() === 'asc' ? 'ascending' : 'descending'
                      : 'none'
                  }
                >
                  {flexRender(header.column.columnDef.header, header.getContext())}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map(row => (
            <tr key={row.id}>
              {row.getVisibleCells().map(cell => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      {/* Pagination */}
      <div role="navigation" aria-label="Table pagination">
        <button
          onClick={() => table.previousPage()}
          disabled={!table.getCanPreviousPage()}
        >
          Previous
        </button>
        <span>
          Page {table.getState().pagination.pageIndex + 1} of{' '}
          {table.getPageCount()}
        </span>
        <button
          onClick={() => table.nextPage()}
          disabled={!table.getCanNextPage()}
        >
          Next
        </button>
        <select
          value={table.getState().pagination.pageSize}
          onChange={e => table.setPageSize(Number(e.target.value))}
          aria-label="Rows per page"
        >
          {[10, 25, 50, 100].map(size => (
            <option key={size} value={size}>{size} rows</option>
          ))}
        </select>
      </div>

      <p aria-live="polite">
        Showing {table.getRowModel().rows.length} of {data.length} results
      </p>
    </div>
  );
}
```

---

## Server-Side Table Pattern

For large datasets (10K+ rows), client-side sorting/filtering is too
slow. Move the logic to the server:

```tsx
function ServerTable() {
  const [sorting, setSorting] = useState<SortingState>([]);
  const [pagination, setPagination] = useState({ pageIndex: 0, pageSize: 25 });
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);

  // TanStack Query handles the server request
  const { data, isLoading, isFetching } = useQuery({
    queryKey: ['users', { sorting, pagination, columnFilters }],
    queryFn: () => fetchUsers({
      sortBy: sorting[0]?.id,
      sortDir: sorting[0]?.desc ? 'desc' : 'asc',
      page: pagination.pageIndex,
      pageSize: pagination.pageSize,
      filters: columnFilters.reduce((acc, f) => ({
        ...acc, [f.id]: f.value
      }), {}),
    }),
    placeholderData: keepPreviousData,  // show old data while loading new
  });

  const table = useReactTable({
    data: data?.rows ?? [],
    columns,
    rowCount: data?.totalCount ?? 0,    // server tells us total
    state: { sorting, pagination, columnFilters },
    onSortingChange: setSorting,
    onPaginationChange: setPagination,
    onColumnFiltersChange: setColumnFilters,
    getCoreRowModel: getCoreRowModel(),
    manualSorting: true,      // don't sort client-side
    manualPagination: true,   // don't paginate client-side
    manualFiltering: true,    // don't filter client-side
  });

  return (
    <div style={{ opacity: isFetching ? 0.6 : 1 }}>
      {/* Same table JSX as above */}
    </div>
  );
}
```

**The key props:** `manualSorting`, `manualPagination`, `manualFiltering`
tell TanStack Table that the server handles these operations. The table
just renders what you give it and reports state changes via the `on*`
callbacks. Your query key includes the table state, so changing sort/
page/filter triggers a new server request automatically.

---

## Virtualized Table for Massive Datasets

When you have 50K+ rows and can't paginate (user needs to scroll freely):

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualizedTable({ data }: { data: User[] }) {
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  const { rows } = table.getRowModel();

  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,  // estimated row height
    overscan: 10,             // render 10 extra rows above/below viewport
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <table>
        <thead>{/* normal header rendering */}</thead>
        <tbody style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
          {virtualizer.getVirtualItems().map(virtualRow => {
            const row = rows[virtualRow.index];
            return (
              <tr
                key={row.id}
                style={{
                  position: 'absolute',
                  top: `${virtualRow.start}px`,
                  height: `${virtualRow.size}px`,
                  width: '100%',
                }}
              >
                {row.getVisibleCells().map(cell => (
                  <td key={cell.id}>
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            );
          })}
        </tbody>
      </table>
    </div>
  );
}
```

**50K rows → only ~25 DOM nodes** at any time. Sorting is instant because
only visible rows re-render.

---

## Row Selection

```tsx
const [rowSelection, setRowSelection] = useState<RowSelectionState>({});

const table = useReactTable({
  // ...
  state: { rowSelection },
  onRowSelectionChange: setRowSelection,
  enableRowSelection: true,
  // enableRowSelection: (row) => row.original.role !== 'admin', // conditional
});

// Checkbox column
const selectionColumn: ColumnDef<User> = {
  id: 'select',
  header: ({ table }) => (
    <input
      type="checkbox"
      checked={table.getIsAllPageRowsSelected()}
      onChange={table.getToggleAllPageRowsSelectedHandler()}
      aria-label="Select all rows"
    />
  ),
  cell: ({ row }) => (
    <input
      type="checkbox"
      checked={row.getIsSelected()}
      onChange={row.getToggleSelectedHandler()}
      aria-label={`Select ${row.original.name}`}
    />
  ),
};

// Get selected rows
const selectedRows = table.getSelectedRowModel().rows;
```

---

## Table Accessibility (CRITICAL for interviews)

Most candidates skip this. Senior engineers don't.

```tsx
// 1. Use semantic table elements
<table role="grid">       {/* grid role for interactive tables */}
  <thead>
    <tr>
      <th scope="col">    {/* scope identifies header direction */}

// 2. Sort buttons must be keyboard accessible
<th scope="col" aria-sort={sortDir}>
  <button onClick={toggleSort}>  {/* button, not just onClick on th */}
    Name {sortDir === 'asc' ? '▲' : '▼'}
  </button>
</th>

// 3. Action buttons need labels
<button aria-label={`Edit ${row.original.name}`}>
  <EditIcon />            {/* icon-only buttons MUST have aria-label */}
</button>

// 4. Live region for result counts
<p aria-live="polite">
  Showing {filteredCount} of {totalCount} results
</p>

// 5. Selection checkboxes need labels
<input
  type="checkbox"
  aria-label={`Select ${row.original.name}`}
/>
```

---

## Interview Questions

---

**Q: Why would you use TanStack Table instead of building your own?**

**How to answer:** Don't just say "it saves time." Explain the complexity
cliff that custom tables hit.

**Answer:**

**What most candidates say:** "It handles sorting, filtering, and
pagination." Feature list.

**What a senior candidate says:** Custom tables work fine for the first
version — map rows, add a sort button, ship it. The problem is the
**complexity cliff** on the second feature request.

After sorting, you need filtering. But filtered + sorted means you sort
the filtered set, not the full set. Then you need pagination of the
filtered + sorted set. Then column resizing — which means tracking
widths in state and handling drag events. Then row selection — which
needs to persist across pages. Then column reordering. Then virtualization
because the client sent 50K rows.

Each feature interacts with every other feature. Sorting + filtering +
pagination + selection + virtualization is a **combinatorial explosion**
of state management. Does selection persist when you change pages? When
you filter? When you sort? What about "select all" — does it select all
visible rows or all rows?

TanStack Table has solved all of these interactions. It's headless, so
you don't fight a component library's CSS. You control every pixel of
rendering. And the state management is tested against every combination
of features.

**The cost of rolling your own:** It's not 1 week of work — it's 1 week
now, then a week per feature request, with growing bug surface area.
TanStack Table is 0 weeks for all features.

---

**Q: How do you handle server-side sorting and filtering?**

**How to answer:** Show the architecture, not just the code.

**Answer:**

**What most candidates say:** "Send the sort column and direction to
the API."

**What a senior candidate says:** The architecture has three layers:

**1. Table state lives in React** — TanStack Table manages sorting,
filtering, and pagination state. When the user clicks a header,
`onSortingChange` fires.

**2. Query key encodes the state** — TanStack Query's key includes the
table state: `['users', { sort, page, filters }]`. When any state
changes, the key changes, triggering a new fetch.

**3. Server handles the actual operation** — your API receives sort/
filter/page params and applies them in the SQL query. The server returns
a page of data plus the total count.

```
User clicks "Name ▲"
  → setSorting([{ id: 'name', desc: false }])
  → query key changes: ['users', { sortBy: 'name', dir: 'asc', page: 0 }]
  → TanStack Query fetches /api/users?sortBy=name&dir=asc&page=0
  → Server: SELECT * FROM users ORDER BY name ASC LIMIT 25 OFFSET 0
  → Response: { rows: [...], totalCount: 1234 }
  → Table renders new data
```

**`placeholderData: keepPreviousData`** is critical — it shows the
current page (slightly dimmed) while the new data loads. Without it,
every sort/filter/page change shows a loading skeleton.

**Debounce text filters** — if the user types in a filter input, don't
send a request per keystroke. Debounce by 300ms so you send one request
for "alice" instead of five for "a", "al", "ali", "alic", "alice".

---

**Q: How do you make data tables accessible?**

**How to answer:** List the specific ARIA attributes and keyboard
interactions required.

**Answer:**

**What most candidates say:** "Add aria-labels." Vague.

**What a senior candidate says:** Data table accessibility has specific
requirements:

1. **Semantic HTML** — use `<table>`, `<thead>`, `<tbody>`, `<th>`, `<td>`.
   Not divs with `display: table`. Semantic elements give screen readers
   the row/column structure for free.

2. **Column headers** — `<th scope="col">` tells screen readers this
   header applies to a column. Row headers get `scope="row"`.

3. **Sort indication** — `aria-sort="ascending"`, `"descending"`, or
   `"none"` on `<th>`. Screen readers announce "Name, column header,
   sorted ascending."

4. **Sort buttons** — `onClick` on `<th>` is NOT keyboard accessible.
   Put a `<button>` inside the `<th>`. Buttons are focusable and
   respond to Enter/Space by default.

5. **Action buttons** — icon-only buttons (edit, delete) MUST have
   `aria-label`. `<button><TrashIcon /></button>` reads as just "button"
   to a screen reader. Add `aria-label="Delete Alice"`.

6. **Selection checkboxes** — each checkbox needs `aria-label` identifying
   the row it controls. The "select all" checkbox needs a label too.

7. **Live regions** — result count (`aria-live="polite"`) announces
   changes when filters update. "Showing 5 of 100 results."

8. **Focus management** — after deleting a row, focus should move to
   the next row, not jump to `<body>`.

Most developers skip all of this and ship tables that are completely
unusable with keyboards or screen readers. Mentioning these specifics
signals real production experience.

---

**Q: Client-side vs server-side tables — how do you decide?**

**How to answer:** Give the threshold and the trade-offs.

**Answer:**

**What most candidates say:** "Server-side for large data."

**What a senior candidate says:** The decision depends on three factors:

**Data size:** Under ~5,000 rows, client-side is fine. The browser sorts
5K rows in <10ms. Over 5K, client-side sorting/filtering becomes
perceptibly slow, and sending all that data over the network is wasteful.

**Data freshness:** If the data changes frequently (real-time dashboard,
live orders), server-side is better — you always query the current state.
Client-side tables can show stale data if the user doesn't refresh.

**Filter complexity:** If users need full-text search, fuzzy matching,
or joins across tables, the server (with its database indexes) is orders
of magnitude faster than client-side JavaScript.

```
Client-side (manualSorting: false):
  ✓ Instant sort/filter (no network round-trip)
  ✓ Works offline
  ✓ Simpler architecture (no API changes needed)
  ✗ Must load ALL data upfront (slow initial load for large sets)
  ✗ Browser memory for large datasets
  ✗ Can't leverage database indexes

Server-side (manualSorting: true):
  ✓ Only transfers one page of data at a time
  ✓ Database handles sort/filter (fast, indexed)
  ✓ Works for any dataset size
  ✗ Network round-trip on every sort/filter/page change
  ✗ More complex architecture (API must support sort/filter params)
  ✗ Needs placeholderData for good UX
```

**Hybrid approach:** Load the first page server-side, let the user sort/
filter server-side for large datasets, but cache previously-visited
pages in TanStack Query so going back to page 1 is instant.
