# Sprint Plan - OJT Session: Visit List Module Refactoring

## Overview

This document outlines the sprint-by-sprint implementation plan for the OJT session focused on refactoring the **Visit List Module** only. This is a **partial implementation** that does NOT use the "V2" method - changes will be applied directly to the existing codebase to give examples of how the pattern will be implemented.

The plan follows a point-based estimation system where:

- **1 point = 1 day of work**
- **1 week = 3.5 points of work + 1.5 points buffer = 5.0 points total**
- Tasks can span across multiple weeks

## Module Points Summary

| Module                          | Points  | Description                                          |
| ------------------------------- | ------- | ---------------------------------------------------- |
| **Global Modal Implementation** | 1.0     | Migrate key visit list modals to global modal system |
| **GraphQL Query Migration**     | 1.0     | Replace request factory with direct useQuery hooks   |
| **Code Modularization**         | 1.5     | Apply basic modularization pattern to visit list     |
| **Total Work Points**           | **3.5** |                                                      |

## Week-by-Week Work Breakdown

| Week       | Module                             | Points | Cumulative | Buffer | Total Week |
| ---------- | ---------------------------------- | ------ | ---------- | ------ | ---------- |
| **Week 1** | Global Modal: Setup + Key modals   | 1.0    | 1.0        | 1.5    | 5.0        |
|            | GraphQL: Setup + Migrate queries   | 1.0    | 2.0        |        |            |
|            | Code Modularization: Extract hooks | 1.5    | 3.5        |        |            |

## Weekly Points Summary

| Week      | Work Points | Buffer Points | Total Points |
| --------- | ----------- | ------------- | ------------ |
| Week 1    | 3.5         | 1.5           | 5.0          |
| **Total** | **3.5**     | **1.5**       | **5.0**      |

## Detailed Week Breakdown

### Week 1 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: Global Modal + GraphQL Migration + Code Modularization**

| Task                                            | Points  | Status |
| ----------------------------------------------- | ------- | ------ |
| Global Modal: Setup store/component (if needed) | 0.3     | ⬜     |
| Global Modal: Migrate Visit Detail Modal        | 0.7     | ⬜     |
| GraphQL: Setup `gqlhooks/visits/` folder        | 0.3     | ⬜     |
| GraphQL: Create useVisitsByFiltersQuery hook    | 0.4     | ⬜     |
| GraphQL: Migrate 1 visit list store             | 0.3     | ⬜     |
| Code Modularization: Extract Visit List hooks   | 1.5     | ⬜     |
| **Buffer**                                      | **1.5** |        |

**Deliverables:**

- Global modal store/component created
- Visit Detail modal migrated to global modal system
- GraphQL hooks folder structure created
- At least 1 visit list query migrated from request factory to useQuery hooks
- Visit List component hooks extracted and modularized

---

## Visit List Module Scope

### Components Included

- `src/containers/visits/visit-list/index.tsx` - Main Visit List component
- `src/components/shared/visits/visit-detail/` - Visit Detail modals

### Stores to Migrate

**Modal Stores:**

- `src/store/visits/visitDetail.ts` - Visit Detail modals (6+ modals)
- `src/store/report-issue/visit-issue/visitIssue.ts` - Visit Issue modal

**GraphQL Stores:**

- `src/store/visits/next7dayVisits.ts` - Next 7 days visits query
- `src/store/visits/next30dayVisits.ts` - Next 30 days visits query
- `src/store/visits/unscheduledVisits.ts` - Unscheduled visits query
- `src/store/visits/past30dayVisits.ts` - Past 30 days visits query
- `src/store/visits/visitStore.ts` - Single visit query

### GraphQL Queries to Migrate

- `VisitsByFiltersQuery` - Used by all visit list stores
- `VisitQuery` - Used by visit detail store

---

## Risk Mitigation

- **Buffer time:** 1.5 points allocated for discussions, unexpected issues, and code reviews
- **Focused scope:** Limited to essential tasks to ensure completion within 1 week
- **Isolated changes:** Visit List module changes are isolated and won't affect other modules
- **No V2 complexity:** Direct changes to existing codebase reduce migration complexity
- **Partial migration:** Only migrate 1 store as proof of concept, not all stores

---

## Success Criteria

### Global Modal Implementation

- ✅ At least 1 visit list modal (Visit Detail) uses global modal system
- ✅ Modal pattern established for future migrations
- ✅ Consistent modal behavior demonstrated

### GraphQL Query Migration

- ✅ At least 1 visit list query uses direct useQuery hooks
- ✅ GraphQL hooks pattern established
- ✅ Proper error handling and loading states demonstrated

### Code Modularization

- ✅ Visit List component uses extracted hooks
- ✅ Code is more maintainable and testable
- ✅ Clear separation of concerns demonstrated

---

## Notes

- **Scope:** This plan is ONLY for the Visit List module. Other modules remain unchanged.
- **No V2 Method:** All changes are applied directly to existing codebase structure.
- **Partial Implementation:** This is a learning exercise demonstrating the three concepts:
  - Global modal implementation (1 modal as proof of concept)
  - GraphQL query migration (1 store as proof of concept)
  - Code modularization (Visit List + hook extraction)
- **Testing:** Manual testing required after completion.
- **Time Constraint:** Limited to 3.5 work points (1 week) to focus on learning the patterns rather than complete migration.

---

**Last Updated:** [Date]
**Owned By:** OJT Session Participant
**Mentor:** Luce.sg (Gio, Fathul)
