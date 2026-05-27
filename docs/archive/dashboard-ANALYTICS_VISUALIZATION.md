# Session Analytics Visualization Strategy

## Overview

This document defines the metrics display and visualization strategy for the crewkit Session Analytics Dashboard. The design prioritizes actionable insights, fast comprehension, and drill-down exploration.

---

## Dashboard Layout

### Primary Layout Structure

```
┌─────────────────────────────────────────────────────────────┐
│ Filters Bar (Date Range, Agent, User, Project)             │
├─────────────────────────────────────────────────────────────┤
│ KPI Cards Row (4-6 cards)                                   │
├─────────────────────────────────────────────────────────────┤
│ Primary Charts (2x2 grid)                                   │
│ ┌─────────────────────┬─────────────────────┐              │
│ │ Sessions Over Time  │ Success Rate Trend  │              │
│ ├─────────────────────┼─────────────────────┤              │
│ │ Cost Breakdown      │ Agent Performance   │              │
│ └─────────────────────┴─────────────────────┘              │
├─────────────────────────────────────────────────────────────┤
│ Secondary Insights (Tables & Detailed Breakdowns)           │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. KPI Cards

### Card Selection & Priority

Display **6 primary KPIs** in a responsive grid (3 cols on desktop, 2 on tablet, 1 on mobile):

#### Row 1 - Volume & Success Metrics
1. **Total Sessions**
   - Value: Large number with compact formatting (e.g., "1.2K", "45.3K")
   - Trend: % change vs previous period (green ↑ / red ↓)
   - Sparkline: Mini line chart (last 7 days)

2. **Success Rate**
   - Value: Percentage (e.g., "87.3%")
   - Trend: Percentage point change (e.g., "+2.4pp")
   - Visual: Progress ring or donut chart (green/red fill)

3. **Completion Rate**
   - Value: Percentage (e.g., "92.1%")
   - Trend: Percentage point change
   - Visual: Progress bar

#### Row 2 - Cost & Performance Metrics
4. **Total Cost**
   - Value: Currency formatted (e.g., "$1,234.56")
   - Trend: % change vs previous period
   - Sparkline: Cost trend (last 7 days)

5. **Avg Cost per Session**
   - Value: Currency (e.g., "$0.45")
   - Trend: % change
   - Visual: Comparison to target (e.g., "15% below target")

6. **Avg Duration**
   - Value: Time formatted (e.g., "2m 34s")
   - Trend: % change
   - Visual: Color-coded (green if faster, amber if slower)

### KPI Card Component Structure

```tsx
interface KPICardProps {
  title: string
  value: string | number
  trend?: {
    value: number
    direction: 'up' | 'down' | 'neutral'
    label: string // "vs last 7 days"
  }
  sparkline?: number[] // Time-series data for mini chart
  visual?: 'progress-ring' | 'progress-bar' | 'sparkline'
  target?: number // Optional target for comparison
}
```

### Data Mapping

```typescript
// From /api/v1/:org_id/sessions/analytics/summary
const kpiData = {
  totalSessions: summary.total_sessions,
  successRate: (summary.successful_sessions / summary.total_sessions) * 100,
  completionRate: (summary.completed_sessions / summary.total_sessions) * 100,
  totalCost: summary.total_cost,
  avgCostPerSession: summary.avg_cost_per_session,
  avgDuration: summary.avg_duration_seconds
}
```

---

## 2. Chart Types & Configurations

### Chart 1: Sessions Over Time (Primary)
**Chart Type:** Area Chart with gradient fill
**Data Source:** `GET /analytics/timeseries?granularity=daily`
**Purpose:** Show session volume trends

**Configuration:**
```typescript
{
  type: 'area',
  xAxis: {
    dataKey: 'date',
    tickFormatter: (value) => format(value, 'MMM dd')
  },
  yAxis: {
    label: 'Sessions',
    tickFormatter: compactNumber // 1K, 2.5K, etc.
  },
  series: [
    {
      dataKey: 'total_sessions',
      fill: 'url(#gradient-primary)',
      stroke: 'hsl(var(--primary))',
      strokeWidth: 2
    }
  ],
  tooltip: {
    labelFormatter: (value) => format(value, 'PPP'), // "Jan 15, 2026"
    formatter: (value, name) => [value, 'Sessions']
  },
  gradient: {
    id: 'gradient-primary',
    stops: [
      { offset: '0%', stopColor: 'hsl(var(--primary))', stopOpacity: 0.8 },
      { offset: '100%', stopColor: 'hsl(var(--primary))', stopOpacity: 0.1 }
    ]
  }
}
```

**Enhancements:**
- Stacked breakdown by status (successful/failed) when toggled
- Brush component for date range selection
- Reference lines for milestones

**Recharts Component:**
```tsx
<ResponsiveContainer width="100%" height={300}>
  <AreaChart data={timeseriesData}>
    <defs>
      <linearGradient id="gradient-primary" x1="0" y1="0" x2="0" y2="1">
        <stop offset="0%" stopColor="hsl(var(--primary))" stopOpacity={0.8}/>
        <stop offset="100%" stopColor="hsl(var(--primary))" stopOpacity={0.1}/>
      </linearGradient>
    </defs>
    <CartesianGrid strokeDasharray="3 3" opacity={0.3} />
    <XAxis dataKey="date" tickFormatter={formatDate} />
    <YAxis tickFormatter={compactNumber} />
    <Tooltip content={<CustomTooltip />} />
    <Area
      type="monotone"
      dataKey="total_sessions"
      stroke="hsl(var(--primary))"
      fill="url(#gradient-primary)"
    />
  </AreaChart>
