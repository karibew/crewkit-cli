---
doc_tier: 1
doc_type: spec
doc_status: active
created: 2025-11-01
last_reviewed: 2026-01-06
owner: platform-team
tags: [dashboard, agents, ui, mockup]
---

# Agent Configuration UI - Visual Layout Reference

## Edit Page Layout

### 3-Panel Layout (Organization-Level)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Breadcrumbs: Home > Organizations > Acme Corp > Agents > rails-expert │
├─────────────────────────────────────────────────────────────────────┤
│  Edit rails-expert                                                   │
│  Organization-level agent customization                             │
├─────────────────────────────────────────────────────────────────────┤
│  ℹ️  Explanation: How configuration inheritance works               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┬─────────────────┬─────────────────┐          │
│  │  Base Config    │  Org Override   │  Effective      │          │
│  │  [Read-only]    │  [Editable]     │  [Preview]      │          │
│  │  Gray bg        │  White bg       │  Blue bg        │          │
│  │                 │                 │                 │          │
│  │  # rails-expert │ <textarea>      │ [Merged view]   │          │
│  │                 │ Add your custom │ Updates as you  │          │
│  │  You are a Rails│ izations here.. │ type (500ms     │          │
│  │  expert agent...│                 │ debounce)       │          │
│  │                 │                 │                 │          │
│  │  [Scrollable]   │  [Scrollable]   │  [Scrollable]   │          │
│  │                 │                 │                 │          │
│  └─────────────────┴─────────────────┴─────────────────┘          │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Additional Settings:                                               │
│  Description: [_______________________________________________]     │
│  ☑ Active                                                           │
├─────────────────────────────────────────────────────────────────────┤
│  [Cancel]                              [Delete] [Save Changes]      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4-Panel Layout (Project-Level with Org Override)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Breadcrumbs: Home > Orgs > Acme > Projects > Portal > Agents > rails │
├─────────────────────────────────────────────────────────────────────┤
│  Edit rails-expert                                                   │
│  Project-level agent customization                                  │
├─────────────────────────────────────────────────────────────────────┤
│  ℹ️  Explanation: Your project override merges with base + org      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┬──────────┬──────────┬──────────┐                    │
│  │  Base    │  Org     │  Project │ Effective│                    │
│  │  Config  │ Override │ Override │ Config   │                    │
│  │ [Read]   │ [Read]   │ [Edit]   │ [Preview]│                    │
│  │ Gray bg  │ Gray bg  │ White bg │ Blue bg  │                    │
│  │          │          │          │          │                    │
│  │ # rails  │ ## Acme  │<textarea>│ [Merged] │                    │
│  │ expert   │ Standards│ Add proj │ view of  │                    │
│  │          │          │ specific │ all 3    │                    │
│  │ You are  │ Always   │ customiz │ configs  │                    │
│  │ a Rails  │ use Type │ ations.. │ combined │                    │
│  │ expert...│ Script...│          │          │                    │
│  │          │          │          │          │                    │
│  │ [Scroll] │ [Scroll] │ [Scroll] │ [Scroll] │                    │
│  │          │          │          │          │                    │
│  └──────────┴──────────┴──────────┴──────────┘                    │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Additional Settings:                                               │
│  Description: [_______________________________________________]     │
│  ☑ Active                                                           │
├─────────────────────────────────────────────────────────────────────┤
│  [Cancel]                              [Delete] [Save Changes]      │
└─────────────────────────────────────────────────────────────────────┘
```

## Index Page Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Breadcrumbs: Home > Organizations > Acme Corp > Agent Configs     │
├─────────────────────────────────────────────────────────────────────┤
│  Organization Agent Configurations                                  │
│  Set organization-wide agent defaults                              │
├─────────────────────────────────────────────────────────────────────┤
│  ℹ️  Agent Configuration Inheritance explanation                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Configured Agents                                                  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ ⚙️  Rails Expert                        Updated 2 hours ago  >│  │
│  │     Custom settings for Rails development                    │  │
│  ├─────────────────────────────────────────────────────────────┤  │
│  │ ⚙️  Frontend Expert                     Updated 1 day ago    >│  │
│  │     Tailwind and React configuration                         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Available Agents                                                   │
│  ┌──────────────┬──────────────┬──────────────┐                   │
│  │ ⚙️  API      │ ⚙️  Data     │ ⚙️  CLI       │                   │
│  │   Designer   │   Analyst    │   Designer    │                   │
│  │              │              │               │                   │
│  │ Base agent   │ Base agent   │ Base agent    │                   │
│  │              │              │               │                   │
│  │ [Configure]  │ [Configure]  │ [Configure]   │                   │
│  └──────────────┴──────────────┴──────────────┘                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Color Coding

### Panel Colors
- **Gray background** (`bg-gray-50`) = Read-only content
- **White background** (`bg-white`) = Editable content
- **Blue background** (`bg-blue-50`) = Live preview/effective config

### Status Badges
- **Gray badge** = Base only (no override)
- **Blue badge** = Override exists
- **Green badge** = Editable field
- **Yellow badge** = Read-only reference

## Responsive Behavior

### Desktop (>= 1024px)
- 3-panel: Side-by-side horizontal layout
- 4-panel: Four equal columns
- All panels visible simultaneously

### Mobile (< 1024px)
- Panels stack vertically
- Full width for each panel
- Scroll between panels
- Same content, different layout

## Interactive Behaviors

### Live Preview
1. User types in override textarea
2. Wait 500ms after last keystroke (debounce)
3. Fetch preview from API
4. Show loading spinner
5. Update preview panel
6. Handle errors gracefully

### Form Submission
1. Click "Save Changes"
2. Button text changes to "Saving..."
3. Button disabled during submission
4. Turbo submits form via AJAX
5. On success: Redirect or show success message
6. On error: Re-render form with inline errors

### Delete Action
1. Click "Delete" button
2. Show confirmation dialog
3. If confirmed: Send DELETE request
4. Redirect to index with success message

## Accessibility Features

- **ARIA labels**: All interactive elements labeled
- **ARIA live regions**: Preview updates announced
- **ARIA roles**: Alerts use `role="alert"`
- **Keyboard navigation**: Tab order is logical
- **Focus states**: Visible focus rings
- **Screen reader text**: `sr-only` class for context
- **Semantic HTML**: Proper heading hierarchy
- **High contrast**: 4.5:1 minimum ratio

## Icons Used (Heroicons)

- **Settings/Config**: Cog icon (gear)
- **Info**: Information circle
- **Delete**: Trash icon
- **Loading**: Spinning circle
- **Success**: Check circle
- **Error**: X circle

## Typography

- **Page title**: `text-3xl font-bold` (30px)
- **Section headings**: `text-xl font-semibold` (20px)
- **Panel headers**: `text-sm font-semibold` (14px)
- **Body text**: `text-sm` (14px)
- **Code/config**: `text-xs font-mono` (12px)
- **Helper text**: `text-xs text-gray-500` (12px)

## Spacing

- **Page padding**: `py-8 px-4 sm:px-6 lg:px-8`
- **Panel gap**: `gap-4` (16px)
- **Section margin**: `mb-6` (24px)
- **Field spacing**: `space-y-4` (16px)
- **Button padding**: `px-4 py-2` (16px/8px)

---

**This mockup represents the implemented UI. All layouts are responsive and accessible.**
