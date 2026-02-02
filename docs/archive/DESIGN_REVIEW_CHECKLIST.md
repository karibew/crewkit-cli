---
doc_tier: 1
doc_type: plan
doc_status: archived
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
related_docs:
  - docs/archive/DESIGN_REVIEW_PROJECT_DETAIL.md
tags: [dashboard, checklist, frontend]
---

# Design Review Checklist: Project Details Page

**Status:** Archived (implementation checklist)
**Page:** `/kit/projects/[id]/page.tsx`
**Reviewer:** Design System (crewkit-design-expert agent)
**Date:** 2026-01-03

---

## Pre-Implementation Review

### Current Issues Found

- [x] Breadcrumb shows UUID instead of project name
- [x] Redundant header information (custom header + "Project Details" card)
- [x] Inconsistent with Resources detail page pattern
- [x] No error state using Alert component

### Design System Inconsistencies

- [x] Not using DashboardShell for unified layout
- [x] Card title doesn't clarify content purpose ("Project Details" too generic)
- [x] Custom header instead of standardized component pattern
- [x] Missing consistent back navigation pattern

---

## Implementation Checklist

### Phase 1: Layout Refactoring

- [ ] Import DashboardShell component
- [ ] Move project name + slug to DashboardShell title prop
- [ ] Move Settings button to DashboardShell actions prop
- [ ] Remove custom header div (lines 65-91)
- [ ] Update title format to include icon:
  ```tsx
  title={
    <div className="flex items-center gap-3">
      <FolderGit2 className="h-6 w-6 text-slate-600" />
      <div>
        <h1 className="text-3xl font-bold tracking-tight">{project.name}</h1>
        <p className="text-muted-foreground font-mono text-sm">{project.slug}</p>
      </div>
    </div>
  }
  ```

### Phase 2: Content Cards

- [ ] Rename "Project Details" card title to "Information"
- [ ] Update CardDescription: "Key details about this project"
- [ ] Verify grid layout uses responsive classes: `sm:grid-cols-2 lg:grid-cols-3`
- [ ] Check field spacing: `space-y-1` for each field group
- [ ] Verify icon sizing in labels (h-4 w-4)
- [ ] Keep quick actions grid unchanged (already correct pattern)

### Phase 3: Navigation

- [ ] Add back button below DashboardShell:
  ```tsx
  <Button variant="ghost" size="sm" asChild className="-ml-2">
    <Link href="/kit/projects">
      <ArrowLeft className="mr-2 h-4 w-4" />
      Back to Projects
    </Link>
  </Button>
  ```
- [ ] Verify back button styling (ghost variant, -ml-2 class)
- [ ] Remove any conflicting navigation elements

### Phase 4: Error Handling

- [ ] Update error state to use Alert component:
  ```tsx
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
              The project you're looking for doesn't exist or you
              don't have access.
            </AlertDescription>
          </Alert>
        </div>
      </DashboardShell>
    );
  }
  ```
- [ ] Import Alert and AlertCircle components
- [ ] Verify error message is user-friendly

### Phase 5: Spacing & Layout

- [ ] Verify gap-6 between major sections (DashboardShell to back button to cards)
- [ ] Verify gap-4 between grid items
- [ ] Check space-y-1 within field groups
- [ ] Verify md:grid-cols-2 on quick actions cards
- [ ] No custom gap overrides that conflict with design system

### Phase 6: Loading State

- [ ] Update ProjectDetailLoading skeleton to match new structure
- [ ] Include skeleton for DashboardShell title area
- [ ] Include skeleton for back button
- [ ] Verify skeleton dimensions match content heights

---

## Component Imports Needed

- [x] DashboardShell - `@/components/layouts/dashboard-shell`
- [x] Alert, AlertCircle, AlertTitle, AlertDescription - `@/components/ui/alert`
- [ ] Verify all lucide-react icons are imported:
  - ArrowLeft ✓
  - FolderGit2 ✓
  - Calendar ✓
  - GitBranch ✓
  - Settings ✓
  - Bot ✓
  - FlaskConical ✓

---

## Code Quality Checks

### Before Submission

