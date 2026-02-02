---
doc_tier: 1
doc_type: report
doc_status: archived
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
tags: [dashboard, design-review, frontend]
---

# Design Review: Project Details Page

**Page:** `/kit/projects/[id]`
**Status:** Design consistency review completed
**Date:** 2026-01-03

---

## Issues Identified

### 1. Breadcrumb Shows UUID Instead of Project Name

**Problem:**
The `Header` component uses `getBreadcrumbs()` function that constructs breadcrumbs from URL segments. For dynamic routes like `/kit/projects/[id]`, the UUID is displayed instead of the human-readable project name.

**Current Flow:**
```
/kit/projects/[id] → Header extracts "id" segment → "Id" displayed in breadcrumb
```

**Expected Flow:**
```
/kit/projects/[id] → Header should show project name from data context
```

**Location:**
`dashboard/src/components/layouts/header.tsx`, lines 26-54

**Impact:**
- Poor UX: Users see "Id" (the segment) instead of "My Cool Project"
- Inconsistent with detail pages that use `DashboardShell` (Resources page shows actual names in descriptions)
- Confusing for navigation context

---

### 2. Redundant Header Information

**Problem:**
The project details page has two separate header sections that duplicate information:

**Header Section 1** (lines 65-91 in page.tsx):
```tsx
<div className="flex items-center justify-between">
  <div className="flex items-center gap-4">
    {/* Back button, icon, title, slug */}
  </div>
  <Button variant="outline">Settings</Button>
</div>
```

**Header Section 2** (lines 94-131 in page.tsx):
```tsx
<Card>
  <CardHeader>
    <CardTitle>Project Details</CardTitle>
    <CardDescription>Overview of your project</CardDescription>
  </CardHeader>
  {/* Read-only fields */}
</Card>
```

**Issues:**
- Both sections present the same "project details" conceptually
- First header is custom-built; second uses Card component (inconsistent)
- The Card's title "Project Details" adds no value—it's already shown in the header
- Spacing/typography differs between components
- Creates visual clutter

---

## Recommended Refactoring

### Solution 1: Adopt `DashboardShell` Pattern (Recommended)

The `/kit/resources/[id]` page provides the ideal pattern. Use `DashboardShell` to standardize layout:

