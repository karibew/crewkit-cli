# Analytics Component System - Design Specification

## Overview

Reusable, accessible, minimalist analytics UI components for crewkit dashboard following Vercel-inspired design principles.

**Design Philosophy**: Functional over fancy, consistency creates calm, speed through simplicity.

---

## Component Architecture

### File Structure

```
src/components/analytics/
├── index.ts                      # Barrel exports
├── USAGE.md                      # Usage documentation
├── DESIGN_SPEC.md                # This file
│
├── kpi/
│   ├── kpi-card.tsx             # Individual KPI metric card
│   └── kpi-grid.tsx             # Responsive grid layout
│
├── data-table/
│   └── advanced-table.tsx       # Sortable, searchable, paginated table
│
├── charts/
│   ├── line-chart.tsx           # Line chart wrapper
│   ├── bar-chart.tsx            # Bar chart wrapper
│   └── area-chart.tsx           # Area chart wrapper
│
├── timeline/
│   └── timeline.tsx             # Vertical event timeline
│
└── filter-bar/
    └── filter-bar.tsx           # Unified filter controls
```

---

## Design Tokens

### Colors (shadcn/ui variables)

**Chart Colors**:
```css
--chart-1: 220 70% 50%;   /* Primary (Blue) */
--chart-2: 160 60% 45%;   /* Success (Green) */
--chart-3: 30 80% 55%;    /* Warning (Orange) */
--chart-4: 280 65% 60%;   /* Purple */
--chart-5: 340 75% 55%;   /* Pink */
```

**KPI Variants**:
- `default`: Standard border/background
- `success`: Green 50/200/500 (light bg, border, text)
- `warning`: Yellow 50/200/500
- `danger`: Red 50/200/500

**Timeline Status**:
- `success`: Green
- `error`: Red
- `warning`: Yellow
- `info`: Blue
- `neutral`: Gray

### Typography

**KPI Card**:
- Title: `text-sm font-medium text-muted-foreground`
- Value: `text-2xl font-bold`
- Description: `text-xs text-muted-foreground`
- Trend: `text-xs font-medium` + status color

**Table**:
- Headers: `text-xs text-gray-700 uppercase bg-gray-50`
- Cells: `text-sm`
- Monospace (cost, IDs): `font-mono`

**Timeline**:
- Title: `font-medium`
- Description: `text-sm text-muted-foreground`
- Timestamp: `text-xs text-muted-foreground`

### Spacing

**Consistent Gap Pattern**:
- Grid gap: `gap-4` (16px)
- Card padding: `p-4` (16px header/content)
- Table padding: `px-6 py-4` (24px horizontal, 16px vertical)
- Timeline gap: `gap-4` between icon and content

**Responsive Breakpoints**:
```tsx
sm: 640px   // Mobile landscape
md: 768px   // Tablet
lg: 1024px  // Desktop
xl: 1280px  // Wide desktop
```

### Borders & Shadows

**Cards**:
- Border: `border border-border` (subtle gray)
- Variant borders: `border-2` for colored variants
- Shadow: `shadow-md` on hover (optional)

**Tables**:
- Outer border: `rounded-md border`
- Row borders: `border-b` (TableRow component default)

**Timeline**:
- Vertical line: `w-px bg-border`
- Icon border: `border-2` + status color

---

## Component Specifications

### 1. KPI Card

**Purpose**: Display single metric with trend and optional sparkline

**Anatomy**:
```
┌─────────────────────────────────┐
│ Title                     [Icon]│  ← Header (pb-2)
├─────────────────────────────────┤
│ 1,234                           │  ← Value (text-2xl font-bold)
│ Optional description            │  ← Description (text-xs)
│ ↑ +12.5% vs last month         │  ← Trend (with icon, color)
│ ▁▂▃▄▃▅▆ (sparkline)            │  ← Sparkline (mt-3, h-8)
└─────────────────────────────────┘
```

**States**:
- **Loading**: Skeleton with same layout
- **Error**: Red border, error message in content area
- **Success/Warning/Danger**: Colored background/border

**Accessibility**:
- Icon has `aria-hidden="true"` (decorative)
- Trend direction conveyed via text (`+` prefix) not just color
- Value is actual text (not image or canvas)

