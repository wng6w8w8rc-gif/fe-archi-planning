# Visit Token vs Auth Pattern Consolidation

## Summary

This refactoring consolidates the duplicate visit operation stores that handle both authenticated and token-based access. Currently, there are separate stores for each operation type (visit detail, rate, reschedule, cancel) that differ only in authentication method.

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
export const useRateVisitStore = createRequestFactory<
  RateVisitMutation,
  Response,
  RateVisitMutationVariables
>({
  method: "mutation",
  graphqlDocument: RateVisitDocument,
  // ... configuration
});
```

```typescript:luce-fe/src/store/visits/useRateVisitWithToken.ts
// Identical pattern but uses RateVisitWithTokenDocument
```

**Reschedule Visit** - 2 separate stores
**Cancel Visit** - 2 separate stores

### After: Unified Store Pattern

**Option 1: Factory with Auth Type**

```typescript:luce-fe/src/store/visits/createVisitOperationStore.ts
import { createRequestFactory } from "@/lib/request-factory";

type VisitAuthType = "auth" | "token";

type VisitOperationConfig<
  TQuery,
  TResponse,
  TVariables,
> = {
  authType: VisitAuthType;
  authDocument: any;
  tokenDocument: any;
  transformFunction?: (res: TQuery) => TResponse;
  onFetchSuccess?: (data: TResponse) => void;
};

export function createVisitOperationStore<
  TQuery,
  TResponse,
  TVariables,
>(config: VisitOperationConfig<TQuery, TResponse, TVariables>) {
  const document = config.authType === "auth"
    ? config.authDocument
    : config.tokenDocument;

  return createRequestFactory<TQuery, TResponse, TVariables>({
    method: "query", // or "mutation" based on operation
    graphqlDocument: document,
    transformFunction: config.transformFunction,
    onFetchSuccess: config.onFetchSuccess,
  });
}
```

**Option 2: Unified Store with Runtime Selection (Recommended)**

```typescript:luce-fe/src/store/visits/visitStore.ts
import { createRequestFactory } from "@/lib/request-factory";
import {
  VisitDocument,
  VisitTokenDocument,
  type VisitQuery,
  type VisitQueryVariables,
} from "@/__generated__/graphql";

type VisitStoreConfig = {
  useToken?: boolean;
};

export function createVisitStore(config: VisitStoreConfig = {}) {
  const document = config.useToken ? VisitTokenDocument : VisitDocument;

  return createRequestFactory<
    VisitQuery,
    VisitDetailData,
    VisitQueryVariables
  >({
    method: "query",
    graphqlDocument: document,
    // ... rest of configuration
  });
}

// Export both variants for backward compatibility
export const useVisitStore = createVisitStore({ useToken: false });
export const useVisitTokenStore = createVisitStore({ useToken: true });
```

**Option 3: Single Store with Dynamic Document (Simplest)**

```typescript:luce-fe/src/store/visits/visitStore.ts
import { createRequestFactory } from "@/lib/request-factory";
import {
  VisitDocument,
  VisitTokenDocument,
  type VisitQuery,
  type VisitQueryVariables,
} from "@/__generated__/graphql";

type VisitStoreState = {
  useToken: boolean;
  setUseToken: (useToken: boolean) => void;
};

export const useVisitStore = create<VisitStoreState & RequestStore<...>>((set, get) => ({
  useToken: false,
  setUseToken: (useToken) => set({ useToken }),
  // ... store implementation that uses get().useToken to select document
}));
```

**Updated Usage**

```typescript
// Before
const { data } = useVisitStore();
const { data: tokenData } = useVisitTokenStore();

// After (Option 2)
const { data } = useVisitStore(); // auth by default
const { data: tokenData } = useVisitTokenStore(); // token variant

// Or (Option 3)
const { data, setUseToken } = useVisitStore();
setUseToken(true); // switch to token mode
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
2. Choose consolidation approach (Option 2 recommended)
3. Migrate one operation type at a time (start with visit detail)
4. Test both auth and token flows thoroughly
5. Update all usages across codebase
6. Remove duplicate store files

**Considerations**:

- Token-based operations may have different error handling
- Token operations might not require authentication state
- Consider if separate stores are actually needed for clarity
- May want to keep separate if they diverge significantly in the future

