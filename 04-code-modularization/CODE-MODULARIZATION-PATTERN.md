# Code Modularization Pattern

## Summary

This document outlines the pattern for modularizing React components by extracting responsibilities into separate files. This pattern improves separation of concerns, maintainability, and testability. The pattern is demonstrated using `VisitList` and `InvoiceList` components as examples.

### Key Principles

1. **Configuration** → Extract to `lib.ts` file
2. **Data fetching logic** → Extract to custom hooks (`use-[feature]/index.ts`)
3. **Action handlers** → Extract to separate hooks (`use-[feature]/use-[feature]-actions.ts`)
4. **Component** → Focus only on UI rendering

### Benefits

- **Single responsibility** per file
- **Reusable hooks** across related components
- **Improved maintainability** with clear separation
- **Better testability** - test hooks and components independently
- **Reduced component complexity** - components become simpler and easier to read

## Pattern Structure

```
containers/[feature]/[feature]-list/
├── index.tsx          # UI component only (~60-90 lines)
└── lib.ts             # Configuration and types

components/hooks/use-[feature]/
├── index.ts           # Data fetching hook
└── use-[feature]-actions.ts  # Action handlers hook
```

## Implementation Pattern

### 1. Configuration File: `lib.ts`

**Purpose**: Centralize configuration, types, and constants

**Example from VisitList**:

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
import {
  useNext7DayVisitsStore,
  useNext30DayVisitsStore,
  useUnscheduledVisitsStore,
  usePast30DayVisitsStore,
} from "@/store/visits/visitListStores";
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
  // ... more configurations
};
```

**Example from InvoiceList**:

```typescript:luce-fe/src/containers/profile/invoices/invoice-list/lib.ts
import type { InvoiceFilterStatusEnum } from "@/types/invoice";
import type { InvoiceCardData } from "@/components/shared/invoices/invoice-card";
import type { RequestPaginatedFactoryType } from "@/lib/request-factory";
import type { InvoicesByFiltersQueryVariables } from "@/__generated__/graphql";
import {
  usePaidInvoicesStore,
  useUnpaidInvoicesStore,
} from "@/store/invoices/invoiceListStores";
import { useAuthState } from "@/store/auth";

export type InvoiceListOptions = {
  useStore: RequestPaginatedFactoryType<
    InvoiceCardData,
    InvoicesByFiltersQueryVariables
  >;
  initialFilters: (clientId: string) => InvoicesByFiltersQueryVariables["filters"];
};

export const listType: Record<InvoiceFilterStatusEnum, InvoiceListOptions> = {
  [InvoiceFilterStatusEnum.PAID]: {
    useStore: usePaidInvoicesStore,
    initialFilters: (clientId) => ({
      clientId: [clientId],
      paymentStatus: InvoiceFilterStatusEnum.PAID,
    }),
  },
  [InvoiceFilterStatusEnum.UNPAID]: {
    useStore: useUnpaidInvoicesStore,
    initialFilters: (clientId) => ({
      clientId: [clientId],
      paymentStatus: InvoiceFilterStatusEnum.UNPAID,
    }),
  },
};
```

### 2. Data Fetching Hook: `use-[feature]/index.ts`

**Purpose**: Handle all data fetching, loading states, and pagination logic

**Example from VisitList**:

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
          filters: initialFilters(user.id),
        },
      });
    }
  }, [user.id, type, getVisits, initialFilters]);

  const onLoadMore = useCallback(() => {
    if (user.id) {
      fetchMore({
        requestPayload: {
          filters: initialFilters(user.id),
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

**Example from InvoiceList**:

```typescript:luce-fe/src/components/hooks/use-invoice/index.ts
import { useEffect, useMemo, useCallback } from "react";
import { useShallow } from "zustand/react/shallow";
import { useAuthState } from "@/store/auth";
import { useScrollRefresh } from "@/components/shared/scroll-content/scroll-refresh-context";
import type { InvoiceFilterStatusEnum } from "@/types/invoice";
import type { InvoiceCardData } from "@/components/shared/invoices/invoice-card";
import { listType } from "@/containers/profile/invoices/invoice-list/lib";

