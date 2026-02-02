---
doc_tier: 2
doc_type: reference
doc_status: active
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
related_docs:
  - CLAUDE.md
tags: [dashboard, patterns, frontend, shadcn]
---

# Design Patterns Reference Guide

**Purpose:** Standardized patterns used across crewkit dashboard

---

## Pattern 1: Detail Page Layout (Standard)

Use this pattern for all resource detail pages.

### Pattern Structure

```tsx
"use client";

import { use } from "react";
import Link from "next/link";
import { ArrowLeft, Settings } from "lucide-react";
import { Button } from "@/components/ui/button";
import { DashboardShell } from "@/components/layouts/dashboard-shell";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { Alert, AlertDescription, AlertTitle } from "@/components/ui/alert";

export default function ResourceDetailPage({ params }) {
  const { id } = use(params);
  const { data: resource, isLoading, error } = useResource(id);

  // Loading state
  if (isLoading) return <ResourceDetailSkeleton />;

  // Error state
  if (error || !resource) {
    return (
      <DashboardShell>
        <div className="space-y-6">
          <Button variant="ghost" size="sm" asChild className="-ml-2">
            <Link href="/kit/resources">
              <ArrowLeft className="mr-2 h-4 w-4" />
              Back to Resources
            </Link>
          </Button>
          <Alert variant="destructive">
            <AlertTitle>Resource Not Found</AlertTitle>
            <AlertDescription>
              The resource you're looking for doesn't exist or you don't have access.
            </AlertDescription>
          </Alert>
        </div>
      </DashboardShell>
    );
  }

  // Success state
  return (
    <DashboardShell
      title={
        <div className="flex items-center gap-3">
          <span>{resource.name}</span>
          <Badge variant="outline">{resource.type}</Badge>
        </div>
      }
      description={
        <div className="text-sm">
          <span>Created {formatDate(resource.created_at)}</span>
        </div>
      }
      actions={
        <Button variant="outline" asChild>
          <Link href={`/kit/resources/${id}/settings`}>
            <Settings className="mr-2 h-4 w-4" />
            Settings
          </Link>
        </Button>
      }
    >
      <div className="space-y-6">
        {/* Back button */}
        <Button variant="ghost" size="sm" asChild className="-ml-2">
          <Link href="/kit/resources">
            <ArrowLeft className="mr-2 h-4 w-4" />
            Back to Resources
          </Link>
        </Button>

        {/* Content cards */}
        <Card>
          <CardHeader>
            <CardTitle>Information</CardTitle>
          </CardHeader>
          <CardContent>
            {/* Grid content */}
          </CardContent>
        </Card>

        {/* Action cards */}
        <div className="grid gap-4 md:grid-cols-2">
          <Card className="hover:shadow-md transition-shadow cursor-pointer">
            <Link href={`/kit/resources/${id}/edit`} className="block">
              <CardHeader>
                <CardTitle className="text-lg">Edit</CardTitle>
                <CardDescription>Modify this resource</CardDescription>
              </CardHeader>
            </Link>
          </Card>
        </div>
      </div>
    </DashboardShell>
  );
}
```

### Key Points

1. **DashboardShell**: Unified layout component (title, description, actions)
2. **Three States**: Loading, Error, Success
3. **Back Button**: Via Link in both error and success states
4. **Title Area**: Icon (if needed) + name + badges
5. **Actions**: Primary action (Settings) via DashboardShell actions prop
6. **Content**: Wrapped in space-y-6 for consistent spacing
7. **Error Handling**: Uses Alert component from shadcn

### Spacing Reference

```
DashboardShell (title + actions in header)
  ↓ gap-6
Back button (size="sm", variant="ghost", -ml-2)
  ↓ gap-6
Card (Information)
  ↓ gap-6
Grid/other content
```

---

## Pattern 2: List Page Layout

Use for pages showing collections of items.

```tsx
export default function ResourcesPage() {
  const { data: resources, isLoading } = useResources();

  return (
    <div className="space-y-6">
      {/* Header section */}
      <div className="flex items-center justify-between">
        <div>
          <h1 className="text-3xl font-bold tracking-tight">Resources</h1>
          <p className="text-muted-foreground">
            Manage all your resources
          </p>
        </div>
        <Button asChild>
          <Link href="/kit/resources/new">
            <Plus className="mr-2 h-4 w-4" />
            New Resource
          </Link>
        </Button>
      </div>

      {/* Content grid */}
      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {resources?.map((resource) => (
          <ResourceCard key={resource.id} resource={resource} />
        ))}
      </div>
    </div>
  );
}
```

### Key Points

1. **Custom header** (not DashboardShell) - acceptable for simple layouts
2. **Title + subtitle** pattern
3. **Primary action** on the right (button, not link)
4. **Grid layout** for cards with responsive breakpoints
5. **Consistent gap-6** between sections

---