**Variants**:
```tsx
// Default
<KpiCard variant="default" ... />

// Success (green tint)
<KpiCard variant="success" ... />

// Warning (yellow tint)
<KpiCard variant="warning" ... />

// Danger (red tint)
<KpiCard variant="danger" ... />
```

**Responsive**:
- Mobile: Full width, stacks in `KpiGrid`
- Tablet: 2 columns
- Desktop: 3-4 columns

---

### 2. KPI Grid

**Purpose**: Responsive layout for multiple KPI cards

**Layout**:
```tsx
columns={1} // grid-cols-1
columns={2} // grid-cols-1 md:grid-cols-2
columns={3} // grid-cols-1 md:grid-cols-2 lg:grid-cols-3
columns={4} // grid-cols-1 md:grid-cols-2 lg:grid-cols-4
```

**Gap**: `gap-4` (16px)

---

### 3. Advanced Table

**Purpose**: Full-featured data table with sorting, search, pagination, export

**Features**:
- **Search**: Full-text across searchable columns
- **Sort**: Click headers (asc → desc → none)
- **Pagination**: Configurable page sizes
- **Export**: CSV download via callback

**Anatomy**:
```
┌─────────────────────────────────────────────────────┐
│ [Search input]                      [Export button] │  ← Controls
├─────────────────────────────────────────────────────┤
│ │ Name ↑ │ Status │ Cost ↕ │ Date │               │  ← Table
│ ├─────────┼────────┼─────────┼──────┤               │
│ │ ...     │ ...    │ ...     │ ...  │               │
├─────────────────────────────────────────────────────┤
│ Rows: [10▼]    1-10 of 234    [◀◀ ◀ ▶ ▶▶]        │  ← Pagination
└─────────────────────────────────────────────────────┘
```

**Icons**:
- Unsorted: `ArrowUpDown` (muted)
- Ascending: `ArrowUp`
- Descending: `ArrowDown`
- Navigation: `ChevronsLeft`, `ChevronLeft`, `ChevronRight`, `ChevronsRight`

**Empty State**:
```tsx
<TableRow>
  <TableCell colSpan={columns.length} className="text-center py-8">
    No data available
  </TableCell>
</TableRow>
```

**Loading State**:
```tsx
<TableRow>
  <TableCell colSpan={columns.length} className="text-center py-8">
    Loading...
  </TableCell>
</TableRow>
```

**Accessibility**:
- `role="table"` (implicit from Table component)
- Sort buttons are `<th>` with `cursor-pointer`
- Pagination shows current range: "1-10 of 234"
- Screen reader announces sort direction

---

### 4. Chart Components

**Purpose**: Consistent recharts wrappers with shadcn/ui theming

**Common Pattern**:
```tsx
<Card>
  <CardHeader>
    <CardTitle>{title}</CardTitle>
    <CardDescription>{description}</CardDescription>
  </CardHeader>
  <CardContent>
    <ChartContainer config={config}>
      <ResponsiveContainer>
        <[Chart Type]>
          {/* Chart elements */}
        </[Chart Type]>
      </ResponsiveContainer>
    </ChartContainer>
  </CardContent>
</Card>
```

**Chart Configuration**:
```typescript
const chartConfig = {
  dataKey: {
    label: "Display Name",
    color: "hsl(var(--chart-1))",  // Use CSS variables
  },
} satisfies ChartConfig;
```

**Line Chart**:
- Use for: Trends over time
- Style: 2px stroke, no dots (unless < 10 points)
- Colors: From `--chart-*` variables

**Bar Chart**:
- Use for: Categorical comparisons
- Style: 4px rounded corners (top only)
- Stacking: Optional via `stacked` prop

**Area Chart**:
- Use for: Cumulative metrics
- Style: 20% fill opacity, 2px stroke
- Stacking: Optional via `stacked` prop

**Shared Features**:
- Grid: `strokeDasharray="3 3"` (dashed)
- Axes: No tick lines, no axis lines
- Tooltip: shadcn/ui `ChartTooltipContent`
- Responsive: `ResponsiveContainer` wrapper

---

### 5. Timeline

**Purpose**: Vertical event timeline with expandable details