type UseInvoiceListReturn = {
  invoices: InvoiceCardData[];
  loading: boolean;
  shouldShowLoadMore: boolean;
  onLoadMore: () => void;
  refreshing: boolean;
  setRefreshing: (refreshing: boolean) => void;
};

export function useInvoiceList(type: InvoiceFilterStatusEnum): UseInvoiceListReturn {
  const user = useAuthState((state) => state.data.userInfo);
  const { refreshing, setRefreshing } = useScrollRefresh();
  const { useStore, initialFilters } = listType[type];

  const {
    data: invoices,
    loading,
    fetchMore,
    refetch,
  } = useStore(useShallow((state) => state));

  const shouldShowLoadMore = useMemo(
    () => !!invoices.length,
    [invoices.length],
  );

  useEffect(() => {
    if (user.id) {
      refetch({
        requestPayload: { filters: initialFilters(user.id) },
      });
    }
  }, [user.id, type, initialFilters, refetch]);

  useEffect(() => {
    if (refreshing) {
      refetch();
      setRefreshing(false);
    }
  }, [refreshing, refetch, setRefreshing]);

  const onLoadMore = useCallback(() => {
    if (user.id) {
      fetchMore({
        requestPayload: {
          filters: initialFilters(user.id),
        },
      });
    }
  }, [user.id, fetchMore, initialFilters]);

  return {
    invoices,
    loading,
    shouldShowLoadMore,
    onLoadMore,
    refreshing,
    setRefreshing,
  };
}
```

### 3. Actions Hook: `use-[feature]/use-[feature]-actions.ts`

**Purpose**: Handle all user interactions and side effects

**Example from VisitList**:

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

**Example from InvoiceList**:

```typescript:luce-fe/src/components/hooks/use-invoice/use-invoice-list-actions.ts
import { useCallback } from "react";
import { useRoute } from "@/components/shared/router";
import { useWindowDimensions } from "@/components/hooks/use-window-dimension";
import { PaymentMethodScreenModal } from "@/types/invoice";
import { useInvoiceState } from "@/store/invoices";

type UseInvoiceListActionsReturn = {
  handleViewInvoice: (invoiceId: string) => () => void;
  handlePay: (invoiceId: string) => () => void;
};

export function useInvoiceListActions(): UseInvoiceListActionsReturn {
  const { push } = useRoute();
  const { isMobile } = useWindowDimensions();
  const { setInvoiceId, showPaymentMethodModal } = useInvoiceState();

  const handleViewInvoice = useCallback(
    (invoiceId: string) => () => {
      push({
        pageKey: "invoiceDetail",
        params: {
          id: invoiceId,
        },
      });
    },
    [push],
  );

  const handlePay = useCallback(
    (invoiceId: string) => () => {
      setInvoiceId(invoiceId);
      showPaymentMethodModal(PaymentMethodScreenModal.SELECT_PAYMENT);
      if (isMobile) {
        push({ pageKey: "selectPayment" });
      }
    },
    [setInvoiceId, showPaymentMethodModal, isMobile, push],
  );

  return {
    handleViewInvoice,
    handlePay,
  };
}
```

### 4. Simplified Component: `index.tsx`

**Purpose**: Focus only on UI rendering and composition

**Example from VisitList** (Before: 274 lines → After: ~90 lines):

```typescript:luce-fe/src/containers/visits/visit-list/index.tsx
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

**Example from InvoiceList** (Before: 158 lines → After: ~60 lines):

