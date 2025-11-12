# Generic List Component Pattern

## Summary

This refactoring creates a generic list component that can be reused across different list types (visits, invoices, packages, etc.). All list components follow similar patterns: pagination, loading states, empty states, and data fetching.

### Key Changes

- **Generic `ListContainer` component** replaces duplicate list components
- **Configuration-based approach** with render props for customization
- **Consistent UX** across all list types
- **Reusable hooks** for list data fetching

### Benefits

- **DRY principle** - single source of truth for list UI
- **Consistent UX** across all lists
- **Easier to add new list types** - just provide configuration
- **Reduced code duplication** - eliminate ~200+ lines per list component
- **Easier maintenance** - fix bugs in one place

## Implementation Realization

### Before: Duplicate List Components

**VisitList** (274 lines → 90 lines after hook extraction)

```typescript:luce-fe/src/containers/visits/visit-list/index.tsx
export function VisitList({ type }: { type: VisitListType }) {
  const { visits, loading, shouldShowLoadMore, onLoadMore } = useVisitList(type);
  const { handleViewDetail, handleRebook, ... } = useVisitListActions();

  return (
    <View className="flex flex-col gap-4">
      <ListHeading icon={icon} title={title} subtitle={subtitle} />
      {loading ? (
        <Skeleton ... />
      ) : (
        <IfElse
          if={!!visits?.length}
          else={<EmptyCard ... />}
        >
          <View className="flex flex-col gap-4">
            {visits.map((visit) => (
              <VisitCard
                key={visit.id}
                visitData={visit}
                handleViewDetail={handleViewDetail}
                ...
              />
            ))}
          </View>
        </IfElse>
      )}
      <IfElse if={shouldShowLoadMore}>
        <Button onClick={onLoadMore} ... />
      </IfElse>
    </View>
  );
}
```

**InvoiceList** (158 lines)

```typescript:luce-fe/src/containers/profile/invoices/invoice-list/index.tsx
export function InvoiceList({ type }: { type: InvoiceFilterStatusEnum }) {
  // Similar structure with different hooks and components
  return (
    <View className="mt-4 flex flex-col gap-4 px-4 pb-8 md:px-0">
      {loading ? (
        <Skeleton ... />
      ) : (
        <IfElse
          if={!!invoices?.length}
          else={<EmptyCard ... />}
        >
          <View className="flex flex-col gap-4">
            {invoices.map((invoice) => (
              <InvoiceCard ... />
            ))}
          </View>
        </IfElse>
      )}
      <IfElse if={shouldShowLoadMore}>
        <Button onClick={onLoadMore} ... />
      </IfElse>
    </View>
  );
}
```

**PackageList** - Similar pattern

### After: Generic List Component

**GenericListContainer.tsx** - Reusable component

```typescript:luce-fe/src/components/shared/list/generic-list-container.tsx
import { View } from "@/components/ui";
import { Skeleton } from "@/components/ui";
import { Button } from "@/components/ui";
import { IfElse } from "@/components/shared/if-else";
import { ListHeading } from "@/components/shared/list-header";
import { EmptyCard } from "@/components/shared/empty-card";
import type { Icon } from "@/components/shared/icons";
import { __ } from "@/lib/i18n";

type GenericListConfig<TItem> = {
  title: string;
  subtitle?: string;
  icon?: Icon;
  emptyState: {
    image: string;
    title: string;
    description: string;
  };
  skeletonCount?: number;
  skeletonHeight?: string;
  loadMoreButtonText?: string;
  containerClassName?: string;
};

type GenericListProps<TItem> = {
  config: GenericListConfig<TItem>;
  items: TItem[];
  loading: boolean;
  shouldShowLoadMore: boolean;
  onLoadMore: () => void;
  renderItem: (item: TItem, index: number) => React.ReactNode;
  renderHeader?: () => React.ReactNode;
  renderFooter?: () => React.ReactNode;
};

export function GenericListContainer<TItem extends { id: string | number }>({
  config,
  items,
  loading,
  shouldShowLoadMore,
  onLoadMore,
  renderItem,
  renderHeader,
  renderFooter,
}: GenericListProps<TItem>) {
  const {
    title,
    subtitle,
    icon,
    emptyState,
    skeletonCount = 4,
    skeletonHeight = "h-[200px]",
    loadMoreButtonText = __("Load more"),
    containerClassName = "flex flex-col gap-4",
  } = config;

  return (
    <View className={containerClassName}>
      {icon && (
        <ListHeading
          icon={icon}
          title={__(title)}
          subtitle={subtitle ? __(subtitle) : ""}
        />
      )}
      {renderHeader?.()}
      {loading ? (
        <>
          {Array.from({ length: skeletonCount }).map((_, i) => (
            <Skeleton key={i} className={`${skeletonHeight} w-full`} />
          ))}
        </>
      ) : (
        <IfElse
          if={!!items?.length}
          else={
            <EmptyCard
              src={emptyState.image}
              title={__(emptyState.title)}
              desc={__(emptyState.description)}
            />
          }
        >
          <View className="flex flex-col gap-4 md:p-0">
            {items.map((item, index) => (
              <React.Fragment key={item.id}>
                {renderItem(item, index)}
              </React.Fragment>
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
          children={loadMoreButtonText}
        />
      </IfElse>
      {renderFooter?.()}
    </View>
  );
}
```