```tsx
export default function ProjectDetailPage({ params }: ProjectDetailPageProps) {
  const { id } = use(params);
  const { data: project, isLoading, error } = useProject(id);

  if (isLoading) return <ProjectDetailLoading />;

  if (error || !project) {
    return (
      <DashboardShell>
        <div className="space-y-6">
          <Button variant="ghost" size="sm" asChild className="-ml-2">
            <Link href="/kit/projects">
              <ArrowLeft className="mr-2 h-4 w-4" />
              Back to Projects
            </Link>
          </Button>
          <Alert variant="destructive">
            <AlertCircle className="h-4 w-4" />
            <AlertTitle>Project Not Found</AlertTitle>
            <AlertDescription>
              The project you're looking for doesn't exist or you don't have access.
            </AlertDescription>
          </Alert>
        </div>
      </DashboardShell>
    );
  }

  return (
    <DashboardShell
      title={
        <div className="flex items-center gap-3">
          <FolderGit2 className="h-6 w-6 text-slate-600" />
          <div>
            <h1 className="text-3xl font-bold tracking-tight">{project.name}</h1>
            <p className="text-muted-foreground font-mono text-sm">{project.slug}</p>
          </div>
        </div>
      }
      actions={
        <Button variant="outline" asChild>
          <Link href={`/kit/projects/${id}/settings`}>
            <Settings className="mr-2 h-4 w-4" />
            Settings
          </Link>
        </Button>
      }
    >
      <div className="space-y-6">
        {/* Back button */}
        <Button variant="ghost" size="sm" asChild className="-ml-2">
          <Link href="/kit/projects">
            <ArrowLeft className="mr-2 h-4 w-4" />
            Back to Projects
          </Link>
        </Button>

        {/* Project Info Card - Renamed to clarify purpose */}
        <Card>
          <CardHeader>
            <CardTitle>Information</CardTitle>
            <CardDescription>Key details about this project</CardDescription>
          </CardHeader>
          <CardContent className="space-y-4">
            <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
              <div className="space-y-1">
                <p className="text-sm font-medium text-muted-foreground flex items-center gap-2">
                  <Calendar className="h-4 w-4" />
                  Created
                </p>
                <p className="text-sm">
                  {format(new Date(project.created_at), "MMMM d, yyyy")}
                </p>
              </div>
              {project.git_remote && (
                <div className="space-y-1">
                  <p className="text-sm font-medium text-muted-foreground flex items-center gap-2">
                    <GitBranch className="h-4 w-4" />
                    Git Remote
                  </p>
                  <p className="text-sm font-mono truncate">
                    {project.git_remote}
                  </p>
                </div>
              )}
              <div className="space-y-1">
                <p className="text-sm font-medium text-muted-foreground">
                  Project ID
                </p>
                <p className="text-sm font-mono text-muted-foreground">
                  {project.id}
                </p>
              </div>
            </div>
          </CardContent>
        </Card>

        {/* Quick Actions */}
        <div className="grid gap-4 md:grid-cols-2">
          <Card className="hover:shadow-md transition-shadow cursor-pointer">
            <Link href={`/kit/agents?project=${project.id}`} className="block">
              <CardHeader>
                <CardTitle className="flex items-center gap-2 text-lg">
                  <Bot className="h-5 w-5" />
                  Agents
                </CardTitle>
                <CardDescription>
                  View and manage agent configurations for this project
                </CardDescription>
              </CardHeader>
            </Link>
          </Card>

          <Card className="hover:shadow-md transition-shadow cursor-pointer">
            <Link
              href={`/kit/experiments?project=${project.id}`}
              className="block"
            >
              <CardHeader>
                <CardTitle className="flex items-center gap-2 text-lg">
                  <FlaskConical className="h-5 w-5" />
                  Experiments
                </CardTitle>
                <CardDescription>
                  Run A/B tests on agent configurations in this project
                </CardDescription>
              </CardHeader>
            </Link>
          </Card>
        </div>
      </div>
    </DashboardShell>
  );
}
```

---

## Design System Consistency Analysis

### Current State: Mixed Patterns

**Page Structure Analysis:**

| Aspect | Current | Standard Pattern |
|--------|---------|------------------|
| **Header approach** | Custom divs + manual spacing | `DashboardShell` component |
| **Title display** | Direct text with icon | Icon + text in DashboardShell title |
| **Breadcrumb behavior** | URL-based (shows UUID) | Data-aware (shows resource name) |
| **Back navigation** | Not present | Back button via DashboardShell actions or body |
| **Primary action** | Right-aligned in custom header | DashboardShell actions prop |
| **Info section title** | "Project Details" (generic) | "Information" (clarifies purpose) |
| **Spacing/typography** | Consistent with Tailwind | Consistent ✓ |
| **Loading state** | Dedicated skeleton | Proper ✓ |
| **Error state** | Custom error div | Should use Alert component |

**Comparison with Similar Pages:**

**Resources Detail Page** (`/kit/resources/[id]`) ✓ REFERENCE IMPLEMENTATION:
- Uses `DashboardShell` for unified header
- Shows resource name + metadata in description area
- Clear visual hierarchy
- Consistent with shadcn/ui patterns
- Actions integrated naturally

**Projects Settings Page** (`/kit/projects/[id]/settings`) ✓ PARTIAL:
- Uses custom header (not DashboardShell) but correctly
- Back button + title + subtitle pattern
- Card-based content layout
- Inconsistent with Resources page pattern

**Projects List Page** (`/kit/projects`) ✓ REFERENCE:
- Uses custom header for simple layout (acceptable for non-detailed pages)
- Clear title + subtitle
- Primary action on the right

---

## Specific Design Issues

### 1. Card Title Naming

**Current:** "Project Details"
**Problem:** Too generic, doesn't clarify what information is shown

**Recommendation:** Rename to "Information" (matches Resource page) or "Key Details"

**Location:** Line 96 in page.tsx

---

