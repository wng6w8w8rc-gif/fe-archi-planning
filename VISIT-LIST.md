# VisitList Component Refactoring

## Overview

This document outlines the refactoring of the `VisitList` component to improve separation of concerns, maintainability, and testability by extracting responsibilities into custom hooks.

## Current Issues

### 1. **Too Many Responsibilities**

The component was handling:

- Data fetching and state management
- User authentication state
- Store selection based on visit list type
- Pagination logic
- Multiple action handlers (5+ callbacks)
- UI rendering logic
- Empty state determination

### 2. **Tight Coupling**

- Business logic mixed with presentation
- Hard to test individual concerns
- Difficult to reuse logic in other components

### 3. **Code Organization**

- 274 lines in a single component file
- Configuration mixed with component logic
- Action handlers scattered throughout

## Refactoring Strategy

### Separation of Concerns

Split the component into three main areas:

1. **Data & Fetching** → `@/components/hooks/use-visit/index.ts` hook (primary hook)
2. **Action Handlers** → `@/components/hooks/use-visit/use-visit-list-actions.ts` hook
3. **Configuration** → `lib.ts` file (following codebase pattern)
4. **UI Rendering** → Simplified `index.tsx` component

## File Structure

```
visit-list/
├── index.tsx                   # Main component (UI only)
├── lib.ts                      # Configuration & utilities (list type config)
└── REFACTORING.md              # This file

components/hooks/
└── use-visit/                  # Hooks folder (following existing hooks pattern)
    ├── index.ts                # Primary hook: use-visit-list (data fetching & state management)
    └── use-visit-list-actions.ts  # Action handlers hook
```

**Note**: Following the codebase hooks pattern:

- **Hooks** are placed in `@/components/hooks/` following the existing pattern (e.g., `use-app-version/index.tsx`, `use-booking-route.ts`)
- The primary hook is named `index.ts` for cleaner imports: `import { useVisitList } from "@/components/hooks/use-visit"`
- **Configuration and utilities** go in `lib.ts` within the container (e.g., `homepage/lib.tsx`, `visits/lib.ts`)
- This structure allows the hooks to be easily reused across visit-related components

## Implementation Details

### 1. `lib.ts` - Configuration & Utilities

**Purpose**: Centralizes all visit list type configurations and utilities

**Exports**:

- `VisitListOptions` type
- `listType` configuration object

**Benefits**:

- Single source of truth for list configurations
- Easy to add new visit list types
- Type-safe configuration
- Separated from component logic

**Key Features**:

- Defines title, icon, subtitle for each type
- Specifies which store to use
- Provides filter generation functions
- Configures pagination behavior

### 2. `@/components/hooks/use-visit/index.ts` - Data & Fetching Hook (Primary)

**Purpose**: Manages all data fetching and state management

**Responsibilities**:

- Get user from auth store
- Select appropriate visit store based on type
- Fetch visits with initial filters
- Handle pagination (load more)
- Compute derived state (shouldShowLoadMore)

**Returns**:

```typescript
{
  visits: VisitData[];
  loading: boolean;
  shouldShowLoadMore: boolean;
  onLoadMore: () => void;
}
```

**Key Features**:

- Automatic fetching when user.id or type changes
- Memoized load more visibility calculation
- Encapsulated pagination logic
- Clean separation from UI

### 3. `@/components/hooks/use-visit/use-visit-list-actions.ts` - Action Handlers Hook

**Purpose**: Manages all user interactions and visit actions

**Responsibilities**:

- View visit details
- Rebook a visit
- Set/reschedule visit
- Rate a visit
- Report visit issues

**Returns**:

```typescript
{
  handleViewDetail: (visitId: string) => void;
  handleRebook: (department: PackageDepartmentEnum) => void;
  handleSetSchedule: (visitId: string) => void;
  handleRateVisit: (visitId: string, rate?: number) => void;
  handleReportVisit: (visitId: string) => void;
}
```

**Key Features**:

- All callbacks memoized with `useCallback`
- Centralized action logic
- Easy to test independently
- Reusable across components