**Updated VisitList** - Simple configuration

```typescript:luce-fe/src/containers/visits/visit-list/index.tsx
import { GenericListContainer } from "@/components/shared/list/generic-list-container";
import { useVisitList } from "@/components/hooks/use-visit";
import { useVisitListActions } from "@/components/hooks/use-visit/use-visit-list-actions";
import { VisitCard } from "@/components/shared/visits/visit-card";
import { listType } from "./lib";
import NoVisitsBike from "@/assets/images/3d-illus/no-visit-bike.png";
import NoVisitsVaccum from "@/assets/images/3d-illus/no-visits-vaccum.png";

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

  const emptyState = type === "next30days"
    ? {
        image: NoVisitsVaccum,
        title: "You don't have any history visits",
        description: "Your completed bookings will appear here once the service is done.",
      }
    : {
        image: NoVisitsBike,
        title: "You don't have any upcoming visits",
        description: "Your scheduled services will appear here once booked.",
      };

  return (
    <GenericListContainer
      config={{
        title,
        subtitle,
        icon,
        emptyState,
        skeletonHeight: "h-[200px]",
        loadMoreButtonText: __("Show More"),
      }}
      items={visits}
      loading={loading}
      shouldShowLoadMore={shouldShowLoadMore}
      onLoadMore={onLoadMore}
      renderItem={(visit) => (
        <VisitCard
          key={`${visit.id}-${type}`}
          visitData={visit}
          handleViewDetail={handleViewDetail}
          handleRateVisit={handleRateVisit}
          handleRebook={handleRebook}
          handleSetSchedule={handleSetSchedule}
          handleReportVisit={handleReportVisit}
        />
      )}
    />
  );
}
```

**Updated InvoiceList** - Simple configuration

```typescript:luce-fe/src/containers/profile/invoices/invoice-list/index.tsx
import { GenericListContainer } from "@/components/shared/list/generic-list-container";
import { useInvoiceList } from "@/components/hooks/use-invoice";
import { useInvoiceListActions } from "@/components/hooks/use-invoice/use-invoice-list-actions";
import { InvoiceCard } from "@/components/shared/invoices/invoice-card";
import NoInvoices from "@/assets/images/3d-illus/no-invoices.png";
import { getPlatform } from "@/lib/utils";
import { PaymentMethodContainer } from "../payment-method";

export function InvoiceList({ type }: { type: InvoiceFilterStatusEnum }) {
  const { invoices, loading, shouldShowLoadMore, onLoadMore } = useInvoiceList(type);
  const { handleViewInvoice, handlePay } = useInvoiceListActions();
  const platform = getPlatform();

  return (
    <>
      <GenericListContainer
        config={{
          emptyState: {
            image: NoInvoices,
            title: "No invoices yet!",
            description: "We'll send your invoice at month's end for recurring bookings and the next day for one-time ones.",
          },
          skeletonHeight: "h-[120px]",
          skeletonCount: 1,
          containerClassName: "mt-4 flex flex-col gap-4 px-4 pb-8 md:px-0",
        }}
        items={invoices}
        loading={loading}
        shouldShowLoadMore={shouldShowLoadMore}
        onLoadMore={onLoadMore}
        renderItem={(invoice) => (
          <InvoiceCard
            key={invoice.id}
            {...invoice}
            onClick={handleViewInvoice(invoice.id)}
            onPay={handlePay(invoice.id)}
          />
        )}
        renderFooter={() => platform === "web" && <PaymentMethodContainer />}
      />
    </>
  );
}
```

## Migration Summary

**Code Reduction**:

- Before: ~3 list components × ~150 lines average = ~450 lines
- After: 1 generic component (~120 lines) + ~3 list components × ~50 lines = ~270 lines
- **Net reduction**: ~180 lines (40% reduction) + consistent UX

**Files Affected**:

1. ✅ `containers/visits/visit-list/index.tsx`
2. ✅ `containers/profile/invoices/invoice-list/index.tsx`
3. ✅ `containers/profile/package-list/index.tsx`
4. ✅ Potentially other list components

**Benefits**:

- ✅ DRY principle - single source of truth for list UI
- ✅ Consistent UX across all lists
- ✅ Easy to add new list types
- ✅ Easier maintenance - fix bugs in one place
- ✅ Flexible with render props for customization

**Migration Strategy**:

1. Create `GenericListContainer` component
2. Migrate `VisitList` first (already has hooks extracted)
3. Migrate `InvoiceList` (after hook extraction)
4. Migrate `PackageList`
5. Test all list components thoroughly
6. Consider adding more customization options if needed

**Optional Enhancements**:

- Add pull-to-refresh support
- Add infinite scroll option
- Add sorting/filtering UI
- Add list item animations
