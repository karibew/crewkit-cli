# Analytics Component System Setup

## Installation

The analytics component system has been created. Install the missing dependency:

```bash
cd ~/code/karibew/crewkit/dashboard
npm install react-day-picker
```

## File Structure

```
src/components/analytics/
├── README.md                              # Quick start guide
├── USAGE.md                               # Detailed usage documentation
├── DESIGN_SPEC.md                         # Design specifications
├── index.ts                               # Barrel exports
│
├── kpi/
│   ├── kpi-card.tsx                      # Individual metric card
│   └── kpi-grid.tsx                      # Responsive grid layout
│
├── data-table/
│   └── advanced-table.tsx                # Full-featured data table
│
├── charts/
│   ├── line-chart.tsx                    # Line chart wrapper
│   ├── bar-chart.tsx                     # Bar chart wrapper
│   └── area-chart.tsx                    # Area chart wrapper
│
├── timeline/
│   └── timeline.tsx                      # Vertical event timeline
│
├── filter-bar/
│   └── filter-bar.tsx                    # Unified filter controls
│
└── examples/
    └── session-analytics-example.tsx     # Complete demo page
```

## New UI Components Created

```
src/components/ui/
├── calendar.tsx                          # Date picker calendar
└── popover.tsx                           # Popover component
```

## Quick Start

```tsx
import {
  KpiCard,
  KpiGrid,
  AdvancedTable,
  AnalyticsLineChart,
  Timeline,
  FilterBar,
} from "@/components/analytics";
import { Users, DollarSign, TrendingUp } from "lucide-react";

function Dashboard() {
  return (
    <div className="space-y-8">
      {/* KPI Overview */}
      <KpiGrid columns={3}>
        <KpiCard
          title="Total Sessions"
          value={1234}
          icon={Users}
          trend={{ value: 12.5, label: "vs last month" }}
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
      </KpiGrid>

      {/* Data Table */}
      <AdvancedTable
        data={sessions}
        columns={columns}
        keyExtractor={(row) => row.id}
        pageSize={25}
      />
    </div>
  );
}
```

## Usage Examples

See `src/components/analytics/examples/session-analytics-example.tsx` for a complete working example demonstrating all components.

## Next Steps

1. **Install dependency**: `npm install react-day-picker`
2. **Review documentation**: Read `USAGE.md` for detailed API docs
3. **Test example page**: Import and render `SessionAnalyticsExample` component
4. **Integrate into app**: Use components in your existing pages
5. **Customize themes**: Adjust chart colors in `globals.css`

## Component Summary

### KPI Components
- **KpiCard** - Display metrics with trends, sparklines, loading/error states
- **KpiGrid** - Responsive grid (1-4 columns)

### Data Table
- **AdvancedTable** - Sortable, searchable, paginated table with export

### Charts
- **AnalyticsLineChart** - Line chart with recharts
- **AnalyticsBarChart** - Bar chart (stacked or grouped)
- **AnalyticsAreaChart** - Area chart (stacked or overlapping)

### Timeline
- **Timeline** - Vertical timeline with expandable details, status indicators

### Filters
- **FilterBar** - Unified filter controls (search, select, date range)

## Design Principles

All components follow crewkit's Vercel-inspired design philosophy:

1. **Minimalism** - Remove every unnecessary element
2. **Functional Over Fancy** - Design serves usability
3. **Consistency** - Same component = same treatment everywhere
4. **Speed** - Minimal DOM, fast rendering
5. **Accessibility** - WCAG 2.1 AA compliant

## Features

- ✅ Full TypeScript support
- ✅ Responsive (mobile-first)
- ✅ Accessible (keyboard navigation, ARIA, screen readers)
- ✅ Themeable (shadcn/ui CSS variables)
- ✅ Loading and error states
- ✅ Customizable variants (success, warning, danger)
- ✅ Export functionality (table)
- ✅ Date range filtering
- ✅ Sparklines in KPI cards
- ✅ Expandable timeline details

## Integration Points

Use these components for:
- **Session analytics** - Filter, view, analyze agent sessions
- **Experiment metrics** - Compare A/B test results
- **Agent performance** - Track success rates, costs, durations
- **Cost tracking** - Monitor spending across projects
- **Activity timelines** - Show session event logs
- **Dashboard KPIs** - Display key metrics at a glance

## Support

Questions? Check:
- `README.md` - Quick reference
- `USAGE.md` - Detailed API documentation
- `DESIGN_SPEC.md` - Design decisions and architecture
- `examples/` - Working code examples