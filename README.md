# V2 Refactoring Plan - Parallel Folder Approach

## Table of Contents

- [Summary](#summary)
- [Implementation Strategy](#implementation-strategy)
  - [Phase 1: Infrastructure Setup](#phase-1-infrastructure-setup)
  - [Phase 2: Route Integration](#phase-2-route-integration)
  - [Phase 3: Incremental Migration Strategy](#phase-3-incremental-migration-strategy)
  - [Phase 4: Shared Code Strategy](#phase-4-shared-code-strategy)
  - [Phase 5: Testing Strategy](#phase-5-testing-strategy)
  - [Phase 6: Migration Checklist](#phase-6-migration-checklist)
- [Example: Complete Migration Flow](#example-complete-migration-flow)
- [Version Governance](#version-governance)
- [Rollback Plan](#rollback-plan)
- [Final Migration Steps](#final-migration-steps)
- [Benefits](#benefits)
- [Migration Priority](#migration-priority)
- [Notes](#notes)
- [Future Considerations](#future-considerations)
- [Global Modal Refactoring](#global-modal-refactoring)
- [Provider to Store Migration](#provider-to-store-migration)
- [GraphQL Query Simplification](#graphql-query-simplification)
- [Code Modularization Pattern](#code-modularization-pattern)

## Summary

This document outlines the strategy for executing a BIG refactoring of the luce-fe React Native multiplatform codebase (web and app) using a **Parallel "v2" Folder Approach** with **Incremental Isolation**. This approach allows gradual migration of code while maintaining the existing v1 codebase fully functional, enabling safe, incremental refactoring with zero downtime.

### Key Strategy

- **Parallel Development**: Create `v2/` folders alongside existing code
- **Incremental Migration**: Move features one at a time from v1 to v2
- **Route-Level Switching**: Use route-level configuration to toggle between v1 and v2 routes
- **Zero Downtime**: v1 remains fully functional during migration
- **Gradual Rollout**: Test v2 in isolation, then gradually redirect users to v2 routes
- **Final State**: Keep both v1 and v2 separate forever - redirect routes to v2, move v1 code to `src/v1/` folder

### Files Affected

**New Files:**

- `src/v2/` - Root directory for all v2 code
- `src/v2/app/` - V2 routes (Expo Router for native)
- `src/v2/app-web/` - V2 web routes (React Router)
- `src/v2/components/` - V2 components
- `src/v2/containers/` - V2 containers
- `src/v2/hooks/` - V2 hooks
- `src/v2/store/` - V2 stores
- `src/v2/types/` - V2 types
- `src/v2/config/` - V2 config
- `src/v2/constants/` - V2 constants
- `src/v2/lib/` - V2 utilities
- `src/config/routes.ts` - Route version configuration

**Future Files (Final State):**

- `src/v1/` - Root directory for all v1 code (moved from root `src/` after migration)

**Modified Files:**

- `src/app/_layout.tsx` - Add v2 route handling
- `src/app-web/router.tsx` - Add v2 route handling
- `src/constants/routes.ts` - Add v2 route constants

## Implementation Strategy

### Phase 1: Infrastructure Setup

#### 1.1 Route Version Configuration

Create a simple route version configuration to toggle between v1 and v2 routes.

**File: `src/config/routes.ts`**

```typescript
// Route version configuration
// Set to true to use v2, false to use v1
export const ROUTE_VERSIONS = {
  home: false,
  profile: false,
  visits: false,
  booking: false,
  login: false,
  signup: false,
  rewards: false,
  notifications: false,
  // Add more routes as needed
} as const;

export type RouteName = keyof typeof ROUTE_VERSIONS;

/**
 * Check if a route should use v2
 */
export function useRouteV2(routeName: RouteName): boolean {
  return ROUTE_VERSIONS[routeName] ?? false;
}
```

#### 1.2 Route Constants

**File: `src/constants/routes.ts` (additions)**

```typescript
import { ROUTE_VERSIONS } from "@/config/routes";

// Route component names (for React Native Expo Router)
// Route version configuration determines which version to use
export const routes = {
  home: ROUTE_VERSIONS.home ? "HomeV2" : "HomeV1",
  profile: ROUTE_VERSIONS.profile ? "ProfileV2" : "ProfileV1",
  visits: ROUTE_VERSIONS.visits ? "VisitsV2" : "VisitsV1",
  booking: ROUTE_VERSIONS.booking ? "BookingV2" : "BookingV1",
  login: ROUTE_VERSIONS.login ? "LoginV2" : "LoginV1",
  // Add more routes as needed
} as const;

// Route paths (for Web React Router)
export const ROUTES = {
  Root: "",
  Login: "/login",
  Signup: "/signup",
  Home: "/",
  Profile: {
    Root: "/profile",
    // ... same structure as before
  },
  Visits: {
    Root: "/visits",
    // ...
  },
  // ... existing route structure
} as const;

// V2 routes (for testing during migration - optional)
export const ROUTES_V2 = {
  Root: "/v2",
  Login: "/v2/login",
  Signup: "/v2/signup",
  Profile: {
    Root: "/v2/profile",
    // ... same structure as ROUTES.Profile
  },
  // ... mirror ROUTES structure with /v2 prefix
} as const;

// Note: Route version configuration determines which version (v1 or v2) to use
// During migration, routes can be individually toggled in src/config/routes.ts
// After migration is complete, all routes will use v2 (same paths as v1)
```

#### 1.3 Folder Structure

**During Migration:**

```
src/
├── v2/                         # New v2 code
│   ├── app/
│   ├── app-web/
│   ├── components/
│   ├── containers/
│   ├── hooks/
│   ├── store/
│   ├── types/
│   ├── config/
│   ├── constants/
│   └── lib/
├── app/                        # Existing v1 code (stays in place)
├── app-web/                    # Existing v1 code
├── components/                 # Existing v1 code
├── containers/                 # Existing v1 code
├── store/                      # Existing v1 code
└── ...
```

**Final State (After Migration):**

```
src/
├── v1/                         # Preserved v1 code (moved from root)
│   ├── app/
│   ├── app-web/
│   ├── components/
│   ├── containers/
│   ├── store/
│   └── ...
├── v2/                         # Active v2 code (handles all routes)
│   ├── app/
│   ├── app-web/
│   ├── components/
│   ├── containers/
│   ├── hooks/
│   ├── store/
│   └── ...
├── lib/                        # Shared utilities
├── queries/                    # Shared GraphQL queries
├── assets/                     # Shared assets
└── __generated__/              # Shared generated code
```

### Phase 2: Route Integration

#### 2.1 Native Route Integration (Expo Router)

**File: `src/app/_layout.tsx` (modifications)**

```typescript
// Keep existing v1 layout structure
// Route-level flags determine which components to render
import "../assets/global.css";
// ... existing imports

export const unstable_settings = {
  initialRouteName: "(tabs)",
};

function RootLayout() {
  // Existing v1 layout structure
  // Individual routes will use v1 or v2 components based on route flags
  return (
    <PostHogProvider client={posthogInstance}>
      <ApolloProvider client={ClientInstance}>
        <AuthProvider>
          <NotificationsProvider>
            <SafeAreaProvider>
              {showSplash && <CustomSplashScreen />}
              <Stack screenOptions={{ headerShown: false }}>
                <Stack.Screen name="(tabs)" />
              </Stack>
              {/* Components */}
            </SafeAreaProvider>
          </NotificationsProvider>
        </AuthProvider>
      </ApolloProvider>
    </PostHogProvider>
  );
}

export default Sentry.wrap(RootLayout);
```

**File: `src/app/(tabs)/index.tsx` (route-level example)**

```typescript
import { useRouteV2 } from "@/config/routes";
import { HomeV1 } from "@/containers/homepage";
import { HomeV2 } from "@/v2/containers/homepage";

export default function HomeScreen() {
  const useV2 = useRouteV2("home");

  // Route version configuration determines which component to render
  if (useV2) {
    return <HomeV2 />;
  }

  return <HomeV1 />;
}
```

**File: `src/app/(tabs)/profile.tsx` (route-level example)**

```typescript
import { useRouteV2 } from "@/config/routes";
import { ProfileV1 } from "@/containers/profile";
import { ProfileV2 } from "@/v2/containers/profile";

export default function ProfileScreen() {
  const useV2 = useRouteV2("profile");

  // Route version configuration determines which component to render
  if (useV2) {
    return <ProfileV2 />;
  }

  return <ProfileV1 />;
}
```

**File: `src/v2/app/_layout.tsx`**

```typescript
// V2 version of root layout
// Copy from src/app/_layout.tsx and refactor incrementally
import "../assets/global.css";
// ... imports

export const unstable_settings = {
  initialRouteName: "(tabs)",
};

function V2RootLayout() {
  // V2 implementation with refactored code
  return (
    <PostHogProvider client={posthogInstance}>
      <ApolloProvider client={ClientInstance}>
        <AuthProvider>
          <NotificationsProvider>
            <SafeAreaProvider>
              {showSplash && <CustomSplashScreen />}
              <Stack screenOptions={{ headerShown: false }}>
                <Stack.Screen name="(tabs)" />
              </Stack>
              {/* V2 components */}
            </SafeAreaProvider>
          </NotificationsProvider>
        </AuthProvider>
      </ApolloProvider>
    </PostHogProvider>
  );
}

export default Sentry.wrap(V2RootLayout);
```

#### 2.2 Web Route Integration (React Router)

**File: `src/app-web/router.tsx` (modifications)**

```typescript
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { useRouteV2 } from "@/config/routes";
import { ROUTES } from "@/constants/routes";
// V1 components
import { HomepagePage } from "@/app-web/pages/homepage";
import { ProfilePage } from "@/app-web/pages/profile";
import { VisitsPage } from "./pages/visits";
// V2 components
import { HomepagePage as HomepagePageV2 } from "@/v2/app-web/pages/homepage";
import { ProfilePage as ProfilePageV2 } from "@/v2/app-web/pages/profile";
import { VisitsPage as VisitsPageV2 } from "@/v2/app-web/pages/visits";
// ... other imports

// Route wrapper component that handles v1/v2 switching
function RouteWrapper({
  routeName,
  V1Component,
  V2Component,
}: {
  routeName: string;
  V1Component: React.ComponentType;
  V2Component: React.ComponentType;
}) {
  const useV2 = useRouteV2(routeName as any);
  return useV2 ? <V2Component /> : <V1Component />;
}

export default function App() {
  return (
    <ErrorBoundary fallback={ErrorBoundaryPage}>
      <PostHogProvider client={posthogInstance as unknown as PostHog}>
        <ApolloProvider client={ClientInstance}>
          <AuthProvider>
            <BrowserRouter>
              <Routes>
                {/* Home route - route version config determines v1 or v2 */}
                <Route
                  path={ROUTES.Root}
                  element={<RouteWrapper routeName="home" V1Component={HomepagePage} V2Component={HomepagePageV2} />}
                />
                {/* Profile route - route version config determines v1 or v2 */}
                <Route
                  path={ROUTES.Profile.Root}
                  element={<RouteWrapper routeName="profile" V1Component={ProfilePage} V2Component={ProfilePageV2} />}
                />
                {/* Visits route - route version config determines v1 or v2 */}
                <Route
                  path={ROUTES.Visits.Root}
                  element={<RouteWrapper routeName="visits" V1Component={VisitsPage} V2Component={VisitsPageV2} />}
                />
                {/* Other routes... */}
              </Routes>
            </BrowserRouter>
          </AuthProvider>
        </ApolloProvider>
      </PostHogProvider>
    </ErrorBoundary>
  );
}
```

**Alternative: Direct route-level checks**

```typescript
// src/app-web/router.tsx (alternative approach)
import { useRouteV2 } from "@/config/routes";

export default function App() {
  const homeUseV2 = useRouteV2("home");
  const profileUseV2 = useRouteV2("profile");
  const visitsUseV2 = useRouteV2("visits");

  return (
    <BrowserRouter>
      <Routes>
        <Route path={ROUTES.Root} element={homeUseV2 ? <HomepagePageV2 /> : <HomepagePage />} />
        <Route path={ROUTES.Profile.Root} element={profileUseV2 ? <ProfilePageV2 /> : <ProfilePage />} />
        <Route path={ROUTES.Visits.Root} element={visitsUseV2 ? <VisitsPageV2 /> : <VisitsPage />} />
        {/* Other routes... */}
      </Routes>
    </BrowserRouter>
  );
}
```

**After Full Migration - Redirect Strategy:**

```typescript
// src/app-web/router.tsx (final state)
import { V2AppRouter } from "@/v2/app-web/router";
import { Navigate, Routes, Route } from "react-router-dom";
import { ROUTES } from "@/constants/routes";

export default function App() {
  // All routes now redirect to v2
  // v1 code is preserved in src/v1/ but not used
  return (
    <ErrorBoundary fallback={ErrorBoundaryPage}>
      <V2AppRouter />
      {/* Optional: Keep v1 routes accessible for reference */}
      <Routes>
        <Route path="/v1/*" element={<V1AppRouter />} />
      </Routes>
    </ErrorBoundary>
  );
}
```

**File: `src/v2/app-web/router.tsx`**

```typescript
// V2 version of web router
// Copy from src/app-web/router.tsx and refactor incrementally
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { ROUTES_V2 } from "@/constants/routes";
// ... imports

export function V2AppRouter() {
  return (
    <ErrorBoundary fallback={ErrorBoundaryPage}>
      <PostHogProvider client={posthogInstance as unknown as PostHog}>
        <ApolloProvider client={ClientInstance}>
          <AuthProvider>
            <BrowserRouter>
              <Routes>
                <Route path={ROUTES_V2.Root} element={<V2HomepagePage />} />
                <Route path={ROUTES_V2.Login} element={<V2LoginPageWrapper />} />
                {/* V2 routes */}
              </Routes>
            </BrowserRouter>
          </AuthProvider>
        </ApolloProvider>
      </PostHogProvider>
    </ErrorBoundary>
  );
}
```

### Phase 3: Incremental Migration Strategy

#### 3.1 Migration Workflow

For each feature/module to migrate:

1. **Create V2 Structure**

   - Copy existing code to `v2/` folder
   - Update imports to use v2 paths
   - Keep v1 code untouched

2. **Refactor in V2**

   - Apply refactoring patterns (hooks, stores, etc.)
   - Improve code structure
   - Add tests

3. **Test V2 in Isolation**

   - Engineers test v2 routes by toggling route version config
   - Compare behavior with v1
   - Fix issues

4. **Gradual Rollout**

   - Enable for more routes by updating route version config
   - Monitor for issues
   - Gradually expand user base
   - Redirect more routes to v2

5. **Final Migration**
   - Once v2 is stable, redirect all routes to v2
   - Keep v1 code intact (don't delete or modify)
   - Move v1 code to `src/v1/` folder
   - Update route handlers to redirect to v2
   - Final structure: `src/v1/` and `src/v2/` coexist

#### 3.2 Example: Migrating Auth Module

**Step 1: Copy to V2**

```
src/containers/auth/          →  src/v2/containers/auth/
src/store/auth/               →  src/v2/store/auth/
src/app/login/                →  src/v2/app/login/
```

**Step 2: Update Imports in V2**

```typescript
// src/v2/containers/auth/login/index.tsx
// Change imports from:
import { useAuthState } from "@/store/auth";
// To:
import { useAuthState } from "@/v2/store/auth";
```

**Step 3: Refactor in V2**

```typescript
// src/v2/store/auth/index.ts
// Apply refactoring patterns:
// - Extract hooks
// - Consolidate stores
// - Improve type safety
// - Add better error handling
```

**Step 4: Update V2 Routes**

```typescript
// src/v2/app-web/router.tsx
<Route path={ROUTES_V2.Login} element={<V2LoginPageWrapper />} />
```

**Step 5: Test**

- Engineer logs in with allowed email
- Tests v2 login flow
- Compares with v1 behavior

### Phase 4: Shared Code Strategy

#### 4.1 Shared Utilities

Some code can be shared between v1 and v2:

```
src/
├── lib/              # Shared utilities (never moved)
│   ├── graphql/
│   ├── monitoring/
│   └── ...
├── queries/          # Shared GraphQL queries (never moved)
├── assets/           # Shared assets (never moved)
└── __generated__/    # Shared generated code (never moved)
```

**V2 can import shared code:**

```typescript
// src/v2/components/example.tsx
import { ClientInstance } from "@/lib/graphql/apollo"; // Shared
import { useCustomHook } from "@/v2/hooks/custom"; // V2 specific
```

**V1 can also import shared code (even after moved to src/v1/):**

```typescript
// src/v1/components/example.tsx
import { ClientInstance } from "@/lib/graphql/apollo"; // Shared
import { useCustomHook } from "@/v1/hooks/custom"; // V1 specific
```

#### 4.2 Shared Code Strategy

- **Keep shared code in root `src/`**: Never move to v1 or v2 folders
- **Both v1 and v2 import from root**: Shared utilities remain accessible to both
- **No duplication**: Shared code is never copied, always imported

### Phase 5: Testing Strategy

#### 5.1 Route Version Testing

**For Engineers:**

1. Update `ROUTE_VERSIONS` in `src/config/routes.ts` to enable v2 for a route
2. Test the v2 route functionality
3. Compare with v1 behavior
4. Report issues
5. Verify fixes

**For Engineers (Additional Testing):**

1. Toggle route versions one at a time
2. Test v2 routes thoroughly
3. Compare v1 and v2 behavior side-by-side
4. Document any differences or issues

#### 5.2 A/B Testing

Once v2 is stable, can implement percentage-based rollout by extending the route version configuration.

### Phase 6: Migration Checklist

For each module/feature:

- [ ] Copy code to `v2/` folder
- [ ] Update imports in v2 code
- [ ] Create v2 routes
- [ ] Refactor code in v2
- [ ] Test v2 functionality
- [ ] Compare with v1 behavior
- [ ] Fix issues
- [ ] Document changes
- [ ] Get team review
- [ ] Enable route versions in config for testing
- [ ] Monitor for issues
- [ ] Gradually expand user base
- [ ] Redirect all routes to v2 when stable
- [ ] Move v1 code to `src/v1/` folder
- [ ] Update route handlers to redirect to v2
- [ ] Keep v1 code preserved in `src/v1/` for reference
- [ ] Update documentation

## Example: Complete Migration Flow

### Example: Migrating Visit List Feature

**Step 1: Copy Structure**

```bash
# Copy containers
cp -r src/containers/visits src/v2/containers/visits

# Copy stores
cp -r src/store/visits src/v2/store/visits

# Copy hooks (if any)
cp -r src/components/hooks/use-visit src/v2/hooks/use-visit

# Copy types
cp src/types/visit.ts src/v2/types/visit.ts
```

**Step 2: Update Imports**

```typescript
// src/v2/containers/visits/visit-list/index.tsx
// Before:
import { useVisitList } from "@/components/hooks/use-visit";
import { useVisitStore } from "@/store/visits/visitStore";

// After:
import { useVisitList } from "@/v2/hooks/use-visit";
import { useVisitStore } from "@/v2/store/visits/visitStore";
```

**Step 3: Refactor in V2**

```typescript
// src/v2/containers/visits/visit-list/index.tsx
// Apply refactoring patterns:
// - Extract configuration
// - Simplify component
// - Improve hooks
// - Better error handling
```

**Step 4: Add V2 Route**

```typescript
// src/v2/app-web/router.tsx
<Route
  path={ROUTES_V2.Visits.Root}
  element={
    <PrivateRoute>
      <V2VisitsPage />
    </PrivateRoute>
  }
>
  <Route path={ROUTES_V2.Visits.Children.Upcoming} element={<V2VisitsUpcoming />} />
  <Route path={ROUTES_V2.Visits.Children.History} element={<V2VisitsHistory />} />
</Route>
```

**Step 5: Test**

- Engineer logs in with allowed email
- Navigates to `/v2/visits`
- Tests all functionality
- Compares with `/visits` (v1)

**Step 6: Iterate**

- Fix issues found
- Improve code
- Test again
- Repeat until stable

**Step 7: Final Migration (When All Features Are Migrated)**

Once v2 is stable and all features are migrated:

```bash
# Move v1 code to v1 folder (preserve, don't delete)
mkdir -p src/v1
mv src/containers src/v1/containers
mv src/store src/v1/store
mv src/app src/v1/app
mv src/app-web src/v1/app-web
# ... move all v1 code to src/v1/

# Update route handlers to redirect to v2
# v2 routes now handle the same paths as v1 used to
# v1 code is preserved in src/v1/ for reference
```

**Final Route Structure:**

```typescript
// src/app-web/router.tsx (final state)
// All routes now point to v2, v1 code preserved in src/v1/
import { V2AppRouter } from "@/v2/app-web/router";

export default function App() {
  // All users now use v2 routes
  // v1 code is in src/v1/ but not actively used
  return <V2AppRouter />;
}
```

## Benefits

### Safety

- ✅ **Zero Downtime**: v1 remains fully functional
- ✅ **Isolated Testing**: Test v2 without affecting v1
- ✅ **Easy Rollback**: Toggle route version config to revert
- ✅ **Gradual Migration**: Move at your own pace

### Development Experience

- ✅ **Clear Separation**: v1 and v2 code clearly separated
- ✅ **Incremental Progress**: Migrate one feature at a time
- ✅ **Team Collaboration**: Multiple engineers can work on different v2 features
- ✅ **Easy Comparison**: Side-by-side comparison of v1 vs v2

### Quality

- ✅ **Thorough Testing**: Test v2 thoroughly before replacing v1
- ✅ **Better Code**: Apply learnings and best practices in v2
- ✅ **Reduced Risk**: Lower risk of breaking existing functionality

## Migration Priority

Suggested order for migration:

1. **Infrastructure** (Phase 1-2)

   - Route version configuration
   - Route integration
   - Basic v2 structure

2. **Simple Features** (Phase 3)

   - Static pages
   - Simple components
   - Utility functions

3. **Medium Complexity** (Phase 3)

   - Forms
   - Lists
   - Modals

4. **Complex Features** (Phase 3)

   - Booking flow
   - Payment flow
   - Authentication

5. **Core Features** (Phase 3)
   - Navigation
   - State management
   - Data fetching

## Notes

- **Don't modify v1 code**: Once code is moved to v2, don't update v1 (except critical bug fixes)
- **Preserve v1 code**: v1 code is moved to `src/v1/` and preserved for reference
- **Document changes**: Document what was refactored and why in v2
- **Code reviews**: Review v2 code before enabling for users
- **Performance**: Monitor v2 performance compared to v1
- **User feedback**: Collect feedback from engineers testing v2
- **Final state**: Both v1 and v2 folders coexist - v2 handles all routes, v1 is preserved
- **Consistency**: Ensure file paths and folder names in this document match the actual codebase structure. If code structure changes, update this document accordingly.

## Version Governance

This section clarifies how v1 and v2 changes are tracked and managed to prevent confusion and maintain code quality.

### v1 (Legacy)

- **Status**: Frozen except for critical bug fixes and urgent patches
- **Purpose**: Maintain stability for users during migration period
- **Allowed Changes**:
  - Critical security fixes
  - Urgent production bugs that affect user experience
  - No new features or enhancements
- **Code Ownership**: Maintained by fallback engineers for bug support

### v2 (Active)

- **Status**: Receives all new features, refactors, and architectural improvements
- **Purpose**: Modern, refactored codebase with improved patterns and structure
- **Allowed Changes**:
  - All new features
  - Refactoring and architectural improvements
  - Performance optimizations
  - Bug fixes
- **Code Ownership**: Owned by core refactor team (Gio, Fathul / Luce.sg)

### Shared Modules

Shared modules (`lib/`, `queries/`, `assets/`, `__generated__/`) must remain backward-compatible until v1 is fully deprecated:

- **Backward Compatibility**: Changes to shared modules must not break v1 code
- **Version-Aware Changes**: If breaking changes are needed, create version-specific implementations
- **Testing**: Test shared module changes against both v1 and v2 code paths

### Code Ownership

- **v1**: Maintained by fallback engineers for bug support
- **v2**: Owned by core refactor team (Gio, Fathul / Luce.sg)

### Preventing "v1 Resurrection"

To prevent accidentally adding new features to v1:

1. **Code Review**: All PRs touching `src/v1/` or root-level v1 folders require explicit approval
2. **Linting Rules**: Consider adding lint rules to warn when modifying v1 code
3. **Documentation**: Clear folder structure and comments indicate v1 is legacy
4. **Team Awareness**: Regular reminders that v1 is frozen

## Rollback Plan

Even though route version configuration and zero downtime are in place, it's important to have an explicit rollback plan in case critical issues are discovered in v2.

### To Revert to v1

1. **Update Route Version Config**

   - Set the route's version to `false` in `src/config/routes.ts` (e.g., `home: false`)
   - Commit and deploy immediately

2. **Redeploy**
   - v1 routes become active again automatically
   - All users will be routed back to v1 code paths

### Maintaining v1 Readiness

To ensure smooth rollback:

- **Keep v1 Buildable**: Ensure v1 code remains buildable at all times

  - Don't break v1 imports or dependencies
  - Test v1 build process regularly
  - Keep v1 dependencies up to date (security patches only)

- **Keep v1 Tests Active**: Maintain tests and configs for v1 until full migration

  - Run v1 tests in CI/CD pipeline
  - Ensure v1 test suite passes
  - Document v1 test coverage

- **Monitor v1 Health**: Even when v2 is active, monitor v1 for issues
  - Keep v1 error tracking active
  - Monitor v1 performance metrics
  - Be ready to switch back if needed

### Rollback Scenarios

**Scenario 1: Critical Bug in v2**

- Set route version to `false` in config immediately
- Investigate issue in v2
- Fix and test thoroughly
- Re-enable route version after fix

**Scenario 2: Performance Degradation**

- Set route version to `false` for affected route
- Investigate performance issues
- Optimize v2 code
- Re-enable with monitoring

**Scenario 3: User Experience Issues**

- Set route version to `false`
- Gather user feedback
- Make necessary adjustments in v2
- Re-enable with improved UX

## Final Migration Steps

When all features are migrated and v2 is stable:

1. **Redirect all routes to v2**

   - Update route handlers to point to v2
   - Set all route versions to `true` in config (all routes use v2)
   - v2 routes now handle the same paths as v1

2. **Move v1 code to `src/v1/`**

   - Move all v1 folders to `src/v1/`
   - Update imports in v1 code to use `@/v1/` prefix
   - Keep v1 code intact for reference

3. **Final structure**

   - `src/v1/` - Preserved v1 code (not actively used)
   - `src/v2/` - Active v2 code (handles all routes)
   - `src/lib/`, `src/queries/`, etc. - Shared code (never moved)

4. **Optional: Keep v1 accessible**
   - Can keep v1 routes accessible at `/v1/*` for reference
   - Useful for debugging or comparison

## Future Considerations

These are optional enhancements that can be implemented if needed:

- **Percentage-based rollout**: During migration, enable for percentage of users
- **A/B testing**: Compare v1 vs v2 metrics during migration
- **Automated migration**: Tools to help migrate code from v1 to v2
- **Migration scripts**: Scripts to update imports, routes, etc.
- **Monitoring dashboards**: Track v1 vs v2 performance metrics

---

## Global Modal Store Refactoring

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

### Usage Examples

**Example 1: Opening a Simple Dialog Modal**

```typescript
import { Button } from "@/components/ui";
import { useGlobalModalStore } from "@/store/modal/globalModalStore";
import { LoginModal } from "@/containers/auth/login";
import { __ } from "@/lib/i18n";

function LoginButton() {
  const openModal = useGlobalModalStore((state) => state.openModal);

  const handleLogin = () => {
    openModal(<LoginModal />, { type: "dialog" });
  };

  return <Button variant="primary" onClick={handleLogin} children={__("Login")} />;
}
```

**Example 2: Opening a Bottom Drawer with Props**

```typescript
import { Button } from "@/components/ui";
import { useGlobalModalStore } from "@/store/modal/globalModalStore";
import { VisitDetailModal } from "@/containers/visits/visit-detail";
import { __ } from "@/lib/i18n";

function VisitCard({ visitId }: { visitId: string }) {
  const openModal = useGlobalModalStore((state) => state.openModal);

  const handleViewDetail = () => {
    openModal(<VisitDetailModal visitId={visitId} />, {
      type: "bottom-drawer",
      animation: "slide-up",
    });
  };

  return <Button variant="tertiary" onClick={handleViewDetail} children={__("View Details")} />;
}
```

**Example 3: Opening a Fullscreen Modal**

```typescript
import { Button } from "@/components/ui";
import { useGlobalModalStore } from "@/store/modal/globalModalStore";
import { BookingFlowModal } from "@/containers/booking";
import { __ } from "@/lib/i18n";

function BookServiceButton() {
  const openModal = useGlobalModalStore((state) => state.openModal);

  const handleBook = () => {
    openModal(<BookingFlowModal serviceId="123" />, {
      type: "fullscreen",
      animation: "fade",
    });
  };

  return <Button variant="primary" onClick={handleBook} children={__("Book Now")} />;
}
```

**Example 4: Closing a Modal**

```typescript
import { View, Button } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useGlobalModalStore } from "@/store/modal/globalModalStore";
import { __ } from "@/lib/i18n";

function ModalContent() {
  const closeModal = useGlobalModalStore((state) => state.closeModal);

  const handleClose = () => {
    closeModal(); // Closes the last opened modal
  };

  return (
    <View>
      <Typography variant="h1">{__("Modal Content")}</Typography>
      <Button variant="tertiary" onClick={handleClose} children={__("Close")} />
    </View>
  );
}
```

**Example 5: Opening Multiple Modals (Stacking)**

```typescript
import { Button } from "@/components/ui";
import { useGlobalModalStore } from "@/store/modal/globalModalStore";
import { __ } from "@/lib/i18n";

function ParentComponent() {
  const openModal = useGlobalModalStore((state) => state.openModal);

  const handleOpenNested = () => {
    // First modal
    openModal(<FirstModal />, { type: "dialog" });

    // Second modal (stacked on top)
    setTimeout(() => {
      openModal(<SecondModal />, { type: "dialog" });
    }, 100);
  };

  return <Button variant="primary" onClick={handleOpenNested} children={__("Open Nested Modals")} />;
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

---

## Provider to Store Migration

## App Version Provider to Zustand Store Migration

## Summary

This refactoring converts `AppVersionProvider` (React Context) to `useAppVersionStore` (Zustand store), following the existing store pattern in the codebase. Version check logic is extracted to `useAppVersionCheck` hook, and initialization status is passed to `useSplashScreen`.

### Files Affected

**New Files:**

- `store/app-version/appVersionStore.ts` - Zustand store for app version
- `components/hooks/use-app-version-check/index.tsx` - Version check hook

**Removed Files:**

- `components/hooks/use-app-version/index.tsx` - Provider component removed (or converted to hook only)

**Modified Files:**

- `app/_layout.tsx` - Remove AppVersionProvider, add useAppVersionCheck hook
- All files using AppVersionContext - Update to use new store
- `components/hooks/use-splash-screen/index.tsx` - Add appVersionInitDone parameter

### Key Changes

- **AppVersionProvider** → `useAppVersionStore` (Zustand store)
- **Version check logic** extracted to `useAppVersionCheck` hook
- **Provider component removed** from `_layout.tsx`
- **Initialization status** passed to `useSplashScreen`

### Benefits

- **Consistent state management** - Uses Zustand pattern like other stores
- **No provider nesting** - Eliminates React Context provider
- **Better performance** - Zustand subscriptions are more efficient
- **Centralized initialization** - Version check logic in `useSplashScreen`
- **Reusable hooks** - Version logic can be used anywhere

## Implementation Realization

### Before: AppVersionProvider with React Context

```typescript:luce-fe/src/components/hooks/use-app-version/index.tsx
import * as Application from "expo-application";
import React, { useEffect, useMemo, useState } from "react";
import { useQuery } from "@apollo/client";
import { MobileAppVersionDocument } from "@/__generated__/graphql";
import { useFocusScreen } from "../use-focus-screen";
import { collectError } from "@/lib/monitoring/sentry";

interface AppVersionData {
  build_version: number;
  appOutDated: boolean;
}

const defaultAppVersionData: AppVersionData = {
  build_version: Number(Application.nativeBuildVersion),
  appOutDated: false,
};

const AppVersionContext = React.createContext<AppVersionData>(defaultAppVersionData);

AppVersionContext.displayName = "AppVersionContext";

export const AppVersionProvider = ({ defaults, children }: AppVersionProviderProps) => {
  const [appVersionData, setAppVersionData] = useState<AppVersionData>({
    ...defaultAppVersionData,
    ...defaults,
  });

  const { data, error, refetch } = useQuery(MobileAppVersionDocument);

  useEffect(() => {
    if (data?.mobileAppVersion?.clientAppBuildNumber) {
      setAppVersionData((prevConfig) => ({
        ...prevConfig,
        build_version: data.mobileAppVersion?.clientAppBuildNumber ?? prevConfig.build_version,
      }));
    }
  }, [data?.mobileAppVersion?.clientAppBuildNumber]);

  useEffect(() => {
    if (error) {
      collectError(error);
    }
  }, [error]);

  useFocusScreen(() => {
    if (refetch && data?.mobileAppVersion) {
      refetch();
    }
  });

  const values = useMemo(() => {
    const currentBuildNumber = parseInt(Application.nativeBuildVersion ?? "0");
    const isOutdated = appVersionData.build_version > currentBuildNumber;

    return {
      ...appVersionData,
      appOutDated: isOutdated,
    };
  }, [appVersionData]);

  return (
    <AppVersionContext.Provider value={values}>
      {children}
    </AppVersionContext.Provider>
  );
};

export const useAppVersion = (): AppVersionData => {
  const context = React.useContext(AppVersionContext);
  if (context === undefined) {
    throw new Error("useAppVersion must be used within a AppVersionProvider");
  }
  return context;
};
```

### After: Zustand Store + Custom Hooks

**Store**: `luce-fe/src/store/app-version/appVersionStore.ts`

```typescript
import { create } from "zustand";
import { log } from "@/lib/store-middleware";
import * as Application from "expo-application";
import { collectError } from "@/lib/monitoring/sentry";

type AppVersionState = {
  build_version: number;
  appOutDated: boolean;
  isLoading: boolean;
  error: Error | null;
};

type AppVersionActions = {
  setBuildVersion: (version: number) => void;
  setAppOutDated: (outdated: boolean) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: Error | null) => void;
};

export const useAppVersionStore = create(
  log<
    {
      data: AppVersionState;
    } & AppVersionActions
  >((set, get) => ({
    data: {
      build_version: Number(Application.nativeBuildVersion),
      appOutDated: false,
      isLoading: false,
      error: null,
    },
    setBuildVersion: (version: number) => {
      set({ data: { ...get().data, build_version: version } });
    },
    setAppOutDated: (outdated: boolean) => {
      set({ data: { ...get().data, appOutDated: outdated } });
    },
    setLoading: (loading: boolean) => {
      set({ data: { ...get().data, isLoading: loading } });
    },
    setError: (error: Error | null) => {
      set({ data: { ...get().data, error } });
    },
  }))
);
```

**Custom Hook**: `luce-fe/src/components/hooks/use-app-version/index.tsx`

```typescript
import { useAppVersionStore } from "@/store/app-version/appVersionStore";

export function useAppVersion() {
  const build_version = useAppVersionStore((state) => state.data.build_version);
  const appOutDated = useAppVersionStore((state) => state.data.appOutDated);

  return {
    build_version,
    appOutDated,
  };
}
```

**Version Check Hook**: `luce-fe/src/components/hooks/use-app-version-check/index.tsx`

```typescript
import { useEffect, useState } from "react";
import * as Application from "expo-application";
import { useQuery } from "@apollo/client";
import { MobileAppVersionDocument } from "@/__generated__/graphql";
import { useFocusScreen } from "../use-focus-screen";
import { collectError } from "@/lib/monitoring/sentry";
import { useAppVersionStore } from "@/store/app-version/appVersionStore";

export function useAppVersionCheck() {
  const [initDone, setInitDone] = useState(false);
  const { setBuildVersion, setError, setAppOutDated } = useAppVersionStore();
  const { data, error, refetch } = useQuery(MobileAppVersionDocument);

  useEffect(() => {
    if (data?.mobileAppVersion?.clientAppBuildNumber) {
      setBuildVersion(data.mobileAppVersion.clientAppBuildNumber);
      setInitDone(true);
    }
  }, [data?.mobileAppVersion?.clientAppBuildNumber, setBuildVersion]);

  useEffect(() => {
    if (error) {
      collectError(error);
      setError(error);
    }
  }, [error, setError]);

  useFocusScreen(() => {
    if (refetch && data?.mobileAppVersion) {
      refetch();
    }
  });

  const build_version = useAppVersionStore((state) => state.data.build_version);

  useEffect(() => {
    const currentBuildNumber = parseInt(Application.nativeBuildVersion ?? "0");
    const isOutdated = build_version > currentBuildNumber;
    setAppOutDated(isOutdated);
  }, [build_version, setAppOutDated]);

  return { initDone };
}
```

**Updated \_layout.tsx**:

```typescript:luce-fe/src/app/_layout.tsx
// Before
<AppVersionProvider>
  <AuthProvider>
    <NotificationsProvider>
      <SafeAreaProvider>
        {/* ... */}
      </SafeAreaProvider>
    </NotificationsProvider>
  </AuthProvider>
</AppVersionProvider>

// After
function RootLayout() {
  const { initDone: appVersionInitDone } = useAppVersionCheck();
  const { showSplash } = useSplashScreen(
    loaded,
    startupInitDone,
    authInitDone,
    appVersionInitDone,
    notificationsInitDone,
  );

  return (
    <PostHogProvider client={posthogInstance}>
      <ApolloProvider client={ClientInstance}>
        <SafeAreaProvider>
          {/* ... */}
        </SafeAreaProvider>
      </ApolloProvider>
    </PostHogProvider>
  );
}
```

### Usage Examples

**Example 1: Checking App Version in Components**

```typescript
import { View, Button } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useAppVersionStore } from "@/store/app-version/appVersionStore";
import { __ } from "@/lib/i18n";

function AppVersionInfo() {
  const buildVersion = useAppVersionStore((state) => state.data.build_version);
  const appOutDated = useAppVersionStore((state) => state.data.appOutDated);

  return (
    <View className="flex flex-col gap-4">
      <Typography variant="body-md">
        {__("Build Version")}: {buildVersion}
      </Typography>
      {appOutDated && (
        <View className="p-4 border border-warning rounded-lg">
          <Typography variant="body-md" color="warning">
            {__("Your app is outdated. Please update to the latest version.")}
          </Typography>
          <Button
            variant="primary"
            onClick={() => {
              // Open app store
            }}
            children={__("Update Now")}
          />
        </View>
      )}
    </View>
  );
}
```

**Example 2: Showing Update Prompt**

```typescript
import { useEffect } from "react";
import { useAppVersionStore } from "@/store/app-version/appVersionStore";
import { useGlobalModalStore } from "@/store/modal/globalModalStore";
import { UpdateRequiredModal } from "@/components/shared/update-required-modal";

function AppVersionChecker() {
  const appOutDated = useAppVersionStore((state) => state.data.appOutDated);
  const openModal = useGlobalModalStore((state) => state.openModal);

  useEffect(() => {
    if (appOutDated) {
      openModal(<UpdateRequiredModal />, { type: "dialog" });
    }
  }, [appOutDated, openModal]);

  return null;
}
```

**Example 3: Accessing Version Data**

```typescript
import { View } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useAppVersionStore } from "@/store/app-version/appVersionStore";
import { __ } from "@/lib/i18n";

function AboutPage() {
  const { build_version, appOutDated } = useAppVersionStore((state) => state.data);

  return (
    <View className="flex flex-col gap-4">
      <Typography variant="h1">{__("About")}</Typography>
      <Typography variant="body-md">
        {__("Version")}: {build_version}
      </Typography>
      {appOutDated && (
        <Typography variant="body-sm" color="warning">
          {__("Update available")}
        </Typography>
      )}
    </View>
  );
}
```

## Migration Summary

**Code Reduction**:

- **Before**: Provider component with React Context (~100 lines)
- **After**: Zustand store (~80 lines) + hook (~50 lines) = ~130 lines
- **Net change**: Slight increase but better architecture

**Benefits**:

- ✅ Consistent state management pattern
- ✅ No provider nesting
- ✅ Better performance with Zustand subscriptions
- ✅ Reusable hooks

**Migration Strategy**:

1. Create `useAppVersionStore` Zustand store
2. Create `useAppVersionCheck` hook for initialization
3. Update `_layout.tsx` to use hook instead of provider
4. Update all components using AppVersionContext
5. Remove `AppVersionProvider` component

## Auth Provider to Zustand Store Migration

## Summary

This refactoring converts `AuthProvider` (React Context) to `useAuthStore` (Zustand store), following the existing store pattern in the codebase. Session restoration logic is extracted to `useAuthSession` hook, and initialization status is passed to `useSplashScreen`.

### Files Affected

**New Files:**

- `store/auth/authStore.ts` - Zustand store for auth state
- `components/hooks/use-auth/index.tsx` - Custom hook (replaces `useAuth()`)
- `components/hooks/use-auth-session/index.tsx` - Session restoration hook

**Removed Files:**

- `components/shared/auth-provider/index.tsx` - Provider component removed

**Modified Files:**

- `app/_layout.tsx` - Remove AuthProvider, add useAuthSession hook
- `app-web/router.tsx` - Remove AuthProvider, add useAuthSession hook
- All files using `useAuth()` from auth-provider - Update to use new hook
- `components/hooks/use-splash-screen/index.tsx` - Add authInitDone parameter

### Key Changes

- **AuthProvider** → `useAuthStore` (Zustand store)
- **Session restoration logic** extracted to `useAuthSession` hook
- **Provider component removed** from `_layout.tsx`
- **Initialization status** passed to `useSplashScreen`

### Benefits

- **Consistent state management** - Uses Zustand pattern like other stores
- **No provider nesting** - Eliminates React Context provider
- **Better performance** - Zustand subscriptions are more efficient
- **Centralized initialization** - Session check logic in `useSplashScreen`
- **Reusable hooks** - Session logic can be used anywhere

## Implementation Realization

### Before: AuthProvider with React Context

```typescript:luce-fe/src/components/shared/auth-provider/index.tsx
import { createContext, useContext, useState, useEffect } from "react";
import { useApolloClient, useMutation } from "@apollo/client";
import { SignOutDocument } from "@/__generated__/graphql";
import { tokenManager } from "@/lib/token-manager";
import { storage } from "@/lib/storage";
import { useClientStore } from "@/store/auth/client";
import { useClientUnreadNotificationCountQuery } from "@/gqlhooks/notification/useClientUnreadNotificationCountQuery";

const CLIENT_ID_KEY = "clientId";

type AuthSession = {
  login: (args: LoginArgs) => void;
  logout: () => Promise<void>;
  isLoading: boolean;
  isAuthenticated: boolean;
  isGuest: boolean;
  clientId: string;
};

const AuthContext = createContext<AuthSession>({
  login: () => {},
  logout: async () => {},
  isLoading: true,
  isAuthenticated: false,
  isGuest: true,
  clientId: "",
});

export function useAuth(): AuthSession {
  return useContext<AuthSession>(AuthContext);
}

export const AuthProvider = ({ children }: Props) => {
  const apolloClient = useApolloClient();
  const [isLoading, setLoading] = useState<boolean>(true);
  const [isGuest, setGuest] = useState<boolean>(true);
  const [clientId, setClientId] = useState<string>("");
  const [_signOut, { loading: logoutLoading }] = useMutation(SignOutDocument);

  useClientUnreadNotificationCountQuery(isGuest);

  const restoreSession = async (): Promise<void> => {
    try {
      const _clientId = await storage.getItem<string>(CLIENT_ID_KEY);
      if (_clientId) {
        setClientId(_clientId);
      }
      const result = await tokenManager.isTokenAvailable();
      if (result) {
        setGuest(false);
        await useClientStore.getState().fetch({
          requestPayload: { id: _clientId },
        });
      }
    } catch (error) {
      console.log("session: restore session", error);
    }
    setLoading(false);
  };

  const logoutCallback = (): void => {
    setLoading(true);
    setGuest(true);
    tokenManager.clearTokens();
    apolloClient.clearStore();
    useClientStore.getState().clear();
    useVisitStore.getState().clear();
    usePackageListStore.getState().clear();
    usePackageStore.getState().clear();
    useUnpaidInvoicesStore.getState().clear();
    usePaidInvoicesStore.getState().clear();
    const { setUserInfo, setUnreadNotificationCount } = useAuthState.getState();
    setUserInfo(DEFAULT_USER_DATA);
    setUnreadNotificationCount(0);
    storage.deleteAll();
    setLoading(false);
  };

  useEffect(() => {
    tokenManager.setLogoutCallback(logoutCallback);
    restoreSession();
  }, []);

  const login = ({ jwt, refreshToken, clientId }: LoginArgs): void => {
    tokenManager.setTokens({ token: jwt, refreshToken });
    storage.setItem(CLIENT_ID_KEY, clientId);
    setClientId(clientId);
    setGuest(false);
    setLoading(false);
    useClientStore.getState().fetch({
      requestPayload: { id: clientId },
    });
  };

  const logout = async (): Promise<void> => {
    try {
      await _signOut();
    } catch (error) {
      console.log("session: logout error", error);
    } finally {
      logoutCallback();
      resetUserContext();
      resetAnalyticUser();
    }
  };

  return (
    <AuthContext.Provider
      value={{
        isLoading,
        isGuest,
        isAuthenticated: !isGuest,
        login,
        logout,
        clientId,
      }}
    >
      <LoadingDialog open={logoutLoading} />
      {children}
    </AuthContext.Provider>
  );
};
```

### After: Zustand Store + Custom Hooks

**Store**: `luce-fe/src/store/auth/authStore.ts`

```typescript
import { create } from "zustand";
import { log } from "@/lib/store-middleware";
import { tokenManager } from "@/lib/token-manager";
import { storage } from "@/lib/storage";
import { useClientStore } from "@/store/auth/client";
import { useVisitStore } from "@/store/visits/visitStore";
import { usePackageListStore } from "@/store/packages/packageListStore";
import { usePackageStore } from "@/store/packages/packageStore";
import { useUnpaidInvoicesStore } from "@/store/invoices/unpaidInvoices";
import { usePaidInvoicesStore } from "@/store/invoices/paidInvoices";
import { useAuthState } from "@/store/auth";
import { DEFAULT_USER_DATA } from "@/constants";

const CLIENT_ID_KEY = "clientId";

type AuthStoreState = {
  isLoading: boolean;
  isGuest: boolean;
  clientId: string;
  logoutLoading: boolean;
};

type AuthStoreActions = {
  setLoading: (loading: boolean) => void;
  setGuest: (isGuest: boolean) => void;
  setClientId: (clientId: string) => void;
  setLogoutLoading: (loading: boolean) => void;
  login: (args: LoginArgs) => void;
  logout: () => Promise<void>;
  logoutCallback: () => void;
  restoreSession: () => Promise<void>;
};

export const useAuthStore = create(
  log<
    {
      data: AuthStoreState;
    } & AuthStoreActions
  >((set, get) => ({
    data: {
      isLoading: true,
      isGuest: true,
      clientId: "",
      logoutLoading: false,
    },
    setLoading: (loading: boolean) => {
      set({ data: { ...get().data, isLoading: loading } });
    },
    setGuest: (isGuest: boolean) => {
      set({ data: { ...get().data, isGuest } });
    },
    setClientId: (clientId: string) => {
      set({ data: { ...get().data, clientId } });
    },
    setLogoutLoading: (loading: boolean) => {
      set({ data: { ...get().data, logoutLoading: loading } });
    },
    login: ({ jwt, refreshToken, clientId }: LoginArgs) => {
      tokenManager.setTokens({ token: jwt, refreshToken });
      storage.setItem(CLIENT_ID_KEY, clientId);
      set({
        data: {
          ...get().data,
          isGuest: false,
          isLoading: false,
          clientId,
        },
      });
      useClientStore.getState().fetch({ requestPayload: { id: clientId } });
    },
    logout: async () => {
      const { setLogoutLoading, logoutCallback } = get();
      setLogoutLoading(true);
      try {
        await signOutMutation();
      } catch (error) {
        console.log("session: logout error", error);
      } finally {
        logoutCallback();
        resetUserContext();
        resetAnalyticUser();
        setLogoutLoading(false);
      }
    },
    logoutCallback: () => {
      set({ data: { ...get().data, isLoading: true, isGuest: true } });
      tokenManager.clearTokens();
      useClientStore.getState().clear();
      useVisitStore.getState().clear();
      usePackageListStore.getState().clear();
      usePackageStore.getState().clear();
      useUnpaidInvoicesStore.getState().clear();
      usePaidInvoicesStore.getState().clear();
      const { setUserInfo, setUnreadNotificationCount } = useAuthState.getState();
      setUserInfo(DEFAULT_USER_DATA);
      setUnreadNotificationCount(0);
      storage.deleteAll();
      set({ data: { ...get().data, isLoading: false } });
    },
    restoreSession: async () => {
      const { setLoading, setGuest, setClientId } = get();
      try {
        const _clientId = await storage.getItem<string>(CLIENT_ID_KEY);
        if (_clientId) {
          setClientId(_clientId);
        }
        const result = await tokenManager.isTokenAvailable();
        if (result) {
          setGuest(false);
          await useClientStore.getState().fetch({
            requestPayload: { id: _clientId },
          });
        }
      } catch (error) {
        console.log("session: restore session", error);
      } finally {
        setLoading(false);
      }
    },
  }))
);
```

**Custom Hook**: `luce-fe/src/components/hooks/use-auth/index.tsx`

```typescript
import { useAuthStore } from "@/store/auth/authStore";

export function useAuth() {
  const isLoading = useAuthStore((state) => state.data.isLoading);
  const isGuest = useAuthStore((state) => state.data.isGuest);
  const isAuthenticated = !isGuest;
  const clientId = useAuthStore((state) => state.data.clientId);
  const login = useAuthStore((state) => state.login);
  const logout = useAuthStore((state) => state.logout);

  return {
    isLoading,
    isGuest,
    isAuthenticated,
    clientId,
    login,
    logout,
  };
}
```

**Session Hook**: `luce-fe/src/components/hooks/use-auth-session/index.tsx`

```typescript
import { useEffect, useState } from "react";
import { useApolloClient } from "@apollo/client";
import { useAuthStore } from "@/store/auth/authStore";
import { tokenManager } from "@/lib/token-manager";
import { useClientUnreadNotificationCountQuery } from "@/gqlhooks/notification/useClientUnreadNotificationCountQuery";

export function useAuthSession() {
  const [initDone, setInitDone] = useState(false);
  const { restoreSession, logoutCallback } = useAuthStore();
  const apolloClient = useApolloClient();

  useEffect(() => {
    const init = async () => {
      tokenManager.setLogoutCallback(() => {
        logoutCallback();
        apolloClient.clearStore();
      });
      await restoreSession();
      setInitDone(true);
    };
    init();
  }, []);

  const isGuest = useAuthStore((state) => state.data.isGuest);
  useClientUnreadNotificationCountQuery(isGuest);

  return { initDone };
}
```

**Updated \_layout.tsx**:

```typescript:luce-fe/src/app/_layout.tsx
// Before
<AppVersionProvider>
  <AuthProvider>
    <NotificationsProvider>
      <SafeAreaProvider>
        {/* ... */}
      </SafeAreaProvider>
    </NotificationsProvider>
  </AuthProvider>
</AppVersionProvider>

// After
function RootLayout() {
  const { initDone: authInitDone } = useAuthSession();
  const { showSplash } = useSplashScreen(
    loaded,
    startupInitDone,
    authInitDone,
    appVersionInitDone,
    notificationsInitDone,
  );

  return (
    <PostHogProvider client={posthogInstance}>
      <ApolloProvider client={ClientInstance}>
        <SafeAreaProvider>
          {/* ... */}
        </SafeAreaProvider>
      </ApolloProvider>
    </PostHogProvider>
  );
}
```

### Usage Examples

**Example 1: Using Auth Hook in Components**

```typescript
import { View, Skeleton } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useAuth } from "@/components/hooks/use-auth";
import { LoginPrompt } from "@/components/shared/login-prompt";

function ProtectedComponent() {
  const { isAuthenticated, isLoading, isGuest, clientId } = useAuth();

  if (isLoading) {
    return (
      <>
        <Skeleton className="h-[200px] w-full" />
      </>
    );
  }
  if (isGuest) return <LoginPrompt />;

  return (
    <View>
      <Typography variant="h1">Welcome! Client ID: {clientId}</Typography>
      {/* Protected content */}
    </View>
  );
}
```

**Example 2: Login Flow**

```typescript
import { View, Button } from "@/components/ui";
import { useAuth } from "@/components/hooks/use-auth";
import { useSignInMutation } from "@/gqlhooks/auth/useSignInMutation";
import { showToast } from "@/components/ui/toast/show-toast";
import { getErrorMessage } from "@/lib/helpers/string";
import { __ } from "@/lib/i18n";

function LoginForm() {
  const { login } = useAuth();
  const [signIn, { loading }] = useSignInMutation();

  const handleLogin = async (email: string, password: string) => {
    try {
      const response = await signIn({
        variables: {
          input: { email, password },
        },
      });

      if (response.data?.signIn) {
        login({
          jwt: response.data.signIn.token,
          refreshToken: response.data.signIn.refreshToken,
          clientId: response.data.signIn.clientId,
        });
      }
    } catch (error) {
      showToast({
        type: "error",
        title: getErrorMessage(error, __("Login failed")),
      });
    }
  };

  return (
    <View>
      {/* Form fields */}
      <Button variant="primary" onClick={() => handleLogin(email, password)} loading={loading} children={__("Login")} />
    </View>
  );
}
```

**Example 3: Logout Flow**

```typescript
import { Button } from "@/components/ui";
import { useAuth } from "@/components/hooks/use-auth";
import { useRoute } from "@/components/shared/router";
import { __ } from "@/lib/i18n";

function LogoutButton() {
  const { logout } = useAuth();
  const { push } = useRoute();

  const handleLogout = async () => {
    await logout();
    push({ pageKey: "login" });
  };

  return <Button variant="tertiary" onClick={handleLogout} children={__("Logout")} />;
}
```

**Example 4: Accessing Auth State Directly from Store**

```typescript
import { View } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useAuthStore } from "@/store/auth/authStore";

function SomeComponent() {
  // Direct store access if needed
  const clientId = useAuthStore((state) => state.data.clientId);
  const isGuest = useAuthStore((state) => state.data.isGuest);

  return (
    <View>
      <Typography variant="body-md">Client: {clientId}</Typography>
      <Typography variant="body-sm">Guest: {isGuest ? "Yes" : "No"}</Typography>
    </View>
  );
}
```

## Migration Summary

**Code Reduction**:

- **Before**: Provider component with React Context (~150 lines)
- **After**: Zustand store (~150 lines) + hooks (~50 lines) = ~200 lines
- **Net change**: Slight increase but better architecture and performance

**Benefits**:

- ✅ Consistent state management pattern
- ✅ No provider nesting
- ✅ Better performance with Zustand subscriptions
- ✅ Reusable hooks

**Migration Strategy**:

1. Create `useAuthStore` Zustand store
2. Create `useAuth` hook
3. Create `useAuthSession` hook for initialization
4. Update `_layout.tsx` to use hooks instead of provider
5. Update all components using `useAuth()` from provider
6. Remove `AuthProvider` component

## Notification Provider to Zustand Store Migration

## Summary

This refactoring converts `NotificationsProvider` (React Context) to `useNotificationsStore` (Zustand store), following the existing store pattern in the codebase. Notification initialization logic is extracted to `useNotificationsInit` hook, and initialization status is passed to `useSplashScreen`.

### Files Affected

**New Files:**

- `store/notifications/notificationsStore.ts` - Zustand store for notifications
- `components/hooks/use-notifications/index.tsx` - Custom hook (replaces context usage)
- `components/hooks/use-notifications-init/index.tsx` - Notification initialization hook

**Removed Files:**

- `containers/notifications/notification-provider/index.native.tsx` - Provider component removed
- `containers/notifications/notification-provider/index.web.tsx` - Provider component removed (if exists)
- `containers/notifications/notification-provider/lib.tsx` - Context file removed

**Modified Files:**

- `app/_layout.tsx` - Remove NotificationsProvider, add useNotificationsInit hook
- All files using NotificationsContext - Update to use new hook
- `components/hooks/use-splash-screen/index.tsx` - Add notificationsInitDone parameter

### Key Changes

- **NotificationsProvider** → `useNotificationsStore` (Zustand store)
- **Notification initialization logic** extracted to `useNotificationsInit` hook
- **Provider component removed** from `_layout.tsx`
- **Initialization status** passed to `useSplashScreen`

### Benefits

- **Consistent state management** - Uses Zustand pattern like other stores
- **No provider nesting** - Eliminates React Context provider
- **Better performance** - Zustand subscriptions are more efficient
- **Centralized initialization** - Notification logic in `useSplashScreen`
- **Reusable hooks** - Notification logic can be used anywhere

## Implementation Realization

### Before: NotificationsProvider with React Context

```typescript:luce-fe/src/containers/notifications/notification-provider/index.native.tsx
import type { ReactNode } from "react";
import { useEffect, useState } from "react";
import * as Notifications from "expo-notifications";
import { useUpdateInstallationMutation } from "@/gqlhooks/notification/useUpdateInstallationMutation";
import { Alert, Linking, AppState } from "react-native";
import { getErrorMessage } from "@/lib/helpers/string";
import { registerForPushNotificationsAsync } from "./register";
import { Href, router } from "expo-router";
import { useAuth } from "@/components/shared/auth-provider";
import { useAuthState } from "@/store/auth";
import { NotificationsContext } from "./lib";
import { PosthogAnalytics } from "@/lib/monitoring/posthog";

export const NotificationsProvider = ({ children }: Props) => {
  const [permissionStatus, setPermissionStatus] = useState<Notifications.PermissionStatus | null>(null);
  const [expoPushToken, setExpoPushToken] = useState<string | null>(null);
  const [notification, setNotification] = useState<Notifications.Notification | null>(null);
  const [error, setError] = useState<Error | null>(null);

  const { updateToken } = useUpdateInstallationMutation();
  const { isAuthenticated, clientId } = useAuth();
  const userPreferencePushEnabled = useAuthState(
    (state) => state.data.userInfo.notificationSettings.PUSH_NOTIF,
  );

  const checkPermissionStatus = async () => {
    const { status } = await Notifications.getPermissionsAsync();
    setPermissionStatus(status);
    return status;
  };

  const getTokenIfGranted = async () => {
    try {
      const token = await registerForPushNotificationsAsync();
      setExpoPushToken(token);
      await updateToken(token);
      setError(null);
    } catch (error) {
      setError(error as Error);
      throw error;
    }
  };

  const requestPermissionAndGetToken = async (showAlertOnDenied: boolean) => {
    try {
      const token = await registerForPushNotificationsAsync();
      setExpoPushToken(token);
      await updateToken(token);
      setError(null);
      await checkPermissionStatus();
      return true;
    } catch (error) {
      setError(error as Error);
      await checkPermissionStatus();
      if (showAlertOnDenied) {
        Alert.alert(
          "Please allow notifications",
          "Go to the System Settings > Notifications > Allow Notifications",
          [
            { text: "Go to Settings", onPress: () => Linking.openSettings() },
            { text: "Cancel" },
          ],
        );
      }
      return false;
    }
  };

  const turnOn = async () => {
    const status = await checkPermissionStatus();
    if (status === Notifications.PermissionStatus.GRANTED) {
      try {
        if (!expoPushToken) {
          await getTokenIfGranted();
        }
        return true;
      } catch {
        return false;
      }
    }
    return await requestPermissionAndGetToken(true);
  };

  const turnOff = async () => {
    try {
      await updateToken("");
      setExpoPushToken(null);
    } catch (error: unknown) {
      Alert.alert(
        "Something went wrong!",
        getErrorMessage(error, "Error when disabling notifications. Please try again later."),
      );
    }
  };

  useEffect(() => {
    if (!isAuthenticated || !clientId) return;

    const initialize = async () => {
      const status = await checkPermissionStatus();
      if (status === Notifications.PermissionStatus.UNDETERMINED) {
        await requestPermissionAndGetToken(false);
      } else if (
        status === Notifications.PermissionStatus.GRANTED &&
        userPreferencePushEnabled
      ) {
        try {
          await getTokenIfGranted();
        } catch {
          // Silent fail
        }
      }
    };

    initialize();

    const subscription = AppState.addEventListener("change", (nextAppState) => {
      if (nextAppState === "active") {
        checkPermissionStatus();
      }
    });

    const notificationListener = Notifications.addNotificationReceivedListener(
      (notification) => {
        setNotification(notification);
      },
    );

    const responseListener = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        let navigateTo: Href = "/notifications";
        if ("navigate_to" in response.notification.request.content.data) {
          navigateTo = response.notification.request.content.data["navigate_to"] as Href;
        }
        PosthogAnalytics.track("push_notification_click");
        router.push(navigateTo);
      },
    );

    return () => {
      subscription.remove();
      notificationListener.remove();
      responseListener.remove();
    };
  }, [isAuthenticated, clientId]);

  useEffect(() => {
    if (!isAuthenticated || !clientId) return;

    const syncWithBackend = async () => {
      const status = permissionStatus;
      if (status !== Notifications.PermissionStatus.GRANTED) {
        return;
      }
      if (userPreferencePushEnabled && !expoPushToken) {
        try {
          await getTokenIfGranted();
        } catch {
          // Silent fail
        }
      } else if (!userPreferencePushEnabled && expoPushToken) {
        await turnOff();
      }
    };

    syncWithBackend();
  }, [userPreferencePushEnabled, permissionStatus, isAuthenticated, clientId]);

  return (
    <NotificationsContext.Provider
      value={{
        enabled: userPreferencePushEnabled && permissionStatus === Notifications.PermissionStatus.GRANTED,
        permissionStatus,
        turnOn,
        turnOff,
        expoPushToken,
        latestNotification: notification,
        error,
      }}
    >
      {children}
    </NotificationsContext.Provider>
  );
};
```

### After: Zustand Store + Custom Hooks

**Store**: `luce-fe/src/store/notifications/notificationsStore.ts`

```typescript
import { create } from "zustand";
import { log } from "@/lib/store-middleware";
import * as Notifications from "expo-notifications";
import { Alert, Linking } from "react-native";
import { getErrorMessage } from "@/lib/helpers/string";
import { registerForPushNotificationsAsync } from "@/containers/notifications/notification-provider/register";

type NotificationsState = {
  enabled: boolean;
  permissionStatus: Notifications.PermissionStatus | null;
  expoPushToken: string | null;
  latestNotification: Notifications.Notification | null;
  error: Error | null;
};

type NotificationsActions = {
  setEnabled: (enabled: boolean) => void;
  setPermissionStatus: (status: Notifications.PermissionStatus | null) => void;
  setExpoPushToken: (token: string | null) => void;
  setLatestNotification: (notification: Notifications.Notification | null) => void;
  setError: (error: Error | null) => void;
  turnOn: () => Promise<boolean>;
  turnOff: () => Promise<void>;
  checkPermissionStatus: () => Promise<Notifications.PermissionStatus>;
  getTokenIfGranted: () => Promise<string | null>;
  requestPermissionAndGetToken: (showAlertOnDenied: boolean) => Promise<boolean>;
};

export const useNotificationsStore = create(
  log<
    {
      data: NotificationsState;
    } & NotificationsActions
  >((set, get) => ({
    data: {
      enabled: false,
      permissionStatus: null,
      expoPushToken: null,
      latestNotification: null,
      error: null,
    },
    setEnabled: (enabled: boolean) => {
      set({ data: { ...get().data, enabled } });
    },
    setPermissionStatus: (status: Notifications.PermissionStatus | null) => {
      set({ data: { ...get().data, permissionStatus: status } });
    },
    setExpoPushToken: (token: string | null) => {
      set({ data: { ...get().data, expoPushToken: token } });
    },
    setLatestNotification: (notification: Notifications.Notification | null) => {
      set({ data: { ...get().data, latestNotification: notification } });
    },
    setError: (error: Error | null) => {
      set({ data: { ...get().data, error } });
    },
    checkPermissionStatus: async () => {
      const { status } = await Notifications.getPermissionsAsync();
      get().setPermissionStatus(status);
      return status;
    },
    getTokenIfGranted: async () => {
      try {
        const token = await registerForPushNotificationsAsync();
        get().setExpoPushToken(token);
        get().setError(null);
        return token;
      } catch (error) {
        get().setError(error as Error);
        throw error;
      }
    },
    requestPermissionAndGetToken: async (showAlertOnDenied: boolean) => {
      try {
        const token = await registerForPushNotificationsAsync();
        get().setExpoPushToken(token);
        get().setError(null);
        await get().checkPermissionStatus();
        return true;
      } catch (error) {
        get().setError(error as Error);
        await get().checkPermissionStatus();
        if (showAlertOnDenied) {
          Alert.alert("Please allow notifications", "Go to the System Settings > Notifications > Allow Notifications", [
            { text: "Go to Settings", onPress: () => Linking.openSettings() },
            { text: "Cancel" },
          ]);
        }
        return false;
      }
    },
    turnOn: async () => {
      const status = await get().checkPermissionStatus();
      if (status === Notifications.PermissionStatus.GRANTED) {
        try {
          if (!get().data.expoPushToken) {
            await get().getTokenIfGranted();
          }
          return true;
        } catch {
          return false;
        }
      }
      return await get().requestPermissionAndGetToken(true);
    },
    turnOff: async () => {
      try {
        get().setExpoPushToken(null);
      } catch (error: unknown) {
        Alert.alert(
          "Something went wrong!",
          getErrorMessage(error, "Error when disabling notifications. Please try again later.")
        );
      }
    },
  }))
);
```

**Custom Hook**: `luce-fe/src/components/hooks/use-notifications/index.tsx`

```typescript
import { useNotificationsStore } from "@/store/notifications/notificationsStore";
import { useAuthState } from "@/store/auth";
import * as Notifications from "expo-notifications";

export function useNotifications() {
  const enabled = useNotificationsStore((state) => state.data.enabled);
  const permissionStatus = useNotificationsStore((state) => state.data.permissionStatus);
  const expoPushToken = useNotificationsStore((state) => state.data.expoPushToken);
  const latestNotification = useNotificationsStore((state) => state.data.latestNotification);
  const error = useNotificationsStore((state) => state.data.error);
  const turnOn = useNotificationsStore((state) => state.turnOn);
  const turnOff = useNotificationsStore((state) => state.turnOff);

  const userPreferencePushEnabled = useAuthState((state) => state.data.userInfo.notificationSettings.PUSH_NOTIF);
  const isEnabled = userPreferencePushEnabled && permissionStatus === Notifications.PermissionStatus.GRANTED;

  return {
    enabled: isEnabled,
    permissionStatus,
    turnOn,
    turnOff,
    expoPushToken,
    latestNotification,
    error,
  };
}
```

**Initialization Hook**: `luce-fe/src/components/hooks/use-notifications-init/index.tsx`

```typescript
import { useEffect, useState } from "react";
import * as Notifications from "expo-notifications";
import { AppState, Alert, Linking } from "react-native";
import { Href, router } from "expo-router";
import { useNotificationsStore } from "@/store/notifications/notificationsStore";
import { useAuth } from "@/components/hooks/use-auth";
import { useAuthState } from "@/store/auth";
import { useUpdateInstallationMutation } from "@/gqlhooks/notification/useUpdateInstallationMutation";
import { PosthogAnalytics } from "@/lib/monitoring/posthog";

export function useNotificationsInit() {
  const [initDone, setInitDone] = useState(false);
  const {
    checkPermissionStatus,
    getTokenIfGranted,
    requestPermissionAndGetToken,
    turnOff,
    setLatestNotification,
    setEnabled,
  } = useNotificationsStore();

  const { isAuthenticated, clientId } = useAuth();
  const userPreferencePushEnabled = useAuthState((state) => state.data.userInfo.notificationSettings.PUSH_NOTIF);
  const permissionStatus = useNotificationsStore((state) => state.data.permissionStatus);
  const expoPushToken = useNotificationsStore((state) => state.data.expoPushToken);
  const { updateToken } = useUpdateInstallationMutation();

  useEffect(() => {
    if (!isAuthenticated || !clientId) {
      setInitDone(true);
      return;
    }

    const initialize = async () => {
      const status = await checkPermissionStatus();

      if (status === Notifications.PermissionStatus.UNDETERMINED) {
        const token = await requestPermissionAndGetToken(false);
        if (token) {
          await updateToken(token);
        }
      } else if (status === Notifications.PermissionStatus.GRANTED && userPreferencePushEnabled) {
        try {
          const token = await getTokenIfGranted();
          if (token) {
            await updateToken(token);
          }
        } catch {
          // Silent fail
        }
      }
      setInitDone(true);
    };

    initialize();

    Notifications.setNotificationHandler({
      handleNotification: () => {
        return Promise.resolve({
          shouldPlaySound: userPreferencePushEnabled,
          shouldSetBadge: userPreferencePushEnabled,
          shouldShowBanner: userPreferencePushEnabled,
          shouldShowList: userPreferencePushEnabled,
        });
      },
    });

    const subscription = AppState.addEventListener("change", (nextAppState) => {
      if (nextAppState === "active") {
        checkPermissionStatus();
      }
    });

    const notificationListener = Notifications.addNotificationReceivedListener((notification) => {
      setLatestNotification(notification);
    });

    const responseListener = Notifications.addNotificationResponseReceivedListener((response) => {
      let navigateTo: Href = "/notifications";
      if ("navigate_to" in response.notification.request.content.data) {
        navigateTo = response.notification.request.content.data["navigate_to"] as Href;
      }
      PosthogAnalytics.track("push_notification_click");
      router.push(navigateTo);
    });

    return () => {
      subscription.remove();
      notificationListener.remove();
      responseListener.remove();
    };
  }, [isAuthenticated, clientId]);

  useEffect(() => {
    if (!isAuthenticated || !clientId) return;

    const syncWithBackend = async () => {
      if (permissionStatus !== Notifications.PermissionStatus.GRANTED) {
        return;
      }

      if (userPreferencePushEnabled && !expoPushToken) {
        try {
          const token = await getTokenIfGranted();
          if (token) {
            await updateToken(token);
          }
        } catch {
          // Silent fail
        }
      } else if (!userPreferencePushEnabled && expoPushToken) {
        await updateToken("");
        await turnOff();
      }
    };

    syncWithBackend();
  }, [userPreferencePushEnabled, permissionStatus, isAuthenticated, clientId, expoPushToken]);

  useEffect(() => {
    const enabled = userPreferencePushEnabled && permissionStatus === Notifications.PermissionStatus.GRANTED;
    setEnabled(enabled);
  }, [userPreferencePushEnabled, permissionStatus, setEnabled]);

  return { initDone };
}
```

### Usage Examples

**Example 1: Using Notifications Hook in Components**

```typescript
import { View, Button } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useNotifications } from "@/components/hooks/use-notifications";
import { __ } from "@/lib/i18n";

function NotificationSettings() {
  const { enabled, permissionStatus, turnOn, turnOff } = useNotifications();

  return (
    <View className="flex flex-col gap-4">
      <Typography variant="h2">{__("Notification Settings")}</Typography>
      <Typography variant="body-md">
        {__("Status")}: {permissionStatus}
      </Typography>
      <Typography variant="body-md">
        {__("Enabled")}: {enabled ? __("Yes") : __("No")}
      </Typography>

      {!enabled ? (
        <Button variant="primary" onClick={turnOn} children={__("Enable Notifications")} />
      ) : (
        <Button variant="tertiary" onClick={turnOff} children={__("Disable Notifications")} />
      )}
    </View>
  );
}
```

**Example 2: Checking Notification Permission**

```typescript
import { View, Button } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useNotifications } from "@/components/hooks/use-notifications";
import * as Notifications from "expo-notifications";
import { __ } from "@/lib/i18n";

function NotificationButton() {
  const { enabled, permissionStatus } = useNotifications();

  if (permissionStatus === Notifications.PermissionStatus.DENIED) {
    return (
      <Typography variant="body-sm" color="muted-foreground">
        {__("Notifications are disabled. Please enable in settings.")}
      </Typography>
    );
  }

  return (
    <Button
      variant="primary"
      disabled={!enabled}
      children={enabled ? __("Send Notification") : __("Enable Notifications First")}
    />
  );
}
```

**Example 3: Accessing Latest Notification**

```typescript
import { View } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useNotifications } from "@/components/hooks/use-notifications";

function NotificationBadge() {
  const { latestNotification } = useNotifications();

  if (!latestNotification) return null;

  return (
    <View className="p-4 border rounded-lg">
      <Typography variant="body-md">{latestNotification.request.content.title}</Typography>
      <Typography variant="body-sm" color="muted-foreground">
        {latestNotification.request.content.body}
      </Typography>
    </View>
  );
}
```

**Example 4: Direct Store Access**

```typescript
import { View, Button } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useNotificationsStore } from "@/store/notifications/notificationsStore";
import { __ } from "@/lib/i18n";

function CustomNotificationComponent() {
  const expoPushToken = useNotificationsStore((state) => state.data.expoPushToken);
  const checkPermission = useNotificationsStore((state) => state.checkPermissionStatus);

  const handleCheck = async () => {
    const status = await checkPermission();
    console.log("Permission status:", status);
  };

  return (
    <View className="flex flex-col gap-2">
      <Typography variant="body-md">
        {__("Push Token")}: {expoPushToken || __("Not available")}
      </Typography>
      <Button variant="tertiary" onClick={handleCheck} children={__("Check Permission")} />
    </View>
  );
}
```

**Updated \_layout.tsx**:

```typescript:luce-fe/src/app/_layout.tsx
// Before
<AppVersionProvider>
  <AuthProvider>
    <NotificationsProvider>
      <SafeAreaProvider>
        {/* ... */}
      </SafeAreaProvider>
    </NotificationsProvider>
  </AuthProvider>
</AppVersionProvider>

// After
function RootLayout() {
  const { initDone: notificationsInitDone } = useNotificationsInit();
  const { showSplash } = useSplashScreen(
    loaded,
    startupInitDone,
    authInitDone,
    appVersionInitDone,
    notificationsInitDone,
  );

  return (
    <PostHogProvider client={posthogInstance}>
      <ApolloProvider client={ClientInstance}>
        <SafeAreaProvider>
          {/* ... */}
        </SafeAreaProvider>
      </ApolloProvider>
    </PostHogProvider>
  );
}
```

---

## GraphQL Query Simplification with Codegen

## Summary

This refactoring simplifies GraphQL query implementation by replacing the custom `createRequestFactory` abstraction with direct `useQuery` hooks from Apollo Client, leveraging GraphQL Codegen's generated typed hooks. This reduces boilerplate from ~20-30 lines per query to just 2-3 lines.

**⚠️ Note**: This proposal demonstrates the pattern and method. We're not proposing to change all files immediately, but to adopt this approach for new queries and gradually migrate existing ones.

### Files Affected

**New Files:**

- `gqlhooks/` - All GraphQL hooks will be organized in this folder (flat structure is acceptable)
- `lib/graphql/defaultOptions.ts` - Default Apollo Client query options configuration (optional)

**Modified Files:**

- `lib/graphql/apollo.ts` - Add default options to Apollo Client
- Individual query stores (gradual migration) - Replace `createRequestFactory` with `useQuery` hooks in `gqlhooks/`

**Removed Files (Future):**

- `lib/request-factory.tsx` - Can be removed once all queries are migrated (397 lines)

### Key Changes

- **Direct `useQuery` usage** instead of Zustand stores with `createRequestFactory`
- **Folderization**: All hooks organized in `gqlhooks/` folder (flat structure is acceptable - simplicity over complex folder hierarchies)
- **Default options configuration** at Apollo Client level for consistent behavior
- **Type-safe hooks** from GraphQL Codegen (already generated)
- **Simplified error handling** using Apollo Client's built-in error handling
- **Reduced boilerplate** from ~20-30 lines to 2-3 lines per query
- **Community-standard patterns** - Uses the most common Apollo Client patterns that any React developer can understand immediately

### Benefits

- **~90% code reduction** per query (20-30 lines → 2-3 lines)
- **Eliminate custom abstraction** - use Apollo Client directly as intended
- **Better type safety** with Codegen-generated hooks
- **Standard Apollo patterns** - uses the most common patterns in the React/GraphQL community
- **Easy to understand** - any developer familiar with Apollo Client can read and understand the code immediately
- **Predictable structure** - all hooks in `gqlhooks/` folder, easy to find and maintain
- **Maintainable** - follows community best practices, easier to onboard new developers
- **Automatic caching** and refetching from Apollo Client
- **Future removal** of 397-line `request-factory.tsx` file

**Important Principles**:

- **Simplicity over complexity** - flat folder structure in `gqlhooks/` is acceptable
- **Predictability** - consistent pattern across all queries makes code easy to navigate
- **Maintainability** - standard patterns mean less custom code to maintain
- **Community alignment** - using common patterns means better documentation, examples, and community support

## Implementation Realization

### Before: Using createRequestFactory

**Example 1: Simple Query (Visit Detail)**

```typescript:luce-fe/src/store/visits/visitStore.ts
import { useFragment } from "@/__generated__";
import type { VisitQuery, VisitQueryVariables } from "@/__generated__/graphql";
import {
  VisitDocument,
  VisitClientFragmentFragmentDoc,
} from "@/__generated__/graphql";
import type { VisitDetailData } from "@/components/shared/visits/visit-detail";
import { mapToVisitDetailData } from "@/components/shared/visits/visit-detail";
import { createRequestFactory } from "@/lib/request-factory";

export const useVisitStore = createRequestFactory<
  VisitQuery,
  VisitDetailData,
  VisitQueryVariables
>({
  method: "query",
  graphqlDocument: VisitDocument,
  transformFunction(data) {
    const visit = useFragment(VisitClientFragmentFragmentDoc, data.visit);
    return mapToVisitDetailData(visit);
  },
});
```

**Example 2: Intermediate Query (Client Confirm Phone Number)**

```typescript:luce-fe/src/store/auth/clientConfirmPhoneNumber.ts
import { createRequestFactory } from "@/lib/request-factory";
import {
  ClientConfirmPhoneNumberDocument,
  type ClientConfirmPhoneNumberMutationVariables,
  type ClientConfirmPhoneNumberMutation,
} from "@/__generated__/graphql";
import { storage } from "@/lib/storage";

type Response = {
  success: boolean;
};

export const useClientConfirmPhoneNumberStore = createRequestFactory<
  ClientConfirmPhoneNumberMutation,
  Response,
  ClientConfirmPhoneNumberMutationVariables
>({
  method: "mutation",
  graphqlDocument: ClientConfirmPhoneNumberDocument,
  transformFunction: (res) => {
    return { success: !!res.clientConfirmPhoneNumber?.result };
  },
  onFetchSuccess({ success }) {
    if (success) {
      storage.setItem("ONBOARDING", "true").catch((e) => {
        console.log("unable to set ONBOARDING. Error:", e);
      });
    }
  },
});
```

**Example 3: Paginated Query (Paid Invoices)**

```typescript:luce-fe/src/store/invoices/paidInvoices.ts
import type {
  InvoicesByFiltersQuery,
  InvoicesByFiltersQueryVariables,
} from "@/__generated__/graphql";
import { InvoicesByFiltersDocument } from "@/__generated__/graphql";
import type { InvoiceCardData } from "@/components/shared/invoices/invoice-card";
import { mapToInvoiceCard } from "@/components/shared/invoices/invoice-card";
import { createPaginatedFactory } from "@/lib/request-factory";

export const usePaidInvoicesStore = createPaginatedFactory<
  InvoicesByFiltersQuery,
  InvoiceCardData,
  InvoicesByFiltersQueryVariables
>({
  method: "query",
  graphqlDocument: InvoicesByFiltersDocument,
  fetchPolicy: "network-only",
  getCountFromResponse: (res) => {
    return res.invoicesByFilters.count;
  },
  transformFunction(data) {
    return data.invoicesByFilters.invoices.map(mapToInvoiceCard);
  },
});
```

### After: Using useQuery Directly

**Example 1: Simple Query (Visit Detail)**

```typescript:luce-fe/src/gqlhooks/visits/useVisit.ts
import { useQuery } from "@apollo/client";
import { VisitDocument, type VisitQueryVariables } from "@/__generated__/graphql";
import { useFragment } from "@/__generated__";
import { VisitClientFragmentFragmentDoc } from "@/__generated__/graphql";
import type { VisitDetailData } from "@/components/shared/visits/visit-detail";
import { mapToVisitDetailData } from "@/components/shared/visits/visit-detail";

export function useVisit(variables: VisitQueryVariables) {
  const { data, loading, error } = useQuery(VisitDocument, { variables });
  const visit = data?.visit ? useFragment(VisitClientFragmentFragmentDoc, data.visit) : null;
  return { data: visit ? mapToVisitDetailData(visit) : null, loading, error };
}
```

**Example 2: Intermediate Query (Client Confirm Phone Number)**

```typescript:luce-fe/src/gqlhooks/auth/useClientConfirmPhoneNumber.ts
import { useMutation } from "@apollo/client";
import { ClientConfirmPhoneNumberDocument } from "@/__generated__/graphql";
import { storage } from "@/lib/storage";

export function useClientConfirmPhoneNumber() {
  const [mutate, { loading, error }] = useMutation(ClientConfirmPhoneNumberDocument, {
    onCompleted: (data) => {
      if (data.clientConfirmPhoneNumber?.result) {
        storage.setItem("ONBOARDING", "true").catch(console.error);
      }
    },
  });
  return { mutate, loading, error };
}
```

**Example 3: Paginated Query (Paid Invoices)**

```typescript:luce-fe/src/gqlhooks/invoices/usePaidInvoices.ts
import { useQuery } from "@apollo/client";
import {
  InvoicesByFiltersDocument,
  type InvoicesByFiltersQueryVariables,
} from "@/__generated__/graphql";
import type { InvoiceCardData } from "@/components/shared/invoices/invoice-card";
import { mapToInvoiceCard } from "@/components/shared/invoices/invoice-card";
import { useMemo } from "react";

export function usePaidInvoices(filters: InvoicesByFiltersQueryVariables["filters"]) {
  const { data, loading, error, fetchMore } = useQuery(InvoicesByFiltersDocument, {
    variables: {
      filters: {
        ...filters,
        offset: 0,
        limit: 20,
      },
    },
    fetchPolicy: "network-only",
  });

  const invoices = useMemo(() => {
    return data?.invoicesByFilters.invoices.map(mapToInvoiceCard) ?? [];
  }, [data]);

  const total = data?.invoicesByFilters.count ?? 0;
  const hasMore = invoices.length < total;

  const loadMore = () => {
    if (!hasMore || loading) return;

    fetchMore({
      variables: {
        filters: {
          ...filters,
          offset: invoices.length,
          limit: 20,
        },
      },
    });
  };

  return { invoices, total, loading, error, loadMore, hasMore };
}
```

### Default Options Configuration

**Apollo Client Configuration**

```typescript:luce-fe/src/lib/graphql/apollo.ts
import {
  ApolloClient,
  ApolloLink,
  HttpLink,
  type ServerError,
  type DefaultOptions,
} from "@apollo/client";
// ... existing imports ...

// Default options for all queries
const defaultOptions: DefaultOptions = {
  watchQuery: {
    fetchPolicy: "cache-and-network",
    errorPolicy: "all",
  },
  query: {
    fetchPolicy: "cache-and-network",
    errorPolicy: "all",
  },
  mutate: {
    errorPolicy: "all",
  },
};

function initClientInstance(): ApolloClient<NormalizedCacheObject> {
  // ... existing link setup ...

  return new ApolloClient({
    connectToDevTools: app.ENV !== "production",
    link: ApolloLink.from([
      tokenManager.authHeaderLink(),
      errorLink,
      tokenManager.errorLink(),
      link,
    ]),
    cache,
    defaultOptions, // Add default options
  });
}

export const ClientInstance = initClientInstance();
```

### Usage Examples

**Example 1: Using Simple Query in Component**

```typescript
import { View, Skeleton } from "@/components/ui";
import { Typography } from "@/components/shared/typography";
import { useVisit } from "@/gqlhooks/visits/useVisit";

function VisitDetail({ visitId }: { visitId: string }) {
  const { data: visit, loading, error } = useVisit({ id: visitId });

  if (loading) {
    return <Skeleton className="h-[200px] w-full" />;
  }

  if (error) {
    return <Typography variant="body-md">Error: {error.message}</Typography>;
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

**Example 2: Using Mutation in Component**

```typescript
import { Button } from "@/components/ui";
import { useClientConfirmPhoneNumber } from "@/gqlhooks/auth/useClientConfirmPhoneNumber";
import { showToast } from "@/components/ui/toast/show-toast";
import { __ } from "@/lib/i18n";

function ConfirmPhoneButton({ input }: { input: { phoneNumber: string; code: string } }) {
  const { mutate, loading, error } = useClientConfirmPhoneNumber();

  const handleConfirm = async () => {
    try {
      const result = await mutate({ variables: { input } });
      if (result.data?.clientConfirmPhoneNumber?.result) {
        showToast({
          type: "success",
          title: __("Phone number confirmed"),
        });
      }
    } catch (err) {
      // Error is already handled by Apollo Client error link
      showToast({
        type: "error",
        title: __("Failed to confirm phone number"),
      });
    }
  };

  return (
    <Button variant="primary" onClick={handleConfirm} loading={loading}>
      {__("Confirm")}
    </Button>
  );
}
```

**Example 3: Using Paginated Query in Component**

```typescript
import { View, Button, Skeleton } from "@/components/ui";
import { InvoiceCard } from "@/components/shared/invoices/invoice-card";
import { usePaidInvoices } from "@/gqlhooks/invoices/usePaidInvoices";
import { InvoiceFilterStatusEnum } from "@/__generated__/graphql";

function PaidInvoicesList() {
  const { invoices, loading, error, loadMore, hasMore } = usePaidInvoices({
    paymentStatus: InvoiceFilterStatusEnum.Paid,
  });

  if (loading && invoices.length === 0) {
    return <Skeleton className="h-[200px] w-full" />;
  }

  if (error) {
    return <View>Error: {error.message}</View>;
  }

  return (
    <View className="flex flex-col gap-4">
      {invoices.map((invoice) => (
        <InvoiceCard key={invoice.id} invoice={invoice} />
      ))}
      {hasMore && (
        <Button variant="tertiary" onClick={loadMore} loading={loading}>
          Load More
        </Button>
      )}
    </View>
  );
}
```

## Migration Summary

**Code Reduction**:

- **Before**: ~20-30 lines per query store (factory setup, types, transform functions)
- **After**: ~2-3 lines per query hook (direct useQuery/useMutation call)
- **Net reduction**: ~85-90% per query
- **Total potential reduction**: ~397 lines from removing `request-factory.tsx` + ~15-25 lines per migrated query

**Example LOC Comparison**:

- **Simple Query (Visit)**: 22 lines → 8 lines (64% reduction)
- **Intermediate Query (Confirm Phone)**: 30 lines → 12 lines (60% reduction)
- **Paginated Query (Paid Invoices)**: 25 lines → 35 lines (but eliminates 397-line factory + better maintainability)

**Files Affected**:

1. ✅ `lib/graphql/apollo.ts` - Add default options
2. ✅ Individual query stores → Convert to hooks (gradual migration)
3. ⏳ `lib/request-factory.tsx` - Remove after all migrations (397 lines)

**Benefits**:

- ✅ Standard Apollo Client patterns
- ✅ Better type safety with Codegen
- ✅ Automatic caching and refetching
- ✅ Reduced boilerplate (90% reduction)
- ✅ Easier to understand for new developers
- ✅ Future-proof with Apollo Client best practices

**Migration Strategy**:

1. Add default options to Apollo Client configuration
2. Create example hooks in `gqlhooks/` folder (flat structure is acceptable - organize by domain if needed, but don't over-engineer)
3. Migrate new queries to use `useQuery`/`useMutation` directly in `gqlhooks/`
4. Gradually migrate existing queries (start with simple ones)
5. Remove `createRequestFactory` usage from paginated queries (keep `createPaginatedFactory` for now)
6. Eventually remove `request-factory.tsx` once all queries are migrated

**Folder Structure**:

- All GraphQL hooks go in `gqlhooks/` folder
- Flat structure is acceptable (e.g., `gqlhooks/useVisit.ts`, `gqlhooks/useClientConfirmPhoneNumber.ts`)
- Optional subfolder organization by domain if it improves clarity (e.g., `gqlhooks/visits/`, `gqlhooks/auth/`)
- **Key principle**: Keep it simple, predictable, and maintainable. Don't over-engineer the folder structure.

**Considerations**:

- **Pagination**: Uses Apollo Client's `fetchMore` with offset-based pagination
- **Error handling**: Done via Apollo Client error link (already configured)
- **Transform functions**: Moved to hooks using `useMemo` for performance
- **`onFetchSuccess` callbacks**: Use `onCompleted` in mutations or `useEffect` in queries
- **Pagination state**: Managed directly in the hook (offset, hasMore) - simpler than Zustand store pagination state

### Examples for Considerations

**1. Pagination with fetchMore**

Uses Apollo Client's `fetchMore` for offset-based pagination:

```typescript:luce-fe/src/gqlhooks/invoices/usePaidInvoices.ts
export function usePaidInvoices(filters: InvoicesByFiltersQueryVariables["filters"]) {
  const { data, loading, error, fetchMore } = useQuery(InvoicesByFiltersDocument, {
    variables: {
      filters: {
        ...filters,
        offset: 0,
        limit: 20,
      },
    },
  });

  const invoices = useMemo(() => {
    return data?.invoicesByFilters.invoices.map(mapToInvoiceCard) ?? [];
  }, [data]);

  const total = data?.invoicesByFilters.count ?? 0;
  const hasMore = invoices.length < total;

  const loadMore = () => {
    if (!hasMore || loading) return;

    fetchMore({
      variables: {
        filters: {
          ...filters,
          offset: invoices.length,
          limit: 20,
        },
      },
    });
  };

  return { invoices, total, loading, error, loadMore, hasMore };
}
```

**2. Error Handling via Apollo Client Error Link**

Error handling is centralized in Apollo Client error link (already configured):

```typescript:luce-fe/src/lib/graphql/apollo.ts
const onErrorCallback: ErrorLink.ErrorHandler = ({
  graphQLErrors,
  networkError,
}) => {
  // Network errors (503, 504, etc.) are handled here
  // GraphQL errors are logged here
  // Toast notifications can be shown here if needed
};

const errorLink = onError(onErrorCallback);
```

In components, errors are available from the hook:

```typescript
function MyComponent() {
  const { data, loading, error } = useQuery(MyDocument, { variables });

  if (error) {
    // Error is already logged by error link
    // Show user-friendly error message
    return <ErrorView message={error.message} />;
  }

  // ... rest of component
}
```

**3. Transform Functions with useMemo**

Transform functions are moved to hooks using `useMemo` for performance:

```typescript:luce-fe/src/gqlhooks/visits/useVisit.ts
import { useQuery } from "@apollo/client";
import { VisitDocument, type VisitQueryVariables } from "@/__generated__/graphql";
import { useFragment } from "@/__generated__";
import { VisitClientFragmentFragmentDoc } from "@/__generated__/graphql";
import type { VisitDetailData } from "@/components/shared/visits/visit-detail";
import { mapToVisitDetailData } from "@/components/shared/visits/visit-detail";
import { useMemo } from "react";

export function useVisit(variables: VisitQueryVariables) {
  const { data, loading, error } = useQuery(VisitDocument, { variables });

  // Extract fragment data first (useFragment is a hook, must be at top level)
  const fragmentData = data?.visit
    ? useFragment(VisitClientFragmentFragmentDoc, data.visit)
    : null;

  // Transform function wrapped in useMemo for performance
  const visit = useMemo(() => {
    if (!fragmentData) return null;
    return mapToVisitDetailData(fragmentData);
  }, [fragmentData]);

  return { data: visit, loading, error };
}
```

**Alternative: Transform in useMemo without fragment** (if fragment is not needed):

```typescript:luce-fe/src/gqlhooks/invoices/usePaidInvoices.ts
export function usePaidInvoices(filters: InvoicesByFiltersQueryVariables["filters"]) {
  const { data, loading, error, fetchMore } = useQuery(InvoicesByFiltersDocument, {
    variables: {
      filters: {
        ...filters,
        offset: 0,
        limit: 20,
      },
    },
  });

  // Transform function with useMemo - no hooks inside
  const invoices = useMemo(() => {
    return data?.invoicesByFilters.invoices.map(mapToInvoiceCard) ?? [];
  }, [data]);

  return { invoices, loading, error, fetchMore };
}
```

**4. onFetchSuccess Callbacks**

**For Mutations** - Use `onCompleted`:

```typescript:luce-fe/src/gqlhooks/auth/useClientConfirmPhoneNumber.ts
import { useMutation } from "@apollo/client";
import { ClientConfirmPhoneNumberDocument } from "@/__generated__/graphql";
import { storage } from "@/lib/storage";

export function useClientConfirmPhoneNumber() {
  const [mutate, { loading, error }] = useMutation(ClientConfirmPhoneNumberDocument, {
    // onFetchSuccess equivalent - runs after successful mutation
    onCompleted: (data) => {
      if (data.clientConfirmPhoneNumber?.result) {
        storage.setItem("ONBOARDING", "true").catch(console.error);
      }
    },
  });
  return { mutate, loading, error };
}
```

**For Queries** - Use `useEffect`:

```typescript:luce-fe/src/gqlhooks/visits/useVisit.ts
import { useEffect } from "react";
import { useQuery } from "@apollo/client";
import { VisitDocument, type VisitQueryVariables } from "@/__generated__/graphql";

export function useVisit(variables: VisitQueryVariables) {
  const { data, loading, error } = useQuery(VisitDocument, { variables });

  // onFetchSuccess equivalent - runs when data changes
  useEffect(() => {
    if (data?.visit && !loading) {
      // Side effect after successful fetch
      // e.g., update analytics, update other stores, etc.
      console.log("Visit loaded:", data.visit.id);
    }
  }, [data, loading]);

  return { data, loading, error };
}
```

**5. Pagination State Management**

Pagination state is managed directly in the hook - simpler than Zustand store:

```typescript:luce-fe/src/gqlhooks/invoices/usePaidInvoices.ts
import { useQuery } from "@apollo/client";
import { InvoicesByFiltersDocument, type InvoicesByFiltersQueryVariables } from "@/__generated__/graphql";
import { useMemo } from "react";

export function usePaidInvoices(filters: InvoicesByFiltersQueryVariables["filters"]) {
  const { data, loading, error, fetchMore } = useQuery(InvoicesByFiltersDocument, {
    variables: {
      filters: {
        ...filters,
        offset: 0,
        limit: 20,
      },
    },
  });

  const invoices = useMemo(() => {
    return data?.invoicesByFilters.invoices.map(mapToInvoiceCard) ?? [];
  }, [data]);

  // Pagination state computed from data - no separate Zustand store needed
  const total = data?.invoicesByFilters.count ?? 0;
  const hasMore = invoices.length < total;
  const currentOffset = invoices.length;

  const loadMore = () => {
    if (!hasMore || loading) return;

    fetchMore({
      variables: {
        filters: {
          ...filters,
          offset: currentOffset, // Simple offset calculation
          limit: 20,
        },
      },
    });
  };

  return {
    invoices,
    total,
    loading,
    error,
    loadMore,
    hasMore, // Simple boolean - no complex pagination object
    currentOffset, // Simple number - no pagination state object
  };
}
```

**Comparison with Zustand Store Pagination**:

```typescript
// Before: Complex Zustand pagination state
const { data, pagination, fetchMore } = usePaidInvoicesStore();
// pagination = { offset: 0, limit: 20, total: 0, enabled: true }

// After: Simple computed values
const { invoices, total, hasMore, loadMore } = usePaidInvoices(filters);
// hasMore = invoices.length < total (simple boolean)
```

---

## Code Modularization Pattern

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