</ResponsiveContainer>
```

---

### Chart 2: Success Rate Trend
**Chart Type:** Line Chart with dual y-axis
**Data Source:** `GET /analytics/success-trend?granularity=daily`
**Purpose:** Track success rate percentage over time

**Configuration:**
```typescript
{
  type: 'line',
  xAxis: { dataKey: 'date' },
  yAxis: [
    {
      yAxisId: 'left',
      label: 'Success Rate (%)',
      domain: [0, 100]
    }
  ],
  series: [
    {
      yAxisId: 'left',
      dataKey: 'success_rate',
      stroke: 'hsl(var(--success))',
      strokeWidth: 3,
      dot: { r: 4 }
    }
  ],
  referenceLine: {
    y: 80, // Target success rate
    label: 'Target',
    stroke: 'hsl(var(--muted-foreground))',
    strokeDasharray: '5 5'
  }
}
```

**Visual Indicators:**
- Green zone (>80% success)
- Amber zone (60-80% success)
- Red zone (<60% success)
- Shaded background regions

**Recharts Component:**
```tsx
<ResponsiveContainer width="100%" height={300}>
  <LineChart data={successTrendData}>
    <CartesianGrid strokeDasharray="3 3" opacity={0.3} />
    <XAxis dataKey="date" tickFormatter={formatDate} />
    <YAxis domain={[0, 100]} tickFormatter={(v) => `${v}%`} />
    <Tooltip content={<CustomTooltip />} />
    <ReferenceLine y={80} stroke="hsl(var(--muted-foreground))" strokeDasharray="5 5" />
    <Line
      type="monotone"
      dataKey="success_rate"
      stroke="hsl(var(--success))"
      strokeWidth={3}
      dot={{ r: 4 }}
    />
  </LineChart>
