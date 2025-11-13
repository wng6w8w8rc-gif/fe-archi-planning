# Global Modal Store Refactoring

## Summary

This refactoring consolidates all modal state management from multiple scattered stores into a single global modal store (`useGlobalModalStore`) with a unified modal component rendered at the layout level. This eliminates modal state duplication, standardizes modal behavior, and simplifies modal management across the codebase.

### Files Affected

**New Files:**

- `store/modal/globalModalStore.ts` - Global modal store
- `components/shared/global-modal/index.tsx` - Unified modal renderer component

**Modified Files:**

- `store/visits/visitDetail.ts` - Remove modal states and methods
- `store/auth/index.ts` - Remove modal states and `showModal` method
- `store/profile/index.ts` - Remove modal states and `showModal` method
- `store/invoices/index.ts` - Remove modal states and `showPaymentMethodModal` method
- `store/booking/useBookingState.ts` - Remove modal boolean states
- `store/report-issue/visit-issue/visitIssue.ts` - Remove `showModal` state
- `store/report-issue/package-issue/packageIssue.ts` - Remove `showModal` state
- `store/sharer/index.ts` - Remove entire store (if only used for modal)
- `containers/visits/visit-detail/index.tsx` - Remove conditional modal rendering
- `containers/auth/index.tsx` - Remove conditional modal rendering
- `app/_layout.tsx` - Add GlobalModal component, remove individual modal containers
- `app-web/router.tsx` - Add GlobalModal component (if applicable)

**Removed Files:**

- `containers/visits/visit-detail/index.tsx` - May be simplified or removed if only used for modals
- Individual modal container files that are no longer needed

### Key Changes

- **Single global modal store** replaces 8+ modal stores across the codebase
- **Unified modal component** rendered at layout level (web/native)
- **Direct component passing** - no registry needed, pass components directly
- **Support for all modal types**: Dialog, Drawer, BottomDrawer, Fullscreen
- **Animation and positioning** configuration per modal
- **Modal props/context** passed through store for dynamic content

### Benefits

- **Single source of truth** for modal state management
- **Consistent modal behavior** across all use cases
- **Easier maintenance** - modal logic in one place
- **Simpler API** - pass components directly, no registration needed
- **Reduced code duplication** - eliminate 8+ modal stores
- **Simplified component structure** - modals rendered at layout level

## Implementation Realization

### Before: Scattered Modal State Management

**visitDetail.ts** - Multiple modal states in one store

```typescript:luce-fe/src/store/visits/visitDetail.ts
type VisitDetailState = {
  showModal: boolean;
  showRateVisitModal: boolean;
  showSkipVisitModal: boolean;
  showRescheduleVisitConfirmation: boolean;
  showContactSupportToRescheduleModal: boolean;
  showSkipVisitSurveySuccess: boolean;
  visitId: string | null;
  visitToken: string | null;
  rescheduleVisitWorkerType: RescheduleVisitWorkerType | null;
};

export const useVisitDetailStore = create<VisitDetailStore>((set, get) => ({
  data: initialData,
  openVisitModal: (visitId: string, token?: string) => {
    set({
      data: {
        ...get().data,
        showModal: true,
        visitId,
        visitToken: token ?? null,
      },
    });
    // ... fetch logic
  },
  closeVisitModal: () => {
    set({
      data: {
        ...currentData,
        showModal: false,
        showRateVisitModal: false,
        showSkipVisitModal: false,
        // ... reset all modal states
      },
    });
  },
  // ... 10+ more modal methods
}));
```

**auth/index.ts** - Enum-based modal management

```typescript:luce-fe/src/store/auth/index.ts
type AuthState = {
  loginModalOpen: boolean;
  otpModalOpen: boolean;
  signUpModalOpen: boolean;
  emailLoginModalOpen: boolean;
  forgotPasswordModalOpen: boolean;
  addressModalOpen: boolean;
  logoutModalOpen: boolean;
  welcomeModalOpen: boolean;
};

showModal: (...modalNames: (AuthModals | null)[]) => {
  set({
    data: {
      ...get().data,
      loginModalOpen: modalNames.includes(AuthModals.LOGIN),
      otpModalOpen: modalNames.includes(AuthModals.OTP),
      signUpModalOpen: modalNames.includes(AuthModals.SIGN_UP),
      // ... 5+ more boolean assignments
    },
  });
},
```

**invoices/index.ts** - Payment method modals

```typescript:luce-fe/src/store/invoices/index.ts
type InvoiceState = {
  selectPaymentScreenModal: boolean;
  paynowScreenModal: boolean;
  creditCardScreenModal: boolean;
  addCreditCardScreenModal: boolean;
  showVoucherModal: boolean;
};

showPaymentMethodModal: (...modalNames: (PaymentMethodScreenModal | null)[]) => {
  set({
    data: {
      ...get().data,
      selectPaymentScreenModal: modalNames.includes(PaymentMethodScreenModal.SELECT_PAYMENT),
      paynowScreenModal: modalNames.includes(PaymentMethodScreenModal.PAYNOW),
      // ... more boolean assignments
    },
  });
},
```

