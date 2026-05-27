# Analytics Component System - Visual Summary

## Component Tree

```
Analytics Component System
│
├── KPI Components
│   ├── KpiCard
│   │   ├── Props: title, value, description, icon, variant, trend, sparklineData
│   │   ├── Variants: default, success, warning, danger
│   │   ├── States: normal, loading, error
│   │   └── Features: trend indicators, sparklines, icons
│   │
│   └── KpiGrid
│       ├── Props: children, columns (1-4), className
│       └── Responsive: Mobile (1 col) → Tablet (2 col) → Desktop (3-4 col)
│
├── Data Table
│   └── AdvancedTable<T>
│       ├── Props: data, columns, keyExtractor, pageSize, onExport
│       ├── Features: sorting, searching, pagination, export
│       ├── States: normal, loading, empty
│       └── Column Config: sortable, searchable, custom render
│
├── Charts (Recharts Wrappers)
│   ├── AnalyticsLineChart
│   │   ├── Props: data, xKey, yKeys, config, height
│   │   └── Use Case: Trends over time
│   │
│   ├── AnalyticsBarChart
│   │   ├── Props: data, xKey, yKeys, config, stacked
│   │   └── Use Case: Categorical comparisons
│   │
│   └── AnalyticsAreaChart
│       ├── Props: data, xKey, yKeys, config, stacked
│       └── Use Case: Cumulative metrics
│
├── Timeline
│   └── Timeline
│       ├── Props: items (TimelineItemData[])
│       ├── Item Props: id, title, description, timestamp, icon, status, details, metadata
│       ├── Status: success, error, warning, info, neutral
│       └── Features: expandable details, metadata display, icons, vertical line
│
└── Filter Bar
    └── FilterBar
        ├── Props: filters, values, onChange, onClear
        ├── Filter Types: search, select, multi-select, date-range
        └── Features: responsive layout, clear all button, dynamic visibility
```

## Visual Layout Examples

### KPI Grid (4 columns, desktop)

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ Total        │ Total Cost   │ Success Rate │ Avg Duration │
│ Sessions     │              │              │              │
│              │              │              │              │
│ 1,234        │ $456.78      │ 94.2%        │ 12m 34s      │
│              │              │              │              │
│ ↑ +12.5%     │ ↓ -8.3%      │ ↑ +2.1%      │ ↓ -5.2%      │
│ vs last mo   │ cost red.    │              │ faster       │
│              │              │              │              │
│ ▁▂▃▄▃▅▆     │ ▅▄▅▃▃▂▁     │              │              │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

### Advanced Table

```
┌────────────────────────────────────────────────────────────┐
│ [🔍 Search...]                          [⬇ Export]         │
├────────────────────────────────────────────────────────────┤
│ Agent ↑       │ Outcome   │ Cost    │ Turns │ Started     │
├───────────────┼───────────┼─────────┼───────┼─────────────┤
│ 🤖 rails-exp │ Success ✓ │ $0.4523 │ 12    │ 2h ago      │
│ 🤖 frontend  │ Success ✓ │ $0.3241 │ 8     │ 4h ago      │
│ 🤖 api-desgn │ Failure ✗ │ $0.1234 │ 15    │ 6h ago      │
├────────────────────────────────────────────────────────────┤
│ Rows: [10▼]        1-10 of 234        [◀◀ ◀ ▶ ▶▶]        │
└────────────────────────────────────────────────────────────┘
```

### Line Chart

```
┌────────────────────────────────────────────────────────────┐
│ Session Activity                                           │
│ Sessions and cost over the last week                       │
├────────────────────────────────────────────────────────────┤
│                                                            │
│   60 ┤                       ●                            │
│      │                   ●       ●                        │
│   40 ┤               ●               ●                    │
│      │           ●                       ●                │
│   20 ┤       ●                                            │
│      │   ●                                                │
│    0 ┴────────────────────────────────────────            │
│       Mon  Tue  Wed  Thu  Fri  Sat  Sun                   │
│                                                            │
│       ── Sessions    ── Cost ($)                          │
└────────────────────────────────────────────────────────────┘
```

### Timeline

