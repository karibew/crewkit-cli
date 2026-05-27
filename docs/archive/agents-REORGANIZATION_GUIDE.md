# Agent Reorganization Guide

## Overview

This guide provides a systematic approach to reorganize the remaining 12 agents into crewkit's three-tier configuration inheritance structure.

**Status**: 5/17 agents complete (29%)

**Completed**:
- ✅ agent-creator.md
- ✅ svg-designer.md
- ✅ rails-expert.md
- ✅ task-orchestrator.md
- ✅ api-designer.md

**Remaining** (in recommended order):
1. pattern-detection-expert.md (294 lines)
2. streaming-ingestion-expert.md (321 lines)
3. ml-engineer.md (397 lines)
4. batch-ingestion-expert.md (468 lines)
5. marketing-specialist.md (487 lines)
6. schema-architect.md (496 lines)
7. data-analyst-expert.md (524 lines)
8. cli-expert.md (830 lines)
9. cli-ux-designer.md (866 lines)
10. product-strategist.md (899 lines)
11. design-system-expert.md (909 lines)
12. frontend-expert.md (1,175 lines)

---

## Three-Tier Structure

### BASE TIER (Generic Expertise)
**Purpose**: Content applicable to ANY organization using this technology

**Contains**:
- Universal best practices
- Industry-standard patterns
- Technology-agnostic principles
- Generic code examples (use generic class names, not crewkit-specific)
- Reusable templates
- Testing patterns applicable anywhere

**Examples**:
- "Multi-tenant SaaS pattern" (not "crewkit multi-tenancy")
- "REST API best practices" (not "crewkit API design")
- "Statistical significance testing" (not "crewkit experiment analysis")

**Markers**:
```markdown
<!-- BASE_TIER_START -->
# BASE TIER: Generic [Domain] Expertise

[All generic content here]

<!-- BASE_TIER_END -->
```

---

### ORGANIZATION TIER (crewkit Standards)
**Purpose**: Company-specific rules, conventions, and standards

**Contains**:
- crewkit development workflow
- crewkit naming conventions
- crewkit brand guidelines
- Role-based system (entry_level, junior, etc.)
- 3-word experiment naming (adjective-color-noun)
- API-first approach
- Agent collaboration patterns
- Triggers (when to invoke this agent)

**Examples**:
- "crewkit uses 3-word slugs for experiments"
- "API-FIRST workflow is MANDATORY"
- "Role modifiers: junior → coaching mode"
- "Always lowercase 'crewkit' (not 'Crewkit')"

**Markers**:
```markdown
<!-- ORGANIZATION_TIER_START -->
# ORGANIZATION TIER: crewkit [Domain] Standards

[All crewkit-specific standards here]

<!-- ORGANIZATION_TIER_END -->
```

---

### PROJECT TIER (crewkit Implementation)
**Purpose**: Specific to this codebase, file paths, current implementation

**Contains**:
- Actual file paths (`api/app/controllers/api/v1/`)
- Specific model names (Organization, Project, AgentExperiment)
- Current development commands (`cd ~/code/karibew/crewkit/api && bin/dev`)
- Integration with OTHER specific agents (rails-expert, frontend-expert)
- Exact API endpoints (`http://localhost:3050/api/v1/agents/...`)
- Current tech stack versions (Rails 8.0, Ruby 3.3.2)
- Existing service names (AgentConfigurationService)

**Examples**:
- "Files located at: `api/app/services/agent_configuration_service.rb`"
- "Run: `curl http://localhost:3050/api/v1/experiments`"
- "Invokes: **rails-expert**, **frontend-expert**"
- "Models: Organization, Project, AgentConfiguration"

**Markers**:
```markdown
<!-- PROJECT_TIER_START -->
# PROJECT TIER: crewkit Platform Implementation

[All project-specific implementation details here]

<!-- PROJECT_TIER_END -->
```

---

## Classification Decision Tree

Use this flowchart to classify each section of content:

```
Is this content specific to the crewkit codebase?
├─ YES → Is it about file paths, specific models, or exact commands?
│         ├─ YES → PROJECT TIER
│         └─ NO  → Is it about crewkit standards/conventions?
│                  ├─ YES → ORGANIZATION TIER
│                  └─ NO  → BASE TIER (shouldn't happen if first answer was YES)
└─ NO  → Could another company reuse this content as-is?
          ├─ YES → BASE TIER
          └─ NO  → Re-evaluate, might be ORGANIZATION TIER
```

---

## Step-by-Step Process

### Step 1: Read the Agent File
```bash
cat .claude/agents/[agent-name].md
```

### Step 2: Identify Content Categories