- [ ] No console errors or warnings
- [ ] All imports are used
- [ ] No unused variables
- [ ] TypeScript types are correct
- [ ] No duplicate code
- [ ] Line length reasonable (< 100 chars preferred)
- [ ] Comments updated if component behavior changed

### Linting & Formatting

- [ ] Run prettier: `npm run format`
- [ ] Run eslint: `npm run lint`
- [ ] No TypeScript errors: `npm run type-check`

---

## Testing Checklist

### Functional Testing

- [ ] Page loads successfully with project data
- [ ] Project name displays in title (not UUID)
- [ ] Project slug displays in subtitle
- [ ] Settings button navigates to `/kit/projects/[id]/settings`
- [ ] Back button navigates to `/kit/projects`
- [ ] "Agents" card links to `/kit/agents?project=[id]`
- [ ] "Experiments" card links to `/kit/experiments?project=[id]`
- [ ] All project information fields display correctly
- [ ] Hover effect on quick actions cards (shadow)

### Error Handling

- [ ] Invalid project ID shows error state
- [ ] Error message is user-friendly
- [ ] Back button works from error state
- [ ] No console errors on error page

### Loading States

- [ ] Loading skeleton displays initially
- [ ] Skeleton structure matches content
- [ ] Smooth transition from skeleton to content

### Responsive Design

- [ ] Mobile (375px): Stack layout works
- [ ] Tablet (768px): Grid uses 2 columns for actions
- [ ] Desktop (1440px): Grid uses responsive spacing
- [ ] No text overflow or truncation issues
- [ ] Icons scale appropriately
- [ ] Padding/margins consistent across breakpoints

### Accessibility

- [ ] Back button has sr-only text or aria-label
- [ ] Links have proper href attributes
- [ ] Buttons have descriptive text (or sr-only for icons)
- [ ] Color contrast meets WCAG AA (checked via audit)
- [ ] Keyboard navigation works (Tab through elements)
- [ ] No missing alt text for icons (icons should have sr-only label if icon-only)

### Browser Compatibility

- [ ] Chrome/Edge latest
- [ ] Firefox latest
- [ ] Safari latest
- [ ] Mobile Safari (iOS)
- [ ] Chrome Mobile

---

## Design System Compliance

### Spacing ✓ (No changes needed)
- [ ] gap-6 between major sections
- [ ] gap-4 between grid items
- [ ] space-y-1 within field groups
- [ ] -ml-2 on back button (correct negative margin)

### Typography ✓ (No changes needed)
- [ ] text-3xl font-bold tracking-tight for main title
- [ ] text-muted-foreground for secondary text
- [ ] text-sm for labels and values
- [ ] font-mono for slugs and IDs

### Colors ✓ (No changes needed)
- [ ] bg-slate-100 for icon background
- [ ] text-slate-600 for icon color
- [ ] text-muted-foreground for secondary content
- [ ] No color overrides or custom colors

### Components
- [ ] DashboardShell used correctly
- [ ] Card/CardHeader/CardTitle/CardDescription structure
- [ ] Button variants match usage (outline for Settings, ghost for back)
- [ ] Alert component for error states
- [ ] Lucide icons sized correctly (h-6 w-6 for title, h-4 w-4 for fields)

### Patterns
- [ ] Follows Resources detail page pattern
- [ ] Consistent with other detail pages
- [ ] Back button pattern matches design system
- [ ] Error state pattern matches design system
- [ ] Loading state pattern matches design system

---

## Visual Audit Criteria

When page is complete, verify visually:

### Header Area
- [ ] Project icon displays (FolderGit2)
- [ ] Project name is visible and readable
- [ ] Project slug is below name in smaller, mono font
- [ ] Settings button is right-aligned and visible
- [ ] Overall header looks similar to Resources detail page

### Content Area
- [ ] Back button is visible and properly styled
- [ ] "Information" card has correct title (not "Project Details")
- [ ] Grid items are properly aligned
- [ ] Created date, Git remote, and Project ID display
- [ ] No missing or truncated content

### Quick Actions
- [ ] Two cards side-by-side on desktop
- [ ] Each card shows icon + title + description
- [ ] "Agents" card on left, "Experiments" on right
- [ ] Cards have hover shadow effect
- [ ] Cards are clickable

