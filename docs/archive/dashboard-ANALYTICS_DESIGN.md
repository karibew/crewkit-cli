# Session Analytics & KPIs Dashboard Design

## Overview

Comprehensive analytics dashboard for monitoring agent session performance, costs, and outcomes in crewkit.

## Architecture

### New Pages

#### `/kit/analytics` - Main Analytics Dashboard
- **Location**: `dashboard/src/app/kit/analytics/page.tsx`
- **Purpose**: High-level analytics with charts and KPIs
- **Features**:
  - Date range filtering (7d, 30d, 90d, 6mo, 1yr presets)
  - Agent filtering
  - Tabbed interface (Overview, Performance, Costs)
  - Real-time data updates
  - Responsive grid layouts

### Component Hierarchy

```
/kit/analytics/
├── AnalyticsKpiCards (9 KPI cards)
│   ├── Total Sessions
│   ├── Success Rate
│   ├── Avg Cost
│   ├── Avg Turns
│   ├── Avg Duration
│   ├── P95 Cost
│   ├── Success Count
│   ├── Failure Count
│   └── Abandoned Count
├── DateRangePicker (filter)
├── AgentFilterSelect (filter)
└── Tabs
    ├── Overview Tab
    │   ├── SessionTimeSeriesChart (line chart)
    │   └── OutcomeDistributionChart (pie chart)
    ├── Performance Tab
    │   └── AgentPerformanceChart (bar chart)
    └── Costs Tab
        └── CostAnalysisChart (area chart)
```

### Updated Existing Pages

#### `/kit/sessions` - Session List
- Already exists with basic KPI cards
- No changes needed (already has filters and pagination)

#### `/kit/sessions/[id]` - Session Detail
- Already exists with session summary
- No changes needed

## API Integration

### New API Endpoints (Backend TODO)

These endpoints need to be implemented in the Rails API:

#### 1. Time Series Data
```
GET /api/v1/:org_id/sessions/time_series
Parameters:
  - agent_name (optional)
  - project_external_id (optional)
  - start_date (required, YYYY-MM-DD)
  - end_date (required, YYYY-MM-DD)
  - interval (optional, default: "day", values: "day" | "week" | "month")

Response:
{
  "time_series": [
    {
      "date": "2026-01-01",
      "total_sessions": 45,
      "success_count": 38,
      "failure_count": 5,
      "abandoned_count": 2,
      "avg_cost": 0.0234,
      "total_cost": 1.053,
      "avg_duration": 342.5,
      "avg_turns": 12.3
    }
  ],
  "meta": {
    "start_date": "2025-12-01",
    "end_date": "2026-01-01",
    "interval": "day"
  }
}
```

#### 2. Agent Performance Comparison
```
GET /api/v1/:org_id/sessions/agent_performance
Parameters:
  - start_date (optional, YYYY-MM-DD)
  - end_date (optional, YYYY-MM-DD)

Response:
{
  "agents": [
    {
      "agent_name": "rails-expert",
      "total_sessions": 145,
      "success_rate": 84.8,
      "avg_turns": 15.2,
      "avg_duration": 456.7,
      "avg_cost": 0.0312,
      "total_cost": 4.524
    }
  ]
}
```

### Existing API Endpoints (Already Working)

```
GET /api/v1/:org_id/sessions/statistics
GET /api/v1/:org_id/sessions
GET /api/v1/:org_id/sessions/:id/summary
```

## Type Definitions

### New Types Added to `types/api.ts`

```typescript
export interface SessionTimeSeriesDataPoint {
  date: string;
  total_sessions: number;
  success_count: number;
  failure_count: number;
  abandoned_count: number;
  avg_cost: number;
  total_cost: number;
  avg_duration: number;
  avg_turns: number;
}

export interface SessionTimeSeriesResponse {
  time_series: SessionTimeSeriesDataPoint[];
  meta: {
    start_date: string;
    end_date: string;
    interval: "day" | "week" | "month";
  };
}

export interface AgentPerformanceMetrics {
  agent_name: string;
  total_sessions: number;
  success_rate: number;
  avg_turns: number;
  avg_duration: number;
  avg_cost: number;
  total_cost: number;
}

export interface AgentPerformanceResponse {
  agents: AgentPerformanceMetrics[];
}
```

## New Components

### Analytics Components (`components/features/analytics/`)

#### 1. `analytics-kpi-cards.tsx`
- **Purpose**: Display key performance indicators
- **Props**: `{ stats?: SessionStatistics }`
- **Features**:
  - 6 primary KPI cards (Sessions, Success Rate, Avg Cost, Avg Turns, Avg Duration, P95 Cost)
  - 3 outcome cards (Success, Failure, Abandoned)
  - Color-coded icons
  - Percentage calculations
  - Duration formatting helper