Go through the file section by section and mark each with a tier label:

**Example Markings**:
```
[INTRO] - Usually ORGANIZATION (agent identity in crewkit)
## Best Practices - Usually BASE (generic principles)
## crewkit Context - ORGANIZATION
## File Paths - PROJECT
## Example Code with generic classes - BASE
## Example Code with crewkit models - PROJECT
## Integration with Other Agents - PROJECT (specific agent names)
```

### Step 3: Create New Structure

Start with this template:

```markdown
---
name: agent-name
description: [Keep original description]
---

<!-- BASE_TIER_START -->
# BASE TIER: Generic [Domain] Expertise

## [Generic Section 1]
[Content from original file that's universal]

## [Generic Section 2]
[More universal content]

<!-- BASE_TIER_END -->

<!-- ORGANIZATION_TIER_START -->
# ORGANIZATION TIER: crewkit [Domain] Standards

[Agent's role description at crewkit]

## Triggers (Proactive)
[When this agent should be invoked]

## crewkit-Specific [Topic]
[Company standards, conventions, workflows]

<!-- ORGANIZATION_TIER_END -->

<!-- PROJECT_TIER_START -->
# PROJECT TIER: crewkit Platform Implementation

## File Paths
[Exact paths in this codebase]

## Commands
[Actual commands with full paths]

## Integration with Other Agents

### Invokes
- **agent-name** - Description

### Invoked By
- **agent-name** - Description

**Goal**: [One-sentence ultimate objective]

<!-- PROJECT_TIER_END -->
```

### Step 4: Move Content Into Tiers

**For BASE TIER**:
- Strip crewkit-specific references
- Make examples generic
- Keep only universal principles
- Ensure it could be used by any company

**For ORGANIZATION TIER**:
- Include crewkit standards
- Keep brand guidelines
- Explain workflows
- Define conventions

**For PROJECT TIER**:
- Use exact file paths
- Reference specific models
- Include actual commands
- Name other agents specifically

### Step 5: Verify Completeness

Check that:
- [ ] All original content is present (no loss)
- [ ] YAML frontmatter unchanged
- [ ] Each tier has appropriate content
- [ ] HTML comment markers are correct
- [ ] Tier headers follow pattern
- [ ] No content duplicated across tiers
- [ ] Line count increased only by markers/headers (~20-50 lines)

### Step 6: Save and Test

```bash
# Save the file
# Then verify line count
wc -l .claude/agents/[agent-name].md

# The line count should increase by ~20-50 lines (for markers/headers only)
```

---

## Common Patterns by Agent Type

### For Data/Analytics Agents
**BASE**: Statistical methods, metrics formulas, A/B testing math
**ORG**: crewkit experiment conventions, significance thresholds
**PROJECT**: ExperimentMetricsService, AgentSession model

### For CLI Agents
**BASE**: CLI UX principles, error message patterns, autocomplete design
**ORG**: crewkit CLI conventions, Rust/clap standards
**PROJECT**: cli/src/ paths, actual crewkit commands

### For Infrastructure Agents
**BASE**: Streaming patterns, batch processing principles, data pipeline design
**ORG**: crewkit ingestion standards, data retention policies
**PROJECT**: Solid Queue configuration, actual job classes

### For Frontend Agents
**BASE**: Hotwire patterns, Stimulus principles, Tailwind patterns
**ORG**: crewkit design system, action naming standards
**PROJECT**: api/app/views/ paths, specific partials, controller integration

---

## Examples from Completed Agents

### Example 1: rails-expert.md

**BASE TIER** included:
- Multi-tenant SaaS pattern (universal)
- Configuration inheritance pattern (generic)
- A/B testing pattern (universal)
- Rails 8 Solid* suite (Rails 8 standard)
- Hotwire patterns (framework standard)

**ORGANIZATION TIER** included:
- crewkit API-FIRST workflow (company standard)
- crewkit role system (company convention)
- crewkit 3-word slugs (company convention)
- crewkit design system standards (company guideline)

**PROJECT TIER** included:
- Actual models: Organization, Project, AgentExperiment
- File paths: `api/app/controllers/api/v1/`
- Commands: `cd ~/code/karibew/crewkit/api && bin/dev`
- Specific services: AgentConfigurationService
- Integration with: api-designer, frontend-expert, design-system-expert

### Example 2: api-designer.md

**BASE TIER** included:
- REST API best practices (universal)
- Vercel performance principles (industry standard)
- Error message patterns (universal)
- Performance optimization checklist (universal)

**ORGANIZATION TIER** included:
- crewkit API-FIRST workflow (company standard)
- crewkit domain patterns (company conventions)
- crewkit performance requirements (company targets)

