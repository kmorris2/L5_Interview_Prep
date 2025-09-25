# V8 DataTable Tech interview 



## Executive summary (30-sec opener)
* What it is: A v8 DataTable built on TanStack Table v8, wrapped with Canvas design tokens + CSS Modules, featuring client/server sorting, row & cell hover modes, row selection, column resizing, fixed header, grid, empty state, and accessible tooltips & controls.
* What’s new vs v7:
    1. Predictable state via context + reducer;
    2. Customizable sort cycles (['asc','desc',null], or per-column) to solve the v7 “unhelpful null” complaint;
    3. Styling isolation with CSS Modules + tokens;
    4. More robust UX (keyboard-resizable columns, improved hover scoping, consistent icons).
* Why it matters: Fewer edge-case bugs, clearer API for teams (server/client), easier theming, and a path to future features (row grouping, virtualization, sticky columns).

## Architecture at a glance
* **Core:** useReactTable() from TanStack v8; row model + sorting model configured per sortMode.
* **State:** TableContext (useReducer) tracks sorting, rowSelection, and tableInstance. Supports controlled (server) and uncontrolled (client) patterns.
* **Columns:** getMergedColumns() composes defaultColDef + column defs + selection column; per-column meta used for width hints.
* **Sorting:**
    * Default cycle: ['asc','desc',null].
    * Global & per-column override via sortOrderCycle and columnSortOrderCycles.
    * Seeded initial sort when cycle excludes null (prevents “first click actually sorts” mismatch).
* **Resizing:**
    * Opt-in enableColumnResizing; resizer element + keyboard buttons; CSS draws a single vertical line (no double borders with grid).
* **Styling:** CSS Modules + Canvas tokens; hover scoped by hoverHighlight prop ('none' | 'row' | 'cell'), fixed header uses position: sticky.

## Key technical decisions & trade-offs
* TanStack v8 vs. rolling custom table
    * Why TanStack: declarative column model, strong plugin surface (sorting, grouping, resizing), battle-tested performance patterns (row models).
    * Trade-off: learning curve + headless API; we add our own opinions for a simpler Canvas API.
* Controlled vs uncontrolled sorting
    * Why both: client tables need quick wins; server tables must reflect backend order and analytics queries.
    * Trade-off: dual code paths; solved via a single onSortingChange dispatcher that routes to context or external prop.
* Custom sort cycles
    * Why: v7 feedback—“null is unhelpful when data is already sorted.” We expose a global and per-column cycle to match business rules.
    * Trade-off: slightly more API surface, but prevents hacks and UX confusion.
* CSS Modules + tokens
    * Why: zero global leakage, theming via tokens, predictable specificity.
    * Trade-off: no global utility classes; solved via small local helpers and semantic classes.

## Performance & scale (what you’ll be asked)
* Rendering
    * Memoized column defs (useMemo), minimal state writes, flexRender only where needed.
    * Avoid prop churn; container style objects kept stable.
* Potential next steps for scale
    * Row virtualization (TanStack Virtual or react-window) for 10k+ rows.
    * Column virtualization for wide tables.
    * Debounced column resize writes; passive pointer listeners.
    * Pure render cells (memoized cell renderers for expensive content).
    * SSR/streaming support for initial paint.
* Sorting at scale
    * Client mode: stable sort with type-aware comparators; slice/paginate before render.
    * Server mode: push sort keys + directions upstream; keep table stateless and cheap.

## Accessibility & UX (be ready with specifics)
* Sort control
    * Hit target is a button with aria-label reflecting the next action (“Sort ascending/descending/Clear sort”).
    * Icon reflects current state; ChevronUpDown used for unsorted.
* ARIA
    * aria-sort can be set on <th> ('ascending'|'descending'|undefined) for SRs.
    * Tooltip triggers are focusable; aria-hidden on decorative icons only.
* Keyboard resizing
    * Resizer buttons provide +/- width changes; live region updates can announce new width (optional).

