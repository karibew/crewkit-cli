# Analytics Components Quick Reference

## Component Tree

```
/kit/analytics
├── DashboardShell
│   ├── Filters Row
│   │   ├── DateRangePicker (Popover with presets)
│   │   └── AgentFilterSelect (Select dropdown)
│   │
│   ├── KPI Section
│   │   └── AnalyticsKpiCards
│   │       ├── Grid: 6 primary KPIs
│   │       │   ├── Total Sessions (Activity icon)
│   │       │   ├── Success Rate (TrendingUp icon)
│   │       │   ├── Avg Cost (DollarSign icon)
│   │       │   ├── Avg Turns (MessageSquare icon)
│   │       │   ├── Avg Duration (Clock icon)
│   │       │   └── P95 Cost (DollarSign icon)
│   │       └── Grid: 3 outcome cards
│   │           ├── Success (CheckCircle2 icon)
│   │           ├── Failure (XCircle icon)
│   │           └── Abandoned (AlertCircle icon)
│   │
│   └── Tabs
│       ├── Overview Tab
│       │   ├── SessionTimeSeriesChart (2/3 width)
│       │   │   └── LineChart (4 lines: total, success, failure, abandoned)
│       │   └── OutcomeDistributionChart (1/3 width)
│       │       └── PieChart (outcome breakdown)
│       │
│       ├── Performance Tab
│       │   └── AgentPerformanceChart (full width)
│       │       └── BarChart (multi-bar: success rate, avg turns, sessions)
│       │
│       └── Costs Tab
│           └── CostAnalysisChart (full width)
│               └── AreaChart (dual area: avg cost, total cost)
```

## Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                        Analytics Page                        │
│  State: dateRange, selectedAgent                            │
└─────────────────────────────────────────────────────────────┘
                          │
                          ├──── filterParams { agent_name, start_date, end_date }
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ useSession   │  │ useSession   │  │ useAgent     │
│ Statistics   │  │ TimeSeries   │  │ Performance  │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ API: GET     │  │ API: GET     │  │ API: GET     │
│ /statistics  │  │ /time_series │  │ /agent_perf  │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                          ▼
                   ┌──────────────┐
                   │  PostgreSQL  │
                   └──────────────┘
```

## Props Interface

### AnalyticsKpiCards
```typescript
interface Props {
  stats?: SessionStatistics;
}
```

### SessionTimeSeriesChart
```typescript
interface Props {
  data: SessionTimeSeriesDataPoint[];
}
```

### OutcomeDistributionChart
```typescript
interface Props {
  data: Record<string, number>;
}
```

### AgentPerformanceChart
```typescript
interface Props {
  orgId: string;
  dateRange?: {
    start_date?: string;
    end_date?: string;
  };
}
```

### CostAnalysisChart
```typescript
interface Props {
  data: SessionTimeSeriesDataPoint[];
}
```

### DateRangePicker
```typescript
interface Props {
  value: {
    start: Date;
    end: Date;
  };
  onChange: (range: { start: Date; end: Date }) => void;
}
```

### AgentFilterSelect
```typescript
interface Props {
  value: string;
  onChange: (value: string) => void;
}
```

### LiveActivityFeed
```typescript
interface Props {
  orgId: string;
  limit?: number; // default: 10
}
```

## Usage Examples

### Basic Analytics Page
```tsx
import { AnalyticsKpiCards } from "@/components/features/analytics/analytics-kpi-cards";
import { useSessionStatistics } from "@/hooks/use-sessions";

function MyAnalytics() {
  const { data } = useSessionStatistics(orgId, filters);
  return <AnalyticsKpiCards stats={data?.statistics} />;
}
```

### Time Series Chart
```tsx
import { SessionTimeSeriesChart } from "@/components/features/analytics/session-time-series-chart";
import { useSessionTimeSeries } from "@/hooks/use-sessions";

function MyChart() {
  const { data } = useSessionTimeSeries(orgId, filters);
  return <SessionTimeSeriesChart data={data?.time_series || []} />;
}
```

### Agent Performance
```tsx
import { AgentPerformanceChart } from "@/components/features/analytics/agent-performance-chart";

function MyPerformance() {
  return (
    <AgentPerformanceChart
      orgId={orgId}
      dateRange={{ start_date: "2026-01-01", end_date: "2026-01-31" }}
    />
  );
}
```

### Date Range Filter
```tsx
import { DateRangePicker } from "@/components/features/analytics/date-range-picker";
import { useState } from "react";
import { subDays } from "date-fns";

function MyFilters() {
  const [range, setRange] = useState({
    start: subDays(new Date(), 30),
    end: new Date(),
  });

  return <DateRangePicker value={range} onChange={setRange} />;
}
```

### Live Activity Feed
```tsx
import { LiveActivityFeed } from "@/components/features/analytics/live-activity-feed";

function MySidebar() {
  return <LiveActivityFeed orgId={orgId} limit={5} />;
}
```

## Styling Classes

### Responsive Grid
```tsx
// KPI Cards (6 columns on desktop)
<div className="grid gap-4 md:grid-cols-3 lg:grid-cols-6">