## Pattern 3: Settings/Form Page Layout

Use for pages with forms and configuration.

```tsx
export default function ResourceSettingsPage({ params }) {
  const { id } = use(params);
  const { data: resource } = useResource(id);

  return (
    <div className="space-y-6">
      {/* Header */}
      <div className="flex items-center gap-4">
        <Button variant="ghost" size="icon" asChild>
          <Link href={`/kit/resources/${id}`}>
            <ArrowLeft className="h-4 w-4" />
          </Link>
        </Button>
        <div>
          <h1 className="text-3xl font-bold tracking-tight">
            Settings
          </h1>
          <p className="text-muted-foreground">
            Manage {resource?.name}
          </p>
        </div>
      </div>

      {/* Form content */}
      <Card>
        <CardHeader>
          <CardTitle>Basic Information</CardTitle>
        </CardHeader>
        <CardContent>
          {/* Form fields */}
        </CardContent>
      </Card>

      {/* Danger zone */}
      <Card className="border-red-200">
        <CardHeader>
          <CardTitle className="text-red-600">Danger Zone</CardTitle>
        </CardHeader>
        <CardContent>
          {/* Destructive actions */}
        </CardContent>
      </Card>
    </div>
  );
}
```

### Key Points

1. **Back button** with icon next to title
2. **Forms in Cards** with clear sections
3. **Danger Zone** with red border for destructive actions
4. **Gap-6** between main sections
5. **Separator** component between distinct sections (optional)

---

## Pattern 4: Grid-based Information Cards

Used in detail pages to display related information.

### 2-Column Layout (Responsive)

```tsx
<Card>
  <CardHeader>
    <CardTitle>Information</CardTitle>
    <CardDescription>Key details</CardDescription>
  </CardHeader>
  <CardContent className="space-y-4">
    <div className="grid gap-4 sm:grid-cols-2">
      <div className="space-y-1">
        <p className="text-sm font-medium text-muted-foreground flex items-center gap-2">
          <Icon className="h-4 w-4" />
          Label
        </p>
        <p className="text-sm">Value</p>
      </div>
      <div className="space-y-1">
        <p className="text-sm font-medium text-muted-foreground">
          Another Label
        </p>
        <p className="text-sm">Another Value</p>
      </div>
    </div>
  </CardContent>
</Card>
```

### 3-Column Layout

```tsx
<div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
  {/* Three columns on desktop, 2 on tablet, 1 on mobile */}
</div>
```

### Key Points

1. **Container gap**: `gap-4` between fields
2. **Field spacing**: `space-y-1` (label above value)
3. **Label styling**: `text-sm font-medium text-muted-foreground`
4. **Value styling**: `text-sm` (default), `font-mono` for technical values
5. **Icons**: 4x4 in label (h-4 w-4)
6. **Responsive**: `sm:grid-cols-2 lg:grid-cols-3`

---

## Pattern 5: Action Card (Clickable Navigation)

Used for contextual action cards that link to related sections.

```tsx
<Card className="hover:shadow-md transition-shadow cursor-pointer">
  <Link href={`/kit/agents?project=${id}`} className="block">
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
```

### Key Points

1. **Link wraps Card** (className="block" to make entire card clickable)
2. **Hover effect**: `hover:shadow-md transition-shadow`
3. **Cursor pointer**: Makes interaction intent clear
4. **Icon + text** in title (h-5 w-5 for prominent icons)
5. **Description** explains the action
6. **Grid layout**: Usually `md:grid-cols-2` for 2 cards

---

## Pattern 6: Breadcrumb Navigation

Current implementation shows breadcrumbs in the Header component.

### Current Behavior

```tsx
// Breadcrumbs generated from URL path
/kit/projects → "Projects"
/kit/projects/123 → "Projects" > "Id" ❌ (should be project name)
/kit/projects/123/settings → "Projects" > "Id" > "Settings"
```

### Recommended Enhancement

Add context-aware breadcrumbs for detail pages:

```tsx
export default function ProjectDetailPage({ params }) {
  const { id } = use(params);
  const { data: project } = useProject(id);

  // Pass custom breadcrumbs to Header (future enhancement)
  return (
    <>
      {/* Header will use these breadcrumbs instead of URL parsing */}
      {/* Showing: "Projects" > "My Cool Project" */}

      <div className="space-y-6">
        {/* Page content */}
      </div>
    </>
  );
}
```

### Implementation Note

Header component would need modification to accept optional breadcrumbs prop while maintaining backward compatibility.

---

## Pattern 7: Loading States (Skeleton)

Use dedicated skeleton components for loading states.