## Styling details they may probe
* No double borders when enableColumnResizing is on: the right edge shows only the resizer line; grid lines suppressed on the last header cell.
* Fixed header vs grid lines bug: swap the logic so resizing controls vertical lines, not fixedHeader. We added a .resizable class to the <table> and targeted .resizer vs .grid to avoid conflicts.
* Hover scoping
    * hoverHighlight='row' → apply background on the row only;
    * 'cell' → apply to the hovered cell only;
    * Alternate rows use --color-background-subdued; we ensure the hover color uses --color-background-subdued-hover.

## The sorting cycle deep dive (you will likely get asked)
* Problem (v7): Users with pre-sorted data found the “null” cycle step confusing; first click felt like a no-op.
* Solution (v8):
    * Expose sortOrderCycle?: ('asc'|'desc'|null)[] globally and columnSortOrderCycles per column.
    * Initial seeding: if cycle excludes null, auto-apply the first allowed order to the first sortable column on mount—so the icon and data match immediately.
* Edge cases handled
    * Single state only (['asc'] or ['desc']) → icon shows that state; clicking keeps it consistent.
    * ['null'] only → button disabled (optional) or remains noop; icon stays unsorted.
    * Per-column override: e.g., lastName uses ['desc','asc'] while others use defaults.

## Testing strategy (they’ll want proof)
* **Unit (Jest/RTL)**
    * Renders with empty data → header count & empty state copy.
    * Sort cycle transitions: given a cycle, clicking the button emits the expected onSortChange payloads and updates icons.
    * Resizing: width updates propagate to table.setColumnSizing and resizer line visible.
* **E2E (Cypress)**
    * Server mode: clicking sort triggers network call / mock handler; table order updates.
    * Keyboard resizing: focus resizer buttons and adjust width; visual regression on column boundary.
    * Sticky header: scroll retains header; grid lines remain correct.
* **A11y checks**
    * aria-label correctness; tab order; tooltip open/close on focus/blur.

## Extensions they might ask for (have opinions ready)
* Row grouping / aggregation
    * Use TanStack group row model; reuse hover/selection; keep API consistent (groupBy prop or column meta).
* Pinned columns / sticky columns
    * Use CSS position: sticky per cell + extra shadow/border tokens; ensure resizer positioning stays correct.
* Column reordering / hide-show
    * Keep column ID stable; expose a controlled visibleColumns or columnOrder prop; update reducer pattern.
* Inline edit
    * Controlled cell editors with optimistic updates; commit handler in meta.updateData; ESC to revert, Enter to save.
* Pagination & virtualization
    * Prefer server pagination for huge datasets; windowed rows with dynamic row height guardrails.

Common interview questions → succinct answers
* Q: Why TanStack v8?
    * A: Headless, composable, and stable row/column models. It let me keep Canvas APIs simple while leveraging robust internals (sorting, resizing, grouping). Much faster/safer than reinventing grids.
* Q: How did you resolve the v7 “null sort” pain?
    * A: I exposed configurable sort cycles (global & per-column) and auto-seed the first allowed order if null isn’t allowed, so the icon and the data match on initial paint.
* Q: How do you handle server vs client sorting?
    * A: One onChange path. If sortMode='server' and onSortChange is provided, I forward the new SortingState. Otherwise I update context state and let TanStack compute the client result.
* Q: What about performance at 10k rows?
    * A: Memoized column defs + minimal state writes today; for 10k we’d add row virtualization (TanStack Virtual), consider column virtualization for wide tables, and debounced resize events.
* Q: Accessibility?
    * A: Sort controls are buttons with accurate labels describing the next state; aria-sort on headers reflects the current state. Tooltip triggers are focusable; resize is keyboard operable.
* Q: How did you avoid the double borders with resizing?
    * A: Resizer gets its own thin line via .resizable .resizer. When resizing is enabled, I suppress the grid’s right border on header cells (especially the last child) to avoid visual doubling.

______________________________________________________________

## Information about DataTable.tsx:  Architecture & Design
### Purpose / role
This DataTable is a headless-ish UI wrapper around TanStack Table v8 that adds Canvas styling, icons, and UX features (sorting, selection, resizing, hover states, empty state, etc.). It supports both client and server sorting modes and exposes enough hooks (props + context) to scale to advanced use cases (e.g., row grouping/pagination/virtualization later).
### High-level structure
* DataTable<TData>: generic public component; wraps InnerDataTable with a TableProvider so table state can live in React Context (selection/sorting/table instance).
* InnerDataTable<TData>: builds TanStack’s table instance and renders the UI. It:
    * Merges column definitions (getMergedColumns + defaultColDef)
    * Optionally injects a selection column (buildSelectionColumn) when rowSelection is on
    * Chooses sorting mode (client vs server)
    * Wires column resize behavior
    * Renders header / body with flexRender
    * Applies Canvas CSS-module classes
