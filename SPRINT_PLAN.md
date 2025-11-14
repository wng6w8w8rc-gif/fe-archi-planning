# Sprint Plan - V2 Refactoring Implementation

## Overview

This document outlines the sprint-by-sprint implementation plan for the V2 refactoring project. The plan follows a point-based estimation system where:

- **1 point = 1 day of work**
- **1 week = 3.5 points of work + 1.5 points buffer = 5.0 points total**
- Tasks can span across multiple weeks

## Module Points Summary

| Module                             | Points  | Description                                                 |
| ---------------------------------- | ------- | ----------------------------------------------------------- |
| **V2 Infrastructure Setup**        | 3       | Create v2 folder structure, route config, route integration |
| **Auth Provider → Store**          | 2       | Migrate AuthProvider to Zustand store                       |
| **App Version Provider → Store**   | 1       | Migrate AppVersionProvider to Zustand store                 |
| **Notifications Provider → Store** | 3       | Migrate NotificationsProvider to Zustand store              |
| **Global Modal Refactoring**       | 9       | Consolidate 8+ modal stores into single global modal        |
| **GraphQL Query Simplification**   | 10      | Replace createRequestFactory with direct useQuery hooks     |
| **Total Work Points**              | **28**  |                                                             |
| **Code Modularization**            | Ongoing | 2 points per component (not included in total)              |

## Week-by-Week Work Breakdown

| Week       | Module                                    | Points | Cumulative | Buffer | Total Week |
| ---------- | ----------------------------------------- | ------ | ---------- | ------ | ---------- |
| **Week 1** | V2 Infrastructure Setup                   | 3.0    | 3.0        | 1.5    | 5.0        |
|            | Auth Provider → Store (start)             | 0.5    | 3.5        |        |            |
| **Week 2** | Auth Provider → Store (continue)          | 1.5    | 5.0        | 1.5    | 5.0        |
|            | App Version Provider → Store              | 1.0    | 6.0        |        |            |
|            | Notifications Provider → Store (start)    | 1.0    | 7.0        |        |            |
| **Week 3** | Notifications Provider → Store (continue) | 2.0    | 9.0        | 1.5    | 5.0        |
|            | Global Modal Refactoring (start)          | 1.5    | 10.5       |        |            |
| **Week 4** | Global Modal Refactoring (continue)       | 3.5    | 14.0       | 1.5    | 5.0        |
| **Week 5** | Global Modal Refactoring (continue)       | 4.0    | 18.0       | 1.5    | 5.0        |
| **Week 6** | GraphQL Query Simplification (start)      | 3.5    | 21.5       | 1.5    | 5.0        |
| **Week 7** | GraphQL Query Simplification (continue)   | 3.5    | 25.0       | 1.5    | 5.0        |
| **Week 8** | GraphQL Query Simplification (continue)   | 3.0    | 28.0       | 1.5    | 5.0        |

## Weekly Points Summary

| Week      | Work Points | Buffer Points | Total Points |
| --------- | ----------- | ------------- | ------------ |
| Week 1    | 3.5         | 1.5           | 5.0          |
| Week 2    | 3.5         | 1.5           | 5.0          |
| Week 3    | 3.5         | 1.5           | 5.0          |
| Week 4    | 3.5         | 1.5           | 5.0          |
| Week 5    | 3.5         | 1.5           | 5.0          |
| Week 6    | 3.5         | 1.5           | 5.0          |
| Week 7    | 3.5         | 1.5           | 5.0          |
| Week 8    | 3.5         | 1.5           | 5.0          |
| **Total** | **28.0**    | **12.0**      | **40.0**     |

## Detailed Week Breakdown

### Week 1 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: V2 Infrastructure Setup + Auth Provider Start**

| Task                              | Points  | Status |
| --------------------------------- | ------- | ------ |
| Create `src/v2/` folder structure | 0.5     | ⬜     |
| Create `src/config/routes.ts`     | 0.5     | ⬜     |
| Update `src/constants/routes.ts`  | 0.5     | ⬜     |
| Update `src/app/_layout.tsx`      | 1.0     | ⬜     |
| Update `src/app-web/router.tsx`   | 0.5     | ⬜     |
| Auth Provider: Create store       | 0.5     | ⬜     |
| **Buffer**                        | **1.5** |        |

**Deliverables:**

- V2 folder structure created
- Route version configuration system in place
- Route integration setup complete
- Auth store created

---

### Week 2 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: Complete Provider Migrations**

| Task                                | Points  | Status |
| ----------------------------------- | ------- | ------ |
| Auth Provider: Create hooks         | 0.5     | ⬜     |
| Auth Provider: Migration + testing  | 1.0     | ⬜     |
| App Version: Create store           | 0.25    | ⬜     |
| App Version: Create hooks           | 0.25    | ⬜     |
| App Version: Migration + testing    | 0.5     | ⬜     |
| Notifications: Create store (start) | 1.0     | ⬜     |
| **Buffer**                          | **1.5** |        |

