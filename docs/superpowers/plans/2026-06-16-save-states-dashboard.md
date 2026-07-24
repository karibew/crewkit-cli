# Save States — Dashboard Implementation Plan (Plan 4 of 4)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.

**Goal:** A "Save States" panel on the project page: view my personal save state and the team daily + weekend digests; managers can edit or delete a team digest.

**Architecture:** Regenerate API types from the Plan-1 OpenAPI, add a `save-states` API module + React Query hooks (mirroring `artifacts`/`inbound-messages`), and a `SaveStatesPanel` feature component rendered on the project page. Manager-only edit/delete gated via the current user's org `permission_level`.

**Tech Stack:** Next.js 16, React Query, shadcn/ui (Card, Sheet, AlertDialog, Dialog, Textarea, Badge, Skeleton), MSW + Vitest + RTL.

**Depends on:** Plan 1 (endpoints + OpenAPI live). Read-only for personal; edit/delete for team digests (which exist once Plan 2 runs — the panel handles the empty case).

> **Contract alignment:** use the REAL Plan-1 shape — `kind` (`personal`|`team_daily`|`team_weekend`), `handoff`, `saved_by{id,email,name}`, org-scoped routes with a `project_id` query param, `PATCH /:org/save_states/team/:kind`, `DELETE /:org/save_states/:id`. (There is no `state_type`/`name`/`data` field.)

## File structure

- Modify: `dashboard/src/types/api-generated.d.ts` — regenerated from OpenAPI.
- Create: `dashboard/src/lib/api/save-states.ts` — client + types.
- Create: `dashboard/src/hooks/use-save-states.ts` — query + mutation hooks.
- Create: `dashboard/src/components/features/projects/save-states-panel.tsx` — the panel.
- Modify: `dashboard/src/app/kit/projects/[id]/page.tsx` — render the panel.
- Modify: `dashboard/src/test/mocks/data.ts` — `mockSaveState` factory.
- Modify: `dashboard/src/test/mocks/handlers.ts` — `saveStateHandlers`.
- Create: `dashboard/src/components/features/projects/__tests__/save-states-panel.test.tsx`

> Run from `dashboard/`. Pre-merge: `npm run lint && npm run typecheck && npx vitest run && npm run validate:api-contract`.

---

### Task 1: Regenerate API types + contract

**Files:** Modify `dashboard/src/types/api-generated.d.ts`.

- [ ] **Step 1: Regenerate** (Plan 1 added the `save_states` paths to `api/swagger/v1/openapi.yaml`):

Run: `cd dashboard && npm run generate:api-types`
Expected: `src/types/api-generated.d.ts` now contains the `/{organization_id}/save_states*` paths.

- [ ] **Step 2: Validate contract**

Run: `npm run validate:api-contract`
Expected: exits 0 (generated types match the spec).

- [ ] **Step 3: Commit**

```bash
git add dashboard/src/types/api-generated.d.ts
git commit -m "chore(dashboard): regenerate api types for save_states"
```

---

### Task 2: API client module

**Files:** Create `dashboard/src/lib/api/save-states.ts`. Test: via hooks (Task 3) + MSW.

- [ ] **Step 1: Implement** (mirror `lib/api/artifacts.ts` — `apiClient`, `buildQueryParams`, `data` unwrap):