</ResponsiveContainer>
```

---

### Chart 3: Cost Breakdown
**Chart Type:** Stacked Bar Chart or Donut Chart
**Data Source:** `GET /analytics/cost-breakdown?dimension=agent`
**Purpose:** Visualize cost distribution

**Option A: Stacked Bar (by time)**
```typescript
{
  type: 'bar',
  stacked: true,
  xAxis: { dataKey: 'date' },
  yAxis: {
    label: 'Cost ($)',
    tickFormatter: currencyCompact // $1K, $2.5K
  },
  series: agents.map(agent => ({
    dataKey: agent.name,
    stackId: 'cost',
    fill: agent.color
  })),
  legend: {
    position: 'bottom',
    iconType: 'square'
  }
}
```

**Option B: Donut Chart (current period)**
```typescript
{
  type: 'pie',
  innerRadius: '60%',
  outerRadius: '80%',
  data: costBreakdownData.map(item => ({
    name: item.dimension_value,
    value: item.total_cost,
    fill: getColorForAgent(item.dimension_value)
  })),
  label: {
    position: 'outside',
    formatter: (entry) => `${entry.name}: ${formatCurrency(entry.value)}`
  },
  centerLabel: {
    value: formatCurrency(totalCost),
    label: 'Total Cost'
  }
}
```

**Recommendation:** Use **Donut Chart** for current period snapshot with option to toggle to stacked bar for trends.

**Recharts Component:**
```tsx
<ResponsiveContainer width="100%" height={300}>
  <PieChart>
    <Pie
      data={costBreakdownData}
      cx="50%"
      cy="50%"
      innerRadius={60}
      outerRadius={80}
      fill="hsl(var(--primary))"
      dataKey="value"
      label={({ name, value }) => `${name}: ${formatCurrency(value)}`}
    >
      {costBreakdownData.map((entry, index) => (
        <Cell key={`cell-${index}`} fill={getColorForAgent(entry.name)} />
      ))}
    </Pie>
    <Tooltip formatter={(value) => formatCurrency(value)} />
    <Legend />
  </PieChart>
</ResponsiveContainer>
```

---

### Chart 4: Agent Performance
**Chart Type:** Horizontal Bar Chart with dual metrics
**Data Source:** `GET /analytics/by-agent`
**Purpose:** Compare agents by sessions and success rate

**Configuration:**
```typescript
{
  type: 'bar',
  layout: 'horizontal',
  xAxis: { type: 'number' },
  yAxis: {
    type: 'category',
    dataKey: 'agent_name',
    width: 120
  },
  series: [
    {
      dataKey: 'session_count',
      fill: 'hsl(var(--primary))',
      name: 'Sessions'
    },
    {
      dataKey: 'success_rate',
      fill: 'hsl(var(--success))',
      name: 'Success Rate (%)',
      yAxisId: 'right'
    }
  ],
  legend: { position: 'top' }
}
```

**Recharts Component:**
```tsx
<ResponsiveContainer width="100%" height={400}>
  <BarChart data={agentData} layout="horizontal">
    <CartesianGrid strokeDasharray="3 3" opacity={0.3} />
    <XAxis type="number" />
    <YAxis type="category" dataKey="agent_name" width={120} />
    <Tooltip content={<CustomTooltip />} />
    <Legend />
    <Bar dataKey="session_count" fill="hsl(var(--primary))" name="Sessions" />
    <Bar dataKey="success_rate" fill="hsl(var(--success))" name="Success Rate (%)" />
  </BarChart>
</ResponsiveContainer>
```

---

### Chart 5: User Engagement Table
**Chart Type:** Data Table with inline sparklines
**Data Source:** `GET /analytics/by-user`
**Purpose:** Detailed user activity breakdown

**Columns:**
1. User Name
2. Total Sessions (sortable)
3. Success Rate (sortable, color-coded)
4. Avg Duration (sortable)
5. Total Cost (sortable)
6. Last 7 Days Activity (sparkline)

**Table Component:**
```tsx
<Table>
  <TableHeader>
    <TableRow>
      <TableHead onClick={() => handleSort('user_name')}>User</TableHead>
      <TableHead onClick={() => handleSort('session_count')}>Sessions</TableHead>
      <TableHead onClick={() => handleSort('success_rate')}>Success Rate</TableHead>
      <TableHead onClick={() => handleSort('avg_duration')}>Avg Duration</TableHead>
      <TableHead onClick={() => handleSort('total_cost')}>Total Cost</TableHead>
      <TableHead>Activity</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    {userData.map(user => (
      <TableRow key={user.user_id}>
        <TableCell>{user.user_name}</TableCell>
        <TableCell>{compactNumber(user.session_count)}</TableCell>
        <TableCell>
          <Badge variant={getSuccessBadgeVariant(user.success_rate)}>
            {user.success_rate.toFixed(1)}%
          </Badge>
        </TableCell>
        <TableCell>{formatDuration(user.avg_duration)}</TableCell>
        <TableCell>{formatCurrency(user.total_cost)}</TableCell>
        <TableCell>
          <MiniSparkline data={user.activity_trend} />
        </TableCell>
      </TableRow>
    ))}
  </TableBody>
