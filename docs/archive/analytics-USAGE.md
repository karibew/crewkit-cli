# Analytics Component System - Usage Guide

Complete analytics component library for crewkit dashboard with consistent styling, accessibility, and responsive behavior.

## Components Overview

### 1. KPI Cards - Metric Display

Display key performance indicators with trends, sparklines, and variants.

#### Basic Usage

```tsx
import { KpiCard, KpiGrid } from "@/components/analytics";
import { DollarSign, Users, TrendingUp } from "lucide-react";

function SessionMetrics() {
  return (
    <KpiGrid columns={3}>
      <KpiCard
        title="Total Sessions"
        value={1234}
        description="Last 30 days"
        icon={Users}
        trend={{ value: 12.5, label: "vs last month" }}
      />

      <KpiCard
        title="Total Cost"
        value="$456.78"
        icon={DollarSign}
        variant="success"
        trend={{ value: -8.3, direction: "down", label: "cost reduction" }}
        sparklineData={[45, 52, 48, 61, 55, 63, 58]}
      />

      <KpiCard
        title="Success Rate"
        value="94.2%"
        icon={TrendingUp}
        variant="success"
        trend={{ value: 2.1 }}
      />
    </KpiGrid>
  );
}
```

#### Props API

```typescript
interface KpiCardProps {
  title: string;                    // Card title
  value: string | number;           // Main metric value
  description?: string;             // Optional description
  icon?: LucideIcon;               // Lucide icon component
  variant?: "default" | "success" | "warning" | "danger";
  trend?: {
    value: number;                 // Percentage change
    direction?: "up" | "down" | "neutral"; // Auto-calculated if omitted
    label?: string;                // "vs last month", etc.
  };
  sparklineData?: number[];        // Array of values for sparkline
  loading?: boolean;               // Show skeleton loader
  error?: string;                  // Error message
  className?: string;
}
```

#### States

**Loading**:
```tsx
<KpiCard title="Sessions" value="0" loading />
```

**Error**:
```tsx
<KpiCard title="Sessions" value="0" error="Failed to load data" />
```

**Variants**:
```tsx
<KpiCard variant="success" ... />  // Green background
<KpiCard variant="warning" ... />  // Yellow background
<KpiCard variant="danger" ... />   // Red background
```

---

### 2. Advanced Data Table

Sortable, searchable, paginated table with export functionality.

#### Basic Usage

```tsx
import { AdvancedTable, ColumnDef } from "@/components/analytics";
import { SessionOutcomeBadge } from "@/components/features/sessions";

interface Session {
  id: string;
  agent_name: string;
  total_cost: number;
  outcome: string;
  started_at: string;
}

const columns: ColumnDef<Session>[] = [
  {
    key: "agent_name",
    header: "Agent",
    sortable: true,
    searchable: true,
  },
  {
    key: "outcome",
    header: "Outcome",
    render: (row) => <SessionOutcomeBadge outcome={row.outcome} />,
  },
  {
    key: "total_cost",
    header: "Cost",
    sortable: true,
    render: (row) => `$${row.total_cost.toFixed(4)}`,
    className: "text-right font-mono",
  },
  {
    key: "started_at",
    header: "Started",
    sortable: true,
    render: (row) => new Date(row.started_at).toLocaleDateString(),
  },
];

function SessionsTable({ sessions }: { sessions: Session[] }) {
  const handleExport = (data: Session[]) => {
    // Convert to CSV and download
    const csv = convertToCSV(data);
    downloadFile(csv, "sessions.csv");
  };

  return (
    <AdvancedTable
      data={sessions}
      columns={columns}
      keyExtractor={(row) => row.id}
      searchPlaceholder="Search sessions..."
      onExport={handleExport}
      pageSize={25}
      pageSizeOptions={[10, 25, 50, 100]}
    />
  );
}
```

#### Column Definition

