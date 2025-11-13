# Visit Token vs Auth Pattern Consolidation

## Summary

This refactoring consolidates the duplicate visit operation stores that handle both authenticated and token-based access. Currently, there are separate stores for each operation type (visit detail, rate, reschedule, cancel) that differ only in authentication method.

### Files Affected

**New Files:**

- `store/visits/visitStore.ts` - Unified store with factory function

**Removed Files (if using factory approach):**

- `store/visits/visitTokenStore.ts` (consolidated into `visitStore.ts`)
- `store/visits/useRateVisitWithToken.ts` (consolidated into `useRateVisit.ts`)
- `store/visits/useClientRescheduleWithToken.ts` (consolidated into `useClientRescheduleVisit.ts`)
- `store/visits/useCancelVisitWithToken.ts` (consolidated into `useClientCancelVisit.ts`)

**Modified Files:**

- `store/visits/visitStore.ts` - Add token support
- `store/visits/useRateVisit.ts` - Add token support
- `store/visits/useClientRescheduleVisit.ts` - Add token support
- `store/visits/useClientCancelVisit.ts` - Add token support
- All files using the token-based stores

### Key Changes

- **Unified store pattern** that handles both auth and token variants
- **Configuration-based approach** with authentication type
- **Consistent behavior** across both authentication methods
- **Reduced code duplication** for visit operations

### Benefits

- **~50% code reduction** (8 files → 4 unified stores)
- **DRY principle** - single source of truth for visit operations
- **Easier maintenance** - fix bugs in one place
- **Consistent behavior** across auth and token flows
- **Type safety** maintained with configuration

## Implementation Realization

### Before: Duplicate Stores for Auth and Token

**Visit Detail** - 2 separate stores

```typescript:luce-fe/src/store/visits/visitStore.ts
import { createRequestFactory } from "@/lib/request-factory";
import {
  VisitDocument,
  type VisitQuery,
  type VisitQueryVariables,
} from "@/__generated__/graphql";

export const useVisitStore = createRequestFactory<
  VisitQuery,
  VisitDetailData,
  VisitQueryVariables
>({
  method: "query",
  graphqlDocument: VisitDocument,
  // ... configuration
});
```

```typescript:luce-fe/src/store/visits/visitTokenStore.ts
// Identical pattern but uses VisitTokenDocument
// Different GraphQL document, same structure
```

**Rate Visit** - 2 separate stores

```typescript:luce-fe/src/store/visits/useRateVisit.ts
import { createRequestFactory } from "@/lib/request-factory";
import {
  RateVisitDocument,
  type RateVisitMutation,
  type RateVisitMutationVariables,
} from "@/__generated__/graphql";

export const useRateVisitStore = createRequestFactory<
  RateVisitMutation,
  RateVisitMutation,
  RateVisitMutationVariables
>({
  method: "mutation",
  graphqlDocument: RateVisitDocument,
  hasUpload: true,
});
```

```typescript:luce-fe/src/store/visits/useRateVisitWithToken.ts
import { createRequestFactory } from "@/lib/request-factory";
import {
  RateVisitWithTokenDocument,
  type RateVisitWithTokenMutation,
  type RateVisitWithTokenMutationVariables,
} from "@/__generated__/graphql";

export const useRateVisitWithTokenStore = createRequestFactory<
  RateVisitWithTokenMutation,
  RateVisitWithTokenMutation,
  RateVisitWithTokenMutationVariables
>({
  method: "mutation",
  graphqlDocument: RateVisitWithTokenDocument,
  hasUpload: true,
});
```

**Reschedule Visit** - 2 separate stores
**Cancel Visit** - 2 separate stores

### After: Unified Store Pattern