**booking/useBookingState.ts** - Booking-related modals

```typescript:luce-fe/src/store/booking/useBookingState.ts
type BookingState = {
  airconSalesModalOpen: boolean;
  homeCleaningDurationModalOpen: boolean;
  chooseServicePackageModal: boolean;
  slotExpiredModalOpen: boolean;
  addToCalendarModalOpen: boolean;
  homeMassagePackageModalOpen: boolean;
  petGroomingPackageModalOpen: boolean;
  openClientServiceProfileForm: boolean;
};
```

**Multiple Container Files** - Conditional modal rendering

```typescript:luce-fe/src/containers/visits/visit-detail/index.tsx
export function VisitDetailContainer() {
  const {
    data: {
      showModal,
      showRateVisitModal,
      showSkipVisitModal,
      showRescheduleVisitConfirmation,
      // ... more modal states
    },
  } = useVisitDetailStore();

  if (showContactSupportToRescheduleModal) {
    return <ContactUsToRescheduleModal />;
  }
  if (showRateVisitModal && data) {
    return <VisitRateModalContainer visitData={data} />;
  }
  if (showRescheduleVisitConfirmation && data) {
    return <RescheduleSkipVisitConfirmation {...props} />;
  }
  if (showModal) {
    return <VisitDetailModal {...props} />;
  }
  // ... more conditional renders
  return null;
}
```

**Layout Files** - Multiple modal components

```typescript:luce-fe/src/app/_layout.tsx
<AuthContainer />
<VisitDetailContainer />
<LogoutDialog />
<ShareReferralModal />
```

### After: Unified Global Modal System

**globalModalStore.ts** - Single modal store with direct component passing

```typescript:luce-fe/src/store/modal/globalModalStore.ts
type ModalConfig = {
  animation?: "fade" | "slide-up" | "slide-down" | "none";
  position?: "center" | "bottom" | "fullscreen";
  type?: "dialog" | "drawer" | "bottom-drawer" | "fullscreen";
  closeOnBackdrop?: boolean;
  closeOnEscape?: boolean;
};

type ModalState = {
  isOpen: boolean;
  content: React.ReactNode;
  config?: ModalConfig;
};

type GlobalModalStore = {
  modals: ModalState[];
  openModal: (content: React.ReactNode, config?: ModalConfig) => void;
  closeModal: (index?: number) => void;
  closeAllModals: () => void;
  updateModal: (index: number, updates: Partial<ModalState>) => void;
};

export const useGlobalModalStore = create<GlobalModalStore>((set, get) => ({
  modals: [],
  openModal: (content, config) => {
    set({
      modals: [
        ...get().modals,
        {
          isOpen: true,
          content,
          config,
        },
      ],
    });
  },
  closeModal: (index) => {
    if (typeof index === "number") {
      set({
        modals: get().modals.filter((_, i) => i !== index),
      });
    } else {
      // Close last modal
      const modals = get().modals;
      if (modals.length > 0) {
        set({ modals: modals.slice(0, -1) });
      }
    }
  },
  closeAllModals: () => set({ modals: [] }),
  updateModal: (index, updates) => {
    set({
      modals: get().modals.map((modal, i) =>
        i === index ? { ...modal, ...updates } : modal
      ),
    });
  },
}));
```

**GlobalModal Component** - Unified modal renderer

```typescript:luce-fe/src/components/shared/global-modal/index.tsx
export function GlobalModal() {
  const modals = useGlobalModalStore((state) => state.modals);
  const closeModal = useGlobalModalStore((state) => state.closeModal);

  return (
    <>
      {modals.map((modal, index) => {
        if (!modal.isOpen) return null;

        const { content, config = {} } = modal;

        return (
          <ModalRenderer
            key={index}
            content={content}
            config={config}
            onClose={() => closeModal(index)}
          />
        );
      })}
    </>
  );
}

function ModalRenderer({ content, config, onClose }) {
  switch (config.type) {
    case "bottom-drawer":
      return (
        <BottomDrawerModal open onOpenChange={onClose}>
          {content}
        </BottomDrawerModal>
      );
    case "fullscreen":
      return (
        <MobileFullscreenModal open onClose={onClose}>
          {content}
        </MobileFullscreenModal>
      );
    case "dialog":
    default:
      return (
        <Dialog open onOpenChange={onClose}>
          <DialogContent>
            {content}
          </DialogContent>
        </Dialog>
      );
  }
}
```

**Updated Store Usage** - Simple modal opening with props in content