</Table>
```

---

### Chart 6: Project Analytics Table
**Chart Type:** Grouped data table with metrics
**Data Source:** `GET /analytics/by-project`
**Purpose:** Project-level performance insights

**Similar structure to User Engagement Table with project-specific columns.**

---

### Chart 7: Coaching Mode Effectiveness
**Chart Type:** Comparative Bar Chart
**Data Source:** `GET /analytics/coaching`
**Purpose:** Compare coaching vs autonomous mode

**Configuration:**
```typescript
{
  type: 'bar',
  xAxis: { dataKey: 'metric' }, // ['Sessions', 'Success Rate', 'Avg Duration']
  yAxis: { label: 'Value' },
  series: [
    {
      dataKey: 'coaching_mode',
      fill: 'hsl(var(--chart-1))',
      name: 'Coaching Mode'
    },
    {
      dataKey: 'autonomous_mode',
      fill: 'hsl(var(--chart-2))',
      name: 'Autonomous Mode'
    }
  ],
  legend: { position: 'top' }
}
```

---

## 3. Filter UI Design

### Filter Bar Component

```tsx
<div className="flex flex-wrap gap-4 p-4 border-b">
  {/* Date Range Picker */}
  <Popover>
    <PopoverTrigger asChild>
      <Button variant="outline" className="w-[280px] justify-start">
        <CalendarIcon className="mr-2 h-4 w-4" />
        {dateRange.from ? (
          dateRange.to ? (
            <>
              {format(dateRange.from, "LLL dd, y")} -{" "}
              {format(dateRange.to, "LLL dd, y")}
            </>
          ) : (
            format(dateRange.from, "LLL dd, y")
          )
        ) : (
          <span>Pick a date range</span>
        )}
      </Button>
    </PopoverTrigger>
    <PopoverContent className="w-auto p-0" align="start">
      <Calendar
        initialFocus
        mode="range"
        selected={dateRange}
        onSelect={setDateRange}
      />
    </PopoverContent>
  </Popover>

  {/* Granularity Selector */}
  <Select value={granularity} onValueChange={setGranularity}>
    <SelectTrigger className="w-[180px]">
      <SelectValue placeholder="Granularity" />
    </SelectTrigger>
    <SelectContent>
      <SelectItem value="hourly">Hourly</SelectItem>
      <SelectItem value="daily">Daily</SelectItem>
      <SelectItem value="weekly">Weekly</SelectItem>
      <SelectItem value="monthly">Monthly</SelectItem>
    </SelectContent>
  </Select>

  {/* Agent Multi-Select */}
  <MultiSelect
    options={agents}
    value={selectedAgents}
    onChange={setSelectedAgents}
    placeholder="Filter by agent"
  />

  {/* User Multi-Select */}
  <MultiSelect
    options={users}
    value={selectedUsers}
    onChange={setSelectedUsers}
    placeholder="Filter by user"
  />

  {/* Project Multi-Select */}
  <MultiSelect
    options={projects}
    value={selectedProjects}
    onChange={setSelectedProjects}
    placeholder="Filter by project"
  />

  {/* Clear Filters */}
  <Button variant="ghost" onClick={clearAllFilters}>
    Clear All
  </Button>
</div>
```

### Filter State Management

```typescript
interface AnalyticsFilters {
  dateRange: {
    from: Date
    to: Date
  }
  granularity: 'hourly' | 'daily' | 'weekly' | 'monthly'
  agents: string[]
  users: string[]
  projects: string[]
}

