# Notification Provider to Zustand Store Migration

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
- **Easier testing** - Store logic can be tested independently

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
        <Button
          variant="primary"
          onClick={turnOn}
          children={__("Enable Notifications")}
        />
      ) : (
        <Button
          variant="tertiary"
          onClick={turnOff}
          children={__("Disable Notifications")}
        />
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
      <Typography variant="body-md">
        {latestNotification.request.content.title}
      </Typography>
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
      <Button
        variant="tertiary"
        onClick={handleCheck}
        children={__("Check Permission")}
      />
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
