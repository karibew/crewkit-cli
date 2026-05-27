# Transcript Screen Simplification Plan

**Target**: `dashboard/src/components/features/sessions/transcript-viewer/` and related files
**Method**: Musk 5-Step Process
**Status**: Approved through Step 5. Ready to execute.

---

## Step 1: Requirements (Confirmed)

### Real requirements
1. View conversation turns (user prompts, assistant responses, system messages)
2. See tool calls paired with results (collapsible)
3. See tasks as consolidated collapsible cards (Option A — keep current UX)
4. Navigate via JumpToPrompt dropdown + tree sidebar (keep for big convos)
5. Paginate with infinite scroll
6. Server-side search (not client-side)
7. Export JSONL
8. Single view mode (conversation only)

### Eliminated
- Detailed view mode — merged into conversation mode
- Client-side search — replaced by server-side
- Progressive disclosure via separate API call — send full content

---

## Step 2: Delete [ ]

| What | Location | Lines | Risk |
|---|---|---|---|
| Detailed view mode + `DetailedItemRenderer` | `detailed-item-renderer.tsx` (59), toggle UI in `index.tsx` (~30), `showTools` state (~20) | ~109 | None |
| `TaskBoundaryItem` type + rendering | `types.ts` (5), `exchange-card.tsx:103-114` (12) | ~17 | None (dead code) |
| `detail: "full"` API param | `use-session-transcript.ts` (1) | ~1 | None (unused) |
| Client-side search | `pipeline.ts:501-550` (50), `loadAllPages` (~20), debounce (~10) | ~80 | Needs server-side replacement first |
| `extractToolNameFromContent()` | `pipeline.ts:51-60` (10) | ~10 | None (inline 1 fallback) |
| `useTranscriptPipeline` hook file | `use-transcript-pipeline.ts` (106) | ~106 | None (inline into index.tsx) |
| `useScrollToHighlight` hook file | `use-scroll-to-highlight.ts` (42) | ~42 | None (5-line utility) |
| `RoleBadge` component file | `role-badge.tsx` (23) | ~23 | None (inline 1-liner) |
| **Total** | | **~385** | |

---

## Step 3: Simplify [ ]

### 3a. Pipeline: 4 stages -> 1 pass (~550 -> ~200 lines)
- Single `buildExchanges()` function, one loop over turns
- Skip noise inline (replaces `preprocessTurns`)
- Pair tool calls by `tool_use_id` with simple Map (replaces 111-line grouper)
- Track current task by `subagent_session_id` changes (replaces 253-line consolidator)
- Emit exchanges on user prompts (replaces `buildExchanges`)
- Types reduced: 5 item kinds -> 3 (`tool_pair`, `task_group`, `single`)

### 3b. Turn rendering: 4 role variants -> 1 config-driven component (~464 -> ~150 lines)
- Single `TurnDisplay` with role config map
- `SubagentTranscriptTurn` (114 lines) becomes AssistantTurn + indigo CSS class
- Drop 90% duplication between role variants

### 3c. ExchangeCard: move tool counting to pipeline (~137 -> ~80 lines)
- Pipeline splits direct tools vs task tools
- Component just renders what it gets

### 3d. Tool rendering: drop ToolGroupDetailed (~287 -> ~100 lines)
- All tool pairs collapsed by default (same visual as tool groups)
- Use shadcn `Collapsible` directly instead of ai-elements wrappers
- Drop `ai-elements/tool.tsx` (175 lines) and `ai-elements/task.tsx` (88 lines) as intermediaries

### 3e. Tree sidebar: flatten (~319 -> ~120 lines)
- Delete `exchangesToTree` converter (76 lines), compute in render
- Merge `TreeNode` and `TaskNode` into one recursive component

### Summary

| Area | Before | After | Saved |
|---|---|---|---|
| Pipeline + types + hook | 728 | 230 | 498 |
| Turn rendering | 464 | 150 | 314 |
| ExchangeCard | 137 | 80 | 57 |
| Tool components + ai-elements | 550 | 130 | 420 |
| Tree sidebar | 319 | 120 | 199 |
| **Total** | **2,198** | **710** | **~1,488 (68%)** |

---

## Step 4: Optimize [ ]

1. **Client-side search -> server-side search**: Current search only works on loaded pages. Replace with API full-text search on `session_turns.content`.
2. **Progressive disclosure API calls**: Evaluate sending full content by default (~1KB/turn). Eliminates `useTurnDetail` hook + expand/fetch/cache cycle.

---

## Step 5: Automate [ ]

### Tests to add (behavior, not implementation)
1. Exchanges render with user prompt at top, assistant response below
2. Tool calls are paired correctly (call + result)
3. Tasks are grouped into collapsible cards
4. Infinite scroll loads next page when sentinel is visible
5. Search filters exchanges (once server-side is wired)
6. Tree sidebar reflects exchange structure

### Tests to NOT write
- Pipeline internals (will be a single function)
- Component rendering details (snapshot tests are brittle)
- CSS classes or layout

---

## Execution Order

1. [ ] **Delete** dead code (TaskBoundaryItem, detail mode, unused params)
2. [ ] **Rewrite pipeline** as single-pass `buildExchanges()`
3. [ ] **Unify turn rendering** into config-driven component
4. [ ] **Simplify tool rendering** — drop ai-elements wrappers, use Collapsible directly
5. [ ] **Flatten tree sidebar**
6. [ ] **Wire server-side search** (requires API endpoint)
7. [ ] **Add behavioral tests**