```typescript
interface ColumnDef<T> {
  key: string;                         // Property name in data
  header: string;                      // Column header text
  sortable?: boolean;                  // Enable sorting
  searchable?: boolean;                // Include in search
  render?: (row: T) => React.ReactNode; // Custom cell renderer
  className?: string;                  // Cell CSS classes
}
```

#### Features

- **Sorting**: Click column headers (shows arrow icons)
- **Search**: Full-text search across searchable columns
- **Pagination**: Configurable page sizes
- **Export**: CSV/JSON export via callback
- **Empty State**: Customizable empty message
- **Loading State**: Built-in skeleton

---

### 3. Chart Components

Recharts wrappers with consistent styling and shadcn/ui integration.

#### Line Chart

```tsx
import { AnalyticsLineChart } from "@/components/analytics";
import { ChartConfig } from "@/components/ui/chart";

const chartData = [
  { date: "Jan", sessions: 120, cost: 45.2 },
  { date: "Feb", sessions: 145, cost: 52.1 },
  { date: "Mar", sessions: 132, cost: 48.7 },
];

const chartConfig = {
  sessions: {
    label: "Sessions",
    color: "hsl(var(--chart-1))",
  },
  cost: {
    label: "Cost ($)",
    color: "hsl(var(--chart-2))",
  },
} satisfies ChartConfig;

<AnalyticsLineChart
  title="Session Trends"
  description="Last 3 months"
  data={chartData}
  xKey="date"
  yKeys={["sessions", "cost"]}
  config={chartConfig}
  height={300}
/>
```

#### Bar Chart

```tsx
import { AnalyticsBarChart } from "@/components/analytics";

<AnalyticsBarChart
  title="Outcome Distribution"
  data={outcomeData}
  xKey="outcome"
  yKeys={["count"]}
  config={chartConfig}
  stacked={false}  // Set true for stacked bars
/>
```

#### Area Chart

```tsx
import { AnalyticsAreaChart } from "@/components/analytics";

<AnalyticsAreaChart
  title="Cost Over Time"
  data={costData}
  xKey="date"
  yKeys={["cost"]}
  config={chartConfig}
  stacked={false}
/>
```

#### Chart Props (Common)

```typescript
interface ChartProps {
  title?: string;
  description?: string;
  data: Record<string, unknown>[];
  xKey: string;                    // X-axis data key
  yKeys: string[];                 // Y-axis data keys (multiple lines/bars)
  config: ChartConfig;             // Color/label configuration
  height?: number;                 // Chart height (default: 350)
  showGrid?: boolean;              // Show grid lines (default: true)
  showTooltip?: boolean;           // Show tooltip (default: true)
  stacked?: boolean;               // Stack bars/areas (default: false)
  className?: string;
}
```

---

### 4. Timeline

Vertical timeline with expandable details and status indicators.

#### Basic Usage

```tsx
import { Timeline } from "@/components/analytics";
import { Clock, CheckCircle, XCircle, AlertTriangle } from "lucide-react";

const timelineItems = [
  {
    id: "1",
    title: "Session started",
    description: "rails-expert agent initialized",
    timestamp: "2 hours ago",
    icon: Clock,
    status: "info" as const,
    metadata: {
      "Agent": "rails-expert",
      "Version": "v1.2.0",
    },
  },
  {
    id: "2",
    title: "Code modification",
    description: "Updated user_controller.rb",
    timestamp: "1 hour ago",
    icon: CheckCircle,
    status: "success" as const,
    details: (
      <pre className="text-xs">
        + Added validation for email field
        + Improved error handling
      </pre>
    ),
  },
  {
    id: "3",
    title: "Test failure",
    description: "RSpec tests failed",
    timestamp: "30 minutes ago",
    icon: XCircle,
    status: "error" as const,
    metadata: {
      "Failed": "3/45",
      "Duration": "12.4s",
    },
  },
  {
    id: "4",
    title: "Session completed",
    timestamp: "10 minutes ago",
    icon: CheckCircle,
    status: "success" as const,
  },
];

<Timeline items={timelineItems} />
```