**Anatomy**:
```
    ┌───┐
    │ ● │  Event Title [Status Badge]          2h ago
    │ │ │  Event description
    │ │ │  [▶ Show details]
    │ │ │
    ┌─┴─┐
    │ ● │  Another Event                        1h ago
    │ │ │  Description
    │ │ │  [▼ Hide details]
    │ │ │  ┌──────────────────────┐
    │ │ │  │ Expanded content     │
    │ │ │  │ Key: Value           │
    │ │ │  └──────────────────────┘
    │ │ │
    ┌─┴─┐
    │ ● │  Latest Event                         10m ago
    └───┘
```

**Icon Sizes**:
- Container: `h-9 w-9` (36px circle)
- Icon: `h-4 w-4` (16px)
- Line: `w-px` (1px vertical)

**Expandable Details**:
- Button: `text-sm text-muted-foreground hover:text-foreground`
- Icon: `ChevronRight` (collapsed) / `ChevronDown` (expanded)
- Content: `Card` with `bg-muted/50` background

**Status Colors** (icon background + border):
- Success: Green
- Error: Red
- Warning: Yellow
- Info: Blue
- Neutral: Gray

**Metadata Display**:
```tsx
<dl className="grid grid-cols-2 gap-2 text-sm">
  <dt className="text-muted-foreground">Key</dt>
  <dd className="font-medium font-mono">Value</dd>
</dl>
```

---

### 6. Filter Bar

**Purpose**: Unified filter controls with consistent layout

**Anatomy**:
```
┌──────────────────────────────────────────────────────┐
│ [🔍 Search...] [Agent ▼] [Status ▼] [Date Range ▼] │
│                                      [✕ Clear all]   │
└──────────────────────────────────────────────────────┘
```

**Filter Types**:

**Search**:
- Icon: `Search` (left-aligned, `pl-9`)
- Debounce: 300ms (implement in parent component)

**Select**:
- Component: shadcn/ui `Select`
- Placeholder: Shows when no value selected
- Width: `min-w-[180px]`

**Date Range**:
- Component: shadcn/ui `Calendar` in `Popover`
- Format: "Jan 15, 2025 - Feb 14, 2025"
- Width: `min-w-[260px]` (fits formatted range)

**Clear All Button**:
- Visible: Only when filters active
- Icon: `X`
- Variant: `ghost`

**Responsive**:
- Mobile: Stacks vertically (`flex-col`)
- Desktop: Horizontal row (`sm:flex-row`)

---

## Accessibility Standards

All components meet **WCAG 2.1 AA** standards.

### Keyboard Navigation

**Table**:
- Tab: Navigate between controls (search, page size, pagination)
- Enter: Activate button/select
- Arrow Keys: Navigate select options

**Filters**:
- Tab: Move between inputs
- Arrow Keys: Navigate select/calendar
- Escape: Close popover

**Timeline**:
- Tab: Focus expand/collapse buttons
- Enter/Space: Toggle details

### Screen Reader Support

**KPI Card**:
```tsx
<CardTitle className="text-sm font-medium">
  Total Sessions
</CardTitle>
<Icon className="..." aria-hidden="true" />  {/* Decorative */}
<div className="text-2xl font-bold">1,234</div>
```

**Table**:
```tsx
<TableHead onClick={handleSort} aria-sort={sortDirection}>
  Name
  <SortIcon />  {/* Visual only, direction in aria-sort */}
</TableHead>
```

**Timeline**:
```tsx
<button aria-expanded={expanded}>
  {expanded ? "Hide" : "Show"} details
</button>
```

### Focus Management

- **Visible Focus**: Default browser outline or `focus-visible:ring-2`
- **Focus Trap**: Modals/popovers trap focus
- **Focus Return**: Returns to trigger after close

### Color Contrast

**Minimum Ratios** (WCAG AA):
- Normal text: 4.5:1
- Large text (18pt+): 3:1
- UI components: 3:1

**Tested Combinations**:
- `text-foreground` on `bg-background`: 15:1 ✓
- `text-muted-foreground` on `bg-background`: 7:1 ✓
- Chart colors on grid: 5:1+ ✓

---

## Performance Considerations

### Table

**Optimization**:
- Client-side pagination (< 1000 rows)
- Server-side pagination (1000+ rows)
- Virtualization (10,000+ rows) - use `@tanstack/react-virtual`

**Re-render Prevention**:
```tsx
const columns = useMemo(() => [...], []);  // Stable reference
const filteredData = useMemo(() => ..., [data, filters]);
```

### Charts