// URL-synced filters for sharing links
const [filters, setFilters] = useQueryParams<AnalyticsFilters>({
  dateRange: {
    from: subDays(new Date(), 30),
    to: new Date()
  },
  granularity: 'daily',
  agents: [],
  users: [],
  projects: []
})
```

---

## 4. Drill-Down Interactions

### Interaction Patterns

1. **Chart Click → Filtered View**
   - Click agent in bar chart → Filter all charts to that agent
   - Click date in timeline → Show sessions for that day in modal
   - Click cost segment → Show breakdown by sub-dimension

2. **KPI Card Click → Detailed Modal**
   - Click "Total Sessions" → Modal with sessions table
   - Click "Success Rate" → Modal with success/failure breakdown
   - Click "Total Cost" → Modal with cost details

3. **Table Row Click → Detail Panel**
   - Click user row → Slide-out panel with user session history
   - Click project row → Panel with project-specific metrics
   - Click agent row → Panel with agent configuration details

### Example: Agent Drill-Down

```tsx
const handleAgentClick = (agentName: string) => {
  // Update filters to focus on this agent
  setFilters(prev => ({
    ...prev,
    agents: [agentName]
  }))

  // Scroll to agent detail section
  scrollToElement('#agent-details')

  // Track analytics event
  trackEvent('dashboard_agent_drilldown', { agent: agentName })
}

// Agent Detail Panel
<Sheet open={!!selectedAgent} onOpenChange={() => setSelectedAgent(null)}>
  <SheetContent className="w-[600px]">
    <SheetHeader>
      <SheetTitle>{selectedAgent?.name} Performance</SheetTitle>
    </SheetHeader>
    <div className="space-y-6">
      {/* Agent-specific KPIs */}
      <div className="grid grid-cols-2 gap-4">
        <KPICard title="Sessions" value={agentStats.session_count} />
        <KPICard title="Success Rate" value={`${agentStats.success_rate}%`} />
      </div>

      {/* Recent sessions table */}
      <SessionsTable filters={{ agent: selectedAgent.name }} />

      {/* Configuration link */}
      <Button asChild>
        <Link href={`/agents/${selectedAgent.id}/config`}>
          View Configuration
        </Link>
      </Button>
    </div>
  </SheetContent>
</Sheet>
```

---

## 5. Empty States & Loading States

### Empty States

**No Data Available:**
```tsx
<div className="flex flex-col items-center justify-center h-[400px] text-center">
  <BarChartIcon className="h-16 w-16 text-muted-foreground mb-4" />
  <h3 className="text-lg font-semibold mb-2">No data available</h3>
  <p className="text-sm text-muted-foreground max-w-md">
    There are no sessions matching your current filters.
    Try adjusting the date range or clearing filters.
  </p>
  <Button variant="outline" onClick={clearFilters} className="mt-4">
    Clear Filters
  </Button>
</div>
```

**No Sessions Yet:**
```tsx
<div className="flex flex-col items-center justify-center h-[400px] text-center">
  <RocketIcon className="h-16 w-16 text-muted-foreground mb-4" />
  <h3 className="text-lg font-semibold mb-2">Get started with crewkit</h3>
  <p className="text-sm text-muted-foreground max-w-md mb-4">
    Start your first agent session to see analytics appear here.
  </p>
  <Button asChild>
    <Link href="/docs/quickstart">View Quickstart Guide</Link>
  </Button>
</div>
```

### Loading States

**Skeleton Loaders:**
```tsx
// KPI Card Skeleton
<Card>
  <CardHeader>
    <Skeleton className="h-4 w-24" />
  </CardHeader>
  <CardContent>
    <Skeleton className="h-8 w-32 mb-2" />
    <Skeleton className="h-3 w-20" />
  </CardContent>
</Card>

// Chart Skeleton
<div className="space-y-4">
  <Skeleton className="h-6 w-48" />
  <Skeleton className="h-[300px] w-full" />