#### 2. `session-time-series-chart.tsx`
- **Purpose**: Line chart showing session activity over time
- **Props**: `{ data: SessionTimeSeriesDataPoint[] }`
- **Library**: recharts
- **Features**:
  - Multi-line chart (Total, Success, Failure, Abandoned)
  - Date formatting (MMM d)
  - Responsive container
  - Custom tooltip styling
  - Empty state handling

#### 3. `outcome-distribution-chart.tsx`
- **Purpose**: Pie chart showing outcome breakdown
- **Props**: `{ data: Record<string, number> }`
- **Library**: recharts
- **Features**:
  - Color-coded outcomes
  - Percentage labels
  - Legend
  - Custom tooltip with counts and percentages
  - Empty state handling

#### 4. `agent-performance-chart.tsx`
- **Purpose**: Bar chart comparing agent performance
- **Props**: `{ orgId: string; dateRange?: {...} }`
- **Library**: recharts
- **Features**:
  - Multi-bar chart (Success Rate, Avg Turns, Sessions)
  - Angled x-axis labels for readability
  - Data fetching via hook
  - Loading and error states
  - Cost scaling for visibility

#### 5. `cost-analysis-chart.tsx`
- **Purpose**: Area chart showing cost trends
- **Props**: `{ data: SessionTimeSeriesDataPoint[] }`
- **Library**: recharts
- **Features**:
  - Dual area chart (Avg Cost, Total Cost)
  - Gradient fills
  - Currency formatting
  - Date parsing
  - Empty state handling

#### 6. `date-range-picker.tsx`
- **Purpose**: Date range selection with presets
- **Props**: `{ value: {...}; onChange: (...) => void }`
- **Features**:
  - Predefined ranges (7d, 30d, 90d, 6mo, 1yr)
  - Popover UI
  - Date formatting
  - Responsive width

#### 7. `agent-filter-select.tsx`
- **Purpose**: Agent dropdown filter
- **Props**: `{ value: string; onChange: (value: string) => void }`
- **Features**:
  - "All agents" option
  - Dynamic agent list from API
  - Display name fallback
  - Loading state

## Hooks

### Updated `hooks/use-sessions.ts`

Added two new hooks:

```typescript
export function useSessionTimeSeries(
  orgId: string,
  params?: Parameters<typeof api.getSessionTimeSeries>[1]
)

export function useAgentPerformance(
  orgId: string,
  params?: Parameters<typeof api.getAgentPerformance>[1]
)
```

Both with:
- 1 minute stale time
- Auto-refetch when orgId changes
- TanStack Query caching

## API Client

### Updated `lib/api/sessions.ts`

Added two new functions:

```typescript
export async function getSessionTimeSeries(...)
export async function getAgentPerformance(...)
```

Both with:
- Query parameter building
- Type-safe responses
- Optional filters

## Styling & Design

### Color Palette

- **Success**: `hsl(142, 76%, 36%)` (green)
- **Failure**: `hsl(0, 84%, 60%)` (red)
- **Abandoned**: `hsl(38, 92%, 50%)` (orange)
- **In Progress**: `hsl(217, 91%, 60%)` (blue)
- **Cancelled**: `hsl(215, 20%, 65%)` (gray)
- **Timed Out**: `hsl(280, 65%, 60%)` (purple)
- **Primary**: `hsl(var(--primary))` (brand color)

### Responsive Breakpoints

- Mobile: 1 column
- Tablet (md): 2-3 columns
- Desktop (lg): 6 columns for KPIs

### Chart Heights

- KPI Cards: Auto (based on content)
- Line/Pie Charts: 300px
- Bar/Area Charts: 400px

## Data Flow

```
User selects date range + agent filter
  ↓
Filters update state
  ↓
TanStack Query hooks refetch with new params
  ↓
API client makes requests to Rails API
  ↓
Rails API queries PostgreSQL
  ↓
Response returns through hooks
  ↓
Components render with new data
```

## Real-Time Updates

- **Session list**: Refetch every 10 seconds
- **Statistics**: No auto-refetch (manual refresh via filters)
- **Time series**: 1 minute stale time
- **Agent performance**: 1 minute stale time

## Navigation Integration

To integrate with the sidebar, add to `components/layouts/dashboard-shell.tsx` or sidebar navigation:

```tsx
{
  name: "Analytics",
  href: "/kit/analytics",
  icon: BarChart3,
}
```

## Testing Checklist

### Frontend Tests

- [ ] Empty states render correctly
- [ ] Loading skeletons display
- [ ] Date range presets work
- [ ] Agent filter updates charts
- [ ] Charts render with valid data
- [ ] Responsive layouts on mobile/tablet/desktop
- [ ] Tooltips show correct values
- [ ] Currency formatting is correct
- [ ] Duration formatting handles edge cases