#### Props

```typescript
interface TimelineItemData {
  id: string;
  title: string;
  description?: string;
  timestamp: string;                           // Human-readable time
  icon?: LucideIcon;
  status?: "success" | "error" | "warning" | "info" | "neutral";
  details?: ReactNode;                         // Expandable content
  metadata?: Record<string, string | number>;  // Key-value pairs
}

interface TimelineProps {
  items: TimelineItemData[];
  className?: string;
}
```

---

### 5. Filter Bar

Reusable filter controls with date range, search, and select inputs.

#### Basic Usage

```tsx
import { FilterBar, FilterConfig, FilterValue } from "@/components/analytics";
import { useState } from "react";

const filterConfig: FilterConfig[] = [
  {
    id: "search",
    type: "search",
    label: "Search",
    placeholder: "Search sessions...",
  },
  {
    id: "agent",
    type: "select",
    label: "Agent",
    placeholder: "All agents",
    options: [
      { value: "all", label: "All agents" },
      { value: "rails-expert", label: "rails-expert" },
      { value: "frontend-expert", label: "frontend-expert" },
    ],
  },
  {
    id: "outcome",
    type: "select",
    label: "Outcome",
    placeholder: "All outcomes",
    options: [
      { value: "all", label: "All outcomes" },
      { value: "success", label: "Success" },
      { value: "failure", label: "Failure" },
      { value: "abandoned", label: "Abandoned" },
    ],
  },
  {
    id: "dateRange",
    type: "date-range",
    label: "Date Range",
    placeholder: "Select date range",
  },
];

function SessionFilters() {
  const [filters, setFilters] = useState<FilterValue>({});

  return (
    <FilterBar
      filters={filterConfig}
      values={filters}
      onChange={setFilters}
      onClear={() => setFilters({})}
    />
  );
}
```

#### Filter Types

**Search Input**:
```typescript
{
  id: "search",
  type: "search",
  label: "Search",
  placeholder: "Search...",
}
```

**Select Dropdown**:
```typescript
{
  id: "status",
  type: "select",
  label: "Status",
  placeholder: "All statuses",
  options: [
    { value: "active", label: "Active" },
    { value: "completed", label: "Completed" },
  ],
}
```

**Date Range Picker**:
```typescript
{
  id: "dateRange",
  type: "date-range",
  label: "Date Range",
  placeholder: "Select dates",
}
```

#### Accessing Filter Values

```tsx
const searchTerm = filters.search as string;
const selectedAgent = filters.agent as string;
const dateRange = filters.dateRange as DateRange;

// Use values to filter data
const filteredSessions = sessions.filter((session) => {
  if (searchTerm && !session.agent_name.includes(searchTerm)) return false;
  if (selectedAgent && selectedAgent !== "all" && session.agent_name !== selectedAgent) return false;
  if (dateRange?.from && new Date(session.started_at) < dateRange.from) return false;
  if (dateRange?.to && new Date(session.started_at) > dateRange.to) return false;
  return true;
});
```

---

## Complete Example: Session Analytics Page