### 4. `index.tsx` - Main Component

**Purpose**: Pure presentation component

**Responsibilities**:

- Render UI structure
- Display loading states
- Show empty states
- Render visit cards
- Display load more button

**Key Features**:

- Clean and readable
- Focused on presentation
- Easy to understand at a glance
- Minimal business logic

## Benefits

### 1. **Maintainability**

- Each file has a single, clear responsibility
- Changes to data fetching don't affect UI
- Action handlers can be updated independently

### 2. **Testability**

- Hooks can be tested in isolation
- Mock dependencies easily
- Test business logic without UI

### 3. **Reusability**

- `use-visit-list-actions` can be used in other components
- Configuration can be shared
- Data fetching logic is reusable

### 4. **Readability**

- Component is now ~100 lines (down from 274)
- Clear separation of concerns
- Self-documenting code structure

### 5. **Type Safety**

- All hooks are fully typed
- Configuration is type-safe
- Better IDE autocomplete

## Implementation Realization

This section shows the complete before and after code for each file in the refactoring.

### 1. Configuration File: `lib.ts`

**Before**: Configuration was inline in `index.tsx` (lines 38-106)

**After**: Extracted to `lib.ts`

```typescript:luce-fe/src/containers/visits/visit-list/lib.ts
import type {
  PackageDepartmentEnum,
  VisitsByFiltersQueryVariables,
} from "@/__generated__/graphql";
import { VisitStatusEnum } from "@/__generated__/graphql";
import type { Icon } from "@/components/shared/icons";
import { CalendarDots, CalendarX, MapPinArea } from "@/components/shared/icons";
import { addDays, addMonths, addYears, startOfYesterday } from "date-fns";
import { formatDate } from "@/lib/helpers/date";
import type { VisitData } from "@/components/shared/visits/visit-card";
import type { RequestPaginatedFactoryType } from "@/lib/request-factory";
import type { VisitListType } from "@/store/visits/visitList";
import { useNext7DayVisitsStore } from "@/store/visits/next7dayVisits";
import { useNext30DayVisitsStore } from "@/store/visits/next30dayVisits";
import { useUnscheduledVisitsStore } from "@/store/visits/unscheduledVisits";
import { usePast30DayVisitsStore } from "@/store/visits/past30dayVisits";
import { __ } from "@/lib/i18n";

export type VisitListOptions = {
  title: string;
  tlId: string;
  icon: Icon;
  subtitle?: string;
  hasLoadMore: boolean;
  useStore: RequestPaginatedFactoryType<
    VisitData,
    VisitsByFiltersQueryVariables
  >;
  initialFilters: (
    clientId: string,
  ) => VisitsByFiltersQueryVariables["filters"];
};

export const listType: Record<VisitListType, VisitListOptions> = {
  next7days: {
    title: __("Visits in the next 7 days"),
    tlId: "visits.7Days",
    icon: MapPinArea,
    hasLoadMore: false,
    useStore: useNext7DayVisitsStore,
    initialFilters: (clientId) => ({
      clientId: [clientId],
      from: formatDate(new Date()),
      to: formatDate(addDays(new Date(), 6)),
    }),
  },
  unscheduled: {
    title: __("Unscheduled Visits"),
    tlId: "visits.unscheduled",
    icon: CalendarX,
    hasLoadMore: false,
    useStore: useUnscheduledVisitsStore,
    initialFilters: (clientId) => ({
      clientId: [clientId],
      from: formatDate(new Date()),
      to: formatDate(addMonths(new Date(), 1)),
      status: [
        VisitStatusEnum.PendingClientSchedule,
        VisitStatusEnum.UnableToSchedule,
      ],
    }),
  },
  next30days: {
    title: __("Visits in the next 30 days"),
    tlId: "visits.30Days",
    icon: CalendarDots,
    hasLoadMore: true,
    useStore: useNext30DayVisitsStore,
    initialFilters: (clientId) => ({
      clientId: [clientId],
      from: formatDate(addDays(new Date(), 7)),
      to: formatDate(addMonths(new Date(), 1)),
    }),
  },
  past30days: {
    title: __("Past Visits"),
    tlId: "visits.past",
    icon: MapPinArea,
    hasLoadMore: true,
    useStore: usePast30DayVisitsStore,
    initialFilters: (clientId) => ({
      clientId: [clientId],
      from: formatDate(addYears(startOfYesterday(), -1)),
      to: formatDate(startOfYesterday()),
    }),
  },
};
```

