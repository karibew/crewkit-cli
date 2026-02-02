# Product Strategy: Playbook Convention Enforcement

**Status**: Decision Framework
**Date**: 2026-01-15
**Decision Owner**: Product Strategy

---

## Executive Summary

**Problem**: Teams want Claude Code to follow their conventions (Rails patterns, API standards, security rules). Current `.claude/CLAUDE.md` relies on Claude voluntarily following instructions.

**Hook Limitation**: UserPromptSubmit hook can only block (erasing user's prompt) or inject invisible context. No warn-and-allow mode exists.

**Strategic Question**: Should we build enforcement, or is context injection sufficient?

**Recommendation**: **Phased rollout starting with Context-Only (Phase 1)**, with decision gates based on actual violation data before investing in enforcement infrastructure.

---

## Decision Framework

### Success Metrics Hierarchy

**North Star Metric**: Convention adherence rate
- **Measure**: % of Claude responses that follow playbook conventions
- **Target**: 85%+ adherence without enforcement

**Leading Indicators**:
1. **Violation frequency**: How often Claude ignores conventions?
2. **Violation severity**: Do violations cause bugs, tech debt, or just style issues?
3. **Developer time cost**: Minutes spent fixing convention violations per session
4. **User tolerance**: Do teams want enforcement or is nudging enough?

**Lagging Indicators**:
1. Code quality metrics (if violations cause bugs)
2. PR review time (if reviewers catch violations)
3. Support tickets about "Claude didn't follow our standards"

### Phase 1: Context-Only (Week 1-2)

**Hypothesis**: Injecting conventions.md into every prompt is sufficient. Claude will follow clear, well-structured rules without enforcement.

**Implementation**:
- UserPromptSubmit hook injects playbook conventions as `additionalContext`
- No blocking, no warnings, zero user friction
- Instrument everything for measurement

**Go Criteria** (to proceed to Phase 1):
- ✅ Playbook marketplace is live (prerequisite)
- ✅ At least 10 teams have subscribed to playbooks
- ✅ conventions.md format is stable

**Success Criteria** (Phase 1 validation):
- **85%+ adherence rate** → Stop here, ship it
- **70-84% adherence** → Consider Soft Warnings (Phase 2)
- **<70% adherence** → Need Soft Warnings or Hard Blocking

**Measurement** (via existing session insights):
1. **Manual sampling**: Review 50 LlmSessions, count violations
2. **LLM-as-judge**: Batch API evaluates adherence (use existing Brief system)
3. **User reports**: ConventionAlertMailer sends weekly digests

**Investment**: 2 days development
- Hook implementation (1 day)
- Analytics dashboard additions (1 day)

**Exit Criteria** (decision gate after 2 weeks):
- Have 200+ sessions with playbook subscriptions
- Measured adherence rate with 95% confidence interval
- Qualitative feedback from 5+ teams

---

### Phase 2: Soft Warnings (Week 3-4)

**Trigger**: Phase 1 shows <85% adherence AND violations are costly

**Hypothesis**: Claude mentioning conventions in its response increases adherence without blocking UX.

**Implementation**:
- Hook injects conventions + "Mention if you're deviating from conventions"
- Claude naturally explains deviations in responses
- User sees reasoning, can correct in follow-up

**Example**:
```
User: "Use Twilio for SMS"
Claude: "I see your playbook recommends SNS for messaging.
Should I proceed with Twilio anyway, or would you like SNS?"
```

**Go Criteria** (to proceed to Phase 2):
- ❌ Phase 1 adherence <85%
- ❌ Violations cost >10 min/session to fix (measured)
- ❌ At least 3 teams requested enforcement

**Success Criteria** (Phase 2 validation):
- **90%+ adherence** → Stop here, ship it
- **85-90% adherence** → Good enough, monitor
- **<85% adherence** → Consider Hard Blocking (Phase 3)

**Measurement**:
1. Adherence rate (same as Phase 1)
2. User satisfaction with warnings (survey)
3. Override rate (how often users proceed despite warning)

**Investment**: 3 days development
- Enhanced context injection (1 day)
- Response pattern analysis (1 day)
- Dashboard updates (1 day)

**Exit Criteria** (decision gate after 2 weeks):
- Have 200+ sessions with soft warnings
- Measured improvement over Phase 1
- User feedback on warning helpfulness

---

### Phase 3: Hard Blocking (Week 5-8)

**Trigger**: Phase 2 shows <85% adherence AND violations cause critical issues (security, data loss)

**Hypothesis**: Blocking critical violations is worth the UX friction when mistakes have high blast radius.

**Implementation**:
- Hook caches last prompt before blocking
- Custom `/playbook-override <reason>` command to bypass
- ConventionOverride table logs all overrides
- Weekly digest to managers: "3 overrides this week"

**Go Criteria** (to proceed to Phase 3):
- ❌ Phase 2 adherence <85%
- ❌ At least 1 critical violation caused production incident
- ❌ At least 5 teams explicitly requested blocking
- ❌ Business case: ROI calculation shows positive value

**Success Criteria** (Phase 3 validation):
- **95%+ adherence** (blocking works)
- **<5% override rate** (rules aren't too strict)
- **User satisfaction >3.5/5** (friction acceptable)

**Measurement**:
1. Adherence rate (should be ~100% minus overrides)
2. Override rate and reasons
3. Support tickets about "blocked incorrectly"
4. User churn (do teams disable playbooks?)

**Investment**: 10 days development
- Prompt caching system (3 days)
- `/playbook-override` command (2 days)
- ConventionOverride tracking (2 days)
- Manager digests (2 days)
- Testing and polish (1 day)

**Exit Criteria** (decision gate after 4 weeks):
- Have 500+ sessions with blocking enabled
- Measured adherence rate and override patterns
- User feedback on blocking experience
- Business impact assessment

---

## Key Questions to Answer (Before Starting)

### 1. Do we have evidence Claude violates clear conventions?

**Answer Method**: Lightweight experiment
- Take 5 existing teams using crewkit
- Ask them to define 5 critical conventions in CLAUDE.md
- Review last 20 sessions per team (100 total)
- Count violations manually

**Decision**:
- **<10 violations total** → Don't build this, not a real problem
- **10-30 violations** → Build Phase 1, measure
- **>30 violations** → Build Phase 1, plan for Phase 2

**Timeline**: 1 day (5 hours review + 3 hours analysis)

---

### 2. What's the cost of violations?

**Answer Method**: User interviews + time tracking
- Interview 5 teams currently using crewkit
- Ask: "How much time do you spend fixing Claude's convention violations?"
- Measure: Minutes per session, severity (cosmetic vs critical)

**Decision**:
- **<5 min/session** → Low priority, Phase 1 sufficient
- **5-15 min/session** → Medium priority, proceed to Phase 2 if Phase 1 fails
- **>15 min/session** → High priority, fast-track to Phase 2

**Cost-Benefit Threshold**:
- Phase 1 investment: 2 days = $2k dev cost
- Break-even: Save 5 min/session × 200 sessions/month × $0.50/min = $500/month
- ROI: 4 months to break even

---

### 3. What's the tolerance for enforcement friction?

**Answer Method**: User personas analysis

**Sarah (Senior Engineer)**:
- High tolerance for friction IF it prevents junior mistakes
- Quote: "I'd rather Claude blocks bad patterns than review 10 messy PRs"
- Willingness: 8/10 for blocking

**Mike (Manager)**:
- Low tolerance for friction that slows team down
- Quote: "If blocking costs more time than violations, it's a net negative"
- Willingness: 4/10 for blocking

**Alex (Architect)**:
- High tolerance IF conventions are precise and correct
- Quote: "Blocking is fine for security rules, annoying for style preferences"
- Willingness: 7/10 for critical-only blocking

**Decision**:
- **High-severity rules only** (security, data loss) → Hard Blocking
- **Medium-severity rules** (architecture patterns) → Soft Warnings
- **Low-severity rules** (style, naming) → Context-Only

---

### 4. Is this "must-have" or "nice-to-have"?

**Must-Have Criteria**:
1. Violations cause production incidents (security, data loss)
2. Violations cost >15 min/session to fix
3. At least 30% of teams explicitly request enforcement
4. Competitors offer enforcement (market expectation)

**Nice-to-Have Criteria**:
1. Violations are cosmetic (style, naming)
2. Violations cost <5 min/session to fix
3. Teams are satisfied with current CLAUDE.md approach
4. No competitive pressure

**Current Assessment** (pending evidence):
- Likely **nice-to-have** for most conventions
- Possibly **must-have** for security/compliance rules
- Decision: Start with nice-to-have (Phase 1), upgrade if evidence shows must-have

---

## Validation Plan (Before Phase 1)

### Week 0: Evidence Gathering

**Day 1-2: Manual Review** (5 hours)
- Review 100 sessions from 5 teams
- Count violations by severity
- Calculate time cost estimates

**Day 3: User Interviews** (3 hours)
- Interview 5 teams (30 min each)
- Ask about violation frequency, severity, tolerance
- Gauge interest in enforcement

**Day 4: Cost-Benefit Analysis** (2 hours)
- Calculate: violation_cost × frequency vs development_cost
- Determine ROI for each phase
- Identify break-even points

**Day 5: Decision** (1 hour)
- Review evidence with team
- Decide: Proceed to Phase 1, defer, or kill feature

**Go/No-Go Decision**:
- **GO**: Violations cost >$500/month across customers AND ROI >2x
- **NO-GO**: Violations <10 total OR cost <$200/month OR teams don't care

---

## Minimal Viable First Step

**If we proceed (post-validation):**

### Phase 1A: Silent Monitoring (Week 1)

Before ANY enforcement, measure baseline:

**Implementation** (1 day):
1. UserPromptSubmit hook logs prompts (no injection yet)
2. Batch API evaluates adherence post-session
3. Dashboard shows adherence rate per team

**Goal**: Establish baseline adherence rate with zero intervention

**Decision Gate**:
- **Baseline >85%** → Claude already follows conventions, don't build enforcement
- **Baseline 70-84%** → Proceed to Phase 1B (context injection)
- **Baseline <70%** → Deep dive: Are conventions unclear? Wrong format?

### Phase 1B: Context Injection (Week 2)

After baseline established:

**Implementation** (1 day):
1. Enable additionalContext injection
2. Measure adherence rate change
3. Calculate lift vs baseline

**Decision Gate**:
- **Lift >10%** → Context injection works, ship it
- **Lift 5-10%** → Marginal, proceed to Phase 2 (soft warnings)
- **Lift <5%** → Context injection doesn't work, investigate why

---

## Risk Mitigation

### Risk 1: Users Disable Playbooks (High Friction)

**Mitigation**:
- Feature flag per organization: "enforcement_level" (off/context/warn/block)
- Default to context-only
- Let teams opt into stricter enforcement
- Monitor churn rate by enforcement level

### Risk 2: False Positives (Blocking Valid Patterns)

**Mitigation**:
- Phase 3 only: Override logging reveals false positives
- Weekly review of overrides
- Refine conventions based on override reasons
- Target: <5% override rate

### Risk 3: Development Cost > User Value

**Mitigation**:
- Gate each phase with measurable success criteria
- Kill feature if Phase 1 doesn't show ROI
- User interviews at each gate
- Compare cost vs measured time savings

### Risk 4: Hook Latency Impacts UX

**Mitigation**:
- Measure hook execution time (target: <100ms)
- Cache conventions.md per project
- Use Haiku for analysis (fast, cheap)
- Monitor P95 latency in dashboard

---

## Decision Summary

### Recommended Path

**Week 0**: Evidence gathering (5 days)
- Manual review, interviews, cost-benefit analysis
- **Gate**: Proceed only if violations cost >$500/month

**Week 1**: Phase 1A - Silent monitoring (baseline)
- **Gate**: If baseline >85%, stop here (no problem to solve)

**Week 2**: Phase 1B - Context injection
- **Gate**: If lift >10%, ship and monitor

**Week 3-4**: Phase 2 - Soft warnings (if Phase 1B fails)
- **Gate**: If adherence >85%, stop here

**Week 5-8**: Phase 3 - Hard blocking (if Phase 2 fails AND critical violations exist)
- **Gate**: ROI must be >3x given complexity

### Success Definition

**Phase 1 Success**: 85%+ adherence with context-only, <2ms latency, zero user complaints
**Phase 2 Success**: 90%+ adherence with soft warnings, >4/5 user satisfaction
**Phase 3 Success**: 95%+ adherence with blocking, <5% override rate, positive NPS impact

### Kill Criteria

**Kill if**:
1. Week 0 evidence shows <10 violations per 100 sessions
2. Phase 1A baseline is already >85% (no problem exists)
3. Phase 1B lift is <5% (context injection doesn't work)
4. Phase 2 user satisfaction <3/5 (friction too high)
5. Any phase shows negative ROI

---

## Open Questions for Stakeholders

1. **Priority**: Where does this rank vs other Q1 features (session analytics dashboard, role modifiers)?
2. **Resources**: Can we allocate 1 engineer for 2 weeks for evidence gathering + Phase 1?
3. **Beta Users**: Which 5 teams should we interview for validation?
4. **Success Bar**: Is 85% adherence acceptable, or do we need >90%?
5. **Enforcement Philosophy**: Are we building guardrails (prevent mistakes) or coaching (teach best practices)?

---

## Next Steps

**Immediate** (this week):
1. [ ] Get stakeholder buy-in on phased approach
2. [ ] Identify 5 beta teams for evidence gathering
3. [ ] Define "convention violation" precisely (what counts?)
4. [ ] Create measurement rubric for manual review

**Week 0** (evidence gathering):
1. [ ] Manual review of 100 sessions
2. [ ] User interviews (5 teams)
3. [ ] Cost-benefit analysis
4. [ ] Go/no-go decision

**Week 1** (if GO):
1. [ ] Implement Phase 1A (silent monitoring)
2. [ ] Measure baseline adherence
3. [ ] Decision gate: Proceed to 1B?

---

**Decision Owner**: Product Strategy
**Engineering Lead**: TBD
**Timeline**: 2 weeks evidence gathering + 2-8 weeks implementation (phased)
**Budget**: $2k (Phase 1) → $5k (Phase 2) → $15k (Phase 3)
**ROI Target**: >2x cost savings within 6 months