```typescript:luce-fe/src/store/visits/visitDetail.ts
// Before: Multiple modal states and methods
openVisitModal: (visitId: string, token?: string) => {
  set({ data: { ...get().data, showModal: true, visitId, visitToken: token ?? null } });
  // ... fetch logic
}

// After: Pass component with props directly in JSX
openVisitModal: (visitId: string, token?: string) => {
  useGlobalModalStore.getState().openModal(
    <VisitDetailModal visitId={visitId} token={token} />,
    { type: "bottom-drawer" }
  );
  // Note: Any fetch logic should be done inside VisitDetailModal component
}
```

**Updated Component Usage** - Props passed directly in content

```typescript
// Before: Store method with enum
useAuthState().showModal(AuthModals.LOGIN);

// After: Pass component directly, all logic inside component
useGlobalModalStore().openModal(<LoginModal />, { type: "dialog" });

// Before: Store method with multiple parameters
useVisitDetailStore().openRateVisitModalById(visitId, rate);

// After: Props passed directly in JSX, fetches done inside component
useGlobalModalStore().openModal(
  <VisitRateModalContainer visitId={visitId} rate={rate} />,
  { type: "dialog" }
);

// Example: Modal component handles its own data fetching
function VisitDetailModal({ visitId, token }: { visitId: string; token?: string }) {
  // All store calls, hooks, and fetches happen inside the component
  const { data, loading } = token
    ? useVisitTokenStore()
    : useVisitStore();

  useEffect(() => {
    if (token) {
      useVisitTokenStore.getState().fetch({ requestPayload: { token } });
    } else {
      useVisitStore.getState().fetch({ requestPayload: { id: visitId } });
    }
  }, [visitId, token]);

  return (
    // ... modal content
  );
}
```

**Updated Layout** - Single modal component

```typescript:luce-fe/src/app/_layout.tsx
<GlobalModal />
// All modals are now rendered through this single component
```

**Removed Container Complexity** - No conditional rendering needed

```typescript:luce-fe/src/containers/visits/visit-detail/index.tsx
// Before: 250+ lines with conditional modal rendering
export function VisitDetailContainer() {
  // ... complex conditional logic
  if (showModal) return <VisitDetailModal />;
  if (showRateVisitModal) return <VisitRateModalContainer />;
  // ... more conditionals
}

// After: Container focuses on business logic only
// Modals are handled by GlobalModal component
export function VisitDetailContainer() {
  // ... business logic only
  // Modal rendering happens in GlobalModal
}
```

## Migration Summary

**Code Reduction**:

- **Before**: 8+ stores with modal state management (~500+ lines)
- **After**: 1 global store (~80 lines) + 1 component (~150 lines) = ~230 lines
- **Net reduction**: ~270 lines + elimination of scattered modal logic
- **Container simplification**: Remove ~200+ lines of conditional modal rendering
- **No registry needed** - simpler architecture, pass components directly

**Stores Affected**:

1. ✅ `visitDetail.ts` - Remove 6 modal boolean states + 10+ modal methods
2. ✅ `auth/index.ts` - Remove 8 modal boolean states + `showModal` method
3. ✅ `profile/index.ts` - Remove 4 modal boolean states + `showModal` method
4. ✅ `invoices/index.ts` - Remove 5 modal boolean states + `showPaymentMethodModal` method
5. ✅ `booking/useBookingState.ts` - Remove 8 modal boolean states
6. ✅ `visit-issue/visitIssue.ts` - Remove `showModal` state
7. ✅ `package-issue/packageIssue.ts` - Remove `showModal` state
8. ✅ `sharer/index.ts` - Remove entire store (if only used for modal)

**Benefits**:

- ✅ Single source of truth for modal state management
- ✅ Consistent modal behavior across all use cases
- ✅ Simpler API - pass components directly, no registration step
- ✅ Easier to add new modals - just pass the component when opening
- ✅ Centralized animation and positioning configuration
- ✅ Simplified component structure - no conditional modal rendering
- ✅ Better testability - modal logic isolated in one place
- ✅ Support for modal stacking (nested modals)
- ✅ Reduced bundle size - shared modal rendering logic

**Migration Strategy**:

1. Create global modal system (store + component) - no registry needed
2. Migrate one store at a time to minimize risk
3. Test each migration before moving to next store
4. Update layouts last after all stores are migrated
5. Clean up old modal state code after successful migration

**Key Design Decision**:

The modal store uses a simple array structure where each modal has:

- `isOpen`: boolean to control visibility
- `content`: React component/node to render (with props already included in JSX)
- `config`: optional configuration (type, animation, position, etc.)

**Important**: All props are passed directly in the content JSX (e.g., `<VisitDetailModal visitId={visitId} token={token} />`). Any store calls, custom hooks, or data fetching should be done inside the modal component itself, not in the store method that opens the modal. This keeps the modal store simple and makes each modal component self-contained.
