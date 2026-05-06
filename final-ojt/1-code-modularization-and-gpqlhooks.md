# Visits Query Refactoring

## Summary

Refactored visits queries from Zustand stores to Apollo Client hooks with separate paginated and non-paginated query hooks.

## New Files

### GraphQL Hooks (`src/gqlhooks/visit/`)

- `useGetPaginatedVisitsQuery.tsx` - Paginated queries using `visitsByFilters`
- `useGetAllVisitsQuery.tsx` - Non-paginated queries using `visitsByClient`

### Custom Hooks (`src/components/hooks/use-visit/`)

- `use-visit.tsx` - Main hook managing next7days, next30days, and unscheduled queries
- `use-visit-action.tsx` - Visit actions (view detail, rebook, rate, report)
- `use-visit-previous-booking.tsx` - Previous booking logic
- `use-visit-rebook-action.tsx` - Rebook action handler

### Additional GraphQL Hooks

- `src/gqlhooks/package/useGetClientPackageDetailsQuery.tsx` - Package details query hook
- `src/gqlhooks/review/useGetClientReviewsQuery.tsx` - Client reviews query hook

## Modified Files

### `src/lib/graphql/apollo.ts`

- Added cache policy for `visitsByFilters` with `keyArgs` for `clientId`, `from`, `to`, `status`, `limit`
- Enables proper cache merging for paginated visit queries

### `src/containers/visits/visit-list/index.tsx`

- Changed from accepting `type` prop with Zustand store usage to accepting `query` prop
- Simplified component from 179 to 49 lines
- Now receives query result as prop instead of managing state internally
- Uses `useVisitAction` hook for visit actions

### `src/containers/visits/history/index.tsx`

- Replaced `usePast30DayVisitsStore` Zustand store with `useGetPaginatedVisitsQuery` hook
- Removed prop drilling for `refreshing` and `setRefreshing`
- Direct GraphQL query instead of Zustand store abstraction

### `src/containers/visits/upcoming/index.tsx`

- Replaced multiple Zustand stores (`useNext7DayVisitsStore`, `useNext30DayVisitsStore`, `useUnscheduledVisitsStore`) with `useVisit` hook
- Centralized query management in custom hook
- Removed prop drilling for refresh state

### `src/containers/previous-booking/index.tsx`

- Replaced complex logic with multiple stores using `useVisitPreviousBooking` and `useVisitRebookAction` hooks
- Simplified component, business logic moved to hooks
- Reduced from 72 to 27 lines

### `src/app/(tabs)/visits/upcoming.tsx`

- Removed `refreshing` and `setRefreshing` props
- Simplified component interface, hook manages refresh internally

### `src/components/shared/visits/visit-card/index.tsx`

- Added missing dependencies to `useMemo` hook for visit buttons
- Fixed dependency array to include all callback functions

## Key Changes

1. **Separation of Concerns**: Paginated vs non-paginated queries use different hooks
2. **Cache Management**: Added Apollo cache policy for proper pagination merging
3. **Component Simplification**: Components now receive query results as props
4. **Hook-based Architecture**: Business logic moved to reusable hooks
5. **Type Safety**: Flexible types supporting both paginated and non-paginated queries
6. **Reduced Coupling**: Removed Zustand store dependencies from components
7. **Better Reusability**: GraphQL hooks can be reused across different components
