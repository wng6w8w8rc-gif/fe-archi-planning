# V2 Refactoring Plan - Parallel Folder Approach

## Table of Contents

- [Summary](#summary)
- [Migration Status Board](#migration-status-board) ⚠️ **MANDATORY: Keep Updated**
- [Implementation Strategy](#implementation-strategy)
  - [Phase 1: Infrastructure Setup](#phase-1-infrastructure-setup) ⚠️ **MANDATORY**
  - [Phase 2: Route Integration](#phase-2-route-integration) ⚠️ **MANDATORY**
  - [Phase 3: Incremental Migration Strategy](#phase-3-incremental-migration-strategy) ⚠️ **MANDATORY**
  - [Phase 4: Shared Code Strategy](#phase-4-shared-code-strategy) ⚠️ **MANDATORY**
  - [Phase 5: Testing Strategy](#phase-5-testing-strategy) ⚠️ **MANDATORY**
  - [Phase 6: Migration Checklist](#phase-6-migration-checklist) ⚠️ **MANDATORY**
- [Example: Complete Migration Flow](#example-complete-migration-flow)
- [Version Governance](#version-governance) ⚠️ **MANDATORY**
- [Rollback Plan](#rollback-plan) ⚠️ **MANDATORY**
- [Final Migration Steps](#final-migration-steps) ⚠️ **MANDATORY**
- [Benefits](#benefits)
- [Migration Priority](#migration-priority)
- [Notes](#notes)
- [Future Considerations](#future-considerations) ⚪ **OPTIONAL**

**Legend:**

- ⚠️ **MANDATORY** - Required sections that must be followed
- ⚪ **OPTIONAL** - Nice-to-have sections for future improvements

## Summary

This document outlines the strategy for executing a BIG refactoring of the luce-fe React Native multiplatform codebase (web and app) using a **Parallel "v2" Folder Approach** with **Incremental Isolation**. This approach allows gradual migration of code while maintaining the existing v1 codebase fully functional, enabling safe, incremental refactoring with zero downtime.

### Key Strategy

- **Parallel Development**: Create `v2/` folders alongside existing code
- **Incremental Migration**: Move features one at a time from v1 to v2
- **Feature Flags**: Use user-based feature flags to enable v2 routes for engineers
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
- `src/config/feature-flags.ts` - Feature flag configuration
- `src/lib/feature-flags/index.ts` - Feature flag utilities

**Future Files (Final State):**

- `src/v1/` - Root directory for all v1 code (moved from root `src/` after migration)

**Modified Files:**

- `src/app/_layout.tsx` - Add v2 route handling
- `src/app-web/router.tsx` - Add v2 route handling
- `src/components/shared/auth-provider/index.tsx` - Add feature flag check
- `src/constants/routes.ts` - Add v2 route constants

## Migration Status Board ⚠️ **MANDATORY: Keep Updated**

> **Note**: For detailed migration tracking, see [MIGRATION-STATUS.md](./MIGRATION-STATUS.md)

This board tracks the migration status of each module from v1 to v2. Update this section as modules are migrated.

| Module                     | Status         | Route Flag        | Engineers Testing | Notes                         |
| -------------------------- | -------------- | ----------------- | ----------------- | ----------------------------- |
| **Infrastructure**         |                |                   |                   |                               |
| Feature Flags              | ✅ Complete    | N/A               | All               | Route-level flags implemented |
| Route Integration (Native) | 🔄 In Progress | N/A               | Gio, Fathul       | Basic structure in place      |
| Route Integration (Web)    | 🔄 In Progress | N/A               | Gio, Fathul       | Basic structure in place      |
| **Core Features**          |                |                   |                   |                               |
| Authentication             | ⏳ Not Started | `login`, `signup` | -                 | -                             |
| Homepage                   | ⏳ Not Started | `home`            | -                 | -                             |
| Profile                    | ⏳ Not Started | `profile`         | -                 | -                             |
| Visits                     | ⏳ Not Started | `visits`          | -                 | -                             |
| Booking                    | ⏳ Not Started | `booking`         | -                 | -                             |
| Rewards                    | ⏳ Not Started | `rewards`         | -                 | -                             |
| Notifications              | ⏳ Not Started | `notifications`   | -                 | -                             |
| **Shared Modules**         |                |                   |                   |                               |
| GraphQL Client             | ✅ Complete    | N/A               | All               | Shared, no migration needed   |
| Monitoring                 | ✅ Complete    | N/A               | All               | Shared, no migration needed   |
| Assets                     | ✅ Complete    | N/A               | All               | Shared, no migration needed   |

**Status Legend:**

- ✅ **Complete** - Fully migrated, tested, and in production
- 🔄 **In Progress** - Currently being migrated
- ⏳ **Not Started** - Not yet migrated
- 🐛 **Blocked** - Migration blocked by dependencies or issues
- 🔍 **Review** - Awaiting code review

**Last Updated**: [Update this date when modifying the board]

## Implementation Strategy

### Phase 1: Infrastructure Setup ⚠️ **MANDATORY**

#### 1.1 Feature Flag System

Create a feature flag system that checks user email/clientId to enable v2 routes.

**File: `src/config/feature-flags.ts`**

```typescript
export const FEATURE_FLAGS = {
  // Global v2 routes flag (for app-wide toggle)
  V2_ROUTES: {
    enabled: true,
    // Engineers can access v2 by logging in with these emails
    allowedEmails: ["engineer1@example.com", "engineer2@example.com", "dev@example.com"],
    // Or by clientId
    allowedClientIds: ["test-client-id-1", "test-client-id-2"],
  },
  // Route-level flags (for granular control)
  ROUTES: {
    home: {
      useV2: false, // Toggle individual routes
      allowedEmails: ["engineer1@example.com"],
      allowedClientIds: [],
    },
    profile: {
      useV2: true,
      allowedEmails: ["engineer1@example.com", "engineer2@example.com"],
      allowedClientIds: [],
    },
    visits: {
      useV2: false,
      allowedEmails: [],
      allowedClientIds: [],
    },
    booking: {
      useV2: false,
      allowedEmails: [],
      allowedClientIds: [],
    },
    // Add more routes as needed
  },
} as const;
```

**File: `src/lib/feature-flags/index.ts`**

```typescript
import { FEATURE_FLAGS } from "@/config/feature-flags";
import { useAuth } from "@/components/shared/auth-provider";
import { useAuthState } from "@/store/auth";
import { storage } from "@/lib/storage";

export type FeatureFlag = keyof typeof FEATURE_FLAGS;
export type RouteName = keyof typeof FEATURE_FLAGS.ROUTES;

/**
 * Hook to check if a feature flag is enabled for the current user
 * Use this inside React components
 */
export function useFeatureFlag(flag: FeatureFlag): boolean {
  const { clientId } = useAuth();
  const userInfo = useAuthState((state) => state.data.userInfo);
  const flagConfig = FEATURE_FLAGS[flag];

  if (!flagConfig.enabled) {
    return false;
  }

  // Check email
  if (flagConfig.allowedEmails?.includes(userInfo.user?.email ?? "")) {
    return true;
  }

  // Check clientId
  if (flagConfig.allowedClientIds?.includes(clientId)) {
    return true;
  }

  return false;
}

/**
 * Hook to check if a specific route should use v2
 * Returns true if route should use v2, false for v1
 */
export function useRouteVersion(routeName: RouteName): boolean {
  const { clientId } = useAuth();
  const userInfo = useAuthState((state) => state.data.userInfo);
  const routeConfig = FEATURE_FLAGS.ROUTES[routeName];

  if (!routeConfig) {
    return false; // Default to v1 if route not configured
  }

  // Check if v2 is enabled for this route
  if (!routeConfig.useV2) {
    return false;
  }

  // Check if user has access (if no restrictions, all users get v2)
  const hasEmailAccess =
    routeConfig.allowedEmails?.length === 0 || routeConfig.allowedEmails?.includes(userInfo.user?.email ?? "");
  const hasClientIdAccess =
    routeConfig.allowedClientIds?.length === 0 || routeConfig.allowedClientIds?.includes(clientId);

  return hasEmailAccess || hasClientIdAccess;
}

/**
 * Non-hook version for use outside React components
 * Use this in route guards, middleware, etc.
 */
export async function isV2Enabled(): Promise<boolean> {
  const flagConfig = FEATURE_FLAGS.V2_ROUTES;

  if (!flagConfig.enabled) {
    return false;
  }

  // Get current user info from store
  const userInfo = useAuthState.getState().data.userInfo;

  // Get clientId from storage (same key used in AuthProvider)
  const clientId = (await storage.getItem<string>("clientId")) ?? "";

  return (
    flagConfig.allowedEmails?.includes(userInfo.user?.email ?? "") || flagConfig.allowedClientIds?.includes(clientId)
  );
}

/**
 * Non-hook version to check route version
 * Use this in route configuration files
 */
export function getRouteVersion(routeName: RouteName): boolean {
  const routeConfig = FEATURE_FLAGS.ROUTES[routeName];
  if (!routeConfig || !routeConfig.useV2) {
    return false;
  }

  // For non-hook version, check if route is enabled (user check happens at runtime)
  return routeConfig.useV2;
}
```

#### 1.2 Route Constants

**File: `src/constants/routes.ts` (additions)**

```typescript
import { FEATURE_FLAGS } from "@/config/feature-flags";

// Route component names (for React Native Expo Router)
// Route-level flags determine which version to use
// Note: This is a static check. For user-based flags, use useRouteVersion() hook in components
export const routes = {
  home: FEATURE_FLAGS.ROUTES.home.useV2 ? "HomeV2" : "HomeV1",
  profile: FEATURE_FLAGS.ROUTES.profile.useV2 ? "ProfileV2" : "ProfileV1",
  visits: FEATURE_FLAGS.ROUTES.visits.useV2 ? "VisitsV2" : "VisitsV1",
  booking: FEATURE_FLAGS.ROUTES.booking.useV2 ? "BookingV2" : "BookingV1",
  login: FEATURE_FLAGS.ROUTES.login?.useV2 ? "LoginV2" : "LoginV1",
  // Add more routes as needed
} as const;

// For runtime user-based checks, use the useRouteVersion() hook in route components
// This allows per-user feature flagging based on email/clientId

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

// Note: Route-level flags determine which version (v1 or v2) to use
// During migration, routes can be individually toggled
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

### Phase 2: Route Integration ⚠️ **MANDATORY**

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
import { useRouteVersion } from "@/lib/feature-flags";
import { HomeV1 } from "@/containers/homepage";
import { HomeV2 } from "@/v2/containers/homepage";

export default function HomeScreen() {
  const useV2 = useRouteVersion("home");

  // Route-level flag determines which component to render
  if (useV2) {
    return <HomeV2 />;
  }

  return <HomeV1 />;
}
```

**File: `src/app/(tabs)/profile.tsx` (route-level example)**

```typescript
import { useRouteVersion } from "@/lib/feature-flags";
import { ProfileV1 } from "@/containers/profile";
import { ProfileV2 } from "@/v2/containers/profile";

export default function ProfileScreen() {
  const useV2 = useRouteVersion("profile");

  // Route-level flag determines which component to render
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
import { useRouteVersion } from "@/lib/feature-flags";
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
  const useV2 = useRouteVersion(routeName as any);
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
                {/* Home route - route-level flag determines v1 or v2 */}
                <Route
                  path={ROUTES.Root}
                  element={<RouteWrapper routeName="home" V1Component={HomepagePage} V2Component={HomepagePageV2} />}
                />
                {/* Profile route - route-level flag determines v1 or v2 */}
                <Route
                  path={ROUTES.Profile.Root}
                  element={<RouteWrapper routeName="profile" V1Component={ProfilePage} V2Component={ProfilePageV2} />}
                />
                {/* Visits route - route-level flag determines v1 or v2 */}
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
import { useRouteVersion } from "@/lib/feature-flags";

export default function App() {
  const homeUseV2 = useRouteVersion("home");
  const profileUseV2 = useRouteVersion("profile");
  const visitsUseV2 = useRouteVersion("visits");

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

### Phase 3: Incremental Migration Strategy ⚠️ **MANDATORY**

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

   - Engineers test v2 routes with feature flag
   - Compare behavior with v1
   - Fix issues

4. **Gradual Rollout**

   - Enable for more test users via feature flags
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

### Phase 4: Shared Code Strategy ⚠️ **MANDATORY**

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

### Phase 5: Testing Strategy ⚠️ **MANDATORY**

#### 5.1 Feature Flag Testing

**For Engineers:**

1. Add email to `FEATURE_FLAGS.V2_ROUTES.allowedEmails`
2. Log in with that email
3. Access v2 routes (automatically enabled)
4. Test functionality
5. Compare with v1 behavior

**For Engineers (Additional Testing):**

1. Create test accounts with allowed emails
2. Test v2 routes thoroughly
3. Report issues
4. Verify fixes
5. Compare v1 and v2 behavior side-by-side

#### 5.2 A/B Testing ⚪ **OPTIONAL**

Once v2 is stable, can implement percentage-based rollout:

```typescript
export const FEATURE_FLAGS = {
  V2_ROUTES: {
    enabled: true,
    rolloutPercentage: 10, // 10% of users
    allowedEmails: [...],
  },
};
```

### Phase 6: Migration Checklist ⚠️ **MANDATORY**

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
- [ ] Enable for test users via feature flags
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
- ✅ **Easy Rollback**: Disable feature flag to revert
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

   - Feature flags
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

## Version Governance ⚠️ **MANDATORY**

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

## Rollback Plan ⚠️ **MANDATORY**

Even though feature flags and zero downtime are in place, it's important to have an explicit rollback plan in case critical issues are discovered in v2.

### To Revert to v1

1. **Disable Feature Flag**

   - Set `FEATURE_FLAGS.V2_ROUTES.enabled = false` in `src/config/feature-flags.ts`
   - Commit and deploy immediately

2. **Clear User State (if needed)**

   - Clear local storage or reset clientId for affected users
   - This ensures users don't get stuck in v2 routes
   - Can be done via:
     - Client-side: Clear browser/app storage
     - Server-side: Reset user session flags (if stored)

3. **Redeploy**
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

- Disable feature flag immediately
- Investigate issue in v2
- Fix and test thoroughly
- Re-enable feature flag after fix

**Scenario 2: Performance Degradation**

- Disable feature flag for affected users
- Investigate performance issues
- Optimize v2 code
- Gradual re-enablement with monitoring

**Scenario 3: User Experience Issues**

- Disable feature flag
- Gather user feedback
- Make necessary adjustments in v2
- Re-enable with improved UX

## Final Migration Steps ⚠️ **MANDATORY**

When all features are migrated and v2 is stable:

1. **Redirect all routes to v2**

   - Update route handlers to point to v2
   - Remove feature flag checks (all users use v2)
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

## Future Considerations ⚪ **OPTIONAL**

These are optional enhancements that can be implemented if needed:

- **Percentage-based rollout**: During migration, enable for percentage of users
- **A/B testing**: Compare v1 vs v2 metrics during migration
- **Automated migration**: Tools to help migrate code from v1 to v2
- **Migration scripts**: Scripts to update imports, routes, etc.
- **CI/CD integration**: Automated testing and deployment pipelines for v2
- **Monitoring dashboards**: Track v1 vs v2 performance metrics

## Related Documentation

For more detailed information, see:

- **[MIGRATION-STATUS.md](./MIGRATION-STATUS.md)** - Detailed migration tracking board
- **[FEATURE-FLAGS.md](./FEATURE-FLAGS.md)** - Detailed feature flag design and implementation (if created)
- **[CI-CD-CHANGES.md](./CI-CD-CHANGES.md)** - CI/CD pipeline modifications (if created)