```typescript:luce-fe/src/store/visits/visitStore.ts
import { useFragment } from "@/__generated__";
import { createRequestFactory } from "@/lib/request-factory";
import {
  VisitDocument,
  VisitByClientWithTokenDocument,
  VisitClientFragmentFragmentDoc,
  type VisitQuery,
  type VisitQueryVariables,
  type VisitByClientWithTokenQuery,
  type VisitByClientWithTokenQueryVariables,
} from "@/__generated__/graphql";
import type { VisitDetailData } from "@/components/shared/visits/visit-detail";
import { mapToVisitDetailData } from "@/components/shared/visits/visit-detail";

type VisitStoreConfig = {
  useToken?: boolean;
};

export function createVisitStore(config: VisitStoreConfig = {}) {
  // Conditional document selection: token or auth
  const document = config.useToken
    ? VisitByClientWithTokenDocument
    : VisitDocument;

  return createRequestFactory<
    VisitByClientWithTokenQuery | VisitQuery,
    VisitDetailData,
    VisitByClientWithTokenQueryVariables | VisitQueryVariables
  >({
    method: "query",
    graphqlDocument: document,
    transformFunction(data: any, request?: any) {
      if (config.useToken) {
        // Token-based transform: uses visitByClientWithToken and passes token
        if (!data.visitByClientWithToken) {
          throw new Error("No visit found");
        }
        const visit = useFragment(
          VisitClientFragmentFragmentDoc,
          data.visitByClientWithToken,
        );
        return mapToVisitDetailData(visit, request?.token);
      }

      // Auth-based transform: uses visit
      const visit = useFragment(VisitClientFragmentFragmentDoc, data.visit);
      return mapToVisitDetailData(visit);
    },
  });
}

// Export both variants for backward compatibility
// Auth-based store (useToken: false)
export const useVisitStore = createVisitStore({ useToken: false });
// Token-based store (useToken: true)
export const useVisitTokenStore = createVisitStore({ useToken: true });
```

**Updated Usage**

```typescript
// Before
const { data } = useVisitStore();
const { data: tokenData } = useVisitTokenStore();

// After
// Auth-based usage
const { data, loading, fetch } = useVisitStore();
fetch({ requestPayload: { id: visitId } });

// Token-based usage
const { data: tokenData, loading: tokenLoading, fetch: tokenFetch } = useVisitTokenStore();
tokenFetch({ requestPayload: { token } });
// Both stores are created from the same factory function
```

### Usage Examples

**Example 1: Using Visit Store (Both Auth and Token)**

```typescript
import { useEffect } from "react";
import { View, Skeleton } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useVisitStore } from "@/store/visits/visitStore";
import { useVisitTokenStore } from "@/store/visits/visitStore";
import { useShallow } from "zustand/react/shallow";

// Auth-based usage
function VisitDetail({ visitId }: { visitId: string }) {
  const { data: visit, loading, fetch } = useVisitStore(useShallow((state) => state));

  useEffect(() => {
    fetch({
      requestPayload: { id: visitId },
    });
  }, [visitId, fetch]);

  if (loading) {
    return (
      <>
        <Skeleton className="h-[200px] w-full" />
      </>
    );
  }
  if (!visit) return null;

  return (
    <View>
      <Typography variant="h1">{visit.serviceName}</Typography>
      <Typography variant="body-md">Date: {visit.date}</Typography>
    </View>
  );
}

// Token-based usage
function VisitDetailWithToken({ token }: { token: string }) {
  const { data: visit, loading, fetch } = useVisitTokenStore(useShallow((state) => state));

  useEffect(() => {
    fetch({
      requestPayload: { token },
    });
  }, [token, fetch]);

  if (loading) {
    return (
      <>
        <Skeleton className="h-[200px] w-full" />
      </>
    );
  }
  if (!visit) return null;

  return (
    <View>
      <Typography variant="h1">{visit.serviceName}</Typography>
      <Typography variant="body-md">Date: {visit.date}</Typography>
    </View>
  );
}
```

**Example 2: Unified Component Using Both Stores**

```typescript
import { useEffect } from "react";
import { View, Skeleton } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useVisitStore } from "@/store/visits/visitStore";
import { useVisitTokenStore } from "@/store/visits/visitStore";
import { useShallow } from "zustand/react/shallow";

function VisitDetail({ visitId, token }: { visitId?: string; token?: string }) {
  // Use appropriate store based on available parameter
  const authStore = useVisitStore(useShallow((state) => state));
  const tokenStore = useVisitTokenStore(useShallow((state) => state));

  const { data: visit, loading, fetch } = token ? tokenStore : authStore;

  useEffect(() => {
    if (token) {
      // Token-based fetch
      fetch({ requestPayload: { token } });
    } else if (visitId) {
      // Auth-based fetch
      fetch({ requestPayload: { id: visitId } });
    }
  }, [visitId, token, fetch]);

  if (loading) {
    return (
      <>
        <Skeleton className="h-[200px] w-full" />
      </>
    );
  }
  if (!visit) return null;

  return (
    <View>
      <Typography variant="h1">{visit.serviceName}</Typography>
      <Typography variant="body-md">Date: {visit.date}</Typography>
    </View>
  );
}

// Usage examples:
// <VisitDetail visitId="123" /> // Uses auth store
// <VisitDetail token="abc-token" /> // Uses token store
```