**Data Limits**:
- Line/Area: 100-200 points (smooth)
- Bar: 50-100 bars (readable)
- Larger datasets: Aggregate or sample

**Responsive Container**:
- Uses `ResizeObserver` (built into recharts)
- Debounced resize (< 16ms)

### Timeline

**Pagination**:
- Show 20-30 items by default
- "Load more" button for additional events
- Virtualize if 100+ items

---

## Testing Strategy

### Unit Tests

**KPI Card**:
```tsx
it("renders value and description", () => {
  render(<KpiCard title="Test" value={100} description="desc" />);
  expect(screen.getByText("100")).toBeInTheDocument();
  expect(screen.getByText("desc")).toBeInTheDocument();
});

it("shows loading skeleton", () => {
  render(<KpiCard title="Test" value={0} loading />);
  expect(screen.getByRole("status")).toBeInTheDocument();
});
```

**Table**:
```tsx
it("sorts ascending when clicked", () => {
  render(<AdvancedTable data={data} columns={columns} />);
  const header = screen.getByText("Name");
  fireEvent.click(header);
  expect(screen.getAllByRole("cell")[0]).toHaveTextContent("Alice");
});

it("filters by search term", () => {
  render(<AdvancedTable data={data} columns={columns} />);
  const search = screen.getByPlaceholderText("Search...");
  fireEvent.change(search, { target: { value: "Bob" } });
  expect(screen.getAllByRole("row")).toHaveLength(2); // Header + 1 match
});
```

### Integration Tests

**Page with Analytics**:
```tsx
it("displays KPIs and updates table on filter", async () => {
  render(<SessionAnalyticsPage />);

  // KPIs load
  await waitFor(() => {
    expect(screen.getByText("Total Sessions")).toBeInTheDocument();
  });

  // Filter table
  const agentFilter = screen.getByRole("combobox", { name: /agent/i });
  fireEvent.change(agentFilter, { target: { value: "rails-expert" } });

  await waitFor(() => {
    expect(screen.getByText("rails-expert")).toBeInTheDocument();
    expect(screen.queryByText("frontend-expert")).not.toBeInTheDocument();
  });
});
```

### Accessibility Tests

**Axe Core**:
```tsx
import { axe, toHaveNoViolations } from "jest-axe";

it("has no accessibility violations", async () => {
  const { container } = render(<KpiCard title="Test" value={100} />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

---

## Migration Guide

### From Existing Components

**Stats Card → KPI Card**:
```tsx
// Before
<StatsCard
  title="Sessions"
  value={1234}
  description="Last 30 days"
  icon={Users}
  trend={{ value: 12.5, positive: true }}
/>

// After
<KpiCard
  title="Sessions"
  value={1234}
  description="Last 30 days"
  icon={Users}
  trend={{ value: 12.5 }}  // Direction auto-calculated
/>
```

**Session Table → Advanced Table**:
```tsx
// Before
<SessionTable sessions={sessions} />

// After
const columns: ColumnDef<Session>[] = [
  { key: "agent_name", header: "Agent", sortable: true, searchable: true },
  { key: "total_cost", header: "Cost", sortable: true, render: (row) => `$${row.total_cost.toFixed(4)}` },
];

<AdvancedTable
  data={sessions}
  columns={columns}
  keyExtractor={(row) => row.id}
/>
```

---

## Future Enhancements

### Phase 1 (Current)
- ✅ KPI Cards with trends/sparklines
- ✅ Advanced table with sorting/search/pagination
- ✅ Chart wrappers (Line, Bar, Area)
- ✅ Timeline with expandable details
- ✅ Filter bar with date range

### Phase 2 (Next)
- [ ] Multi-select filter (checkbox dropdown)
- [ ] Table column visibility toggle
- [ ] Table column reordering (drag-drop)
- [ ] Export to JSON (in addition to CSV)
- [ ] Chart legend customization

### Phase 3 (Later)
- [ ] Real-time chart updates (WebSocket)
- [ ] Virtual scrolling for large tables
- [ ] Pie/Donut chart component
- [ ] Heatmap chart component
- [ ] Advanced timeline (branching, grouping)

---

## Related Documentation

- [shadcn/ui Documentation](https://ui.shadcn.com)
- [Recharts Documentation](https://recharts.org)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [crewkit Design System](../../../CLAUDE.md) - Organization-level design standards