### 2. Visual Hierarchy

**Current:**
```
Header (title, icon, slug)
    ↓ (gap-6)
Card: "Project Details" (redundant title)
    ↓ (gap-4)
Card grid with info fields
```

**Improved (using DashboardShell):**
```
DashboardShell
  - Title: Icon + project name + slug
  - Actions: Settings button
    ↓ (gap-6)
Back button
    ↓ (gap-6)
Card: "Information"
    ↓ (gap-4)
Info grid
```

This eliminates the duplicate "Project Details" concept.

---

### 3. Spacing Consistency

**Current spacing analysis:**
- Header to first card: `gap-6` ✓ Correct
- Between cards: `gap-4` (Quick Actions grid) ✓ Correct
- Within card: `gap-4` ✓ Correct

**Issue:** Header section uses manual layout; DashboardShell provides the same with less code.

---

## Breadcrumb Solution Approaches

### Approach A: Context-aware Breadcrumbs (Recommended)

Modify `Header` to accept optional context data:

```tsx
interface Breadcrumb {
  label: string;
  href?: string;
}

interface HeaderProps {
  breadcrumbs?: Breadcrumb[]; // Optional manual breadcrumbs
}

export function Header({ breadcrumbs: manualBreadcrumbs }: HeaderProps) {
  const pathname = usePathname();

  // Use manual breadcrumbs if provided, otherwise generate from path
  const breadcrumbs = manualBreadcrumbs ?? getBreadcrumbs(pathname);

  // ... rest of Header
}
```

**Usage on detail pages:**
```tsx
export default function ProjectDetailPage({ params }: ProjectDetailPageProps) {
  const { id } = use(params);
  const { data: project, isLoading } = useProject(id);

  return (
    <>
      <Header
        breadcrumbs={project && [
          { label: "Projects", href: "/kit/projects" },
          { label: project.name }
        ]}
      />
      {/* Page content */}
    </>
  );
}
```

**Pros:**
- Explicit control over breadcrumb labels
- No breaking changes to existing pages
- Works with any component structure

**Cons:**
- Requires Header modification
- Manual breadcrumb management on each detail page

---

### Approach B: Use DashboardShell for All Detail Pages

Standardize all detail pages (Projects, Resources, Sessions, etc.) on `DashboardShell`:

```tsx
// All detail pages follow this pattern:
<DashboardShell
  title={/* Main title/info */}
  description={/* Optional metadata */}
  actions={/* Action buttons */}
>
  {/* Page content */}
</DashboardShell>
```

**Pros:**
- Consistent pattern across entire dashboard
- No header changes needed
- Cleaner code

**Cons:**
- Requires refactoring existing detail pages
- DashboardShell doesn't include breadcrumbs (but header still does via pathname)

---

### Approach C: Create PageHeader Component (Enhanced)

Create a reusable `PageHeader` component that encapsulates the detail page pattern:

```tsx
interface PageHeaderProps {
  icon: React.ReactNode;
  title: string;
  subtitle?: string;
  breadcrumbs?: Breadcrumb[];
  actions?: React.ReactNode;
}

export function PageHeader({
  icon,
  title,
  subtitle,
  breadcrumbs,
  actions,
}: PageHeaderProps) {
  return (
    <div className="flex items-center justify-between">
      <div className="flex items-center gap-4">
        <div className="flex h-12 w-12 items-center justify-center rounded-lg bg-slate-100">
          {icon}
        </div>
        <div>
          <h1 className="text-3xl font-bold tracking-tight">{title}</h1>
          {subtitle && <p className="text-muted-foreground font-mono text-sm">{subtitle}</p>}
        </div>
      </div>
      {actions && <div className="flex items-center gap-2">{actions}</div>}
    </div>
  );
}
```

---

## Implementation Priority

### High Priority
1. **Adopt DashboardShell pattern** - Standardizes layout, reduces redundancy
2. **Fix breadcrumbs for detail pages** - Improve navigation clarity
3. **Rename Card title to "Information"** - Reduce cognitive load

### Medium Priority
4. Use `Alert` component for error states (consistency)
5. Extract `PageHeader` component if creating other detail pages