### Why generics? What is TData extends RowWithOptionalId?
* TData is the row shape of the table (e.g., { firstName: string; age: number }).
* It’s generic so the component works with any row shape; TypeScript will keep your accessors and cell renderers type-safe.
* RowWithOptionalId is a small constraint that says: your rows may provide either uuid or id (or neither). The component’s getRowId will use those if present; otherwise it falls back to a deterministic string from the row content.

```
getRowId: row =>
  row.uuid !== undefined ? String(row.uuid)
  : row.id !== undefined   ? String(row.id)
  : JSON.stringify(row)
```
* **Why? **Stable React keys are vital for performance and correct behavior (selection/editing). If your data already has identifiers, we use them. If not, we still produce a stable key.
Interview soundbite: “**TData** lets the table be truly reusable across domains while still type-checking accessor keys and cell renderers. The **RowWithOptionalId** constraint guarantees we can derive stable keys without constraining product teams to a single id field name.”

### State management & modes
**Context + reducer (from TableContext)**
* Sorting and selection live in a React Context so child utilities (e.g., selection column, external controls) can read/update table state without prop drilling.
* Actions: `SET_SORTING, SET_ROW_SELECTION, SET_TABLE_INSTANCE.`
**Controlled vs. Uncontrolled sorting**
* sortMode="client" (default): the component manages sorting itself (uncontrolled). It uses TanStack’s getSortedRowModel.
* sortMode="server": the component does not sort. It calls onSortChange(newSorting), and the parent returns new tableData that’s already sorted. This keeps server-truth + big-data scalability.
* The code supports both by computing the “next sorting” and:    if (props.onSortChange) props.onSortChange(nextSorting)
* else dispatch({ type: 'SET_SORTING', payload: nextSorting })

 
Talking point: “This dual-mode is key to scale. Client mode for small local tables; server mode for pagination and large datasets.”

### Columns, selection, and resizing
**Column merging**
* getMergedColumns(columns, defaultColDef, rowSelection, enableColumnResizing) applies defaultColDef (e.g., common sortable, resizable, width) and any per-column overrides. This creates a small, consistent API for consumers.
**Selection column injection**
* If rowSelection is 'single' | 'multiple', we prepend a synthetic checkbox/radio column via buildSelectionColumn so consumers don’t have to hand-roll one.
**Resizing**
* If enableColumnResizing is true, the table turns on TanStack’s column resizing (columnResizeMode: 'onChange'), sets sensible min/size/max, and renders a resizer handle in the header. Widths come from:    const colMeta = header.column.columnDef.meta as { size?: number } | undefined
* const width = enableColumnResizing ? header.getSize() : (colMeta?.size ?? 'auto')
*   
* This keeps behavior predictable whether resizing is on or not.

________________________________________

### The useEffects (interview-worthy)
1. Sync local data with tableData    
`const [data, setData] = useState(() => [...tableData])`
2. `useEffect(() => setData(tableData), [tableData])`
3.   
    * Copies initial data to support in-place updates (meta.updateData) in client mode
    * When the parent changes tableData (server sorting / new fetch), the effect re-syncs the internal state.
    * Tradeoff: If tableData is large and changes frequently by reference only, this re-assignment is still O(n) but unavoidable if you allow local edit paths. If you don’t need local edits, you could render tableData directly and drop the state.
4. Expose table instance to context    
```
useEffect(() => {
dispatch({ type: 'SET_TABLE_INSTANCE', payload: table })
}, [table, dispatch])
```
5.   
    * Makes the TanStack instance available to helpers (e.g., selection column) and external integrations (keyboard shortcuts, debug tooling).
    * **Why effect?** The instance reference changes when columns/data/state change; the context should track the current instance.