```typescript:luce-fe/src/containers/profile/invoices/invoice-list/index.tsx
import type { InvoiceFilterStatusEnum } from "@/types/invoice";
import { InvoiceCard } from "@/components/shared/invoices/invoice-card";
import { Button, Skeleton, View } from "@/components/ui";
import { IfElse } from "@/components/shared/if-else";
import { __ } from "@/lib/i18n";
import NoInvoices from "@/assets/images/3d-illus/no-invoices.png";
import { EmptyCard } from "@/components/shared/empty-card";
import { useInvoiceList } from "@/components/hooks/use-invoice";
import { useInvoiceListActions } from "@/components/hooks/use-invoice/use-invoice-list-actions";
import { getPlatform } from "@/lib/utils";
import { PaymentMethodContainer } from "../payment-method";

export function InvoiceList({ type }: { type: InvoiceFilterStatusEnum }) {
  const { invoices, loading, shouldShowLoadMore, onLoadMore } = useInvoiceList(type);
  const { handleViewInvoice, handlePay } = useInvoiceListActions();
  const platform = getPlatform();

  return (
    <>
      <View className="mt-4 flex flex-col gap-4 px-4 pb-8 md:px-0">
        {loading ? (
          <>
            <Skeleton className="h-[120px] w-full" />
          </>
        ) : (
          <IfElse
            if={!!invoices?.length}
            else={
              <EmptyCard
                src={NoInvoices}
                title={__("No invoices yet!")}
                desc={__(
                  "We'll send your invoice at month's end for recurring bookings and the next day for one-time ones.",
                )}
              />
            }
          >
            <View className="flex flex-col gap-4">
              {invoices.map((invoice) => (
                <InvoiceCard
                  onPay={handlePay(invoice.id)}
                  key={invoice.id}
                  {...invoice}
                  onClick={handleViewInvoice(invoice.id)}
                />
              ))}
            </View>
          </IfElse>
        )}

        <IfElse if={shouldShowLoadMore}>
          <Button
            variant="tertiary"
            color="CTA2"
            fullWidth="full"
            loading={loading}
            onClick={onLoadMore}
            children={__("Load more")}
          />
        </IfElse>
      </View>
      {platform === "web" && <PaymentMethodContainer />}
    </>
  );
}
```

## Migration Checklist

When modularizing a component, follow these steps:

1. **Identify responsibilities**

   - [ ] Configuration and constants
   - [ ] Data fetching and state management
   - [ ] User actions and side effects
   - [ ] UI rendering

2. **Extract configuration**

   - [ ] Create or update `lib.ts` file
   - [ ] Move types, constants, and configuration objects
   - [ ] Update imports in component

3. **Extract data fetching**

   - [ ] Create `use-[feature]/index.ts` hook
   - [ ] Move data fetching logic, loading states, and pagination
   - [ ] Return clean interface for component

4. **Extract actions**

   - [ ] Create `use-[feature]/use-[feature]-actions.ts` hook
   - [ ] Move all event handlers and side effects
   - [ ] Use `useCallback` for memoization

5. **Simplify component**

   - [ ] Remove data fetching logic
   - [ ] Remove action handlers
   - [ ] Keep only UI rendering and composition
   - [ ] Use hooks for data and actions

6. **Test and verify**
   - [ ] Test component functionality
   - [ ] Test hooks independently
   - [ ] Verify no regressions

## Code Reduction Summary

**VisitList Example**:

- **Before**: 274 lines with all logic mixed
- **After**: ~90 lines component + ~80 lines hooks = ~170 lines total
- **Net reduction**: ~104 lines + better organization
- **Import reduction**: 36 imports → 12 imports (67% reduction)

**InvoiceList Example**:

- **Before**: 158 lines with all logic mixed
- **After**: ~60 lines component + ~80 lines hooks = ~140 lines total
- **Net reduction**: ~18 lines + better organization

## Benefits Recap

- ✅ **Single responsibility** - Each file has one clear purpose
- ✅ **Reusable hooks** - Hooks can be used in other components
- ✅ **Easier testing** - Test hooks and components separately
- ✅ **Better maintainability** - Changes are localized to specific files
- ✅ **Improved readability** - Components are easier to understand
- ✅ **Consistent pattern** - Same structure across all list components
