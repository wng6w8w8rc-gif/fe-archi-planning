# App Version Provider to Zustand Store Migration

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
- **Easier testing** - Store logic can be tested independently

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