</div>

// Table Skeleton
<Table>
  <TableHeader>
    <TableRow>
      {columns.map(col => (
        <TableHead key={col}>
          <Skeleton className="h-4 w-20" />
        </TableHead>
      ))}
    </TableRow>
  </TableHeader>
  <TableBody>
    {[...Array(5)].map((_, i) => (
      <TableRow key={i}>
        {columns.map((_, j) => (
          <TableCell key={j}>
            <Skeleton className="h-4 w-24" />
          </TableCell>
        ))}
      </TableRow>
    ))}
  </TableBody>
</Table>
```

**Progressive Loading:**
- Load KPI cards first (fastest query)
- Load primary charts next
- Load tables last (heaviest queries)
- Show stale data with "Refreshing..." indicator

```tsx
const { data: summary, isLoading: summaryLoading } = useAnalytics('summary')
const { data: timeseries, isLoading: timeseriesLoading } = useAnalytics('timeseries')
const { data: byAgent, isLoading: byAgentLoading } = useAnalytics('by-agent')

return (
  <>
    {summaryLoading ? <KPISkeletons /> : <KPICards data={summary} />}
    {timeseriesLoading ? <ChartSkeleton /> : <SessionsChart data={timeseries} />}
    {byAgentLoading ? <TableSkeleton /> : <AgentTable data={byAgent} />}
  </>
)
```

---

## 6. Responsive Behavior

### Desktop (>1024px)
- 2x2 chart grid
- Full-width filter bar
- 6 KPI cards in 3 columns
- Tables with all columns visible

### Tablet (768px - 1024px)
- 1x2 chart grid (stacked)
- Collapsible filter drawer
- 4 KPI cards in 2 columns
- Tables with horizontal scroll

### Mobile (<768px)
- Single column layout
- Bottom sheet for filters
- 2 KPI cards visible, swipeable carousel for rest
- Cards replace tables (tap to expand)

### Responsive Chart Configuration

```tsx
<ResponsiveContainer
  width="100%"
  height={isDesktop ? 300 : isMobile ? 200 : 250}
>
  <AreaChart data={data}>
    {/* Hide labels on mobile */}
    <XAxis
      dataKey="date"
      tickFormatter={isMobile ? formatShortDate : formatDate}
      hide={isMobile}
    />
    {/* Reduce tick count on tablet */}
    <YAxis
      tickCount={isMobile ? 3 : isTablet ? 5 : 7}
    />
    {/* Simpler tooltip on mobile */}
    <Tooltip
      content={isMobile ? <SimplifiedTooltip /> : <DetailedTooltip />}
    />
  </AreaChart>
</ResponsiveContainer>
```

---

## 7. Recharts Component Recommendations

### Core Components to Use

1. **ResponsiveContainer** - Wrapper for all charts
2. **AreaChart** - Sessions over time
3. **LineChart** - Success rate trends
4. **PieChart** - Cost breakdown
5. **BarChart** - Agent performance
6. **ComposedChart** - Multi-metric comparisons

### Custom Components to Build

1. **CustomTooltip** - Styled tooltips with formatted values
2. **CustomLegend** - Interactive legend with toggle
3. **MiniSparkline** - Inline table sparklines
4. **ChartContainer** - Wrapper with loading/error states
5. **AxisTick** - Custom axis labels

### Example Custom Tooltip

```tsx
const CustomTooltip = ({ active, payload, label }) => {
  if (!active || !payload?.length) return null

  return (
    <Card className="p-3 shadow-lg">
      <p className="font-semibold text-sm mb-2">
        {format(new Date(label), 'PPP')}
      </p>
      {payload.map((entry, index) => (
        <div key={index} className="flex items-center gap-2">
          <div
            className="w-3 h-3 rounded-full"
            style={{ backgroundColor: entry.color }}
          />
          <span className="text-sm">
            {entry.name}: {formatValue(entry.value, entry.dataKey)}
          </span>
        </div>
      ))}
    </Card>
  )
}
```

---

## 8. Performance Optimizations

### Data Fetching Strategy

1. **Stale-While-Revalidate** for all endpoints
2. **Cache time:** 5 minutes for aggregate data
3. **Refetch interval:** 60 seconds for active dashboards
4. **Prefetch:** Load summary while user navigates

```typescript
const { data, isLoading } = useQuery({
  queryKey: ['analytics', 'summary', filters],
  queryFn: () => fetchAnalyticsSummary(filters),
  staleTime: 5 * 60 * 1000, // 5 minutes
  cacheTime: 10 * 60 * 1000, // 10 minutes
  refetchInterval: 60 * 1000, // 1 minute
  refetchOnWindowFocus: true
})
```

### Chart Rendering Optimizations

1. **Virtualized Tables** - For >100 rows
2. **Memoized Components** - Wrap charts in React.memo
3. **Debounced Filters** - 300ms delay on filter changes
4. **Lazy Load Charts** - Intersection Observer for below-fold charts

```tsx
const SessionsChart = memo(({ data }) => (
  <ResponsiveContainer>
    <AreaChart data={data}>
      {/* Chart config */}
    </AreaChart>
  </ResponsiveContainer>
))