**Example 3: Rating a Visit (Both Auth and Token)**

```typescript
import { View, Button } from "@/components/ui";
import { useRateVisitStore } from "@/store/visits/useRateVisit";
import { useRateVisitWithTokenStore } from "@/store/visits/useRateVisit";
import { showToast } from "@/components/ui/toast/show-toast";
import { __ } from "@/lib/i18n";

function RateVisitButton({ visitId, token }: { visitId?: string; token?: string }) {
  // After consolidation: Both stores are created from the same factory
  const rateVisit = useRateVisitStore((state) => state.fetch);
  const rateVisitWithToken = useRateVisitWithTokenStore((state) => state.fetch);

  const handleRate = async (rating: number) => {
    try {
      if (token) {
        // Token-based rating
        await rateVisitWithToken({
          requestPayload: { token, rating },
          selfHandleError: true,
        });
      } else if (visitId) {
        // Auth-based rating
        await rateVisit({
          requestPayload: { visitId, rating },
          selfHandleError: true,
        });
      }
      showToast({
        type: "success",
        title: __("Thank you for your feedback"),
      });
    } catch (error) {
      // Error handled by selfHandleError
    }
  };

  return (
    <View className="flex flex-row gap-2">
      {[1, 2, 3, 4, 5].map((star) => (
        <Button key={star} variant="tertiary" onClick={() => handleRate(star)} children="⭐" />
      ))}
    </View>
  );
}

// Usage examples:
// <RateVisitButton visitId="123" /> // Uses auth store
// <RateVisitButton token="abc-token" /> // Uses token store
```

**Example 4: Rate Visit Store Implementation**

```typescript:luce-fe/src/store/visits/useRateVisit.ts
import { createRequestFactory } from "@/lib/request-factory";
import {
  RateVisitDocument,
  RateVisitWithTokenDocument,
  type RateVisitMutation,
  type RateVisitMutationVariables,
  type RateVisitWithTokenMutation,
  type RateVisitWithTokenMutationVariables,
} from "@/__generated__/graphql";

type RateVisitStoreConfig = {
  useToken?: boolean;
};

export function createRateVisitStore(config: RateVisitStoreConfig = {}) {
  // Conditional document selection: token or auth
  const document = config.useToken
    ? RateVisitWithTokenDocument
    : RateVisitDocument;

  return createRequestFactory<
    RateVisitMutation | RateVisitWithTokenMutation,
    RateVisitMutation | RateVisitWithTokenMutation,
    RateVisitMutationVariables | RateVisitWithTokenMutationVariables
  >({
    method: "mutation",
    graphqlDocument: document,
    hasUpload: true,
  });
}

// Export both variants for backward compatibility
// Auth-based store (useToken: false)
export const useRateVisitStore = createRateVisitStore({ useToken: false });
// Token-based store (useToken: true)
export const useRateVisitWithTokenStore = createRateVisitStore({ useToken: true });
```

## Migration Summary

**Code Reduction**:

- Before: 8 files × ~30 lines average = ~240 lines
- After: 4 unified stores × ~40 lines = ~160 lines
- **Net reduction**: ~80 lines (33% reduction) + elimination of duplication

**Files Affected**:

1. ✅ `store/visits/visitStore.ts` + `visitTokenStore.ts` → unified
2. ✅ `store/visits/useRateVisit.ts` + `useRateVisitWithToken.ts` → unified
3. ✅ `store/visits/useClientRescheduleVisit.ts` + `useClientRescheduleWithToken.ts` → unified
4. ✅ `store/visits/useClientCancelVisit.ts` + `useCancelVisitWithToken.ts` → unified

**Benefits**:

- ✅ Single source of truth for visit operations
- ✅ Consistent behavior across auth and token flows
- ✅ Easier maintenance - fix bugs in one place
- ✅ Type-safe with configuration
- ✅ Backward compatible exports

**Migration Strategy**:

1. Analyze differences between auth and token variants
2. Implement unified store with factory function
3. Migrate one operation type at a time (start with visit detail)
4. Test both auth and token flows thoroughly
5. Update all usages across codebase
6. Remove duplicate store files

**Considerations**:

- Token-based operations may have different error handling
- Token operations might not require authentication state
- Consider if separate stores are actually needed for clarity
- May want to keep separate if they diverge significantly in the future