### 2. Primary Hook: `@/components/hooks/use-visit/index.ts`

**Before**: Data fetching logic was in `index.tsx` (lines 111-169)

**After**: Extracted to custom hook

```typescript:luce-fe/src/components/hooks/use-visit/index.ts
import { useEffect, useMemo, useCallback } from "react";
import { useShallow } from "zustand/react/shallow";
import { useAuthState } from "@/store/auth";
import type { VisitListType } from "@/store/visits/visitList";
import type { VisitData } from "@/components/shared/visits/visit-card";
import { listType } from "@/containers/visits/visit-list/lib";

type UseVisitListReturn = {
  visits: VisitData[];
  loading: boolean;
  shouldShowLoadMore: boolean;
  onLoadMore: () => void;
};

export function useVisitList(type: VisitListType): UseVisitListReturn {
  const user = useAuthState((state) => state.data.userInfo);
  const { useStore, initialFilters, hasLoadMore } = listType[type];

  const {
    fetch: getVisits,
    data: visits,
    loading,
    fetchMore,
    pagination,
  } = useStore(useShallow((state) => state));

  const shouldShowLoadMore = useMemo(
    () => hasLoadMore && pagination.total > visits.length,
    [hasLoadMore, pagination.total, visits.length],
  );

  useEffect(() => {
    if (user.id) {
      const filters = initialFilters(user.id);
      getVisits({
        requestPayload: {
          filters,
        },
      });
    }
  }, [user.id, type, getVisits, initialFilters]);

  const onLoadMore = useCallback(() => {
    if (user.id) {
      fetchMore({
        requestPayload: {
          filters: {
            ...initialFilters(user.id),
          },
        },
      });
    }
  }, [user.id, fetchMore, initialFilters]);

  return {
    visits,
    loading,
    shouldShowLoadMore,
    onLoadMore,
  };
}
```

### 3. Actions Hook: `@/components/hooks/use-visit/use-visit-list-actions.ts`

**Before**: Action handlers were in `index.tsx` (lines 108-203)

**After**: Extracted to custom hook

```typescript:luce-fe/src/components/hooks/use-visit/use-visit-list-actions.ts
import { useCallback } from "react";
import { useRoute } from "@/components/shared/router";
import { bookingRoutes } from "@/components/hooks/use-booking-route";
import { getServiceNameFromDepartment } from "@/lib/booking-lib";
import type { PackageDepartmentEnum } from "@/__generated__/graphql";
import { useVisitDetailStore } from "@/store/visits/visitDetail";
import { useVisitStore } from "@/store/visits/visitStore";
import {
  useVisitIssueStore,
  VisitIssueType,
} from "@/store/report-issue/visit-issue/visitIssue";

type UseVisitListActionsReturn = {
  handleViewDetail: (visitId: string) => void;
  handleRebook: (department: PackageDepartmentEnum) => void;
  handleSetSchedule: (visitId: string) => void;
  handleRateVisit: (visitId: string, rate?: number) => void;
  handleReportVisit: (visitId: string) => void;
};

export function useVisitListActions(): UseVisitListActionsReturn {
  const { push } = useRoute();

  const handleViewDetail = useVisitDetailStore(
    (state) => state.openVisitModal,
  );

  const openRateVisitModalById = useVisitDetailStore(
    (state) => state.openRateVisitModalById,
  );

  const openRescheduleVisitConfirmation = useVisitDetailStore(
    (state) => state.openRescheduleVisitConfirmation,
  );

  const openVisitIssueModal = useVisitIssueStore(
    (state) => state.openVisitIssueModal,
  );

  const handleRebook = useCallback(
    (department: PackageDepartmentEnum) => {
      const serviceName = getServiceNameFromDepartment(department);
      push(bookingRoutes[serviceName][0]);
    },
    [push],
  );

  const handleSetSchedule = useCallback(
    (visitId: string) => {
      openRescheduleVisitConfirmation(visitId);
    },
    [openRescheduleVisitConfirmation],
  );

  const handleRateVisit = useCallback(
    (visitId: string, rate?: number) => {
      openRateVisitModalById(visitId, rate);
      useVisitStore.getState().fetch({
        requestPayload: {
          id: visitId,
        },
      });
    },
    [openRateVisitModalById],
  );

  const handleReportVisit = useCallback(
    (visitId: string) => {
      openVisitIssueModal(VisitIssueType.reportVisit, visitId, false);
    },
    [openVisitIssueModal],
  );

  return {
    handleViewDetail,
    handleRebook,
    handleSetSchedule,
    handleRateVisit,
    handleReportVisit,
  };
}
```