```typescript
// dashboard/src/lib/api/save-states.ts
import { apiClient } from "./client";

export type SaveStateKind = "personal" | "team_daily" | "team_weekend";

export interface SaveStateActor { id: string; email: string; name: string }

export interface SaveState {
  id: string;
  kind: SaveStateKind;
  title: string | null;
  handoff: string | null;
  digest_date: string | null;
  period_start: string | null;
  saved_by: SaveStateActor | null;
  reviewed_by: SaveStateActor | null;
  reviewed_at: string | null;
  published_at: string | null;
  source_session_id: string | null;
  created_at: string;
  updated_at: string;
}

export interface SaveStateListResponse { data: SaveState[] }

export const saveStatesApi = {
  list: (orgId: string, projectId: string): Promise<SaveStateListResponse> =>
    apiClient<SaveStateListResponse>(
      `/api/v1/${orgId}/save_states?project_id=${encodeURIComponent(projectId)}`,
      { method: "GET" },
    ),

  updateTeamDigest: async (
    orgId: string, kind: "team_daily" | "team_weekend", projectId: string, handoff: string,
  ): Promise<SaveState> => {
    const res = await apiClient<{ data: SaveState }>(
      `/api/v1/${orgId}/save_states/team/${kind}?project_id=${encodeURIComponent(projectId)}`,
      { method: "PATCH", body: JSON.stringify({ handoff }) },
    );
    return res.data;
  },

  remove: (orgId: string, id: string): Promise<void> =>
    apiClient<void>(`/api/v1/${orgId}/save_states/${id}`, { method: "DELETE" }),
};
```

- [ ] **Step 2: Typecheck.** `cd dashboard && npm run typecheck` → no errors.
- [ ] **Step 3: Commit** `git commit -am "feat(dashboard): save-states api client module"`

---

### Task 3: React Query hooks

**Files:** Create `dashboard/src/hooks/use-save-states.ts`. Test: via the panel test (Task 6).

- [ ] **Step 1: Implement** (mirror `hooks/use-artifacts.ts` keys + query + mutation w/ invalidation + toast):

```typescript
// dashboard/src/hooks/use-save-states.ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";
import { useOrganization } from "@/lib/auth/organization-context";
import { saveStatesApi, type SaveState } from "@/lib/api/save-states";

export const saveStateKeys = {
  all: ["save-states"] as const,
  list: (orgId: string, projectId: string) =>
    [...saveStateKeys.all, "list", orgId, projectId] as const,
};

export function useSaveStates(projectId: string) {
  const { currentOrganization } = useOrganization();
  const orgId = currentOrganization?.id || "";
  return useQuery({
    queryKey: saveStateKeys.list(orgId, projectId),
    queryFn: () => saveStatesApi.list(orgId, projectId),
    enabled: !!orgId && !!projectId,
  });
}

export function useUpdateTeamDigest(projectId: string) {
  const { currentOrganization } = useOrganization();
  const orgId = currentOrganization?.id || "";
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ kind, handoff }: { kind: "team_daily" | "team_weekend"; handoff: string }) =>
      saveStatesApi.updateTeamDigest(orgId, kind, projectId, handoff),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: saveStateKeys.list(orgId, projectId) });
      toast.success("Digest updated");
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : "Failed to update digest"),
  });
}

export function useDeleteSaveState(projectId: string) {
  const { currentOrganization } = useOrganization();
  const orgId = currentOrganization?.id || "";
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => saveStatesApi.remove(orgId, id),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: saveStateKeys.list(orgId, projectId) });
      toast.success("Save state deleted");
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : "Failed to delete"),
  });
}

export function partitionSaveStates(states: SaveState[] = []) {
  return {
    personal: states.find((s) => s.kind === "personal") || null,
    teamDaily: states.find((s) => s.kind === "team_daily") || null,
    teamWeekend: states.find((s) => s.kind === "team_weekend") || null,
  };
}
```

- [ ] **Step 2: Typecheck → clean. Commit** `git commit -am "feat(dashboard): save-states query + mutation hooks"`

---

### Task 4: `SaveStatesPanel` component

**Files:** Create `dashboard/src/components/features/projects/save-states-panel.tsx`. Test: Task 6.

Manager gate: derive `isManager` from the current user's org membership (mirror the `useMembers` + `permission_level` pattern; reuse the existing role hook if the project page already has one — check `page.tsx` for an existing `useMembers`/role usage before adding a new fetch).

- [ ] **Step 1: Implement** (Card + list of the three layers; click → Sheet with full handoff; manager-only Edit dialog for team digests + Delete AlertDialog — mirror `documents/page.tsx` Sheet/AlertDialog/Dialog usage):