```
    ┌───┐
    │ ● │  Session started                              2h ago
    │ │ │  rails-expert agent initialized
    │ │ │
    ┌─┴─┐
    │ ● │  Code modification [Success]                  1h ago
    │ │ │  Updated user_controller.rb
    │ │ │  [▼ Hide details]
    │ │ │  ┌─────────────────────────────────────────┐
    │ │ │  │ + Added email validation                │
    │ │ │  │ + Improved error handling               │
    │ │ │  │                                          │
    │ │ │  │ File: app/controllers/user_controller.rb│
    │ │ │  │ Lines: +24, -8                           │
    │ │ │  └─────────────────────────────────────────┘
    │ │ │
    ┌─┴─┐
    │ ⚠ │  Warning: Performance issue [Warning]        45m ago
    │ │ │  N+1 query detected
    │ │ │  [▶ Show details]
    │ │ │
    ┌─┴─┐
    │ ✓ │  Session completed [Success]                 30m ago
    └───┘
```

### Filter Bar

```
┌────────────────────────────────────────────────────────────┐
│ [🔍 Search sessions...]  [Agent ▼]  [Outcome ▼]           │
│                                                             │
│ [📅 Jan 15, 2025 - Feb 14, 2025 ▼]      [✕ Clear all]    │
└────────────────────────────────────────────────────────────┘
```

## Props API Quick Reference

### KpiCard

```typescript
interface KpiCardProps {
  title: string;
  value: string | number;
  description?: string;
  icon?: LucideIcon;
  variant?: "default" | "success" | "warning" | "danger";
  trend?: {
    value: number;
    direction?: "up" | "down" | "neutral";
    label?: string;
  };
  sparklineData?: number[];
  loading?: boolean;
  error?: string;
}
```

### AdvancedTable

```typescript
interface AdvancedTableProps<T> {
  data: T[];
  columns: ColumnDef<T>[];
  keyExtractor: (row: T) => string | number;
  emptyMessage?: string;
  searchPlaceholder?: string;
  pageSize?: number;
  pageSizeOptions?: number[];
  onExport?: (data: T[]) => void;
  loading?: boolean;
}

interface ColumnDef<T> {
  key: string;
  header: string;
  sortable?: boolean;
  searchable?: boolean;
  render?: (row: T) => React.ReactNode;
  className?: string;
}
```

### Charts (Common Props)

```typescript
interface ChartProps {
  title?: string;
  description?: string;
  data: Record<string, unknown>[];
  xKey: string;
  yKeys: string[];
  config: ChartConfig;
  height?: number;
  showGrid?: boolean;
  showTooltip?: boolean;
  stacked?: boolean;  // Bar/Area charts only
}
```

### Timeline

```typescript
interface TimelineProps {
  items: TimelineItemData[];
}

interface TimelineItemData {
  id: string;
  title: string;
  description?: string;
  timestamp: string;
  icon?: LucideIcon;
  status?: "success" | "error" | "warning" | "info" | "neutral";
  details?: ReactNode;
  metadata?: Record<string, string | number>;
}
```

### FilterBar

```typescript
interface FilterBarProps {
  filters: FilterConfig[];
  values: FilterValue;
  onChange: (values: FilterValue) => void;
  onClear?: () => void;
}

interface FilterConfig {
  id: string;
  type: "select" | "multi-select" | "search" | "date-range";
  label: string;
  placeholder?: string;
  options?: FilterOption[];
}

type FilterValue = {
  [key: string]: string | string[] | DateRange | undefined;
};
```

## Color Palette

### Chart Colors (CSS Variables)

```css
--chart-1: 220 70% 50%;   /* Primary (Blue) */
--chart-2: 160 60% 45%;   /* Success (Green) */
--chart-3: 30 80% 55%;    /* Warning (Orange) */
--chart-4: 280 65% 60%;   /* Purple */
--chart-5: 340 75% 55%;   /* Pink */
```

### KPI Variants

```
Default:  Border: gray-200   Background: white
Success:  Border: green-200  Background: green-50
Warning:  Border: yellow-200 Background: yellow-50
Danger:   Border: red-200    Background: red-50
```

### Timeline Status

```
Success:  Green  (border: green-500,  bg: green-100)
Error:    Red    (border: red-500,    bg: red-100)
Warning:  Yellow (border: yellow-500, bg: yellow-100)
Info:     Blue   (border: blue-500,   bg: blue-100)
Neutral:  Gray   (border: gray-400,   bg: gray-100)
```