### Low Priority
6. Add loading skeleton animation enhancements

---

## Spacing & Typography Audit

### Grid System
- Page level gap: `gap-6` ✓
- Card internal gap: `gap-4` ✓
- Quick Actions grid: `md:grid-cols-2` ✓

### Typography
- Main title: `text-3xl font-bold tracking-tight` ✓
- Secondary text (slug): `text-muted-foreground font-mono` ✓
- Card titles: `text-lg font-semibold` (implicit) ✓
- Card descriptions: `text-sm text-muted-foreground` ✓

### Colors
- Icon background: `bg-slate-100` ✓ (matches shadcn conventions)
- Icon color: `text-slate-600` ✓
- Text: shadcn muted-foreground ✓

**All consistent with shadcn/ui and crewkit design system.**

---

## Component Naming & Structure

### Current Structure
```
ProjectDetailPage
├── Header (custom manual layout)
├── Card (Project Details)
│   └── Grid (created, git remote, ID)
└── Quick Actions (2 cards)
```

### Recommended Structure (using DashboardShell)
```
ProjectDetailPage
├── DashboardShell (provides header wrapper)
│   ├── Title (with icon, name, slug)
│   ├── Actions (Settings button)
│   └── Children:
│       ├── Back button
│       ├── Card (Information)
│       │   └── Grid (created, git remote, ID)
│       └── Quick Actions (2 cards)
```

**Benefits:**
- Consistent structure with Resources page
- Single source of truth for page layout
- Reduces custom styling

---

## Recommendations Summary

| Issue | Solution | Complexity | Impact |
|-------|----------|-----------|--------|
| Redundant header | Use DashboardShell | Medium | High - cleaner, more consistent |
| Breadcrumb shows UUID | Add context-aware breadcrumbs | Medium | High - better navigation |
| Generic "Project Details" title | Rename to "Information" | Trivial | Medium - minor clarity gain |
| Mixed layout patterns | Standardize on DashboardShell | Medium | High - consistency across dashboard |
| Error state styling | Use Alert component | Trivial | Low - consistency |

---

## Files to Update

1. **`dashboard/src/app/kit/projects/[id]/page.tsx`** (Primary)
   - Adopt DashboardShell pattern
   - Remove custom header section
   - Rename Card title
   - Update error state to use Alert

2. **`dashboard/src/components/layouts/header.tsx`** (Optional)
   - Add context-aware breadcrumb support
   - Keep backward compatible

3. **`dashboard/src/components/layouts/dashboard-shell.tsx`** (Optional)
   - Already complete and functional

---

## Reference Pages

- **✓ Best Practice:** `/kit/resources/[id]` - Uses DashboardShell effectively
- **Partial:** `/kit/projects/[id]/settings` - Uses custom header (acceptable for settings)
- **List Page:** `/kit/projects` - Uses custom header (acceptable for lists)

---

## Next Steps

1. Frontend expert applies DashboardShell pattern
2. Review updated page layout for consistency
3. Verify breadcrumb navigation works as expected
4. Test responsive behavior (mobile, tablet, desktop)
5. Test loading and error states

---

## Design System Conventions Used

**From crewkit dashboard design system:**
- `DashboardShell` component for unified page layout
- `Card`, `CardHeader`, `CardTitle`, `CardDescription` for content sections
- `Button` variants: `default` (primary), `outline` (secondary), `ghost` (tertiary)
- Icons from lucide-react
- Spacing: `gap-6` between sections, `gap-4` between items
- Typography: `text-3xl font-bold tracking-tight` for page titles
- Colors: slate-600 for icons, slate-100 for backgrounds
- Responsive grid: `md:grid-cols-2`, `lg:grid-cols-3`, etc.

**All recommendations align with shadcn/ui patterns and existing dashboard implementations.**

---

## Questions for Frontend Expert

1. Should we add badges to project (status, visibility)?
2. Should the Quick Actions cards be clickable links or Cards with nested Links?
3. Should we add project stats (agent count, experiment count) to the Information card?
4. For breadcrumbs, prefer context-aware approach or rely on pathname?