// Charts (2 columns on tablet+)
<div className="grid gap-4 md:grid-cols-2">

// Mixed layout (3 columns on large screens)
<div className="grid gap-4 lg:grid-cols-3">
```

### Chart Container
```tsx
<ResponsiveContainer width="100%" height={300}>
  {/* Chart */}
</ResponsiveContainer>
```

### Empty States
```tsx
<div className="h-[300px] flex items-center justify-center text-muted-foreground">
  No data available
</div>
```

### Loading States
```tsx
<Skeleton className="h-[300px]" />
```

## Color Reference

### Outcome Colors (HSL)
```typescript
const COLORS = {
  success: "hsl(142, 76%, 36%)",      // Green
  failure: "hsl(0, 84%, 60%)",        // Red
  abandoned: "hsl(38, 92%, 50%)",     // Orange
  in_progress: "hsl(217, 91%, 60%)",  // Blue
  cancelled: "hsl(215, 20%, 65%)",    // Gray
  timed_out: "hsl(280, 65%, 60%)",    // Purple
};
```

### KPI Icon Colors
```typescript
const KPI_COLORS = {
  activity: "text-blue-600",
  success: "text-green-600",
  cost: "text-purple-600",
  turns: "text-orange-600",
  duration: "text-cyan-600",
  p95: "text-indigo-600",
};
```

## Chart Configuration

### Line Chart (Time Series)
- **Type**: Line
- **Lines**: 4 (total, success, failure, abandoned)
- **X-Axis**: Date (formatted as "MMM d")
- **Y-Axis**: Count
- **Stroke Width**: 2px
- **Grid**: Dashed (3-3)

### Pie Chart (Outcome Distribution)
- **Type**: Pie
- **Labels**: Name + Percentage
- **Outer Radius**: 80px
- **Legend**: Bottom
- **Tooltip**: Custom styled

### Bar Chart (Agent Performance)
- **Type**: Bar (multi-bar)
- **Bars**: 3 (Success Rate, Avg Turns, Sessions)
- **X-Axis**: Agent names (angled -45°)
- **Y-Axis**: Values
- **Height**: 400px

### Area Chart (Cost Analysis)
- **Type**: Area (dual area)
- **Areas**: 2 (Avg Cost, Total Cost)
- **Fill**: Gradient (top to bottom)
- **Stroke**: Solid color
- **Y-Axis**: Currency formatted ($)

## Hook Configuration

### Query Keys
```typescript
["sessions", "statistics", orgId, params]
["sessions", "time-series", orgId, params]
["sessions", "agent-performance", orgId, params]
```

### Refetch Intervals
- **Statistics**: No auto-refetch
- **Time Series**: 60s stale time
- **Agent Performance**: 60s stale time
- **Session List**: 10s refetch interval

### Enabled Conditions
- All hooks require `!!orgId`
- Hooks only run when orgId is present

## Testing Helpers

### Mock Data
```typescript
const mockStats: SessionStatistics = {
  total_sessions: 150,
  success_rate: 84.5,
  average_duration: 342.5,
  average_turns: 12.3,
  average_cost: 0.0234,
  total_cost: 3.51,
  p95_cost: 0.0456,
  p95_duration: 567.8,
  outcome_breakdown: {
    success: 127,
    failure: 18,
    abandoned: 5,
  },
};

const mockTimeSeries: SessionTimeSeriesDataPoint[] = [
  {
    date: "2026-01-01",
    total_sessions: 45,
    success_count: 38,
    failure_count: 5,
    abandoned_count: 2,
    avg_cost: 0.0234,
    total_cost: 1.053,
    avg_duration: 342.5,
    avg_turns: 12.3,
  },
];
```

### Empty States Test
```typescript
it("renders empty state when no data", () => {
  render(<SessionTimeSeriesChart data={[]} />);
  expect(screen.getByText(/no data available/i)).toBeInTheDocument();
});
```

### Loading States Test
```typescript
it("renders skeleton during load", () => {
  render(<AnalyticsKpiCards stats={undefined} />);
  // Component returns null when stats is undefined
});
```

## Accessibility

### ARIA Labels
- Chart tooltips are keyboard accessible
- Filter controls have proper labels
- Icons are decorative (aria-hidden="true")

### Keyboard Navigation
- Tab through filters
- Enter to select dates/agents
- Arrow keys in select dropdowns

### Screen Readers
- Chart data announced via ARIA live regions
- Loading states announced
- Empty states clear and descriptive

## Performance

### Optimization Techniques
1. **Memoization**: Charts use ResponsiveContainer
2. **Lazy Loading**: Tabs load content on demand
3. **Stale Time**: 60s for analytics queries
4. **Conditional Rendering**: Empty states short-circuit
5. **Skeleton Loading**: Prevents layout shift

### Bundle Size
- recharts: ~130KB gzipped
- date-fns: ~15KB gzipped (only used functions)
- No additional chart libraries needed