```tsx
// dashboard/src/components/features/projects/save-states-panel.tsx
"use client";
import { useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { Sheet, SheetContent, SheetHeader, SheetTitle } from "@/components/ui/sheet";
import { AlertDialog, AlertDialogAction, AlertDialogCancel, AlertDialogContent,
  AlertDialogDescription, AlertDialogFooter, AlertDialogHeader, AlertDialogTitle,
  AlertDialogTrigger } from "@/components/ui/alert-dialog";
import { Dialog, DialogContent, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { Textarea } from "@/components/ui/textarea";
import { useSaveStates, useUpdateTeamDigest, useDeleteSaveState, partitionSaveStates }
  from "@/hooks/use-save-states";
import type { SaveState } from "@/lib/api/save-states";

const LABELS: Record<SaveState["kind"], string> = {
  personal: "My save state",
  team_daily: "Team daily digest",
  team_weekend: "Team weekend digest",
};

export function SaveStatesPanel({ projectId, isManager = false }: { projectId: string; isManager?: boolean }) {
  const { data, isLoading } = useSaveStates(projectId);
  const del = useDeleteSaveState(projectId);
  const edit = useUpdateTeamDigest(projectId);
  const [viewing, setViewing] = useState<SaveState | null>(null);
  const [editing, setEditing] = useState<SaveState | null>(null);
  const [draft, setDraft] = useState("");

  if (isLoading) return <Skeleton className="h-40 w-full" />;
  const { personal, teamDaily, teamWeekend } = partitionSaveStates(data?.data);
  const rows = [personal, teamDaily, teamWeekend].filter(Boolean) as SaveState[];

  return (
    <Card>
      <CardHeader><CardTitle>Save states</CardTitle></CardHeader>
      <CardContent className="space-y-2">
        {rows.length === 0 && (
          <p className="text-sm text-muted-foreground">
            No save states yet. They appear once you save a session or the daily digest runs.
          </p>
        )}
        {rows.map((s) => {
          const isTeam = s.kind !== "personal";
          return (
            <div key={s.id} className="flex items-center justify-between rounded border p-3">
              <button className="text-left" onClick={() => setViewing(s)}>
                <div className="font-medium">{LABELS[s.kind]}</div>
                <div className="text-xs text-muted-foreground">
                  {s.saved_by?.name ?? "crewkit"} · {s.digest_date ?? s.updated_at?.slice(0, 10)}
                </div>
              </button>
              <div className="flex items-center gap-2">
                <Badge variant="secondary">{s.kind}</Badge>
                {isTeam && !s.published_at && <Badge variant="outline">Draft</Badge>}
                {isManager && isTeam && !s.published_at && (
                  <Button size="sm" variant="ghost"
                    onClick={() => edit.mutate({ kind: s.kind as "team_daily" | "team_weekend", handoff: s.handoff ?? "" })}>
                    Publish
                  </Button>
                )}
                {isManager && isTeam && (
                  <Button size="sm" variant="ghost"
                    onClick={() => { setEditing(s); setDraft(s.handoff ?? ""); }}>Edit</Button>
                )}
                {(isManager || s.kind === "personal") && (
                  <AlertDialog>
                    <AlertDialogTrigger asChild>
                      <Button size="sm" variant="ghost">Delete</Button>
                    </AlertDialogTrigger>
                    <AlertDialogContent>
                      <AlertDialogHeader>
                        <AlertDialogTitle>Delete {LABELS[s.kind]}?</AlertDialogTitle>
                        <AlertDialogDescription>This cannot be undone.</AlertDialogDescription>
                      </AlertDialogHeader>
                      <AlertDialogFooter>
                        <AlertDialogCancel>Cancel</AlertDialogCancel>
                        <AlertDialogAction onClick={() => del.mutate(s.id)}>Delete</AlertDialogAction>
                      </AlertDialogFooter>
                    </AlertDialogContent>
                  </AlertDialog>
                )}
              </div>
            </div>
          );
        })}
      </CardContent>

      <Sheet open={!!viewing} onOpenChange={(o) => !o && setViewing(null)}>
        <SheetContent>
          <SheetHeader><SheetTitle>{viewing && LABELS[viewing.kind]}</SheetTitle></SheetHeader>
          <pre className="mt-4 whitespace-pre-wrap text-sm">{viewing?.handoff}</pre>
        </SheetContent>
      </Sheet>

      <Dialog open={!!editing} onOpenChange={(o) => !o && setEditing(null)}>
        <DialogContent>
          <DialogHeader><DialogTitle>Edit {editing && LABELS[editing.kind]}</DialogTitle></DialogHeader>
          <Textarea value={draft} onChange={(e) => setDraft(e.target.value)} rows={12} />
          <DialogFooter>
            <Button variant="outline" onClick={() => setEditing(null)}>Cancel</Button>
            <Button onClick={() => {
              if (editing && editing.kind !== "personal") {
                edit.mutate({ kind: editing.kind, handoff: draft },
                  { onSuccess: () => setEditing(null) });
              }
            }}>Save</Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>
    </Card>
  );
}
```

