# Auth Provider to Zustand Store Migration

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
- **Easier testing** - Store logic can be tested independently

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
