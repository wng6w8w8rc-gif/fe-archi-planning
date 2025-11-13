# InvoiceList Component Refactoring

## Summary

This refactoring extracts responsibilities from the `InvoiceList` component into custom hooks and configuration files to improve separation of concerns, maintainability, and testability. This follows the same pattern as the `VisitList` component refactoring.

### Files Affected

**New Files:**
- `components/hooks/use-invoice/index.ts` - Data fetching hook
- `components/hooks/use-invoice/use-invoice-list-actions.ts` - Action handlers hook

**Modified Files:**
- `containers/profile/invoices/invoice-list/index.tsx` - Simplified to use hooks
- `containers/profile/invoices/invoice-list/lib.ts` - May need enhancement

**Dependencies:**
- Requires `INVOICE-STORE-CONSOLIDATION.md` to be completed first

### Key Changes

- **Configuration** extracted to `lib.ts` (already exists, may need enhancement)
- **Data fetching logic** extracted to `@/components/hooks/use-invoice/index.ts`
- **Action handlers** extracted to `@/components/hooks/use-invoice/use-invoice-list-actions.ts`
- **Component** simplified to focus on UI rendering only

### Benefits

- **Better organization** with single responsibility per file
- **Reusable hooks** that can be used across invoice-related components
- **Easier testing** with isolated business logic
- **Improved maintainability** with clear separation of concerns
- **Consistent pattern** with `VisitList` component

## Implementation Realization

### Before: All Logic in Component

**invoice-list/index.tsx** (158 lines)

```typescript:luce-fe/src/containers/profile/invoices/invoice-list/index.tsx
export function InvoiceList({ type }: { type: InvoiceFilterStatusEnum }) {
  const { refreshing, setRefreshing } = useScrollRefresh();
  const { isMobile } = useWindowDimensions();
  const { push } = useRoute();
  const user = useAuthState((state) => state.data.userInfo);

  const { useStore, initialFilters } = listType[type];

  const {
    data: invoices,
    loading,
    fetchMore,
    refetch,
  } = useStore(useShallow((state) => state));

  const { setInvoiceId, showPaymentMethodModal } = useInvoiceState();

  const shouldShowLoadMore = !!invoices.length;

  useEffect(() => {
    if (user.id) {
      refetch({
        requestPayload: { filters: initialFilters(user.id) },
      });
    }
  }, [user.id, type, initialFilters]);

  useEffect(() => {
    if (refreshing) {
      refetch();
      setRefreshing(false);
    }
  }, [refreshing]);

  const onLoadMore = () => {
    fetchMore({
      requestPayload: {
        filters: initialFilters(user.id),
      },
    });
  };

  const handleViewInvoice = (invoiceId: string) => () => {
    push({
      pageKey: "invoiceDetail",
      params: {
        id: invoiceId,
      },
    });
  };

  const handlePay = (invoiceId: string) => () => {
    setInvoiceId(invoiceId);
    showPaymentMethodModal(PaymentMethodScreenModal.SELECT_PAYMENT);
    if (isMobile) {
      push({ pageKey: "selectPayment" });
    }
  };

  // ... UI rendering
}
```

### After: Extracted Hooks

**1. Primary Hook: `@/components/hooks/use-invoice/index.ts`**

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

**2. Actions Hook: `@/components/hooks/use-invoice/use-invoice-list-actions.ts`**

```typescript:luce-fe/src/components/hooks/use-invoice/use-invoice-list-actions.ts
import { useCallback } from "react";
import { useRoute } from "@/components/shared/router";
import { useWindowDimensions } from "@/components/hooks/use-window-dimension";
import { getPlatform } from "@/lib/utils";
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

**3. Main Component: `index.tsx`**

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

## Migration Summary

**Code Reduction**:

- **Before**: 158 lines with all logic mixed together
- **After**: ~60 lines component + ~80 lines hooks = ~140 lines total
- **Net reduction**: ~18 lines + better organization

**Benefits**:

- ✅ Single responsibility per file
- ✅ Reusable hooks for invoice-related components
- ✅ Easier testing with isolated business logic
- ✅ Consistent pattern with `VisitList` component
- ✅ Better maintainability

**Migration Strategy**:

1. Create `useInvoiceList` hook for data fetching
2. Create `useInvoiceListActions` hook for actions
3. Update `InvoiceList` component to use hooks
4. Test invoice list functionality
5. Consider extracting to generic list pattern if more list components follow this pattern