### 4. Main Component: `index.tsx`

**Before**: 274 lines with all logic mixed together

```typescript:luce-fe/src/containers/visits/visit-list/index.tsx
// BEFORE - 274 lines
import type {
  PackageDepartmentEnum,
  VisitsByFiltersQueryVariables,
} from "@/__generated__/graphql";
import { VisitStatusEnum } from "@/__generated__/graphql";
import { ListHeading } from "@/components/shared/list-header";
import { Button, Skeleton, View } from "@/components/ui";
import type { Icon } from "@/components/shared/icons";
import { CalendarDots, CalendarX, MapPinArea } from "@/components/shared/icons";
import { addDays, addMonths, addYears, startOfYesterday } from "date-fns";
import { formatDate } from "@/lib/helpers/date";
import { useEffect, useCallback, useMemo } from "react";
import { useAuthState } from "@/store/auth";
import { useNext7DayVisitsStore } from "@/store/visits/next7dayVisits";
import { useNext30DayVisitsStore } from "@/store/visits/next30dayVisits";
import { useUnscheduledVisitsStore } from "@/store/visits/unscheduledVisits";
import { usePast30DayVisitsStore } from "@/store/visits/past30dayVisits";
import type { VisitData } from "@/components/shared/visits/visit-card";
import { VisitCard } from "@/components/shared/visits/visit-card";
import type { RequestPaginatedFactoryType } from "@/lib/request-factory";
import type { VisitListType } from "@/store/visits/visitList";
import { useVisitDetailStore } from "@/store/visits/visitDetail";
import { useVisitStore } from "@/store/visits/visitStore";
import { useShallow } from "zustand/react/shallow";
import { useRoute } from "@/components/shared/router";
import { bookingRoutes } from "@/components/hooks/use-booking-route";
import { getServiceNameFromDepartment } from "@/lib/booking-lib";
import { IfElse } from "@/components/shared/if-else";
import {
  useVisitIssueStore,
  VisitIssueType,
} from "@/store/report-issue/visit-issue/visitIssue";
import { __ } from "@/lib/i18n";
import NoVisitsBike from "@/assets/images/3d-illus/no-visit-bike.png";
import NoVisitsVaccum from "@/assets/images/3d-illus/no-visits-vaccum.png";
import { EmptyCard } from "@/components/shared/empty-card";

// Configuration inline (lines 38-106)
type VisitListOptions = { /* ... */ };
export const listType: Record<VisitListType, VisitListOptions> = { /* ... */ };

export function VisitList({ type }: { type: VisitListType }) {
  const { push } = useRoute();
  const user = useAuthState((state) => state.data.userInfo);

  // Store selectors inline
  const handleViewDetail = useVisitDetailStore((state) => state.openVisitModal);
  const openRateVisitModalById = useVisitDetailStore((state) => state.openRateVisitModalById);
  const openRescheduleVisitConfirmation = useVisitDetailStore((state) => state.openRescheduleVisitConfirmation);
  const openVisitIssueModal = useVisitIssueStore((state) => state.openVisitIssueModal);

  // Configuration extraction
  const { title, subtitle, icon, hasLoadMore, initialFilters, useStore } = listType[type];

  // Store usage inline
  const { fetch: getVisits, data: visits, loading, fetchMore, pagination } = useStore(useShallow((state) => state));

  // Derived state calculation inline
  const shouldShowLoadMore = useMemo(
    () => hasLoadMore && pagination.total > visits.length,
    [hasLoadMore, pagination.total, visits.length],
  );

  // Data fetching logic inline
  useEffect(() => {
    if (user.id) {
      const filters = listType[type].initialFilters(user.id);
      getVisits({ requestPayload: { filters } });
    }
  }, [user.id, type, getVisits]);

  // Load more handler inline
  const onLoadMore = useCallback(() => {
    if (user.id) {
      fetchMore({ requestPayload: { filters: { ...initialFilters(user.id) } } });
    }
  }, [user.id, fetchMore, initialFilters]);

  // Action handlers inline (lines 171-203)
  const handleRebook = useCallback((department: PackageDepartmentEnum) => {
    const serviceName = getServiceNameFromDepartment(department);
    push(bookingRoutes[serviceName][0]);
  }, [push]);

  const handleSetSchedule = useCallback((visitId: string) => {
    openRescheduleVisitConfirmation(visitId);
  }, [openRescheduleVisitConfirmation]);

  const handleRateVisit = useCallback((visitId: string, rate?: number) => {
    openRateVisitModalById(visitId, rate);
    useVisitStore.getState().fetch({ requestPayload: { id: visitId } });
  }, [openRateVisitModalById]);

  const handleReportVisit = useCallback((visitId: string) => {
    openVisitIssueModal(VisitIssueType.reportVisit, visitId, false);
  }, [openVisitIssueModal]);

  // UI rendering (lines 205-272)
  return (
    <View className="flex flex-col gap-4">
      {/* ... */}
    </View>
  );
}
```