**Deliverables:**

- Auth Provider fully migrated to Zustand
- App Version Provider fully migrated to Zustand
- Notifications store created

---

### Week 3 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: Complete Notifications + Start Global Modal**

| Task                                   | Points  | Status |
| -------------------------------------- | ------- | ------ |
| Notifications: Create store (continue) | 0.5     | ⬜     |
| Notifications: Create hooks            | 0.5     | ⬜     |
| Notifications: Create init hook        | 1.0     | ⬜     |
| Notifications: Migration + testing     | 0.5     | ⬜     |
| Global Modal: Create store             | 1.0     | ⬜     |
| **Buffer**                             | **1.5** |        |

**Deliverables:**

- Notifications Provider fully migrated to Zustand
- Global Modal store created

---

### Week 4 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: Global Modal Component + Store Migrations**

| Task                                  | Points  | Status |
| ------------------------------------- | ------- | ------ |
| Global Modal: Create component        | 1.5     | ⬜     |
| Global Modal: Migrate stores (part 1) | 2.0     | ⬜     |
| **Buffer**                            | **1.5** |        |

**Deliverables:**

- Global Modal component created
- First batch of modal stores migrated

---

### Week 5 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: Complete Global Modal Migration**

| Task                                  | Points  | Status |
| ------------------------------------- | ------- | ------ |
| Global Modal: Migrate stores (part 2) | 2.0     | ⬜     |
| Global Modal: Update components       | 2.0     | ⬜     |
| **Buffer**                            | **1.5** |        |

**Deliverables:**

- All modal stores migrated
- All components updated to use global modal

---

### Week 6 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: Global Modal Cleanup + Start GraphQL**

| Task                              | Points  | Status |
| --------------------------------- | ------- | ------ |
| Global Modal: Testing + cleanup   | 0.5     | ⬜     |
| GraphQL: Setup `gqlhooks/` folder | 0.5     | ⬜     |
| GraphQL: Simple queries (part 1)  | 2.5     | ⬜     |
| **Buffer**                        | **1.5** |        |

**Deliverables:**

- Global Modal refactoring complete
- GraphQL hooks folder structure created
- First batch of simple queries migrated

---

### Week 7 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: GraphQL Simple + Complex Queries**

| Task                                           | Points  | Status |
| ---------------------------------------------- | ------- | ------ |
| GraphQL: Simple queries (part 2)               | 1.0     | ⬜     |
| GraphQL: Complex queries + pagination (part 1) | 2.5     | ⬜     |
| **Buffer**                                     | **1.5** |        |

**Deliverables:**

- All simple queries migrated
- Complex queries migration started

---

### Week 8 (3.5 work + 1.5 buffer = 5.0 total)

**Focus: Complete GraphQL Migration**

| Task                                           | Points  | Status |
| ---------------------------------------------- | ------- | ------ |
| GraphQL: Complex queries + pagination (part 2) | 1.5     | ⬜     |
| GraphQL: Mutations                             | 2.0     | ⬜     |
| GraphQL: Remove request-factory + cleanup      | 1.0     | ⬜     |
| **Buffer**                                     | **1.5** |        |

**Deliverables:**

- All GraphQL queries migrated
- All mutations migrated
- `request-factory.tsx` removed
- GraphQL simplification complete

---

## Summary Statistics

- **Total work points:** 28.0
- **Total buffer points:** 12.0
- **Total project points:** 40.0
- **Number of weeks:** 8
- **Average work per week:** 3.5
- **Average buffer per week:** 1.5
- **Total capacity per week:** 5.0

## Implementation Order Rationale

The implementation order is optimized for:

1. **Fastest wins first:** Auth Provider migration (2 points) establishes the pattern quickly
2. **Foundation before features:** V2 Infrastructure (3 points) must be done first to enable v2 work
3. **Pattern establishment:** Provider migrations establish Zustand patterns before larger refactors
4. **Core patterns before modularization:** Modal and GraphQL patterns must be established before code modularization
5. **Incremental progress:** Each week delivers working features, not just partial implementations

## Risk Mitigation

- **Buffer time:** 1.5 points per week allocated for discussions, unexpected issues, and code reviews
- **Incremental delivery:** Each week delivers complete features or significant progress
- **Pattern establishment:** Early weeks establish patterns that make later weeks faster
- **Isolated changes:** Each module can be tested independently before moving to the next

## Post-Implementation

After Week 8, the following work can begin:

- **Code Modularization:** Apply modularization pattern to components using new patterns (2 points per component)
- **Feature migration to v2:** Start migrating features to v2 folder using established patterns
- **Performance optimization:** Optimize based on new patterns
- **Documentation:** Update team documentation with new patterns

---

**Last Updated:** [Date]
**Owned By:** Luce.sg (Gio, Fathul)