### Backend Tests (Rails)

- [ ] Time series endpoint returns correct data
- [ ] Agent performance endpoint aggregates correctly
- [ ] Date filtering works
- [ ] Agent filtering works
- [ ] Interval parameter (day/week/month) works
- [ ] Empty results handled gracefully
- [ ] Multi-tenant isolation enforced

## Dependencies

### Already Installed
- `recharts`: Charts library
- `date-fns`: Date formatting
- `@tanstack/react-query`: Data fetching
- `lucide-react`: Icons
- `shadcn/ui`: UI components

### No New Dependencies Required

## Backend Implementation Notes

For the Rails API team:

### Database Queries

**Time Series**:
```sql
SELECT
  DATE_TRUNC('day', started_at) as date,
  COUNT(*) as total_sessions,
  COUNT(*) FILTER (WHERE outcome = 'success') as success_count,
  COUNT(*) FILTER (WHERE outcome = 'failure') as failure_count,
  COUNT(*) FILTER (WHERE outcome = 'abandoned') as abandoned_count,
  AVG(total_cost) as avg_cost,
  SUM(total_cost) as total_cost,
  AVG(EXTRACT(EPOCH FROM (ended_at - started_at))) as avg_duration,
  AVG(total_turns) as avg_turns
FROM agent_sessions
WHERE organization_id = ?
  AND started_at BETWEEN ? AND ?
  AND (agent_name = ? OR ? IS NULL)
GROUP BY DATE_TRUNC('day', started_at)
ORDER BY date ASC
```

**Agent Performance**:
```sql
SELECT
  agent_name,
  COUNT(*) as total_sessions,
  (COUNT(*) FILTER (WHERE outcome = 'success')::float / COUNT(*) * 100) as success_rate,
  AVG(total_turns) as avg_turns,
  AVG(EXTRACT(EPOCH FROM (ended_at - started_at))) as avg_duration,
  AVG(total_cost) as avg_cost,
  SUM(total_cost) as total_cost
FROM agent_sessions
WHERE organization_id = ?
  AND started_at BETWEEN ? AND ?
GROUP BY agent_name
ORDER BY total_sessions DESC
```

### Service Layer

Create `SessionAnalyticsService` with methods:
- `time_series(org_id, filters)`
- `agent_performance(org_id, filters)`

### Controller

Add to `Api::V1::SessionsController`:
- `GET /api/v1/:org_id/sessions/time_series`
- `GET /api/v1/:org_id/sessions/agent_performance`

### Authorization

Use existing `SessionPolicy` - same permissions as `statistics` action.

## Future Enhancements

### Phase 2
- Live activity feed (WebSocket)
- Export analytics to CSV/PDF
- Custom date range picker (calendar UI)
- Drill-down from charts to session list
- Comparison mode (compare two time periods)

### Phase 3
- Anomaly detection
- Predictive cost forecasting
- Benchmarking against organization averages
- Custom dashboards
- Scheduled email reports

## Known Limitations

1. **Browser Compatibility**: recharts requires modern browsers (ES6+)
2. **Data Volume**: Large date ranges may slow down rendering - consider pagination
3. **Timezone**: All dates use client timezone (convert to UTC on backend)
4. **Caching**: 1-minute stale time may show slightly outdated data

## File Structure

```
dashboard/src/
├── app/kit/analytics/
│   └── page.tsx                          # Main analytics page
├── components/features/analytics/
│   ├── analytics-kpi-cards.tsx           # KPI display
│   ├── session-time-series-chart.tsx     # Line chart
│   ├── outcome-distribution-chart.tsx    # Pie chart
│   ├── agent-performance-chart.tsx       # Bar chart
│   ├── cost-analysis-chart.tsx           # Area chart
│   ├── date-range-picker.tsx             # Date filter
│   └── agent-filter-select.tsx           # Agent filter
├── hooks/
│   └── use-sessions.ts                   # Updated with new hooks
├── lib/api/
│   └── sessions.ts                       # Updated with new API calls
└── types/
    └── api.ts                            # Updated with new types
```

## Deployment Checklist

### Before Merging
- [ ] Backend API endpoints implemented
- [ ] Backend tests passing
- [ ] Frontend components built
- [ ] Types aligned with API responses
- [ ] Error handling tested
- [ ] Loading states tested
- [ ] Empty states tested
- [ ] Responsive design verified
- [ ] Navigation updated
- [ ] Documentation complete

### After Merging
- [ ] Verify production API endpoints work
- [ ] Check Sentry for errors
- [ ] Monitor query performance
- [ ] Gather user feedback
- [ ] Plan Phase 2 enhancements