```tsx
function ResourceDetailSkeleton() {
  return (
    <div className="space-y-6">
      {/* Back button skeleton */}
      <Skeleton className="h-10 w-32" />

      {/* Title area skeleton */}
      <div className="space-y-2">
        <Skeleton className="h-10 w-64" />
        <Skeleton className="h-5 w-48" />
      </div>

      {/* Content card skeleton */}
      <Card>
        <CardHeader>
          <Skeleton className="h-6 w-32" />
          <Skeleton className="h-4 w-48" />
        </CardHeader>
        <CardContent>
          <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
            {[...Array(3)].map((_, i) => (
              <div key={i} className="space-y-2">
                <Skeleton className="h-4 w-20" />
                <Skeleton className="h-4 w-32" />
              </div>
            ))}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

### Key Points

1. **Match structure** of actual content
2. **Skeleton className** sizes should match text heights
3. **Use space-y-* classes** to match gaps
4. **Grid layout** matches actual grid
5. **Cards should have skeleton for header + content**

---

## Color & Styling Reference

### Text Colors
- Headings: `text-slate-900` (default text color)
- Secondary: `text-muted-foreground` (gray)
- Links: `text-blue-600 hover:text-blue-700`
- Danger: `text-red-600` (for warnings)

### Background Colors
- Icon background: `bg-slate-100` (light gray)
- Hover state: `hover:shadow-md` (shadow)
- Cards: `bg-white` (default)
- Page background: `bg-slate-50` (managed by layout)

### Borders
- Danger zone: `border-red-200` (light red)
- Normal: `border` (gray-200, default)
- Hover: `transition-shadow` (prefer shadow over border change)

### Spacing
- Between page sections: `gap-6` (24px)
- Between items in grid: `gap-4` (16px)
- Within card content: `gap-4` or `space-y-4`
- Between label and value: `space-y-1` (4px)

---

## Button Variants Reference

### Primary (Default)
- Usage: Main call-to-action
- Classes: `<Button>` (blue background)
- Example: "Create Project", "Save"

### Secondary/Outline
- Usage: Less important actions
- Classes: `<Button variant="outline">`
- Example: "Settings", "Edit", "Cancel"

### Ghost
- Usage: Tertiary, low emphasis
- Classes: `<Button variant="ghost">`
- Example: "Back", "More Options"

### Ghost Icon
- Usage: Icon-only actions
- Classes: `<Button variant="ghost" size="icon">`
- Example: Back arrow, close button

### Danger
- Usage: Destructive actions
- Classes: `<Button variant="outline" className="text-destructive hover:text-destructive">`
- Example: "Delete"

---

## Component Structure Checklist

When creating a new detail page:

- [ ] Use `DashboardShell` for title/actions area
- [ ] Include back button (ghost variant, -ml-2)
- [ ] Use `Card` for content sections
- [ ] Name card title descriptively ("Information", not "Details")
- [ ] Include `CardDescription` when helpful
- [ ] Use grid layout for information fields
- [ ] Match responsive breakpoints: `sm:grid-cols-2 lg:grid-cols-3`
- [ ] Use `Alert` for error states
- [ ] Create dedicated skeleton component for loading
- [ ] Use space-y-6 between main sections
- [ ] Use gap-4 between items
- [ ] Include icons from lucide-react
- [ ] Match typography: `text-3xl font-bold tracking-tight` for page title
- [ ] Use `text-muted-foreground` for secondary text

---

## Common Mistakes to Avoid

❌ **Mixing layout patterns** (custom header + DashboardShell on same page)
❌ **Generic card titles** ("Details", "Information" without context)
❌ **Inconsistent spacing** (gap-6 in one section, gap-8 in another)
❌ **Icon sizing** (using h-6 w-6 in grid fields instead of h-4 w-4)
❌ **Redundant headers** (title in both custom header and card title)
❌ **Not using responsive grid** (fixed columns instead of sm:/lg: breakpoints)
❌ **Breadcrumb UUIDs** (showing ID segment instead of human-readable name)
❌ **Error states without Alert** (using divs instead of Alert component)
❌ **Back button styling** (using outline when ghost is correct)

---

## Files to Reference

- **Detail page example:** `/src/app/kit/resources/[id]/page.tsx` ✓
- **Settings page example:** `/src/app/kit/projects/[id]/settings/page.tsx`
- **List page example:** `/src/app/kit/projects/page.tsx`
- **DashboardShell:** `/src/components/layouts/dashboard-shell.tsx`
- **Header component:** `/src/components/layouts/header.tsx`
- **Button styles:** `/src/components/ui/button.tsx`
- **Card styles:** `/src/components/ui/card.tsx`

---

## Updating Existing Pages

To update `/kit/projects/[id]` to follow these patterns:

1. **Replace custom header** with DashboardShell
2. **Remove "Project Details" Card title** → rename to "Information"
3. **Move Settings button** to DashboardShell actions
4. **Keep back button** in page content (below DashboardShell)
5. **Update error state** to use Alert component
6. **Verify spacing** (gap-6 between sections)

See `DESIGN_REVIEW_PROJECT_DETAIL.md` for detailed refactoring guide.