**After**: ~90 lines, clean and focused on UI

```typescript:luce-fe/src/containers/visits/visit-list/index.tsx
// AFTER - ~90 lines
import type { VisitListType } from "@/store/visits/visitList";
import { ListHeading } from "@/components/shared/list-header";
import { Button, Skeleton, View } from "@/components/ui";
import { VisitCard } from "@/components/shared/visits/visit-card";
import { IfElse } from "@/components/shared/if-else";
import { __ } from "@/lib/i18n";
import NoVisitsBike from "@/assets/images/3d-illus/no-visit-bike.png";
import NoVisitsVaccum from "@/assets/images/3d-illus/no-visits-vaccum.png";
import { EmptyCard } from "@/components/shared/empty-card";
import { useVisitList } from "@/components/hooks/use-visit";
import { useVisitListActions } from "@/components/hooks/use-visit/use-visit-list-actions";
import { listType } from "./lib";

export function VisitList({ type }: { type: VisitListType }) {
  const { title, subtitle, icon } = listType[type];
  const { visits, loading, shouldShowLoadMore, onLoadMore } = useVisitList(type);
  const {
    handleViewDetail,
    handleRebook,
    handleSetSchedule,
    handleRateVisit,
    handleReportVisit,
  } = useVisitListActions();

  const emptyState = type === "next30days" ? (
    <EmptyCard
      src={NoVisitsVaccum}
      title={__("You don't have any history visits")}
      desc={__(
        "Your completed bookings will appear here once the service is done.",
      )}
    />
  ) : (
    <EmptyCard
      src={NoVisitsBike}
      title={__("You don't have any upcoming visits")}
      desc={__(
        "Your scheduled services will appear here once booked.",
      )}
    />
  );

  return (
    <View className="flex flex-col gap-4">
      <ListHeading
        icon={icon}
        title={__(title)}
        subtitle={__(subtitle || "")}
      />
      {loading ? (
        <>
          <Skeleton className="h-[200px] w-full" />
          <Skeleton className="h-[200px] w-full" />
          <Skeleton className="h-[200px] w-full" />
          <Skeleton className="h-[200px] w-full" />
        </>
      ) : (
        <IfElse
          if={!!visits?.length}
          else={emptyState}
        >
          <View className="flex flex-col gap-4 md:p-0">
            {visits.map((visit) => (
              <VisitCard
                key={`${visit.id}-${type}`}
                visitData={visit}
                handleViewDetail={handleViewDetail}
                handleRateVisit={handleRateVisit}
                handleRebook={handleRebook}
                handleSetSchedule={handleSetSchedule}
                handleReportVisit={handleReportVisit}
              />
            ))}
          </View>
        </IfElse>
      )}

      <IfElse if={shouldShowLoadMore}>
        <Button
          variant="tertiary"
          className="w-full"
          loading={loading}
          onClick={onLoadMore}
          iconName="arrowCounterClockwise"
          iconAlignment="end"
          children={__("Show More")}
        />
      </IfElse>
    </View>
  );
}
```