Common interviewer follow-ups:
* Why copy tableData into state? → To support local cell edits and TanStack meta updates. If you don’t need edits, you can avoid the state copy.
* Any pitfalls? → Ensure getRowId is stable; avoid JSON.stringify(row) for very large or non-deterministic objects (but it’s a practical fallback). Consider memoizing heavy cell renderers.
### Extensibility & scalability
* **Extensibility**
    * Adding **row grouping, sticky columns, virtualization, or server pagination** plugs into TanStack’s hooks with minimal changes because state boundaries are already defined (client vs server).
    * Column-level customizations (tooltips, custom headers/cells) flow through columnDef + flexRender.
    * Sort cycle overrides (global + per column) already exist in your later version, letting consumers express domain-specific sort UX.
* **Scalability**
    * **Server mode** avoids loading entire datasets; the table simply emits onSortChange and renders whatever the parent provides.
    * CSS Modules keep styles predictable across many tables on a page.
    * Context-based state makes it easy to add more behaviors without prop drilling.
### Potential improvements / things to be ready to discuss
* **Sorting comparison **
    *  In client mode you rely on TanStack’s default sort types unless you pass sortType. Be ready to mention numeric vs lexical sorts and locale collations.
* **JSON.stringify(row) fallback**
    * It’s fine for demos but can be expensive for very large rows or if the row contains functions/dates (stringifyable but you might get surprises). If teams guarantee IDs, use them.
* **onSortingChange comparison**
    * You’re avoiding redundant dispatches with:

```
const isSame = JSON.stringify(next) === JSON.stringify(current)
 if (isSame) return
 ```

 That’s pragmatic; if performance ever becomes a concern, you can write a tiny shallow comparator for the common single-sort case.


### Quick “what is X” answers
* **TData:** generic row type; keeps accessors and cell renderers type-safe for any domain model.
* **TData extends RowWithOptionalId:** ensures we can derive a stable row key (uuid or id, else content hash).
* **RowWithOptionalId:** a type with optional uuid or id fields (and any other fields your data has).
* **getMergedColumns:** merges consumer columns with defaultColDef, plus feature flags (sortable/resizable/etc.).
* **buildSelectionColumn:** injects a checkbox/radio column when row selection is on.
* **TableProvider / useTableContext:** React Context to share sorting/selection/table instance across subcomponents.
* **flexRender:** TanStack helper to render either strings or JSX cell/header definitions.
* **getCoreRowModel / getSortedRowModel:** TanStack pipelines that compute row order for rendering.
* **manualSorting: sortMode === 'server':** tells TanStack not to sort; you control it externally.


________________________________________________
Information about TableContext.tsx:


What this file is for
It centralizes table UI state (sorting, row selection, and the TanStack table instance) so multiple parts of the DataTable can read/update it without prop-drilling. You’re using a Reducer + Context pattern, which is a solid fit for UI state machines and makes server/client sorting pluggable.
Types & Generics (why they look this way)

TableState<T>

type TableState<T> = {
  sorting: SortingState
  rowSelection: Record<string, boolean>
  tableInstance: Table<T> | null
}


- T is the table row type (same generic used by DataTable<TData>).
- tableInstance is typed as Table<T> | null, so if a consumer knows the row type, they get type-safe access to TanStack APIs (e.g., getRowModel()).

Why createContext<TableContextValue<unknown> | undefined>

const TableContext = createContext<TableContextValue<unknown> | undefined>(undefined)
React contexts cannot truly be generic at runtime—so you store unknown in the context and cast in the hook. That lets each DataTable<TData> instance “treat” the context as TData without forcing a global any.
The hook bridges that gap:

export function useTableContext<T>(): TableContextValue<T> {
  const ctx = useContext(TableContext)
  if (!ctx) throw new Error('useTableContext must be used within TableProvider')
  return ctx as TableContextValue<T>
}
This “generic on the hook” pattern is a common, type-safe compromise in React+TS.
Actions & Reducer

type Action<T> =
  | { type: 'SET_SORTING'; payload: SortingState }
  | { type: 'SET_ROW_SELECTION'; payload: Record<string, boolean> }
  | { type: 'SET_TABLE_INSTANCE'; payload: Table<T> }