```tsx
"use client";

import { useState } from "react";
import {
  KpiCard,
  KpiGrid,
  AdvancedTable,
  AnalyticsLineChart,
  Timeline,
  FilterBar,
  type ColumnDef,
  type FilterConfig,
  type FilterValue,
} from "@/components/analytics";
import { DollarSign, Users, TrendingUp, Clock } from "lucide-react";
import type { Session } from "@/types/api";

export default function SessionAnalyticsPage() {
  const [filters, setFilters] = useState<FilterValue>({});

  // Fetch data (placeholder)
  const sessions: Session[] = []; // From API
  const timelineItems = [];       // From API
  const chartData = [];           // From API

  const filterConfig: FilterConfig[] = [
    {
      id: "search",
      type: "search",
      label: "Search",
      placeholder: "Search sessions...",
    },
    {
      id: "agent",
      type: "select",
      label: "Agent",
      options: [
        { value: "all", label: "All agents" },
        { value: "rails-expert", label: "rails-expert" },
      ],
    },
    {
      id: "dateRange",
      type: "date-range",
      label: "Date Range",
    },
  ];

  const columns: ColumnDef<Session>[] = [
    {
      key: "agent_name",
      header: "Agent",
      sortable: true,
      searchable: true,
    },
    {
      key: "total_cost",
      header: "Cost",
      sortable: true,
      render: (row) => `$${row.total_cost.toFixed(4)}`,
      className: "text-right font-mono",
    },
  ];

  return (
    <div className="space-y-8">
      <h1 className="text-3xl font-bold">Session Analytics</h1>

      {/* KPI Overview */}
      <KpiGrid columns={4}>
        <KpiCard
          title="Total Sessions"
          value={1234}
          icon={Users}
          trend={{ value: 12.5 }}
        />
        <KpiCard
          title="Total Cost"
          value="$456.78"
          icon={DollarSign}
          variant="success"
          trend={{ value: -8.3 }}
        />
        <KpiCard
          title="Success Rate"
          value="94.2%"
          icon={TrendingUp}
          variant="success"
        />
        <KpiCard
          title="Avg Duration"
          value="12m 34s"
          icon={Clock}
        />
      </KpiGrid>

      {/* Chart */}
      <AnalyticsLineChart
        title="Session Trends"
        description="Sessions over time"
        data={chartData}
        xKey="date"
        yKeys={["sessions"]}
        config={{
          sessions: {
            label: "Sessions",
            color: "hsl(var(--chart-1))",
          },
        }}
      />

      {/* Filters + Table */}
      <div className="space-y-4">
        <FilterBar
          filters={filterConfig}
          values={filters}
          onChange={setFilters}
        />

        <AdvancedTable
          data={sessions}
          columns={columns}
          keyExtractor={(row) => row.id}
          pageSize={25}
        />
      </div>

      {/* Timeline */}
      <div>
        <h2 className="text-xl font-semibold mb-4">Recent Activity</h2>
        <Timeline items={timelineItems} />
      </div>
    </div>
  );
}
```

---

## Accessibility

All components include:

- **Semantic HTML**: Proper heading hierarchy, labels, ARIA attributes
- **Keyboard Navigation**: Full keyboard support (Tab, Enter, Arrow keys)
- **Screen Reader Support**: ARIA labels, live regions, descriptions
- **Focus Management**: Visible focus indicators
- **Color Contrast**: WCAG 2.1 AA compliant

---

## Theming

Components use shadcn/ui theming system via CSS variables:

```css
/* Customize in globals.css */
:root {
  --chart-1: 220 70% 50%;  /* Primary chart color */
  --chart-2: 160 60% 45%;  /* Secondary chart color */
  --chart-3: 30 80% 55%;   /* Tertiary chart color */
  --chart-4: 280 65% 60%;
  --chart-5: 340 75% 55%;
}
```

---

## Best Practices

1. **Consistent KPI variants**:
   - `success` = positive metrics (revenue up, errors down)
   - `warning` = needs attention
   - `danger` = critical issues

2. **Table performance**:
   - Limit default page size to 25-50 rows
   - Use pagination for large datasets
   - Implement virtualization for 1000+ rows

3. **Chart data**:
   - Keep data arrays under 100 points for smooth rendering
   - Use area charts for cumulative metrics
   - Use bar charts for categorical comparisons

4. **Timeline**:
   - Most recent events first
   - Use expandable details for verbose content
   - Limit to 20-30 visible items (paginate if needed)

5. **Filters**:
   - Place most-used filters first (left to right)
   - Show "Clear all" only when filters are active
   - Persist filter state in URL params for shareability

---

## Related Components

- `SessionTable` - Feature-specific session table
- `StatsCard` - Existing dashboard stats card (consider migrating to `KpiCard`)
- `ExperimentMetricsCard` - Experiment-specific metrics (uses chart components)