## Migration Summary

**Lines of Code Reduction**:

- Before: 274 lines in single file
- After: ~90 lines (component) + ~70 lines (lib.ts) + ~60 lines (hooks) = ~220 lines total
- **Net reduction**: ~54 lines with better organization

**Import Reduction**:

- Before: 36 imports in component
- After: 12 imports in component (67% reduction)

**Separation of Concerns**:

- ✅ Configuration → `lib.ts`
- ✅ Data fetching → `use-visit/index.ts`
- ✅ Actions → `use-visit/use-visit-list-actions.ts`
- ✅ UI → `index.tsx`

## Dependencies

### External Dependencies

- `zustand` - State management
- `date-fns` - Date utilities
- React hooks (`useEffect`, `useMemo`, `useCallback`)

### Internal Dependencies

- `@/store/auth` - User authentication
- `@/store/visits/*` - Visit stores
- `@/store/visits/visitDetail` - Visit detail modals
- `@/store/report-issue/visit-issue` - Issue reporting
- `@/components/shared/router` - Navigation
- `@/lib/booking-lib` - Booking utilities

## Testing Strategy

### Unit Tests

1. **`@/components/hooks/use-visit/index.ts`**
   - Test data fetching on mount
   - Test pagination logic
   - Test load more visibility calculation
   - Test effect dependencies

2. **`@/components/hooks/use-visit/use-visit-list-actions.ts`**
   - Test each action handler
   - Test navigation calls
   - Test store interactions
   - Test callback memoization

3. **`lib.ts`**
   - Test filter generation
   - Test configuration completeness
   - Test type safety

### Integration Tests

- Test component with all hooks
- Test user interactions
- Test loading states
- Test empty states

## Future Improvements

### Potential Enhancements

1. **Error Handling**
   - Add error state to `use-visit-list`
   - Display error messages in UI
   - Retry mechanisms

2. **Optimization**
   - Add request cancellation
   - Implement optimistic updates
   - Cache management

3. **Accessibility**
   - Add ARIA labels
   - Keyboard navigation
   - Screen reader support

4. **Performance**
   - Virtual scrolling for long lists
   - Lazy loading of visit cards
   - Debounce search/filter

## Code Examples

### Using the Hook in Other Components

```typescript
// In another component
import { useVisitList } from "@/components/hooks/use-visit";
import { useVisitListActions } from "@/components/hooks/use-visit/use-visit-list-actions";

function VisitDashboard() {
  const { visits, loading } = useVisitList("next7days");
  const { handleViewDetail } = useVisitListActions();

  // Use the data and actions
}
```

### Adding a New Visit List Type

```typescript
// In lib.ts
export const listType: Record<VisitListType, VisitListOptions> = {
  // ... existing types
  customType: {
    title: __("Custom Visits"),
    tlId: "visits.custom",
    icon: CustomIcon,
    hasLoadMore: true,
    useStore: useCustomVisitsStore,
    initialFilters: (clientId) => ({
      clientId: [clientId],
      // custom filters
    }),
  },
};
```

## Notes

- All hooks follow React best practices
- Memoization used appropriately to prevent unnecessary re-renders
- Type safety maintained throughout
- No breaking changes to component API
- Backward compatible with existing usage