### Overall
- [ ] Page matches Resources detail page visual style
- [ ] All text is readable
- [ ] Layout is not cluttered
- [ ] Spacing feels consistent
- [ ] No redundant information

---

## Comparison: Before vs After

### Before
```
Header (custom divs)
├── Back button + icon + name + slug
└── Settings button (right-aligned)

"Project Details" Card (redundant title)
├── Created date
├── Git remote
└── Project ID

Quick Actions (2 cards)
├── Agents
└── Experiments
```

### After
```
DashboardShell Header
├── Title (icon + name + slug)
└── Actions (Settings button)

Back button

"Information" Card
├── Created date
├── Git remote
└── Project ID

Quick Actions (2 cards)
├── Agents
└── Experiments
```

**Key Differences:**
- DashboardShell provides unified header layout
- Removes "Project Details" redundancy
- Cleaner, more consistent structure
- Matches Resources page pattern

---

## Edge Cases to Test

- [ ] Project with no git_remote (should not show empty field)
- [ ] Very long project name (should wrap or truncate gracefully)
- [ ] Very long slug (should truncate with ellipsis if needed)
- [ ] Very old created date (format should be readable)
- [ ] Rapid navigation (loading states should handle edge cases)
- [ ] Network error (error state should handle gracefully)

---

## Performance Considerations

- [ ] No unnecessary re-renders
- [ ] useProject hook is optimized
- [ ] No DOM bloat from custom styles
- [ ] Image/icon loading is fast
- [ ] Page transitions smoothly

---

## Accessibility Audit

Required checks:

- [ ] **Keyboard Navigation:** Can I tab through all interactive elements?
- [ ] **Focus Management:** Is focus visible (blue ring)?
- [ ] **Color Contrast:** Text meets WCAG AA (4.5:1 for normal text)
- [ ] **Screen Reader:** All interactive elements are announced correctly
- [ ] **Alt Text:** Icons have aria-label or sr-only text if needed
- [ ] **ARIA Labels:** Buttons with just icons have descriptive labels
- [ ] **Semantic HTML:** Using <button>, <a>, <nav> correctly

**Note:** Back button with icon should have:
```tsx
<Button variant="ghost" size="icon" asChild className="-ml-2">
  <Link href="/kit/projects">
    <ArrowLeft className="h-4 w-4" />
    <span className="sr-only">Back to projects</span>
  </Link>
</Button>
```

---

## Sign-off Checklist

### Ready for Testing
- [ ] All implementation items complete
- [ ] Code passes linting
- [ ] No TypeScript errors
- [ ] Visual appearance matches reference (Resources page)
- [ ] Navigation works correctly

### Ready for Merge
- [ ] All tests pass
- [ ] Design review complete
- [ ] No console errors
- [ ] Accessibility verified
- [ ] Responsive design tested
- [ ] Works on all browsers
- [ ] Performance acceptable

### Post-Merge
- [ ] Monitor error logs for regressions
- [ ] Verify in production
- [ ] Breadcrumb fix addresses UUID issue (next phase)

---

## Additional Notes

### Breadcrumb Fix (Phase 2)

The breadcrumb showing UUID instead of project name will be addressed separately. This requires modifying the `Header` component to accept context-aware breadcrumbs. Current implementation shows path-based breadcrumbs which work for list pages but not for dynamic detail pages.

For now, the header breadcrumb will show "Id" (from the URL segment). This will be improved in a follow-up enhancement.

### Future Improvements

1. Add context-aware breadcrumbs to Header component
2. Consider adding project stats (agent count, experiment count)
3. Add status badge if projects have status values
4. Consider adding quick-copy buttons for project ID

---

## Reference Documents

- See `DESIGN_REVIEW_PROJECT_DETAIL.md` for full analysis
- See `DESIGN_PATTERNS_REFERENCE.md` for pattern guidelines
- Reference page: `/src/app/kit/resources/[id]/page.tsx`

---

## Questions for Reviewer

1. Should we add badges (status, visibility) to the project title?
2. Should we add project stats to the Information card?
3. Should the Settings button appear in both header and as a card?
4. Should git_remote field show a copy-to-clipboard button?

---

**Implementation estimated time:** 30-45 minutes
**Testing estimated time:** 15-20 minutes
**Total estimated time:** 45-65 minutes