function tableReducer<T>(state: TableState<T>, action: Action<T>): TableState<T> {
  // straightforward immutable updates
}
* Clear, isolated mutations make debugging and testing simple.
* Keeping the instance update in the same reducer as sorting/selection guarantees consistent, one-way data flow.
Provider behavior

export function TableProvider<T>({ children }: { children: ReactNode }): JSX.Element {
  const initialState: TableState<T> = { sorting: [], rowSelection: {}, tableInstance: null }

  const [state, dispatch] = useReducer(
    tableReducer as React.Reducer<TableState<T>, Action<T>>,
    initialState,
  )

  const valueForContext = {
    state: state as unknown as TableState<unknown>,
    dispatch: dispatch as unknown as Dispatch<Action<unknown>>,
  }

  return <TableContext.Provider value={valueForContext}>{children}</TableContext.Provider>
}
Why the casts?
Same reason as the unknown above—React context isn’t generic at runtime. You keep the implementation typed and cast at the boundary.
Subtle performance note (good to mention)
valueForContext is a new object each render, which re-renders all consumers on any state change. That’s fine for a table (renders are cheap), but you can optimize:

const valueForContext = useMemo(
  () => ({
    state: state as unknown as TableState<unknown>,
    dispatch: dispatch as unknown as Dispatch<Action<unknown>>,
  }),
  [state, dispatch]
)
Even better, split state & dispatch into two contexts so changing state doesn’t cause components that only use dispatch to re-render:

const StateCtx = createContext<TableState<unknown> | undefined>(undefined)
const DispatchCtx = createContext<Dispatch<Action<unknown>> | undefined>(undefined)

export function TableProvider<T>({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(…) // same as before
  return (
    <DispatchCtx.Provider value={dispatch as Dispatch<Action<unknown>>}>
      <StateCtx.Provider value={state as TableState<unknown>}>{children}</StateCtx.Provider>
    </DispatchCtx.Provider>
  )
}

export function useTableState<T>() { /* read only state */ }
export function useTableDispatch<T>() { /* read only dispatch */ }
This is a common interview follow-up; having it in your pocket shows polish.

How it fits with DataTable.tsx
* InnerDataTable reads and writes sorting & rowSelection through this context (so header buttons, selection cells, and any future controls stay in sync).
* It also publishes the TanStack instance into context:
```
useEffect(() => {
dispatch({ type: 'SET_TABLE_INSTANCE', payload: table })
}, [table, dispatch])
```

*    That makes the instance available to injected columns and future features (keyboard shortcuts, dev tools), without passing it down through props.
Common questions & answers

* Q: Why a reducer instead of multiple useStates?
    * A: Sorting, selection, and the instance are related pieces of UI state; a reducer guarantees updates are predictable and serializable, and gives you a single audit trail (easy to extend with logging or undo/redo if needed).
* Q: Any pitfalls with generics in contexts?
    * A: React can’t carry a runtime generic, so you use unknown at the context boundary and a generic hook to recover types. It’s type-safe at call sites (consumers specify <TData>), and avoids any.
* Q: We saw a “Hooks order changed” warning earlier—what causes that?
    * A: That error appears when hooks are conditionally called or when hook calls are reordered between renders. In this file the order is fixed (useReducer then useContext in consumers). The warning likely came from toggling code in DataTable.tsx (commenting/uncommenting hooks or moving useMemo blocks inside conditions). The rule of thumb: never call hooks inside conditionals or early returns; keep their order static.
* Q: How would you make this more scalable?
    * Split state/dispatch contexts (above) to reduce re-renders.
    * Add selectors (e.g., useSorting(), useRowSelection()) to isolate component subscriptions and improve performance.
    * Accept an optional initialState prop (useful for restoring a saved view).
    * Consider exposing a reset action or a batch helper if multiple updates should be atomic.
Q: Why keep state in context at all? Why not in TanStack only? TanStack owns table mechanics; the Context stores app-level decisions (e.g., single vs. multi select behavior, server vs. client sorting flags, and a stable instance reference). This separation makes it easy to:
    * swap modes (client ↔ server),
    * inject custom columns,
    * and coordinate future features (row grouping, inline edit state) without coupling everything to TanStack internals.