## Responsive Breakpoints

```
Mobile:       < 640px   (sm)
Tablet:       640-1024px (md to lg)
Desktop:      > 1024px  (lg+)
Wide Desktop: > 1280px  (xl+)
```

### Component Behavior

**KpiGrid**:
- Mobile: 1 column (stacked)
- Tablet: 2 columns
- Desktop: 3-4 columns (based on `columns` prop)

**AdvancedTable**:
- Mobile: Horizontal scroll for wide tables
- Tablet: Full table visible
- Desktop: Full table + controls side-by-side

**FilterBar**:
- Mobile: Stacked vertically
- Desktop: Horizontal row

**Charts**:
- All breakpoints: `ResponsiveContainer` maintains aspect ratio

## Accessibility Features

### Keyboard Navigation

- **Tab**: Navigate between interactive elements
- **Enter/Space**: Activate buttons, toggle details
- **Arrow Keys**: Navigate select menus, calendar
- **Escape**: Close popovers, cancel actions

### Screen Reader Support

- Semantic HTML (`<table>`, `<button>`, `<time>`)
- ARIA labels (`aria-label`, `aria-describedby`)
- ARIA states (`aria-expanded`, `aria-sort`)
- Live regions (`aria-live` for dynamic content)

### Color Contrast

All text meets **WCAG 2.1 AA** standards:
- Normal text: 4.5:1 minimum
- Large text: 3:1 minimum
- UI components: 3:1 minimum

## Common Use Cases

### Session Analytics Dashboard

```tsx
<KpiGrid columns={4}>
  <KpiCard title="Sessions" value={total} icon={Users} trend={...} />
  <KpiCard title="Cost" value={cost} icon={DollarSign} variant="success" />
  <KpiCard title="Success Rate" value={rate} icon={TrendingUp} />
  <KpiCard title="Duration" value={duration} icon={Clock} />
</KpiGrid>

<AnalyticsLineChart data={trends} xKey="date" yKeys={["sessions", "cost"]} />

<FilterBar filters={filterConfig} values={filters} onChange={setFilters} />
<AdvancedTable data={sessions} columns={columns} keyExtractor={...} />
```

### Experiment Comparison

```tsx
<KpiGrid columns={2}>
  <KpiCard title="Control Success" value="92%" variant="success" />
  <KpiCard title="Variant Success" value="95%" variant="success" />
</KpiGrid>

<AnalyticsBarChart
  data={comparisonData}
  xKey="metric"
  yKeys={["control", "variant"]}
  stacked={false}
/>
```

### Agent Performance Timeline

```tsx
<Timeline items={sessionEvents} />
```

### Cost Tracking Over Time

```tsx
<AnalyticsAreaChart
  data={costData}
  xKey="date"
  yKeys={["cost"]}
  config={chartConfig}
  stacked={false}
/>
```

## Testing

All components include:
- Unit tests (Vitest + React Testing Library)
- Accessibility tests (jest-axe)
- Integration tests (user flows)

See `DESIGN_SPEC.md` for testing strategy.

## Performance

**Table**:
- Client-side: < 1000 rows
- Server-side: 1000+ rows
- Virtualization: 10,000+ rows

**Charts**:
- Optimal: 100-200 data points
- Maximum: 500 data points

**Timeline**:
- Display: 20-30 items
- Pagination: Load more pattern

## Browser Support

Same as Next.js 16:
- Chrome (latest)
- Firefox (latest)
- Safari (latest)
- Edge (latest)

## Dependencies

```json
{
  "recharts": "^2.15.4",         // Charts
  "date-fns": "^4.1.0",          // Date formatting
  "react-day-picker": "^9.x",    // Date range picker (NEW)
  "lucide-react": "^0.556.0",    // Icons
  "@radix-ui/react-popover": "^1.1.15"  // Popover (NEW)
}
```

## Next Steps

1. Install `react-day-picker`: `npm install react-day-picker`
2. Review `USAGE.md` for detailed API documentation
3. Explore `examples/session-analytics-example.tsx`
4. Integrate components into your pages
5. Customize chart colors in `globals.css`

---

**Questions?** See [USAGE.md](./USAGE.md) or [DESIGN_SPEC.md](./DESIGN_SPEC.md)