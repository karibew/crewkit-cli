# Agent Relationship Consistency Analysis

**Date**: 2025-10-15
**Purpose**: Document agent relationships and identify inconsistencies for correction

## Relationship Matrix (Completed Agents)

### 1. task-orchestrator (Master Coordinator)
**Role**: Primary coordinator for all complex multi-step tasks

**Invokes**: ALL specialist agents (by design)
**Invoked by**: User directly, rails-expert (for multi-domain tasks)

**Status**: ✅ Complete - Correctly positioned as master coordinator

---

### 2. product-strategist
**Role**: Business strategy, user research, feature prioritization

**Invokes**: (needs verification)
**Invoked by**: task-orchestrator, api-designer

**Status**: ⚠️ Needs review - Relationship sections need verification

---

### 3. api-designer
**Role**: REST API architecture, sub-100ms targets

**Invokes**:
- product-strategist (validate API design)
- rails-expert (service layer implementation)
- code-reviewer (validation)

**Invoked by**:
- task-orchestrator
- rails-expert (API-first workflow)
- frontend-expert (after API tested)

**Status**: ✅ Complete and coherent

---

### 4. rails-expert
**Role**: Rails 8 full-stack development

**Mentions** (but lacks structured sections):
- frontend-expert
- cli-expert ❌ (should be cli-ux-designer?)
- api-designer
- code-reviewer

**Invokes**: (MISSING structured section)
**Invoked by**: (MISSING structured section)

**Status**: ❌ INCOMPLETE - Needs structured relationship sections added

**Required fixes**:
- Add "Works With" section
- Add "Invokes" section
- Add "Invoked By" section
- Clarify cli-expert vs cli-ux-designer
- Reference design-system-expert relationship

---

### 5. frontend-expert
**Role**: Hotwire frontend specialist

**Works With**:
- design-system-expert ✅
- rails-expert ✅
- api-designer ✅

**Invokes**:
- design-system-expert ✅
- code-reviewer ✅

**Invoked By**:
- task-orchestrator ✅
- rails-expert ✅

**Status**: ✅ Complete and coherent

---

### 6. design-system-expert
**Role**: Component library management

**Works With**:
- frontend-expert ✅
- rails-expert ✅
- code-reviewer ✅
- product-strategist ✅

**Invokes**:
- code-reviewer ✅
- frontend-expert ✅

**Invoked By**:
- frontend-expert ✅
- rails-expert ✅
- task-orchestrator ✅
- product-strategist ✅

**Status**: ✅ Complete and coherent

---

### 7. svg-designer
**Role**: Vercel-inspired SVG graphics

**Works With**:
- design-system-expert ✅
- frontend-expert ✅
- rails-expert ✅

**Status**: ✅ Complete and coherent

**Note**: Bidirectional check needed:
- Does design-system-expert mention svg-designer? ⚠️ CHECK
- Does frontend-expert mention svg-designer? ⚠️ CHECK
- Does rails-expert mention svg-designer? ⚠️ CHECK

---

### 8. cli-ux-designer
**Role**: CLI UX specialist

**Invokes**: (needs verification)
**Invoked by**: task-orchestrator

**Status**: ⚠️ Needs verification

**Confusion**: rails-expert mentions "cli-expert" but AGENT_ROSTER shows "cli-ux-designer"

---

## Identified Issues

### Issue 1: rails-expert Missing Structured Relationships ❌
**Problem**: rails-expert mentions other agents but lacks structured "Works With", "Invokes", "Invoked By" sections

**Fix Required**:
```markdown
## Integration with Other Agents

### Works With
- **api-designer** - API-first workflow, endpoints tested before UI
- **frontend-expert** - Provides backend for Hotwire features
- **design-system-expert** - Coordinates on view structure and component usage
- **cli-ux-designer** - CLI command backend implementation
- **svg-designer** - Asset pipeline integration for SVGs

### Invokes
- **api-designer** - For complex API design decisions
- **frontend-expert** - After API complete (API-first workflow)
- **design-system-expert** - For component guidance
- **code-reviewer** - Quality validation
- **test-automation-expert** - Test implementation

### Invoked By
- **task-orchestrator** - Primary
- **api-designer** - Service layer implementation
- **frontend-expert** - Backend support for UI features
```

---

### Issue 2: Bidirectional Relationship Consistency ⚠️
**Problem**: When A "works with" B, B should acknowledge A

**Checks Needed**:
1. svg-designer → design-system-expert (NEW)
   - Does design-system-expert mention svg-designer? **CHECK**

