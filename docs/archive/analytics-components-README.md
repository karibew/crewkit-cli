# Analytics Component System

Reusable, accessible analytics UI components for crewkit dashboard.

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

function MyDashboard() {
  return (
    <>
      <KpiGrid columns={3}>
        <KpiCard title="Sessions" value={1234} icon={Users} />
        <KpiCard title="Cost" value="$45.67" icon={DollarSign} />
        <KpiCard title="Success Rate" value="94%" icon={TrendingUp} />
      </KpiGrid>

      <AdvancedTable
        data={sessions}
        columns={columns}
        keyExtractor={(row) => row.id}
      />
    </>
  );
}
```

## Components

### KPI Cards
Display key metrics with trends and sparklines
- **KpiCard** - Individual metric card
- **KpiGrid** - Responsive grid layout

### Data Tables
Advanced table with sorting, search, pagination
- **AdvancedTable** - Full-featured data table

### Charts
Recharts wrappers with consistent styling
- **AnalyticsLineChart** - Line chart
- **AnalyticsBarChart** - Bar chart
- **AnalyticsAreaChart** - Area chart

### Timeline
Vertical event timeline with expandable details
- **Timeline** - Timeline component

### Filters
Unified filter controls
- **FilterBar** - Filter bar with search, select, date range

## Documentation

- **[USAGE.md](./USAGE.md)** - Detailed usage guide with examples
- **[DESIGN_SPEC.md](./DESIGN_SPEC.md)** - Design specifications and architecture
- **[examples/](./examples/)** - Complete working examples

## Features

- **Minimalist Design** - Vercel-inspired, functional over fancy
- **Accessible** - WCAG 2.1 AA compliant
- **Responsive** - Mobile-first, desktop-enhanced
- **Type-Safe** - Full TypeScript support
- **Themeable** - shadcn/ui theming system
- **Performant** - Optimized for large datasets

## Installation

Components use these peer dependencies (already in crewkit):
- `recharts` - Charts
- `date-fns` - Date formatting
- `react-day-picker` - Date range picker
- `lucide-react` - Icons

No additional installation needed.

## Example

See `examples/session-analytics-example.tsx` for a complete implementation demonstrating all components.

## Design Philosophy

1. **Minimalism is a Feature** - Remove every unnecessary element
2. **Functional Over Fancy** - Design serves usability
3. **Consistency Creates Calm** - Same component = same treatment
4. **Speed Through Simplicity** - Minimal DOM, fast rendering
5. **Accessible by Default** - Semantic HTML, keyboard navigation

## Support

Questions? Check:
- [USAGE.md](./USAGE.md) for API documentation
- [DESIGN_SPEC.md](./DESIGN_SPEC.md) for design decisions
- [examples/](./examples/) for working code

## License

Part of crewkit platform. See main LICENSE file.