**PROJECT TIER** included:
- curl commands with `http://localhost:3050`
- Rswag integration specifics
- File paths: `api/app/controllers/api/v1/`
- Context7 usage for crewkit

### Example 3: task-orchestrator.md

**BASE TIER** included:
- Multi-agent orchestration pattern (universal)
- Iteration velocity principles (Vercel-inspired, universal)
- Orchestration patterns (universal)
- Context engineering (universal)

**ORGANIZATION TIER** included:
- crewkit agent selection guide (company-specific keyword map)
- crewkit API-FIRST approach (company workflow)
- Triggers for crewkit domain (company-specific)

**PROJECT TIER** included:
- crewkit file context for planning
- Actual development commands with full paths
- Integration with specific crewkit agents
- crewkit-specific orchestration examples

---

## Quality Checklist

Before considering an agent complete:

- [ ] **Content Preservation**: All original content is present
- [ ] **No Duplication**: No content appears in multiple tiers
- [ ] **Proper Classification**: Each section is in the correct tier
- [ ] **Markers Present**: All 6 HTML comment markers present
- [ ] **Tier Headers**: Each tier has proper header (# BASE TIER: Generic...)
- [ ] **Generic BASE**: BASE tier has no crewkit-specific content
- [ ] **Specific PROJECT**: PROJECT tier has actual file paths/models
- [ ] **Line Count**: Increased only by markers/headers (~20-50 lines)
- [ ] **YAML Intact**: Frontmatter unchanged
- [ ] **Readable**: Each tier flows logically
- [ ] **Parseability**: HTML comments allow easy extraction of tiers

---

## Troubleshooting

### "I'm not sure if this is BASE or ORGANIZATION tier"
**Ask**: Could Shopify use this content as-is for their internal agents?
- YES → BASE tier
- NO, they'd need to adapt it to Shopify conventions → ORGANIZATION tier

### "This content seems to fit in multiple tiers"
**Solution**: Keep it in the MOST SPECIFIC tier only.
- If it's project-specific → PROJECT tier ONLY
- If it's company-specific → ORGANIZATION tier ONLY
- Only put in BASE if it's truly universal

### "The file is getting too long"
**Expected**: Files will increase by ~20-50 lines due to markers and tier headers.
**Not Expected**: Files doubling in size (you're duplicating content).

### "I lost some content"
**Prevention**: Work section-by-section, checking off each as you move it.
**Recovery**: Use `git diff` to see what changed, restore missing content.

---

## Automation Tips

### Quick Classification Helper

```bash
# Search for crewkit-specific terms in a section
grep -i "crewkit\|organization\|project\|agent.*expert" section.txt

# If many matches → ORGANIZATION or PROJECT tier
# If few/no matches → likely BASE tier
```

### Verify No Content Loss

```bash
# Before reorganization
wc -w original-agent.md > before.txt

# After reorganization
wc -w reorganized-agent.md > after.txt

# Word count should be approximately the same (±50 words for markers)
diff before.txt after.txt
```

---

## Commit Strategy

After reorganizing each agent (or batch of 2-3):

```bash
git add .claude/agents/[agent-name].md
git commit -m "refactor(agents): reorganize [agent-name] into three-tier structure

- BASE TIER: [Brief description]
- ORGANIZATION TIER: [Brief description]
- PROJECT TIER: [Brief description]

[X]/17 agents complete ([Y]%)"
```

---

## Final Verification

After all 17 agents are reorganized:

```bash
# Verify all agents have tier markers
grep -l "<!-- BASE_TIER_START -->" .claude/agents/*.md | wc -l
# Should output: 17

# Verify no agents lost content (check line counts increased reasonably)
for file in .claude/agents/*.md; do
  echo "$file: $(wc -l < "$file") lines"
done

# Verify git shows only the expected changes
git diff --stat
```

---

## Timeline Estimate

Per agent:
- Small (300 lines): ~10-15 minutes
- Medium (500 lines): ~15-20 minutes
- Large (900+ lines): ~20-30 minutes

**Total for remaining 12 agents**: ~3-4 hours

**Recommendation**: Work in batches of 3-4 agents, commit between batches.

---

## Need Help?

Reference the completed agents for patterns:
- **agent-creator.md** - Small agent, clear pattern
- **api-designer.md** - Medium agent, good BASE/ORG separation
- **rails-expert.md** - Large agent, comprehensive PROJECT tier
- **task-orchestrator.md** - Coordination agent, workflow-heavy
- **svg-designer.md** - Creative agent, Vercel philosophy in BASE

**Good luck! The pattern is clear, and you have great examples to follow.**