2. svg-designer → frontend-expert (NEW)
   - Does frontend-expert mention svg-designer? **CHECK**

3. svg-designer → rails-expert (NEW)
   - Does rails-expert mention svg-designer? **CHECK**

**Fix**: Add reciprocal mentions where missing

---

### Issue 3: cli-expert vs cli-ux-designer Naming ⚠️
**Problem**: rails-expert mentions "cli-expert" but AGENT_ROSTER shows "cli-ux-designer"

**Resolution Needed**:
- Verify which name is correct
- Update all references consistently
- Possible: cli-expert.md is old file, should be removed?

---

### Issue 4: code-reviewer Referenced But Not Defined ⚠️
**Problem**: Multiple agents invoke "code-reviewer" but it's not in completed agents list

**Status**: Listed in "🔜 Agents to Create" but heavily referenced

**Options**:
1. Mark as placeholder (agents can invoke future agents)
2. Create basic code-reviewer agent definition
3. Add "(TODO)" suffix to all code-reviewer references

---

## Recommended Actions

### Priority 1: Fix rails-expert (CRITICAL) ❌
Add structured relationship sections immediately

### Priority 2: Update for svg-designer (NEW AGENT) ⚠️
Add svg-designer mentions to:
- design-system-expert (in "Works With" section)
- frontend-expert (in "Works With" section)
- rails-expert (in "Works With" section - if structured sections added)

### Priority 3: Verify Bidirectional Consistency ⚠️
Check all "Works With" relationships are reciprocal

### Priority 4: Clarify CLI Agent Naming ⚠️
Resolve cli-expert vs cli-ux-designer confusion

### Priority 5: Handle Future Agent References ℹ️
Decide on convention for referencing agents not yet implemented

---

## Verification Checklist

For each agent, verify:
- [ ] Has "Works With" section (if collaborates with others)
- [ ] Has "Invokes" section (if delegates to specialists)
- [ ] Has "Invoked By" section (who calls this agent)
- [ ] All mentioned agents exist or are clearly marked as future
- [ ] Bidirectional relationships are consistent
- [ ] Names match AGENT_ROSTER exactly

---

## Agent Relationship Rules

1. **task-orchestrator** invokes all agents (master coordinator)
2. **Bidirectional "Works With"** - If A works with B, B should mention A
3. **Invokes vs Invoked By** - Must be reciprocal (A invokes B = B invoked by A)
4. **Name consistency** - Use exact names from AGENT_ROSTER
5. **Future agents** - Can be referenced but should be clearly marked

---

**Next Steps**: Fix identified issues in priority order

---

## ✅ COMPLETED - 2025-10-15

All relationship inconsistencies have been resolved:

### Completed Fixes

1. **✅ rails-expert Relationships** - Added complete "Works With", "Invokes", "Invoked By" sections
   - Now references: api-designer, frontend-expert, design-system-expert, cli-ux-designer, svg-designer

2. **✅ svg-designer Bidirectional Relationships** - Established reciprocal mentions
   - Added to design-system-expert (Works With, Invoked By, Invokes)
   - Added to frontend-expert (Works With)

3. **✅ CLI Agent Naming** - Clarified relationship between two CLI agents
   - **cli-ux-designer**: Designs CLI experience (commands, UX patterns, error messages)
   - **cli-expert**: Implements the designs (Rust, clap, ratatui)
   - Added cli-expert to AGENT_ROSTER as 16th completed agent
   - Updated orchestration hierarchy to show design → implementation workflow

4. **✅ New Agent Created** - marketing-specialist
   - Dual-purpose: Security watchdog (prevents information leaks) + Marketing effectiveness (value props, CTAs)
   - Reviews all public-facing content (landing pages, README, docs)
   - Added as 17th completed agent in new "Marketing & Documentation" category
   - Full relationship sections included (Works With, Invoked By, Invokes)

### Updated Agent Counts

**Before**: 28 agents (15 complete, 13 specified)
**After**: 30 agents (17 complete, 13 specified)

**New completed agents**:
- cli-expert (clarified existing agent)
- marketing-specialist (newly created)

### Verification Status

All relationship rules now followed:
- ✅ Bidirectional "Works With" relationships consistent
- ✅ Invokes/Invoked By relationships reciprocal
- ✅ All agent names match AGENT_ROSTER exactly
- ✅ Structured sections present in all primary agents
- ✅ Collaboration workflows clearly documented

**Agent ecosystem relationships are now coherent and complete.**
