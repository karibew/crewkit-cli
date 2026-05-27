# Session Analytics Dashboard - Component Architecture Design

## Executive Summary

This document provides comprehensive component architecture and design patterns for the Session Analytics Dashboard in crewkit. It outlines reusable components, design system integration, data flow, and implementation strategy.

**Status**: Backend API complete (8 endpoints), Frontend implementation in progress
**Target**: `/kit/analytics` page with KPIs, charts, and filters
**Design Philosophy**: Vercel-inspired minimalism with functional, consistent components

---

## Table of Contents

1. [Design System Audit](#design-system-audit)
2. [Component Hierarchy](#component-hierarchy)
3. [Component Design Patterns](#component-design-patterns)
4. [Page Layout Structure](#page-layout-structure)
5. [Data Fetching Strategy](#data-fetching-strategy)
6. [Reusable Component Library](#reusable-component-library)
7. [Design System Consistency](#design-system-consistency)
8. [File Structure](#file-structure)
9. [Implementation Checklist](#implementation-checklist)
10. [Design Decisions & Rationale](#design-decisions--rationale)

---

## Design System Audit

### Existing UI Primitives (shadcn/ui)

**Card Components** (`components/ui/card.tsx`):
```tsx
Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter, CardAction
```
- Grid-based slot system with `data-slot` attributes
- Container-query support via `@container/card-header`
- Consistent padding (`py-6`, `px-6`) and rounded corners (`rounded-xl`)
- Shadow and border styling

**Chart Components** (`components/ui/chart.tsx`):
```tsx
ChartContainer, ChartTooltip, ChartTooltipContent, ChartLegend, ChartLegendContent
```
- CSS variable-based theming (`--color-{key}`)
- Responsive container with aspect ratio control
- Dark/light mode support via THEMES object
- Recharts integration layer

**Form Components**:
- `Select`, `SelectContent`, `SelectItem`, `SelectTrigger`, `SelectValue`
- `Popover`, `PopoverTrigger`, `PopoverContent`
- `Calendar` (date picker with react-day-picker)
- `Button` with multiple variants

**Feedback Components**:
- `Badge` (variants: default, secondary, destructive, outline)
- `Alert`, `AlertTitle`, `AlertDescription`
- `Skeleton` (loading placeholders)
- `Tabs`, `TabsList`, `TabsTrigger`, `TabsContent`

### Existing Feature Components

**KPI Card Pattern 1** (`features/dashboard/stats-card.tsx`):
```tsx
interface StatsCardProps {
  title: string;
  value: string | number;
  description?: string;
  icon: LucideIcon;
  trend?: { value: number; positive: boolean };
}
```
- Simple, single-purpose
- No loading states
- No variants
- Trend as percentage change

**KPI Card Pattern 2** (`features/admin/admin-stats-cards.tsx`):
```tsx
function StatsCard({
  title, value, description, icon,
  variant?: "default" | "warning",
  highlight?: boolean
}) { ... }
```
- Variant support (warning, highlight)
- Color-coded backgrounds
- Multiple stat grids (overview, activity, roles)
- Section headers

**Enhanced KPI Pattern** (`analytics/kpi/kpi-card.tsx`):
```tsx
export interface KpiCardProps {
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
  className?: string;
}
```
- Most feature-rich KPI component
- 4 variants with color coding
- Trend direction indicators (TrendingUp, TrendingDown, Minus icons)
- Inline sparklines
- Loading and error states
- **CHOSEN PATTERN** for analytics dashboard

### Existing Chart Patterns

**Generic Chart Wrappers** (`analytics/charts/`):
- `line-chart.tsx` - Configurable line chart with title/description
- `bar-chart.tsx` - Configurable bar chart
- `area-chart.tsx` - Configurable area chart

**Pattern**:
```tsx
export function AnalyticsLineChart({
  title,
  description,
  data,
  xKey,
  yKeys,
  config,
  height = 350,
  showGrid = true,
  showTooltip = true,
  className,
}: LineChartProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
        <CardDescription>{description}</CardDescription>
      </CardHeader>
      <CardContent>
        <ChartContainer config={config}>
          <LineChart data={data} height={height}>
            {/* Recharts components */}
          </LineChart>
        </ChartContainer>
      </CardContent>
    </Card>
  );
}
```

**Feature-Specific Charts** (`features/analytics/`):
- `session-time-series-chart.tsx` - Session activity over time
- `agent-performance-chart.tsx` - Agent comparison
- `cost-analysis-chart.tsx` - Cost trends
- `outcome-distribution-chart.tsx` - Outcome pie chart

**Pattern**: Domain-specific charts with built-in data formatting, tooltips, empty states

### Filter Pattern (`features/sessions/session-filters.tsx`)

```tsx
export function SessionFilters({
  status, outcome, agent,
  onStatusChange, onOutcomeChange, onAgentChange,
}: SessionFiltersProps) {
  return (
    <div className="flex flex-col sm:flex-row gap-4">
      <Select value={status} onValueChange={onStatusChange}>
        <SelectTrigger>
          <SelectValue placeholder="All statuses" />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="all">All statuses</SelectItem>
          {/* ... */}
        </SelectContent>
      </Select>
      {/* ... more filters */}
    </div>
  );
}
```

**Pattern**: Controlled components with onChange callbacks, responsive flex layout

---

## Component Hierarchy

### Full Page Structure

```
/kit/analytics/page.tsx
│
├── DashboardShell (layout wrapper)
│   ├── title: "Analytics"
│   ├── description: "Session insights and performance metrics"
│   └── actions: (none)
│
├── Filters Section
│   ├── DateRangePicker
│   │   ├── Popover (shadcn)
│   │   │   ├── Button (trigger)
│   │   │   └── PopoverContent
│   │   │       ├── Preset Buttons (7d, 30d, 90d, 6mo, 1yr)
│   │   │       └── Calendar (react-day-picker)
│   │   └── State: { start: Date, end: Date }
│   │
│   └── AgentFilterSelect
│       ├── Select (shadcn)
│       │   ├── SelectTrigger
│       │   └── SelectContent
│       │       ├── SelectItem value="all" (All agents)
│       │       └── SelectItem for each agent (from API)
│       └── State: string (agent name or "all")
│
├── KPI Section
│   └── AnalyticsKpiCards
│       ├── Primary KPIs (grid: 4 columns)
│       │   ├── KpiCard: Total Sessions (Activity icon)
│       │   ├── KpiCard: Success Rate (TrendingUp icon)
│       │   ├── KpiCard: Avg Cost (DollarSign icon)
│       │   ├── KpiCard: Avg Turns (MessageSquare icon)
│       │   ├── KpiCard: Avg Duration (Clock icon)
│       │   └── KpiCard: P95 Cost (DollarSign icon)
│       │
│       └── Outcome KPIs (grid: 3 columns)
│           ├── KpiCard: Success Count (CheckCircle2 icon, variant="success")
│           ├── KpiCard: Failure Count (XCircle icon, variant="danger")
│           └── KpiCard: Abandoned Count (AlertCircle icon, variant="warning")
│
└── Tabbed Charts Section
    ├── Tabs (shadcn)
    │   ├── TabsList
    │   │   ├── TabsTrigger: "Overview"
    │   │   ├── TabsTrigger: "Performance"
    │   │   └── TabsTrigger: "Costs"
    │   │
    │   ├── TabsContent: "overview"
    │   │   └── Grid (3 columns on desktop)
    │   │       ├── Card (col-span-2)
    │   │       │   ├── CardHeader
    │   │       │   │   ├── CardTitle: "Session Activity"
    │   │       │   │   └── CardDescription: "Sessions over time with outcome breakdown"
    │   │       │   └── CardContent
    │   │       │       └── SessionTimeSeriesChart
    │   │       │           └── LineChart (4 lines: total, success, failure, abandoned)
    │   │       │
    │   │       └── Card (col-span-1)
    │   │           ├── CardHeader
    │   │           │   └── CardTitle: "Outcome Distribution"
    │   │           └── CardContent
    │   │               └── OutcomeDistributionChart
    │   │                   └── PieChart (success, failure, abandoned)
    │   │
    │   ├── TabsContent: "performance"
    │   │   └── Card (full width)
    │   │       ├── CardHeader
    │   │       │   ├── CardTitle: "Agent Performance Comparison"
    │   │       │   └── CardDescription: "Success rates, avg turns, and duration by agent"
    │   │       └── CardContent
    │   │           └── AgentPerformanceChart
    │   │               └── BarChart (multi-bar: success rate, avg turns, sessions)
    │   │
    │   └── TabsContent: "costs"
    │       └── Card (full width)
    │           ├── CardHeader
    │           │   ├── CardTitle: "Cost Analysis"
    │           │   └── CardDescription: "Cost trends and distribution across sessions"
    │           └── CardContent
    │               └── CostAnalysisChart
    │                   └── AreaChart (dual area: avg cost, total cost)
```

---

## Component Design Patterns

### Pattern 1: Enhanced KPI Card

**Component**: `components/analytics/kpi/kpi-card.tsx` (✅ Already exists)

**Usage**:
```tsx
import { KpiCard } from "@/components/analytics/kpi/kpi-card";
import { Activity } from "lucide-react";

<KpiCard
  title="Total Sessions"
  value={stats?.total_sessions || 0}
  description="All sessions in date range"
  icon={Activity}
  variant="default"
  trend={{
    value: 12.5,
    direction: "up",
    label: "vs last period"
  }}
  loading={isLoading}
  error={error?.message}
/>
```

**Features**:
- 4 variants: default (gray), success (green), warning (orange), danger (red)
- Trend indicators with directional icons
- Optional sparkline graphs
- Built-in loading states (Skeleton)
- Built-in error states (red border + error message)
- Flexible icon support

**When to use**:
- Displaying single metrics (totals, averages, percentages)
- Need color-coded status (success/warning/danger)
- Want to show trends over time
- Require loading/error handling

### Pattern 2: Generic Chart Wrapper

**Component**: `components/analytics/charts/line-chart.tsx` (✅ Already exists)

**Usage**:
```tsx
import { AnalyticsLineChart } from "@/components/analytics/charts/line-chart";

const chartConfig = {
  total: { label: "Total", color: "hsl(var(--primary))" },
  success: { label: "Success", color: "hsl(142, 76%, 36%)" },
};

<AnalyticsLineChart
  title="Session Activity"
  description="Sessions over time"
  data={timeSeries}
  xKey="date"
  yKeys={["total", "success"]}
  config={chartConfig}
  height={300}
  showGrid={true}
  showTooltip={true}
/>
```

**Features**:
- Card wrapper with title/description
- Recharts integration via ChartContainer
- Theme-aware colors via CSS variables
- Configurable height, grid, tooltip
- Responsive by default

**When to use**:
- Need a chart in a new feature area (experiments, resources, etc.)
- Want consistent styling without custom chart code
- Require flexibility for different data shapes

### Pattern 3: Feature-Specific Chart

**Component**: `components/features/analytics/session-time-series-chart.tsx` (✅ Already exists)

**Usage**:
```tsx
import { SessionTimeSeriesChart } from "@/components/features/analytics/session-time-series-chart";

<SessionTimeSeriesChart data={timeSeriesResponse?.data || []} />
```

**Features**:
- Domain-specific (knows about sessions, outcomes, dates)
- Pre-configured colors, labels, tooltips
- Built-in empty state handling
- Date formatting (MMM d)
- No configuration needed

**When to use**:
- Displaying session analytics specifically
- Want opinionated, ready-to-use chart
- Don't need to customize behavior

### Pattern 4: Filter Component

**Component**: `components/features/analytics/date-range-picker.tsx` (✅ Already exists)

**Usage**:
```tsx
import { DateRangePicker } from "@/components/features/analytics/date-range-picker";
import { subDays } from "date-fns";

const [dateRange, setDateRange] = useState({
  start: subDays(new Date(), 30),
  end: new Date(),
});

<DateRangePicker value={dateRange} onChange={setDateRange} />
```

**Features**:
- Popover UI with calendar
- Preset buttons (7d, 30d, 90d, 6mo, 1yr)
- Formatted display ("Jan 1 - Jan 31, 2026")
- Controlled component pattern

**When to use**:
- Date range filtering
- Need quick presets + custom range

---

## Page Layout Structure

### Responsive Grid Patterns

**KPI Grid** (6 primary + 3 outcome cards):
```tsx
{/* Primary KPIs: 2 cols on tablet, 4 on desktop */}
<div className="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
  <KpiCard {...} />
  <KpiCard {...} />
  <KpiCard {...} />
  <KpiCard {...} />
  <KpiCard {...} />
  <KpiCard {...} />
</div>

{/* Outcome KPIs: 1 col on mobile, 3 on tablet+ */}
<div className="grid gap-4 md:grid-cols-3">
  <KpiCard variant="success" {...} />
  <KpiCard variant="danger" {...} />
  <KpiCard variant="warning" {...} />
</div>
```

**Chart Grid** (2/3 + 1/3 layout):
```tsx
{/* Overview Tab: Large chart + small chart */}
<div className="grid gap-4 lg:grid-cols-3">
  <Card className="lg:col-span-2">
    {/* SessionTimeSeriesChart */}
  </Card>
  <Card>
    {/* OutcomeDistributionChart */}
  </Card>
</div>
```

**Full-Width Charts**:
```tsx
{/* Performance/Costs Tabs: Single full-width chart */}
<Card>
  <CardHeader>
    <CardTitle>Agent Performance Comparison</CardTitle>
  </CardHeader>
  <CardContent>
    <AgentPerformanceChart {...} />
  </CardContent>
</Card>
```

### Section Spacing

```tsx
<div className="space-y-6">
  {/* Filters */}
  <div className="flex flex-col sm:flex-row gap-4">...</div>

  {/* KPI Cards */}
  <div className="grid gap-4 ...">...</div>

  {/* Tabbed Charts */}
  <Tabs className="space-y-4">...</Tabs>
</div>
```

**Spacing scale**:
- `gap-4` (1rem) - Grid gaps between cards
- `space-y-4` (1rem) - Tab content spacing
- `space-y-6` (1.5rem) - Major section spacing

---

## Data Fetching Strategy

### TanStack Query Hooks

**File**: `hooks/use-analytics.ts` (✅ Already exists)

**Available Hooks**:
```typescript
// Summary KPIs
useAnalyticsSummary(orgId, params?, options?)
// Returns: { total_sessions, success_rate, avg_cost, avg_turns, avg_duration, p95_cost, ... }

// Time-series data
useAnalyticsTimeSeries(orgId, params?, options?)
// Returns: [{ date, total_sessions, success_count, failure_count, ... }]

// Agent breakdown
useAgentStats(orgId, params?, options?)
// Returns: [{ agent_name, total_sessions, success_rate, avg_turns, ... }]

// User breakdown
useUserStats(orgId, params?, options?)

// Project breakdown
useProjectStats(orgId, params?, options?)

// Cost analysis
useCostBreakdown(orgId, params?, options?)

// Success trend
useSuccessTrend(orgId, params?, options?)

// Coaching effectiveness
useCoachingEffectiveness(orgId, params?, options?)
```

**Query Configuration**:
```typescript
{
  queryKey: ["analytics", "summary", orgId, params],
  queryFn: () => api.getAnalyticsSummary(orgId, params),
  enabled: !!orgId,
  staleTime: 60000, // 1 minute
  refetchInterval: undefined, // Manual refresh only
}
```

### Filter State Management

**Page State**:
```tsx
const [dateRange, setDateRange] = useState({
  start: subDays(new Date(), 30),
  end: new Date(),
});
const [selectedAgent, setSelectedAgent] = useState<string>("all");

// Derived filter params
const filterParams = {
  agent_name: selectedAgent !== "all" ? selectedAgent : undefined,
  start_date: format(dateRange.start, "yyyy-MM-dd"),
  end_date: format(dateRange.end, "yyyy-MM-dd"),
};
```

**Data Flow**:
```
User changes filter → State updates → filterParams recomputed
  ↓
TanStack Query detects queryKey change → Automatic refetch
  ↓
Loading state → API call → Data arrives → Components re-render
```

**No manual refetch needed** - TanStack Query handles it via reactive queryKey

### API Client

**File**: `lib/api/analytics.ts` (✅ Already exists)

**Pattern**:
```typescript
export async function getAnalyticsSummary(
  orgId: string,
  params?: {
    start_date?: string;
    end_date?: string;
    agent_name?: string;
    project_external_id?: string;
  },
): Promise<AnalyticsResponse<AnalyticsSummary>> {
  const searchParams = new URLSearchParams();
  if (params?.start_date) searchParams.set("start_date", params.start_date);
  if (params?.end_date) searchParams.set("end_date", params.end_date);
  if (params?.agent_name) searchParams.set("agent_name", params.agent_name);
  if (params?.project_external_id) searchParams.set("project_external_id", params.project_external_id);

  const query = searchParams.toString();
  return apiClient(`/api/v1/${orgId}/sessions/analytics/summary${query ? `?${query}` : ""}`);
}
```

**Benefits**:
- Type-safe parameters
- Automatic query string building
- Optional filters (undefined = omitted)
- Consistent error handling via apiClient

---

## Reusable Component Library

### Generic Components (`components/analytics/`)

**Purpose**: Framework-agnostic, reusable across features

**1. KpiCard** (`kpi/kpi-card.tsx`):
- Display any metric with icon, value, description
- 4 color variants
- Trend indicators
- Loading/error states
- Can be used for experiments, resources, projects, etc.

**2. AnalyticsLineChart** (`charts/line-chart.tsx`):
- Generic line chart with configurable axes
- Works with any time-series data
- Customizable colors via config object

**3. AnalyticsBarChart** (`charts/bar-chart.tsx`):
- Generic bar chart (single or multi-bar)
- Works with any categorical data

**4. AnalyticsAreaChart** (`charts/area-chart.tsx`):
- Generic area chart (single or stacked)
- Works with any continuous data

**When to use**:
- Building analytics for a new feature
- Need consistent styling without domain logic
- Want flexibility to adapt to different data shapes

### Feature-Specific Components (`components/features/analytics/`)

**Purpose**: Domain-aware, opinionated, ready-to-use

**1. AnalyticsKpiCards**:
- Pre-configured grid of session KPIs
- Knows about session statistics structure
- Formats durations, costs, percentages
- No configuration needed

**2. SessionTimeSeriesChart**:
- Line chart for session activity over time
- Pre-configured colors for success/failure/abandoned
- Date formatting
- Empty state handling

**3. AgentPerformanceChart**:
- Bar chart comparing agents
- Fetches data internally
- Multi-metric display (success rate, turns, sessions)

**4. CostAnalysisChart**:
- Area chart for cost trends
- Currency formatting
- Dual areas (avg + total)

**5. OutcomeDistributionChart**:
- Pie chart for outcome breakdown
- Color-coded by outcome type
- Percentage labels

**6. DateRangePicker**:
- Calendar + presets
- Session analytics-specific defaults (last 30 days)

**7. AgentFilterSelect**:
- Dropdown for agent selection
- Fetches agent list from API
- "All agents" option

**When to use**:
- Building session analytics dashboard
- Want pre-configured, opinionated components
- Don't need customization

### Component Selection Guide

```
Need to display a metric?
  ↓
  Is it session-specific? → Use AnalyticsKpiCards (if grid) or KpiCard (if single)
  Is it generic? → Use KpiCard with custom props

Need to display a chart?
  ↓
  Is it session-specific? → Use feature-specific chart (SessionTimeSeriesChart, etc.)
  Is it generic? → Use generic chart wrapper (AnalyticsLineChart, etc.)

Need to filter data?
  ↓
  By date range? → Use DateRangePicker
  By agent? → Use AgentFilterSelect
  Custom filter? → Build using shadcn Select/Popover
```

---

## Design System Consistency

### Color Palette

**Outcome Colors** (aligned with Badge variants):
```typescript
const OUTCOME_COLORS = {
  success: "hsl(142, 76%, 36%)",      // Green-600
  failure: "hsl(0, 84%, 60%)",        // Red-600
  abandoned: "hsl(38, 92%, 50%)",     // Orange-500
  in_progress: "hsl(217, 91%, 60%)",  // Blue-500
  cancelled: "hsl(215, 20%, 65%)",    // Gray-400
  timed_out: "hsl(280, 65%, 60%)",    // Purple-500
};
```

**KPI Variants**:
```typescript
const KPI_VARIANTS = {
  default: "border-border",
  success: "border-green-200 bg-green-50/50",
  warning: "border-yellow-200 bg-yellow-50/50",
  danger: "border-red-200 bg-red-50/50",
};
```

**Chart Colors** (via ChartConfig):
```typescript
const chartConfig = {
  total: { label: "Total", color: "hsl(var(--primary))" },
  success: { label: "Success", color: "hsl(142, 76%, 36%)" },
  failure: { label: "Failure", color: "hsl(0, 84%, 60%)" },
  abandoned: { label: "Abandoned", color: "hsl(38, 92%, 50%)" },
};
```

### Typography

**Card Titles**: `text-sm font-medium text-muted-foreground`
**Card Values**: `text-2xl font-bold`
**Descriptions**: `text-xs text-muted-foreground mt-1`
**Trends**: `text-xs font-medium` + color variant

### Spacing

**Grid Gaps**: `gap-4` (1rem between cards)
**Card Padding**: `py-6 px-6` (1.5rem internal padding)
**Section Spacing**: `space-y-6` (1.5rem between sections)
**Filter Spacing**: `gap-4` (1rem between filter controls)

### Icons (lucide-react)

**KPI Icons**:
- Activity (total sessions)
- TrendingUp (success rate)
- DollarSign (costs)
- MessageSquare (turns)
- Clock (duration)
- CheckCircle2 (success count)
- XCircle (failure count)
- AlertCircle (abandoned count)

**Filter Icons**:
- CalendarIcon (date range picker)
- ChevronDown (select dropdowns)

**Chart Icons**:
- TrendingUp/TrendingDown/Minus (trend indicators)

### Responsive Breakpoints

- **Mobile** (default): 1 column
- **Tablet** (`md:` 768px): 2-3 columns
- **Desktop** (`lg:` 1024px): 4-6 columns

**Grid patterns**:
```tsx
md:grid-cols-2 lg:grid-cols-4  // Primary KPIs
md:grid-cols-3                  // Outcome KPIs
lg:grid-cols-3                  // Chart grids
```

---

## File Structure

```
dashboard/src/
├── app/kit/analytics/
│   └── page.tsx                          # ✅ Main analytics page
│
├── components/
│   ├── analytics/                        # Generic, reusable components
│   │   ├── kpi/
│   │   │   ├── kpi-card.tsx              # ✅ Enhanced KPI card
│   │   │   └── kpi-grid.tsx              # ✅ KPI grid layout
│   │   ├── charts/
│   │   │   ├── line-chart.tsx            # ✅ Generic line chart
│   │   │   ├── bar-chart.tsx             # ✅ Generic bar chart
│   │   │   └── area-chart.tsx            # ✅ Generic area chart
│   │   └── filter-bar/
│   │       └── filter-bar.tsx            # ✅ Filter bar layout
│   │
│   └── features/analytics/               # Session analytics-specific
│       ├── analytics-kpi-cards.tsx       # ✅ Session KPI grid
│       ├── session-time-series-chart.tsx # ✅ Session activity chart
│       ├── outcome-distribution-chart.tsx# ✅ Outcome pie chart
│       ├── agent-performance-chart.tsx   # ✅ Agent comparison chart
│       ├── cost-analysis-chart.tsx       # ✅ Cost trends chart
│       ├── date-range-picker.tsx         # ✅ Date filter
│       ├── agent-filter-select.tsx       # ✅ Agent filter
│       └── live-activity-feed.tsx        # ✅ Real-time feed (future)
│
├── hooks/
│   └── use-analytics.ts                  # ✅ Analytics hooks
│
├── lib/api/
│   └── analytics.ts                      # ✅ Analytics API client
│
└── types/
    └── api.ts                            # ✅ Type definitions
```

**Component Organization**:
- **Generic** (`components/analytics/`) - No domain knowledge, reusable across app
- **Feature-Specific** (`components/features/analytics/`) - Session analytics only
- **Hooks** - Data fetching abstraction
- **API Client** - HTTP request functions
- **Types** - TypeScript interfaces

---

## Implementation Checklist

### Phase 1: Core Components (✅ Complete)
- [x] `analytics/kpi/kpi-card.tsx`
- [x] `analytics/charts/line-chart.tsx`
- [x] `analytics/charts/bar-chart.tsx`
- [x] `analytics/charts/area-chart.tsx`
- [x] `features/analytics/analytics-kpi-cards.tsx`
- [x] `features/analytics/session-time-series-chart.tsx`
- [x] `features/analytics/outcome-distribution-chart.tsx`
- [x] `features/analytics/agent-performance-chart.tsx`
- [x] `features/analytics/cost-analysis-chart.tsx`
- [x] `features/analytics/date-range-picker.tsx`
- [x] `features/analytics/agent-filter-select.tsx`

### Phase 2: Data Layer (✅ Complete)
- [x] `hooks/use-analytics.ts` (8 hooks)
- [x] `lib/api/analytics.ts` (8 API functions)
- [x] `types/api.ts` (Analytics types)

### Phase 3: Page Implementation (✅ Complete)
- [x] `app/kit/analytics/page.tsx`

### Phase 4: Testing (🔲 Pending)
- [ ] Unit tests for KpiCard variants
- [ ] Unit tests for chart components
- [ ] Integration tests for data fetching
- [ ] E2E tests for filter interactions
- [ ] Responsive design testing
- [ ] Empty state testing
- [ ] Error state testing
- [ ] Loading state testing

### Phase 5: Navigation & Polish (🔲 Pending)
- [ ] Add "Analytics" to sidebar navigation
- [ ] Update permissions (same as Sessions page)
- [ ] Add help text/tooltips where needed
- [ ] Verify accessibility (ARIA labels, keyboard nav)

### Phase 6: Documentation (✅ Complete)
- [x] ANALYTICS_ARCHITECTURE.md (this file)
- [x] ANALYTICS_DESIGN.md (backend integration)
- [x] ANALYTICS_COMPONENTS.md (quick reference)

---

## Design Decisions & Rationale

### 1. Why separate generic charts from feature-specific charts?

**Decision**: Two-layer component architecture

**Rationale**:
- **Generic charts** (`analytics/charts/`) are reusable across experiments, resources, projects
- **Feature-specific charts** (`features/analytics/`) are optimized for session analytics
- Separation allows us to DRY (Don't Repeat Yourself) while maintaining domain logic
- Testing is easier: generic charts test rendering, feature charts test business logic

**Alternative considered**: Single layer with all charts in `features/analytics/`
**Why rejected**: Would require duplicating chart code for other features (experiments, etc.)

### 2. Why use existing KpiCard instead of custom component?

**Decision**: Use `analytics/kpi/kpi-card.tsx` for all KPIs

**Rationale**:
- Already supports 4 variants (default, success, warning, danger)
- Already has trend indicators with directional icons
- Already has sparkline support
- Already has loading/error states
- Eliminates need for duplicate code

**Alternative considered**: Create `analytics/kpi/session-kpi-card.tsx`
**Why rejected**: No session-specific logic needed in KPI display

### 3. Why Tabs instead of multiple pages?

**Decision**: Single page with tabbed navigation

**Rationale**:
- Keeps all analytics in one place (fewer clicks)
- Filter state persists across tabs
- Easier to maintain (single page component)
- Common pattern for analytics dashboards (Google Analytics, Mixpanel, etc.)

**Alternative considered**: Separate pages (`/kit/analytics/overview`, `/kit/analytics/performance`)
**Why rejected**: Complicates filter state management, increases navigation friction

### 4. Why no real-time updates?

**Decision**: Manual refresh via filter changes only

**Rationale**:
- Session data changes slowly (minutes to hours, not seconds)
- Polling would waste API calls and database resources
- Users have control via date range filter
- Reduces server load and database queries

**Alternative considered**: 60s polling interval
**Why rejected**: Most changes won't be visible in 60s, wasted requests

### 5. Why shadcn Chart components?

**Decision**: Use `ChartContainer`, `ChartTooltip`, etc. instead of raw Recharts

**Rationale**:
- Consistent theming via CSS variables (`--color-{key}`)
- Dark mode support out of the box
- Accessible tooltips and legends
- Minimal custom styling needed
- Future-proof (theme changes propagate automatically)

**Alternative considered**: Direct Recharts usage
**Why rejected**: Duplicate theming logic, dark mode issues, accessibility gaps

### 6. Why TanStack Query instead of SWR?

**Decision**: Use TanStack Query for data fetching

**Rationale**:
- Already used throughout the app (consistency)
- Better TypeScript support
- More flexible cache management (staleTime, refetchInterval)
- Larger ecosystem and community

**Alternative considered**: SWR
**Why rejected**: Not the existing pattern, migration cost

### 7. Why date-fns instead of moment.js?

**Decision**: Use date-fns for date formatting

**Rationale**:
- Tree-shakeable (only import what you use)
- Smaller bundle size (~15KB vs ~67KB)
- Immutable by default (safer)
- Already used in the app

**Alternative considered**: Day.js
**Why rejected**: date-fns already a dependency

### 8. Why client-side filtering instead of server-side?

**Decision**: Server-side filtering (filters → API params → database query)

**Rationale**:
- Can't fit 90 days of hourly data in initial page load
- Database is optimized for aggregation (indexes, materialized views)
- Reduces client memory usage
- More accurate (no stale data)

**Alternative considered**: Load all data, filter client-side
**Why rejected**: Too much data, slow initial load, stale data issues

---

## Future Enhancements

### Phase 2 (Post-MVP)
- [ ] Live Activity Feed (WebSocket for real-time session updates)
- [ ] Export Analytics (CSV/PDF export for reports)
- [ ] Custom Date Range (calendar UI instead of presets only)
- [ ] Drill-Down (click chart → navigate to filtered session list)
- [ ] Comparison Mode (compare two time periods side-by-side)

### Phase 3 (Advanced)
- [ ] Anomaly Detection (AI-powered alerts for unusual patterns)
- [ ] Predictive Forecasting (cost and usage predictions)
- [ ] Benchmarking (compare org performance to platform averages)
- [ ] Custom Dashboards (user-configurable layouts and saved views)
- [ ] Scheduled Reports (email weekly/monthly summaries)

---

## Known Limitations

**1. Browser Compatibility**: Recharts requires modern browsers (ES6+)
**Mitigation**: crewkit targets modern browsers, not IE11

**2. Data Volume**: Large date ranges (6 months daily) may slow down charts
**Mitigation**: Default to 30 days, use weekly/monthly intervals for longer ranges

**3. Timezone Handling**: All dates use client timezone
**Mitigation**: Backend converts to UTC, frontend displays in user's local time

**4. Caching**: 1-minute staleTime may show slightly outdated data
**Mitigation**: Users can manually refresh by changing filters

**5. No Offline Support**: Requires active internet connection
**Mitigation**: Show clear error message when offline

---

## Dependencies

### Already Installed (No New Dependencies)
- `recharts` (^2.x) - Chart library
- `date-fns` (^2.x) - Date formatting
- `@tanstack/react-query` (^5.x) - Data fetching
- `lucide-react` (^0.x) - Icons
- `shadcn/ui` - UI components
- `react-day-picker` (^8.x) - Calendar
- `@radix-ui/react-popover` (^1.x) - Popover primitive

---

## Summary

The Session Analytics Dashboard uses a **layered component architecture**:

1. **UI Primitives** (`components/ui/`) - shadcn/ui base components
2. **Generic Analytics** (`components/analytics/`) - Reusable KPIs and charts
3. **Feature-Specific** (`components/features/analytics/`) - Session analytics components
4. **Page** (`app/kit/analytics/`) - Layout and orchestration

This approach ensures:
- **Consistency** - All analytics use the same design system
- **Reusability** - Generic components can be used for experiments, resources, etc.
- **Maintainability** - Domain logic isolated in feature components
- **Testability** - Each layer can be tested independently
- **Scalability** - Easy to add new features without breaking existing code

All components follow **Vercel-inspired design principles**:
- Minimalism (fewer elements, more whitespace)
- Functional over fancy (clarity over decoration)
- Consistency creates calm (same patterns everywhere)
- Speed through simplicity (fast rendering, minimal DOM)