const LazyChart = ({ children }) => {
  const [ref, inView] = useInView({ triggerOnce: true })

  return (
    <div ref={ref}>
      {inView ? children : <ChartSkeleton />}
    </div>
  )
}
```

---

## 9. Accessibility Considerations

### ARIA Labels

```tsx
<ResponsiveContainer role="img" aria-label="Sessions over time chart">
  <AreaChart>
    {/* Chart config */}
  </AreaChart>
</ResponsiveContainer>

<Table aria-label="Agent performance table">
  {/* Table content */}
</Table>
```

### Keyboard Navigation

- Filter dropdowns: Arrow keys, Enter to select
- Charts: Tab to focus, Arrow keys to navigate data points
- Tables: Arrow keys to navigate cells, Enter to expand rows

### Color Contrast

- Ensure chart colors meet WCAG AA standards
- Use patterns in addition to colors for critical distinctions
- Provide text alternatives for color-coded metrics

---

## 10. Implementation Checklist

### Phase 1: Foundation
- [ ] Set up Recharts and dependencies
- [ ] Create base chart components (Area, Line, Bar, Pie)
- [ ] Build KPI card component with variants
- [ ] Implement filter bar with all controls
- [ ] Create custom tooltip and legend components

### Phase 2: Chart Integration
- [ ] Sessions over time (AreaChart)
- [ ] Success rate trend (LineChart)
- [ ] Cost breakdown (PieChart)
- [ ] Agent performance (BarChart)
- [ ] User engagement table
- [ ] Project analytics table
- [ ] Coaching effectiveness chart

### Phase 3: Interactions
- [ ] Chart click handlers for drill-down
- [ ] Filter synchronization across charts
- [ ] URL-synced filter state
- [ ] Detail panels/modals for expanded views
- [ ] Export functionality (CSV, PNG)

### Phase 4: Polish
- [ ] Responsive layouts for all breakpoints
- [ ] Empty states for all charts
- [ ] Loading skeletons
- [ ] Error boundaries
- [ ] Accessibility audit
- [ ] Performance optimization

---

## Summary

This visualization strategy provides:
- **Clear hierarchy**: KPIs → Primary charts → Detailed tables
- **Actionable insights**: Drill-down patterns reveal root causes
- **Fast comprehension**: Appropriate chart types for each dataset
- **Responsive design**: Mobile-friendly with progressive enhancement
- **Performance**: Optimized rendering and data fetching

**Key decisions:**
- Use area charts for volume trends (sessions over time)
- Use line charts for percentage metrics (success rate)
- Use donut charts for current breakdowns (cost distribution)
- Use horizontal bar charts for comparisons (agent performance)
- Use tables with sparklines for detailed user/project data
- Prioritize KPIs with trend indicators and sparklines
- Implement progressive loading and stale-while-revalidate caching