- [ ] **Step 2: Typecheck → clean. Commit** `git commit -am "feat(dashboard): SaveStatesPanel component"`

---

### Task 5: Render on the project page + role gating

**Files:** Modify `dashboard/src/app/kit/projects/[id]/page.tsx`.

- [ ] **Step 1: Compute `isManager`** using the existing member/role hook on the page (reuse `useMembers` + current user `permission_level` if already imported; otherwise add the minimal hook per the extraction). Render the panel after `RecentBlueprints` (around line 254):

```tsx
import { SaveStatesPanel } from "@/components/features/projects/save-states-panel";
// ...
const isManager = ["owner", "admin", "manager"].includes(currentPermissionLevel ?? "");
// ...
<SaveStatesPanel projectId={project.id} isManager={isManager} />
```

- [ ] **Step 2: Typecheck + lint.** `cd dashboard && npm run typecheck && npm run lint` → clean.
- [ ] **Step 3: Commit** `git commit -am "feat(dashboard): show save states panel on project page"`

---

### Task 6: MSW handlers, data factory, component test

**Files:** Modify `dashboard/src/test/mocks/data.ts`, `handlers.ts`; create the panel test.

- [ ] **Step 1: Add the factory** to `data.ts`:

```typescript
import type { SaveState } from "@/lib/api/save-states";
export function mockSaveState(overrides: Partial<SaveState> = {}): SaveState {
  return {
    id: nextId(), kind: "personal", title: "My save state",
    handoff: "Left off mid auth refactor.", digest_date: null, period_start: null,
    saved_by: { id: "user-1", email: "dev@example.com", name: "Developer" },
    reviewed_by: null, reviewed_at: null, published_at: "2026-06-16T10:00:00Z",
    source_session_id: null,
    created_at: "2026-06-16T10:00:00Z", updated_at: "2026-06-16T10:00:00Z",
    ...overrides,
  };
}
```

- [ ] **Step 2: Add handlers** to `handlers.ts` (and include `...saveStateHandlers` in the exported `handlers` array):

```typescript
const saveStateHandlers = [
  http.get(`${API_URL}/api/v1/:orgId/save_states`, () =>
    HttpResponse.json({ data: [
      mockSaveState({ kind: "personal", title: "My save state" }),
      mockSaveState({ kind: "team_daily", title: "Team daily digest", saved_by: null,
        handoff: "alice shipped login; bob reviewed api.", digest_date: "2026-06-15" }),
    ] })),
  http.patch(`${API_URL}/api/v1/:orgId/save_states/team/:kind`, () =>
    HttpResponse.json({ data: mockSaveState({ kind: "team_daily", handoff: "edited" }) })),
  http.delete(`${API_URL}/api/v1/:orgId/save_states/:id`, () =>
    HttpResponse.json(null, { status: 204 })),
];
```

- [ ] **Step 3: Write the panel test** (`@/test/test-utils`):

