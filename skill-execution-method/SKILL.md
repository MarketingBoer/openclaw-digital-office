---
name: skill-execution-method
description: |
  Use when choosing an execution method for building skill-sets with Claude Code.
  Routes to parallel subagents, teams, or hybrid approach.
  Triggers: "how to build skills", "parallel or teams", "skill development approach",
  "team setup for skills", "execution method", "how to split the work".
---

# Skill Execution Method

Choose how to execute a multi-skill build project with Claude Code.

## Decision Matrix

| Method | Best For | Speed | Quality | Cost |
|--------|----------|-------|---------|------|
| Parallel | Solo developer, independent skills | High | Medium | Low |
| Teams | Multiple people, compliance/enterprise | Low | High | High |
| Hybrid | Large projects (5+ skills) | Medium-High | High | Medium |

## Parallel Subagents

Opus orchestrator spawns Sonnet subagents per skill group.

- Group skills that share context (max 3-4 per group)
- Each subagent builds independently
- Orchestrator cross-reviews at the end
- Risk: inconsistency between groups, self-review bias
- Mitigation: shared ontology file, cross-reference table

## Teams Approach

Uses Claude Teams/Enterprise plan with role separation.

| Role | Model | Task |
|------|-------|------|
| Builder | Sonnet | Writes SKILL.md + references |
| Reviewer | Opus | Checks correctness, security, completeness |
| Tester | Sonnet | Runs TDD scenarios as fresh agent (no context) |
| Admin | — | Provisions approved skills to team |

Workflow: Builder → Tester → Reviewer → Admin deploy.

## Hybrid (recommended for 5+ skills)

1. **PARALLEL:** Subagents build skills per group (speed)
2. **REVIEW:** Separate reviewer-agent cross-checks all skills (quality)
3. **TEST:** Fresh test-agent runs all TDD scenarios (verification)
4. **DEPLOY:** Admin provisions to team (distribution)

## Model Settings Per Role

| Role | Model | Effort | Thinking |
|------|-------|--------|----------|
| Orchestrator/Planner | Opus | high | true |
| Builder | Sonnet | high | true |
| Reviewer | Opus | high | true |
| Tester | Sonnet | medium | false |

## Lessons Learned

- Subagents for building lose context and produce inconsistent output across groups
- Sequential self-building (one skill at a time) produces higher quality than parallel
- Agents ARE useful for TDD testing (fresh agent = honest verification)
- Always share an ontology file across all builders to prevent semantic drift
