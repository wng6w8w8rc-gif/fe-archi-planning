# Codebase Enhancement Recommendations

This document outlines comprehensive recommendations for enhancing the codebase architecture, patterns, and developer experience.

## Table of Contents

1. [State Management & Store Consolidation](#1-state-management--store-consolidation)
2. [GraphQL Request Factory Enhancements](#2-graphql-request-factory-enhancements)
3. [Visit List Component Refactoring](#3-visit-list-component-refactoring)
4. [Loading & Error State Components](#4-loading--error-state-components)
5. [GraphQL Query Optimization](#5-graphql-query-optimization)
6. [Component Composition Improvements](#6-component-composition-improvements)
7. [Type Safety Improvements](#7-type-safety-improvements)
8. [Performance Optimizations](#8-performance-optimizations)
9. [Testing Infrastructure](#9-testing-infrastructure)
10. [Documentation & Developer Experience](#10-documentation--developer-experience)

---

## 1. State Management & Store Consolidation

### Current Issues

- Multiple separate stores for similar visit data (`next7dayVisits`, `next30dayVisits`, `past30dayVisits`, `unscheduledVisits`)
- Separate store for list empty state (`visitList.ts`)
- Duplicate logic across stores
- Hard to maintain consistency across visit lists

### Implementation Example

```typescript
// src/store/visits/useVisitsStore.ts
import { create } from "zustand";
import { createPaginatedFactory } from "@/lib/request-factory";
import type {
  VisitsByFiltersQuery,
  VisitsByFiltersQueryVariables,
} from "@/__generated__/graphql";
import { VisitsByFiltersDocument } from "@/__generated__/graphql";
import type { VisitData } from "@/components/shared/visits/visit-card";
import { mapToVisitData } from "@/components/shared/visits/visit-card";
import { sortVisitByDateAsc } from "@/containers/visits/lib";
import { useVisitListState } from "./visitList";

export type VisitListType =
  | "next7days"
  | "unscheduled"
  | "next30days"
  | "past30days";

type PaginationState = {
  offset: number;
  limit: number;
  total: number;
  enabled: boolean;
};

type VisitListState = {
  data: VisitData[];
  loading: boolean;
  error: string | null;
  pagination: PaginationState;
  isEmpty: boolean;
};

type VisitsStore = {
  lists: Record<VisitListType, VisitListState>;

  // Unified actions
  fetchList: (
    type: VisitListType,
    filters: VisitsByFiltersQueryVariables["filters"],
  ) => Promise<void>;
  fetchMore: (type: VisitListType) => Promise<void>;
  refetchList: (type: VisitListType) => Promise<void>;
  clearList: (type: VisitListType) => void;

  // Batch operations
  refetchAll: () => Promise<void>;
  clearAll: () => void;

  // Internal store references (for backward compatibility during migration)
  _stores: Record<VisitListType, ReturnType<typeof createPaginatedFactory>>;
};

const createListStore = (type: VisitListType) => {
  return createPaginatedFactory<
    VisitsByFiltersQuery,
    VisitData,
    VisitsByFiltersQueryVariables
  >({
    method: "query",
    graphqlDocument: VisitsByFiltersDocument,
    fetchPolicy: "network-only",
    getCountFromResponse: (res) => res.visitsByFilters.count,
    transformFunction(data) {
      return sortVisitByDateAsc(data.visitsByFilters.visits).map(
        mapToVisitData,
      );
    },
    onFetchSuccess(data) {
      const { setListEmpty } = useVisitListState.getState();
      setListEmpty(type, !data.length);
    },
  });
};

export const useVisitsStore = create<VisitsStore>((set, get) => {
  const stores = {
    next7days: createListStore("next7days"),
    unscheduled: createListStore("unscheduled"),
    next30days: createListStore("next30days"),
    past30days: createListStore("past30days"),
  };

  return {
    lists: {
      next7days: {
        data: [],
        loading: false,
        error: null,
        pagination: { offset: 0, limit: 20, total: 0, enabled: true },
        isEmpty: false,
      },
      unscheduled: {
        data: [],
        loading: false,
        error: null,
        pagination: { offset: 0, limit: 20, total: 0, enabled: true },
        isEmpty: false,
      },
      next30days: {
        data: [],
        loading: false,
        error: null,
        pagination: { offset: 0, limit: 20, total: 0, enabled: true },
        isEmpty: false,
      },
      past30days: {
        data: [],
        loading: false,
        error: null,
        pagination: { offset: 0, limit: 20, total: 0, enabled: true },
        isEmpty: false,
      },
    },
    _stores: stores,

    fetchList: async (type, filters) => {
      const store = stores[type];
      const result = await store.getState().fetch({
        requestPayload: { filters },
      });

      set((state) => ({
        lists: {
          ...state.lists,
          [type]: {
            ...state.lists[type],
            data: result.data || [],
            loading: store.getState().loading,
            error: result.error,
            isEmpty: !result.data?.length,
          },
        },
      }));
    },

    fetchMore: async (type) => {
      const store = stores[type];
      await store.getState().fetchMore();
      // Sync state...
    },

    refetchList: async (type) => {
      const store = stores[type];
      await store.getState().refetch();
      // Sync state...
    },

    clearList: (type) => {
      stores[type].getState().clear();
      set((state) => ({
        lists: {
          ...state.lists,
          [type]: {
            data: [],
            loading: false,
            error: null,
            pagination: { offset: 0, limit: 20, total: 0, enabled: true },
            isEmpty: false,
          },
        },
      }));
    },

    refetchAll: async () => {
      await Promise.all([
        stores.next7days.getState().refetch(),
        stores.unscheduled.getState().refetch(),
        stores.next30days.getState().refetch(),
        stores.past30days.getState().refetch(),
      ]);
    },

    clearAll: () => {
      Object.values(stores).forEach((store) => store.getState().clear());
      set((state) => ({
        lists: {
          next7days: { ...state.lists.next7days, data: [], isEmpty: false },
          unscheduled: { ...state.lists.unscheduled, data: [], isEmpty: false },
          next30days: { ...state.lists.next30days, data: [], isEmpty: false },
          past30days: { ...state.lists.past30days, data: [], isEmpty: false },
        },
      }));
    },
  };
});
```

### Benefits

- ✅ Single source of truth for all visit lists
- ✅ Easier to add new list types
- ✅ Batch operations (refetch all, clear all)
- ✅ Better TypeScript inference
- ✅ Reduced boilerplate code
- ✅ Consistent state structure

---

## 2. GraphQL Request Factory Enhancements

### Current Issues

- Error handling is inconsistent (toast shown by default, but `selfHandleError` option exists)
- No retry mechanism
- No request cancellation
- Pagination factory doesn't handle errors with toast (line 330-349)
- Missing optimistic updates support

### Implementation Example

```typescript
// src/lib/request-factory.tsx (Enhanced version)

type RetryPolicy = {
  maxRetries: number;
  retryDelay: number;
  retryableErrors?: string[];
};

type RequestObject<T, U, P> = {
  url?: string;
  headers?: Record<string, string>;
  method?: "query" | "mutation";
  body?: object | undefined;
  transformFunction?: (data: T, request: P | undefined) => U;
  payload?: P;
  graphqlDocument: TypedDocumentNode<T, P>;
  onFetchSuccess?: (data: U) => void;
  fetchPolicy?: FetchPolicy;
  limit?: number;
  getCountFromResponse?: (data: T) => number;
  hasUpload?: boolean;
  // New enhancements
  retryPolicy?: RetryPolicy;
  optimisticUpdate?: (variables: P) => U;
  onError?: (error: ApolloError, variables: P) => void;
  errorMessage?: string | ((error: ApolloError) => string);
  abortSignal?: AbortSignal;
};

// Enhanced fetch with retry logic
async function fetchWithRetry<T, P extends OperationVariables>(
  requestFn: () => Promise<T>,
  retryPolicy?: RetryPolicy,
  onError?: (error: Error, attempt: number) => void,
): Promise<T> {
  if (!retryPolicy) {
    return requestFn();
  }

  let lastError: Error;
  for (let attempt = 0; attempt <= retryPolicy.maxRetries; attempt++) {
    try {
      return await requestFn();
    } catch (error) {
      lastError = error as Error;
      onError?.(lastError, attempt);

      // Check if error is retryable
      if (retryPolicy.retryableErrors) {
        const isRetryable = retryPolicy.retryableErrors.some((pattern) =>
          lastError.message.includes(pattern),
        );
        if (!isRetryable) {
          throw lastError;
        }
      }

      // Don't retry on last attempt
      if (attempt === retryPolicy.maxRetries) {
        throw lastError;
      }

      // Exponential backoff
      const delay = retryPolicy.retryDelay * Math.pow(2, attempt);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError!;
}

// Enhanced createRequestFactory
export function createRequestFactory<T, U, P extends OperationVariables>(
  requestObject: RequestObject<T, U, P>,
  initialState?: U,
) {
  return create<RequestStore<U, P>>((set, get) => ({
    data: initialState,
    requestPayload: undefined,
    loading: false,
    error: null,
    fetch: async (
      opt?: { requestPayload?: P },
      handleOpt?: { selfHandleError?: boolean },
    ) => {
      const variables = opt?.requestPayload || requestObject.payload;

      // Optimistic update
      if (
        requestObject.optimisticUpdate &&
        requestObject.method === "mutation"
      ) {
        const optimisticData = requestObject.optimisticUpdate(variables);
        set({ data: optimisticData, loading: true, error: null });
      } else {
        set({ loading: true, error: null, requestPayload: variables });
      }

      try {
        const requestFn = async () => {
          // Check for cancellation
          if (requestObject.abortSignal?.aborted) {
            throw new Error("Request cancelled");
          }

          let serverData: T | undefined;
          if (requestObject.method === "query") {
            const res = await ClientInstance.query<T, P>({
              query: requestObject.graphqlDocument,
              variables,
              fetchPolicy: requestObject.fetchPolicy || "cache-first",
              context: {
                fetchOptions: {
                  signal: requestObject.abortSignal,
                },
              },
            });
            serverData = res.data;
          } else if (requestObject.method === "mutation") {
            const res = await ClientInstance.mutate<T, P>({
              mutation: requestObject.graphqlDocument,
              variables,
              context: {
                hasUpload: requestObject.hasUpload,
                fetchOptions: {
                  signal: requestObject.abortSignal,
                },
              },
              fetchPolicy: (requestObject.fetchPolicy ||
                "no-cache") as MutationFetchPolicy,
            });
            serverData = res.data as T;
          } else {
            throw new Error(
              `request method: ${requestObject.method} is not defined in createRequestFactory`,
            );
          }

          return serverData;
        };

        const serverData = await fetchWithRetry(
          requestFn,
          requestObject.retryPolicy,
          (error, attempt) => {
            console.log(
              `Retry attempt ${attempt + 1} for ${requestObject.method}`,
            );
          },
        );

        if (!requestObject.transformFunction) {
          set({
            data: serverData as unknown as U,
            loading: false,
            error: null,
          });
          return { data: serverData as unknown as U, error: null };
        }

        const transformedData = requestObject.transformFunction(
          serverData,
          variables,
        );
        set({
          data: transformedData,
          loading: false,
          error: null,
        });

        const callback = requestObject.onFetchSuccess;
        if (callback) {
          callback(transformedData);
        }
        return { data: transformedData, error: null };
      } catch (err) {
        const error = err as ApolloError;

        // Custom error handler
        if (requestObject.onError) {
          requestObject.onError(error, variables);
        }

        // Get error message
        const errorMsg = requestObject.errorMessage
          ? typeof requestObject.errorMessage === "function"
            ? requestObject.errorMessage(error)
            : requestObject.errorMessage
          : error.message || "unknown error";

        if (!handleOpt?.selfHandleError) {
          showToast({
            title: errorMsg,
            type: "error",
          });
        }

        collectError(err, {
          extra: {
            requestMethod: requestObject.method,
            variables,
            graphqlDocument: requestObject.graphqlDocument,
          },
        });

        set({
          ...get(),
          error: errorMsg,
          loading: false,
        });
        return {
          data: null,
          error: errorMsg,
          graphqlErrors: error.graphQLErrors,
        };
      }
    },
    // ... rest of the implementation
  }));
}

// Fix for paginated factory - add error toast handling
export function createPaginatedFactory<
  T,
  U extends { id: string | number },
  P extends OperationVariables,
>(requestObject: RequestObject<T, U[], P>, initialState?: U[]) {
  return create<RequestStorePaginated<U, P>>((set, get) => ({
    // ... existing state
    fetch: async (
      opt?: { requestPayload: P | undefined },
      handleOpt?: { selfHandleError?: boolean }, // Add this parameter
    ) => {
      // ... existing fetch logic
      try {
        // ... existing try block
      } catch (err) {
        const error = err as ApolloError;

        // FIX: Add error toast handling (consistent with regular factory)
        if (!handleOpt?.selfHandleError) {
          const errorMsg = requestObject.errorMessage
            ? typeof requestObject.errorMessage === "function"
              ? requestObject.errorMessage(error)
              : requestObject.errorMessage
            : error.message || "unknown error";

          showToast({
            title: errorMsg,
            type: "error",
          });
        }

        set({
          ...get(),
          error: (error as Error).message || "unknown error",
          loading: false,
        });

        collectError(err, {
          extra: {
            requestMethod: requestObject.method,
            variables,
            graphqlDocument: requestObject.graphqlDocument,
          },
        });

        return {
          data: null,
          error: (error as Error).message || "unknown error",
          graphqlErrors: error.graphQLErrors,
        };
      }
    },
    // ... rest of the implementation
  }));
}
```

### Usage Example

```typescript
// Example: Store with retry policy and optimistic update
export const useRateVisitStore = createRequestFactory<
  RateVisitMutation,
  RateVisitMutation,
  RateVisitMutationVariables
>({
  method: "mutation",
  graphqlDocument: RateVisitDocument,
  hasUpload: true,
  retryPolicy: {
    maxRetries: 3,
    retryDelay: 1000,
    retryableErrors: ["Network error", "Timeout"],
  },
  optimisticUpdate: (variables) => {
    // Return optimistic data structure
    return {
      rateVisit: {
        result: true,
        // ... optimistic response
      },
    } as RateVisitMutation;
  },
  errorMessage: (error) => {
    if (error.networkError) {
      return "Network error. Please check your connection.";
    }
    return "Failed to rate visit. Please try again.";
  },
});
```

---

## 3. Visit List Component Refactoring

### Current Issues

- `VisitList` component has too many responsibilities
- Multiple `useCallback` hooks that could be extracted
- Direct store access mixed with component logic
- Hard to test

### Implementation Example

```typescript
// src/components/hooks/visits/use-visit-list.ts
import { useEffect, useMemo, useCallback } from "react";
import { useShallow } from "zustand/react/shallow";
import { useAuthState } from "@/store/auth";
import type { VisitListType } from "@/store/visits/visitList";
import { listType } from "@/containers/visits/visit-list";
import type { VisitsByFiltersQueryVariables } from "@/__generated__/graphql";

export const useVisitList = (type: VisitListType) => {
  const user = useAuthState((state) => state.data.userInfo);
  const { title, subtitle, icon, hasLoadMore, initialFilters, useStore } =
    listType[type];

  const {
    fetch: getVisits,
    data: visits,
    loading,
    error,
    fetchMore,
    pagination,
  } = useStore(
    useShallow((state) => ({
      fetch: state.fetch,
      data: state.data,
      loading: state.loading,
      error: state.error,
      fetchMore: state.fetchMore,
      pagination: state.pagination,
    })),
  );

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
    error,
    title,
    subtitle,
    icon,
    shouldShowLoadMore,
    onLoadMore,
  };
};
```

```typescript
// src/components/hooks/visits/use-visit-actions.ts
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

export const useVisitActions = () => {
  const { push } = useRoute();
  const handleViewDetail = useVisitDetailStore((state) => state.openVisitModal);
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
};
```

```typescript
// src/containers/visits/visit-list/index.tsx (Refactored)
import { View } from "@/components/ui";
import { ListHeading } from "@/components/shared/list-header";
import { Button, Skeleton } from "@/components/ui";
import { VisitCard } from "@/components/shared/visits/visit-card";
import { IfElse } from "@/components/shared/if-else";
import { EmptyCard } from "@/components/shared/empty-card";
import { __ } from "@/lib/i18n";
import type { VisitListType } from "@/store/visits/visitList";
import { useVisitList } from "@/components/hooks/visits/use-visit-list";
import { useVisitActions } from "@/components/hooks/visits/use-visit-actions";
import { DataState } from "@/components/shared/data-states";
import NoVisitsBike from "@/assets/images/3d-illus/no-visit-bike.png";
import NoVisitsVaccum from "@/assets/images/3d-illus/no-visits-vaccum.png";

export function VisitList({ type }: { type: VisitListType }) {
  const {
    visits,
    loading,
    error,
    title,
    subtitle,
    icon,
    shouldShowLoadMore,
    onLoadMore,
  } = useVisitList(type);

  const {
    handleViewDetail,
    handleRebook,
    handleSetSchedule,
    handleRateVisit,
    handleReportVisit,
  } = useVisitActions();

  const emptyStateProps = useMemo(() => {
    if (type === "next30days" || type === "past30days") {
      return {
        src: NoVisitsVaccum,
        title: __("You don't have any history visits"),
        desc: __(
          "Your completed bookings will appear here once the service is done.",
        ),
      };
    }
    return {
      src: NoVisitsBike,
      title: __("You don't have any upcoming visits"),
      desc: __("Your scheduled services will appear here once booked."),
    };
  }, [type]);

  return (
    <View className="flex flex-col gap-4">
      <ListHeading
        icon={icon}
        title={__(title)}
        subtitle={__(subtitle || "")}
      />

      {loading ? (
        <DataState.Loading />
      ) : error ? (
        <DataState.Error error={error} />
      ) : !visits?.length ? (
        <DataState.Empty {...emptyStateProps} />
      ) : (
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
      )}

      <IfElse if={shouldShowLoadMore}>
        <Button
          variant="tertiary"
          className="w-full"
          loading={loading}
          onClick={onLoadMore}
          iconName="arrowCounterClockwise"
          iconAlignment="end"
        >
          {__("Show More")}
        </Button>
      </IfElse>
    </View>
  );
}
```

---

## 4. Loading & Error State Components

### Implementation Example

```typescript
// src/components/shared/data-states/index.tsx
import { View } from "@/components/ui";
import { Skeleton } from "@/components/ui";
import { Button } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { EmptyCard } from "@/components/shared/empty-card";
import type { ImageSourcePropType } from "react-native";
import { __ } from "@/lib/i18n";

type DataStateProps = {
  count?: number;
  error?: string | null;
  onRetry?: () => void;
  src?: ImageSourcePropType;
  title?: string;
  desc?: string;
};

export const DataState = {
  Loading: ({ count = 3 }: { count?: number }) => (
    <>
      {Array.from({ length: count }).map((_, i) => (
        <Skeleton key={i} className="h-[200px] w-full" />
      ))}
    </>
  ),

  Error: ({
    error,
    onRetry
  }: {
    error: string | null;
    onRetry?: () => void;
  }) => (
    <View className="flex flex-col items-center gap-4 p-8">
      <Typography variant="body-md" color="error">
        {error || __("An error occurred")}
      </Typography>
      {onRetry && (
        <Button variant="secondary" onClick={onRetry}>
          {__("Retry")}
        </Button>
      )}
    </View>
  ),

  Empty: ({
    src,
    title,
    desc
  }: {
    src: ImageSourcePropType;
    title: string;
    desc: string;
  }) => (
    <EmptyCard src={src} title={title} desc={desc} />
  ),

  // Combined component for common use case
  State: ({
    loading,
    error,
    empty,
    onRetry,
    children,
  }: {
    loading: boolean;
    error: string | null;
    empty: boolean;
    onRetry?: () => void;
    children: React.ReactNode;
  }) => {
    if (loading) return <DataState.Loading />;
    if (error) return <DataState.Error error={error} onRetry={onRetry} />;
    if (empty) return <DataState.Empty {...empty} />;
    return <>{children}</>;
  },
};
```

### Usage Example

```typescript
// In any component
<DataState.State
  loading={loading}
  error={error}
  empty={!data?.length}
  onRetry={() => refetch()}
  empty={{
    src: NoDataImage,
    title: __("No data available"),
    desc: __("Please try again later"),
  }}
>
  {/* Your content */}
  {data.map((item) => (
    <ItemCard key={item.id} item={item} />
  ))}
</DataState.State>
```

---

## 5. GraphQL Query Optimization

### Implementation Example

```typescript
// src/lib/graphql/cache-policies.ts
export const VISIT_QUERY_POLICIES = {
  // For real-time critical data (next 7 days)
  CRITICAL: "cache-and-network" as const,
  // For frequently changing data
  FRESH: "network-only" as const,
  // For stable reference data
  STABLE: "cache-first" as const,
};

// src/lib/graphql/cache.ts (Enhanced)
import { InMemoryCache, type FieldPolicy } from "@apollo/client";
import possibleTypes from "@/__generated__/possibleTypes.json";

export const createFilteredEntityTypePolicy = (
  entityName: string,
  keyArgs: string[],
): FieldPolicy => {
  return {
    keyArgs,
    merge(existing = [], incoming, { args }) {
      // Reset on new filter
      if (args?.filters?.offset === 0) {
        return incoming;
      }
      // Append for pagination
      return [...existing, ...incoming];
    },
  };
};

export const cache = new InMemoryCache({
  possibleTypes,
  typePolicies: {
    FreeTimeslot: {
      keyFields: ["id", "startAt", "endAt"],
    },
    Query: {
      fields: {
        clientNotificationByFilters: createFilteredEntityTypePolicy(
          "notifications",
          ["filters", ["isRead"]],
        ),
        visitsByFilters: createFilteredEntityTypePolicy("visits", [
          "filters",
          ["clientId", "from", "to", "status"],
        ]),
      },
    },
  },
});
```

### Usage Example

```typescript
// Update stores to use consistent policies
export const useNext7DayVisitsStore = createPaginatedFactory({
  method: "query",
  graphqlDocument: VisitsByFiltersDocument,
  fetchPolicy: VISIT_QUERY_POLICIES.CRITICAL, // Instead of "network-only"
  // ... rest of config
});
```

---

## 6. Component Composition Improvements

### Implementation Example

```typescript
// src/components/shared/visits/visit-card/index.tsx (Compound Component Pattern)
import type { ReactNode } from "react";
import { Card, CardContent, View } from "@/components/ui";
import type { VisitData } from "./types";
import { VisitCardHeader } from "./visit-card-header";
import { VisitCardInfo } from "./visit-card-info";
import { VisitCardActions } from "./visit-card-actions";

type VisitCardContextValue = {
  visitData: VisitData;
  handlers: VisitActionHandlers;
};

const VisitCardContext = createContext<VisitCardContextValue | null>(null);

const useVisitCardContext = () => {
  const context = useContext(VisitCardContext);
  if (!context) {
    throw new Error("VisitCard sub-components must be used within VisitCard.Root");
  }
  return context;
};

export const VisitCard = {
  Root: ({
    visitData,
    handlers,
    children
  }: {
    visitData: VisitData;
    handlers: VisitActionHandlers;
    children: ReactNode;
  }) => {
    return (
      <VisitCardContext.Provider value={{ visitData, handlers }}>
        <Card>
          <CardContent>{children}</CardContent>
        </Card>
      </VisitCardContext.Provider>
    );
  },

  Header: () => {
    const { visitData } = useVisitCardContext();
    return <VisitCardHeader visitData={visitData} />;
  },

  Info: () => {
    const { visitData } = useVisitCardContext();
    return <VisitCardInfo visitData={visitData} />;
  },

  Actions: () => {
    const { visitData, handlers } = useVisitCardContext();
    return <VisitCardActions visitData={visitData} handlers={handlers} />;
  },
};

// Usage
<VisitCard.Root visitData={visit} handlers={handlers}>
  <VisitCard.Header />
  <VisitCard.Info />
  <VisitCard.Actions />
</VisitCard.Root>
```

---

## 7. Type Safety Improvements

### Implementation Example

```typescript
// src/types/visits.ts
import { VisitStatusEnum } from "@/__generated__/graphql";
import type { VisitData } from "@/components/shared/visits/visit-card";

export type VisitWithStatus<T extends VisitStatusEnum> = VisitData & {
  status: T;
} & (T extends VisitStatusEnum.Completed
    ? {
        rating: number;
        completedAt: Date;
        canRate: boolean;
      }
    : T extends VisitStatusEnum.PendingClientSchedule
      ? {
          canSchedule: true;
          scheduleDeadline: Date;
        }
      : T extends VisitStatusEnum.Started
        ? {
            startedAt: Date;
            estimatedCompletion: Date;
          }
        : {});

// Type guards
export const isCompletedVisit = (
  visit: VisitData,
): visit is VisitWithStatus<VisitStatusEnum.Completed> => {
  return visit.status === VisitStatusEnum.Completed;
};

export const isSchedulableVisit = (
  visit: VisitData,
): visit is VisitWithStatus<VisitStatusEnum.PendingClientSchedule> => {
  return visit.status === VisitStatusEnum.PendingClientSchedule;
};

// Usage
const handleVisitAction = (visit: VisitData) => {
  if (isCompletedVisit(visit)) {
    // TypeScript knows visit has rating and completedAt
    console.log(visit.rating, visit.completedAt);
  }

  if (isSchedulableVisit(visit)) {
    // TypeScript knows visit has canSchedule and scheduleDeadline
    console.log(visit.canSchedule, visit.scheduleDeadline);
  }
};
```

---

## 8. Performance Optimizations

### Implementation Examples

```typescript
// 1. Memoize expensive computations
// src/containers/visits/visit-list/index.tsx
const sortedVisits = useMemo(
  () => sortVisitByDateAsc(visits),
  [visits]
);

// 2. Virtualize long lists
// src/components/shared/visits/visit-list-virtualized.tsx
import { FlashList } from "@shopify/flash-list";

export const VirtualizedVisitList = ({ visits }: { visits: VisitData[] }) => {
  return (
    <FlashList
      data={visits}
      estimatedItemSize={200}
      renderItem={({ item }) => <VisitCard visitData={item} />}
      keyExtractor={(item) => item.id}
    />
  );
};

// 3. Debounce filter operations
// src/components/hooks/visits/use-visit-filters.ts
import { useDebounceValue } from "@/components/hooks/use-debounce-value";

export const useVisitFilters = () => {
  const [filters, setFilters] = useState<VisitFilters>({});
  const debouncedFilters = useDebounceValue(filters, 300);

  useEffect(() => {
    // Fetch with debounced filters
    fetchVisits(debouncedFilters);
  }, [debouncedFilters]);

  return { filters, setFilters };
};
```

---

## 9. Testing Infrastructure

### Implementation Example

```typescript
// src/lib/request-factory/__tests__/request-factory.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { createRequestFactory } from "../request-factory";
import { ClientInstance } from "@/lib/graphql/apollo";

vi.mock("@/lib/graphql/apollo");

describe("createRequestFactory", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("should handle retry logic", async () => {
    const mockQuery = vi.fn();
    (ClientInstance.query as any) = mockQuery;

    // First two calls fail, third succeeds
    mockQuery
      .mockRejectedValueOnce(new Error("Network error"))
      .mockRejectedValueOnce(new Error("Network error"))
      .mockResolvedValueOnce({ data: { items: [] } });

    const store = createRequestFactory({
      method: "query",
      graphqlDocument: {} as any,
      retryPolicy: {
        maxRetries: 3,
        retryDelay: 100,
        retryableErrors: ["Network error"],
      },
    });

    await store.getState().fetch();

    expect(mockQuery).toHaveBeenCalledTimes(3);
  });

  it("should cancel pending requests", async () => {
    const abortController = new AbortController();
    const store = createRequestFactory({
      method: "query",
      graphqlDocument: {} as any,
      abortSignal: abortController.signal,
    });

    const fetchPromise = store.getState().fetch();
    abortController.abort();

    await expect(fetchPromise).rejects.toThrow("Request cancelled");
  });
});

// src/components/hooks/visits/__tests__/use-visit-list.test.ts
import { renderHook, waitFor } from "@testing-library/react";
import { useVisitList } from "../use-visit-list";

describe("useVisitList", () => {
  it("should fetch visits on mount", async () => {
    const { result } = renderHook(() => useVisitList("next7days"));

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.visits).toBeDefined();
  });
});
```

---

## 10. Documentation & Developer Experience

### Implementation Example

````typescript
// src/lib/request-factory.tsx (With JSDoc)
/**
 * Creates a Zustand store for GraphQL requests with built-in loading/error states,
 * retry logic, optimistic updates, and request cancellation.
 *
 * @template T - Type of the data returned by the GraphQL request
 * @template U - Type of the data stored in the state (after transformation)
 * @template P - Request payload/variables type
 *
 * @param requestObject - Configuration object for the request
 * @param requestObject.method - GraphQL operation type ("query" | "mutation")
 * @param requestObject.graphqlDocument - Typed GraphQL document
 * @param requestObject.transformFunction - Optional function to transform response data
 * @param requestObject.retryPolicy - Optional retry configuration
 * @param requestObject.optimisticUpdate - Optional optimistic update function for mutations
 * @param initialState - Optional initial state value
 *
 * @returns A Zustand store with `fetch`, `refetch`, `clear` methods and `data`, `loading`, `error` state
 *
 * @example
 * ```ts
 * // Basic query
 * const useMyStore = createRequestFactory({
 *   method: "query",
 *   graphqlDocument: MyQueryDocument,
 *   transformFunction: (data) => data.items,
 * });
 *
 * // In component
 * const { data, loading, fetch } = useMyStore();
 * useEffect(() => {
 *   fetch({ requestPayload: { id: "123" } });
 * }, []);
 * ```
 *
 * @example
 * ```ts
 * // Mutation with retry and optimistic update
 * const useCreateStore = createRequestFactory({
 *   method: "mutation",
 *   graphqlDocument: CreateItemDocument,
 *   retryPolicy: {
 *     maxRetries: 3,
 *     retryDelay: 1000,
 *   },
 *   optimisticUpdate: (variables) => ({
 *     createItem: { id: "temp", ...variables.input },
 *   }),
 * });
 * ```
 */
export function createRequestFactory<T, U, P extends OperationVariables>(
  requestObject: RequestObject<T, U, P>,
  initialState?: U,
) {
  // ... implementation
}
````

---

## Priority Implementation Order

### High Priority (Week 1-2)

1. ✅ Fix paginated factory error handling (toast consistency)
2. ✅ Create `useVisitList` and `useVisitActions` hooks
3. ✅ Create reusable `DataState` components

### Medium Priority (Week 3-4)

4. ✅ Consolidate visit stores into unified store
5. ✅ Add retry mechanism to request factory
6. ✅ Improve GraphQL cache policies

### Low Priority (Week 5+)

7. ✅ Compound component pattern
8. ✅ Virtualization for long lists
9. ✅ Enhanced type guards
10. ✅ Testing infrastructure

---

## Migration Strategy

### Phase 1: Non-Breaking Enhancements

- Add new hooks alongside existing code
- Create new components without removing old ones
- Add optional parameters to factories

### Phase 2: Gradual Migration

- Update one component at a time to use new patterns
- Keep old patterns working during transition
- Add feature flags if needed

### Phase 3: Cleanup

- Remove deprecated code
- Update all components to use new patterns
- Update documentation

---

## Notes

- All changes should maintain backward compatibility during migration
- Test thoroughly before removing old patterns
- Consider feature flags for risky changes
- Document breaking changes clearly