```tsx
// dashboard/src/components/features/projects/__tests__/save-states-panel.test.tsx
import { render, screen, waitFor, userEvent } from "@/test/test-utils";
import { SaveStatesPanel } from "../save-states-panel";

describe("SaveStatesPanel", () => {
  it("lists personal and team digest rows", async () => {
    render(<SaveStatesPanel projectId="proj-1" />);
    await waitFor(() => expect(screen.getByText("My save state")).toBeInTheDocument());
    expect(screen.getByText("Team daily digest")).toBeInTheDocument();
  });

  it("hides team Edit for non-managers, shows it for managers", async () => {
    const { rerender } = render(<SaveStatesPanel projectId="proj-1" isManager={false} />);
    await waitFor(() => expect(screen.getByText("Team daily digest")).toBeInTheDocument());
    expect(screen.queryByRole("button", { name: "Edit" })).not.toBeInTheDocument();
    rerender(<SaveStatesPanel projectId="proj-1" isManager={true} />);
    await waitFor(() => expect(screen.getByRole("button", { name: "Edit" })).toBeInTheDocument());
  });

  it("shows a Draft badge + Publish button to a manager for an unpublished team digest", async () => {
    server.use(
      http.get(`${API_URL}/api/v1/:orgId/save_states`, () =>
        HttpResponse.json({ data: [
          mockSaveState({ kind: "team_daily", title: "Team daily digest",
            saved_by: null, published_at: null }),
        ] })),
    );
    render(<SaveStatesPanel projectId="proj-1" isManager={true} />);
    await waitFor(() => expect(screen.getByText("Draft")).toBeInTheDocument());
    expect(screen.getByRole("button", { name: "Publish" })).toBeInTheDocument();
  });

  it("opens the handoff in a sheet on click", async () => {
    const user = userEvent.setup();
    render(<SaveStatesPanel projectId="proj-1" />);
    await waitFor(() => expect(screen.getByText("My save state")).toBeInTheDocument());
    await user.click(screen.getByText("My save state"));
    expect(await screen.findByText("Left off mid auth refactor.")).toBeInTheDocument();
  });
});
```

- [ ] **Step 4: Run the tests.** `cd dashboard && npx vitest run src/components/features/projects/__tests__/save-states-panel.test.tsx`
Expected: PASS.

- [ ] **Step 5: Commit** `git commit -am "test(dashboard): save states panel + msw handlers"`

---

### Task 7: Gate

- [ ] `cd dashboard && npm run lint && npm run typecheck && npx vitest run && npm run validate:api-contract` → all green.

## Self-review

- **Spec coverage:** view personal + team daily + weekend (Task 4 list), view full handoff (Sheet), manager-only edit of team digests (`PATCH team/:kind`, Task 3+4), **Draft badge + Publish action for unpublished (`require_review`-held) team digests — Publish reuses `useUpdateTeamDigest` since the server's `update_team` sets `published_at` on review** (Task 4), delete (own personal or team-if-manager, Task 4), empty state (Task 4), role gating (Task 5), types regen + contract (Task 1), tests + MSW incl. a draft/publish case (Task 6).
- **Contract consistency:** uses `kind`/`handoff`/`saved_by`/`digest_date`, the org-scoped routes with `project_id`, `PATCH /save_states/team/:kind`, and `DELETE /save_states/:id` — exactly the Plan-1 serializer + routes. (Corrected from the earlier `state_type`/`name`/`data` draft.)
- **Naming consistency:** `useSaveStates`/`useUpdateTeamDigest`/`useDeleteSaveState`/`partitionSaveStates`, `saveStatesApi.{list,updateTeamDigest,remove}`, `SaveStatesPanel` props `{projectId, isManager}` used identically across hook, component, page, and test.
- **Assumptions to verify:** the project page's current way of knowing the user's `permission_level` (reuse it; the extraction shows `useMembers` + `permission_level` as the path); `apiClient` DELETE tolerates a 204 empty body (mirror `artifactsApi.delete`); `sonner` `toast` is the project's toast (it is, per use-artifacts).
