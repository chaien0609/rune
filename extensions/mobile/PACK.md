---
name: "@rune/mobile"
description: Mobile development patterns — React Native, Flutter, deep linking, push notifications, OTA updates, app store preparation, native bridge integration, iOS build pipeline, and App Store Connect automation.
metadata:
  author: runedev
  version: "0.3.0"
  layer: L4
  price: "$15"
  target: Mobile developers
---

# @rune/mobile

## Purpose

Mobile development has platform-specific pitfalls that web developers hit repeatedly: navigation stacks that leak memory, FlatList rendering that drops frames, New Architecture migration that silently breaks third-party libraries, deep links that work in dev but fail in production, push notifications that never arrive on iOS, OTA updates that crash on bytecode mismatch, and app store rejections for missing privacy manifests. This pack provides patterns for React Native and Flutter — detect the framework, audit for mobile-specific anti-patterns, and emit fixes that pass platform review.

## Triggers

- Auto-trigger: when `react-native`, `expo`, `flutter`, `android/`, `ios/`, `app.json` (Expo) detected
- `/rune react-native` — audit React Native architecture and performance
- `/rune flutter` — audit Flutter architecture and state management
- `/rune deep-linking` — set up or audit deep linking (Universal Links, App Links)
- `/rune push-notifications` — set up or audit push notification pipeline
- `/rune ota-updates` — set up or audit OTA update strategy
- `/rune app-store-prep` — prepare app store submission
- `/rune native-bridge` — audit or create native module bridges
- `/rune ios-build` — end-to-end iOS build, sign, archive, upload pipeline
- `/rune app-store-connect` — App Store Connect API operations (versions, screenshots, localization, IAPs)
- Called by `cook` (L1) when mobile task detected
- Called by `team` (L1) when porting web to mobile

## Skills Included

### react-native

React Native patterns — New Architecture migration, navigation, state management, native modules, performance optimization, Expo vs bare workflow decisions.

#### Workflow

**Step 1 — Detect React Native setup**
Use Grep to find framework markers: `react-native` in package.json (extract version — 0.82+ = New Architecture mandatory), `expo` config (extract SDK version — 53+ = New Arch default), navigation library (`@react-navigation/native` v6 vs v7, `expo-router` v3 vs v4), state management (`zustand`, `redux`, `jotai`), and native module usage. Read `app.json`/`app.config.js` for Expo configuration.

**Step 2 — Audit New Architecture readiness**
Check for:
- `react-native` >= 0.82 or Expo SDK >= 55: New Architecture is mandatory, no opt-out
- `setNativeProps` usage: incompatible with New Architecture, must migrate to Reanimated or Animated API
- Third-party libraries using legacy Bridge (`NativeModules.X` directly instead of TurboModules): check each against `reactnative.directory` compatibility list
- `react-native-reanimated` version: must be >= 3.8 to avoid Android animation stutter on New Architecture (GitHub #7435)
- Kotlin version in `android/build.gradle`: Reanimated + Kotlin 1.9.25 fails EAS Build (GitHub #7674)
- State batching: New Architecture enables React 18 concurrent batching — components relying on intermediate state between updates silently break

**Step 3 — Audit performance patterns**
Check for: FlatList without `keyExtractor` or with inline `renderItem` (re-renders), images using `react-native-fast-image` (not compatible with New Architecture — migrate to `expo-image`), heavy re-renders from context (missing `useMemo`/`useCallback`), navigation listeners not cleaned up, large JS bundle without lazy loading (`React.lazy` + `Suspense`), `removeClippedSubviews` causing blank cells on fast scroll.

**Step 4 — Audit navigation patterns (React Navigation v7 / Expo Router v4)**
Check for:
- `navigate()` calls: v7 changed semantics — now pushes new screen even if route exists in stack (v6 navigated to existing instance). Audit all `navigation.navigate()` calls
- `useNavigation()` hook: causes re-renders on every route change in Expo Router v4, not just current route (GitHub #35383). Replace with `useRouter()` for navigation-only usage
- Non-unique navigator names: deep links silently fail to resolve (GitHub #9267)
- Authentication + deep link race condition: `NavigationContainer` not ready when initial URL received. Must capture URL, wait for auth, then navigate

**Step 5 — Emit optimized patterns**
For each issue, emit the fix: memoized FlatList item components, `expo-image` migration, proper navigation with typed routes, optimized state selectors, and Hermes engine configuration. For New Architecture migration, emit a phased plan: audit → update libraries → enable → test → fix regressions.

#### Example

```tsx
// BEFORE: FlatList anti-patterns + legacy image library
import FastImage from 'react-native-fast-image'; // ❌ Not New Arch compatible
<FlatList
  data={items}
  renderItem={({ item }) => (
    <View>
      <FastImage source={{ uri: item.image }} />
      <Text>{item.name}</Text>
    </View>
  )}
/>

// AFTER: New Architecture compatible, memoized, proper image caching
import { Image } from 'expo-image'; // ✅ New Arch compatible
import { FlashList } from '@shopify/flash-list'; // ✅ Better than FlatList

const ItemCard = React.memo<{ item: Item; onPress: () => void }>(({ item, onPress }) => (
  <Pressable onPress={onPress}>
    <Image
      source={item.image}
      style={styles.image}
      contentFit="cover"
      placeholder={item.blurhash}
      transition={200}
    />
    <Text>{item.name}</Text>
  </Pressable>
));

const renderItem = useCallback(({ item }: { item: Item }) => (
  <ItemCard item={item} onPress={() => router.push(`/product/${item.id}`)} />
), [router]);

<FlashList
  data={items}
  renderItem={renderItem}
  estimatedItemSize={88} // Required — measure actual item height
  keyExtractor={item => item.id}
/>
```

---

### flutter

Flutter patterns — widget composition, state management (Riverpod, BLoC), platform channels, adaptive layouts.

#### Workflow

**Step 1 — Detect Flutter architecture**
Use Grep to find state management (`riverpod`, `flutter_bloc`, `provider`, `get_it`), routing (`go_router`, `auto_route`), and platform channel usage. Read `pubspec.yaml` for dependencies and `lib/` structure for architecture pattern (feature-first, layer-first).

**Step 2 — Audit widget tree and state**
Check for: `setState` in complex widgets (should use state management), deeply nested widget trees (extract widgets), `BuildContext` passed through many layers (use InheritedWidget or Riverpod), missing `const` constructors (unnecessary rebuilds), platform-specific code without adaptive checks.

**Step 3 — Emit refactored patterns**
For each issue, emit: extracted widget with const constructor, Riverpod provider for state, proper error handling with `AsyncValue`, and adaptive layout using `LayoutBuilder` + breakpoints.

#### Example

```dart
// BEFORE: setState in complex widget, no separation
class HomeScreen extends StatefulWidget { ... }
class _HomeScreenState extends State<HomeScreen> {
  List<Item> items = [];
  bool loading = true;

  @override
  void initState() {
    super.initState();
    fetchItems().then((data) => setState(() { items = data; loading = false; }));
  }
}

// AFTER: Riverpod with AsyncValue, separated concerns
@riverpod
Future<List<Item>> items(Ref ref) async {
  final repo = ref.watch(itemRepositoryProvider);
  return repo.fetchAll();
}

class HomeScreen extends ConsumerWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final itemsAsync = ref.watch(itemsProvider);
    return itemsAsync.when(
      data: (items) => ItemList(items: items),
      loading: () => const ShimmerList(),
      error: (err, stack) => ErrorView(message: err.toString(), onRetry: () => ref.invalidate(itemsProvider)),
    );
  }
}
```

---

### deep-linking

Deep linking setup and debugging — Universal Links (iOS), App Links (Android), deferred deep links, authentication + deep link integration.

#### Workflow

**Step 1 — Detect current deep link setup**
Use Grep to find: `expo-linking`, `Linking.addEventListener`, `useURL`, `expo-router` deep link config, `apple-app-site-association`, `assetlinks.json`, `IntentFilter` in `AndroidManifest.xml`. Check for React Navigation `linking` config or Expo Router file-based deep link handling.

**Step 2 — Audit deep link reliability**
Check for these common failure modes:
- **AASA file redirect**: `.well-known/apple-app-site-association` endpoint must not redirect (HTTP→HTTPS or www→non-www). Any redirect silently breaks Universal Links
- **AASA caching**: Apple CDN caches AASA aggressively (up to 24h). Changes appear correct on server but old version is served to devices
- **SHA-256 mismatch**: Dev/Preview builds use different signing key than Production. `assetlinks.json` must include ALL certificates (upload key + app signing key)
- **Multiple environments**: Staging and production need separate AASA entries with different bundle IDs and team IDs
- **Firebase Dynamic Links dead**: Shut down August 25, 2025. All `page.link` subdomains stopped working. Must migrate to Branch.io, custom server, or standard App Links/Universal Links
- **Simulator limitation**: Universal Links and App Links do not work on simulators. Must test on real physical devices

**Step 3 — Audit authentication + deep link integration**
Check for race condition: deep link arrives before auth state resolves. Pattern: capture initial URL, wait for auth, then navigate. In React Navigation v7, `NAVIGATE` action pushes new screen even for existing routes — deep link handler must check current route before navigating.

**Step 4 — Emit deep link configuration**
For Expo Router: verify file-based route structure matches expected deep link paths. For React Navigation v7: emit typed `linking` config with authentication gate. For server: emit AASA and `assetlinks.json` with correct team ID, bundle ID, and all signing certificates.

#### Example

```typescript
// Expo Router — app/_layout.tsx with auth-gated deep link handling
import { useURL } from 'expo-linking';
import { useRouter, useSegments } from 'expo-router';

export default function RootLayout() {
  const url = useURL();
  const router = useRouter();
  const segments = useSegments();
  const { user, isLoading } = useAuth();
  const pendingDeepLink = useRef<string | null>(null);

  // Capture deep link before auth resolves
  useEffect(() => {
    if (url && isLoading) {
      pendingDeepLink.current = url;
    }
  }, [url, isLoading]);

  // Navigate after auth resolves
  useEffect(() => {
    if (isLoading) return;

    const inAuthGroup = segments[0] === '(auth)';
    if (!user && !inAuthGroup) {
      router.replace('/login');
    } else if (user && inAuthGroup) {
      // Process pending deep link or go to home
      if (pendingDeepLink.current) {
        const path = new URL(pendingDeepLink.current).pathname;
        pendingDeepLink.current = null;
        router.replace(path);
      } else {
        router.replace('/');
      }
    }
  }, [user, isLoading, segments]);

  return <Slot />;
}
```

```json
// .well-known/apple-app-site-association — NO redirects on this endpoint
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAMID.com.example.app", "TEAMID.com.example.app.staging"],
        "components": [
          { "/": "/product/*", "comment": "Product deep links" },
          { "/": "/invite/*", "comment": "Invite deep links" }
        ]
      }
    ]
  }
}
```

```json
// .well-known/assetlinks.json — include BOTH upload key AND app signing key
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.app",
      "sha256_cert_fingerprints": [
        "AA:BB:CC:...:upload_key_fingerprint",
        "DD:EE:FF:...:app_signing_key_fingerprint"
      ]
    }
  }
]
```

---

### push-notifications

Push notification setup — FCM v1, APNs, Expo Notifications, permission handling, scheduling, debugging delivery failures.

#### Workflow

**Step 1 — Detect notification setup**
Use Grep to find: `expo-notifications`, `@react-native-firebase/messaging`, `firebase.messaging`, push token registration, notification listeners. Check `app.json` plugins for `expo-notifications` config. Check for `google-services.json` (Android) and `GoogleService-Info.plist` (iOS).

**Step 2 — Audit FCM v1 migration**
Check for:
- FCM Legacy API usage (server key string instead of service account JSON): Legacy API is fully shut down since June 2024
- `google-services.json` must be FCM v1 version — old files from Legacy API still circulate in repos
- `MismatchSenderId` error: FCM server key and `google-services.json` project_number must match same Firebase project
- Multiple Firebase environments: dev/staging/prod need separate `google-services.json` files with environment-specific project numbers

**Step 3 — Audit iOS-specific gotchas**
Check for:
- `aps-environment` entitlement: works in dev builds but fails in production if `expo-notifications` not in `app.json` plugins array
- `getDevicePushTokenAsync()` race condition: silently never resolves on SDK 53+ for some users (GitHub #37516). Must call after app is fully initialized, not in root layout mount
- iOS requires paid Apple Developer account ($99/yr) for APNs — no way to test push on Simulator or free account
- iOS 18: explicit permission prompt required before scheduling local notifications — call `requestPermissionsAsync()` first
- Push notifications removed from Expo Go on Android (SDK 53+). Must use development build

**Step 4 — Audit permission handling**
Check that permission is requested at contextual moment (not app launch), fallback UI shown when denied, and `Settings.openURL` offered for re-enabling from Settings.

**Step 5 — Emit notification pipeline**
Emit: server-side push via FCM v1 HTTP API with service account auth, client-side token registration with retry, notification listener setup with cleanup, and scheduled notification with proper permission check.

#### Example

```typescript
// Server — FCM v1 push (NOT legacy). Requires service account JSON
import { GoogleAuth } from 'google-auth-library';

const auth = new GoogleAuth({
  keyFile: 'service-account.json', // NOT a server key string
  scopes: ['https://www.googleapis.com/auth/firebase.messaging'],
});

async function sendPush(token: string, title: string, body: string, data?: Record<string, string>) {
  const client = await auth.getClient();
  const projectId = 'your-project-id';

  const response = await client.request({
    url: `https://fcm.googleapis.com/v1/projects/${projectId}/messages:send`,
    method: 'POST',
    data: {
      message: {
        token,
        notification: { title, body },
        data: data ?? {},
        android: { priority: 'high' },
        apns: {
          payload: { aps: { sound: 'default', badge: 1 } },
          headers: { 'apns-priority': '10' },
        },
      },
    },
  });
  return response.data;
}
```

```typescript
// Client — Expo Notifications with proper lifecycle
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';

// Configure BEFORE any notification arrives
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

async function registerForPush(): Promise<string | null> {
  if (!Device.isDevice) {
    console.warn('Push notifications require a physical device');
    return null;
  }

  const { status: existing } = await Notifications.getPermissionsAsync();
  let finalStatus = existing;

  if (existing !== 'granted') {
    // Request at contextual moment, NOT app launch
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') return null;

  // Must be called AFTER app is fully initialized (not in root layout mount)
  const { data: token } = await Notifications.getExpoPushTokenAsync({
    projectId: 'your-expo-project-id', // Required for EAS
  });
  return token;
}
```

---

### ota-updates

OTA update strategy — EAS Update, runtime version management, rollback, staged rollouts, bytecode compatibility.

#### Workflow

**Step 1 — Detect OTA setup**
Use Grep to find: `expo-updates`, `Updates.checkForUpdateAsync`, `runtimeVersion` in `app.json`/`app.config.js`, `eas.json` update channel configuration, custom `UpdatesProvider`.

**Step 2 — Audit OTA safety**
Check for:
- **Runtime version match**: OTA update only applies to builds with exactly matching `runtimeVersion`. A native dependency change bumps runtime version and invalidates all pending OTA updates. Verify `runtimeVersion` strategy (semver auto vs manual)
- **Hermes bytecode compatibility**: Each RN version compiles to specific bytecode version. OTA built against RN 0.79 crashes on binary built with RN 0.78. NEVER OTA across RN version boundaries
- **Update timing**: Updates apply on next cold start, NOT instantly. Users with app in background don't receive updates. For emergency fixes, need custom `UpdatesProvider` with in-session check
- **Rollback gaps**: `eas update:rollback` has syntax bugs with specific flag combinations. Use branch-based rollback: republish previous update to same channel
- **Rollout math**: 10% rollout = 10% of cold-start checks, NOT 10% of users. If 80% of users never cold-start in a week, actual reach is ~2%
- **Native code limitation**: OTA ships JS bundle only. Bugs requiring native changes need full App Store submission

**Step 3 — Emit OTA strategy**
Emit: channel-based configuration (production/staging/preview), runtime version strategy, update check implementation with error handling, and rollback procedure.

#### Example

```json
// eas.json — channel-based OTA configuration
{
  "build": {
    "production": {
      "channel": "production",
      "distribution": "store",
      "autoIncrement": true
    },
    "preview": {
      "channel": "preview",
      "distribution": "internal"
    }
  }
}
```

```typescript
// app.config.ts — runtime version strategy
export default {
  expo: {
    runtimeVersion: {
      policy: 'fingerprintExperimental', // Auto-bumps when native deps change
    },
    updates: {
      url: 'https://u.expo.dev/your-project-id',
      fallbackToCacheTimeout: 3000, // Don't block startup > 3s
    },
  },
};
```

```typescript
// Custom update check with error handling and staged rollout
import * as Updates from 'expo-updates';

async function checkForUpdate() {
  if (__DEV__) return; // Skip in development

  try {
    const update = await Updates.checkForUpdateAsync();
    if (!update.isAvailable) return;

    // Download in background — don't block user
    const result = await Updates.fetchUpdateAsync();
    if (!result.isNew) return;

    // Option A: Apply on next cold start (default, safe)
    // User gets update automatically next time they fully close + reopen

    // Option B: Prompt user to restart (for important fixes)
    Alert.alert(
      'Update Available',
      'A new version is ready. Restart to apply?',
      [
        { text: 'Later', style: 'cancel' },
        { text: 'Restart', onPress: () => Updates.reloadAsync() },
      ]
    );
  } catch (error) {
    // OTA failures should NEVER crash the app
    // Log to error tracking, don't show to user
    console.error('OTA check failed:', error);
  }
}
```

---

### app-store-prep

App store submission preparation — screenshots, metadata, privacy manifests, review guidelines compliance, TestFlight/internal testing.

#### Workflow

**Step 1 — Audit submission readiness**
Check for:
- App icon: 1024x1024 for iOS (no alpha, no rounded corners), adaptive icon for Android
- Splash screen configured
- Privacy policy URL in app config
- Required permissions with specific (not generic) usage descriptions — "App requires access to your camera" gets rejected. Must be specific: "Used to scan QR codes for quick login"
- `PrivacyInfo.xcprivacy` present (mandatory since April 2025): Apple requires privacy manifest for apps using file timestamp, boot time, disk space, or UserDefaults APIs. React Native core and many libraries access these APIs. Missing manifest = auto-rejection
- Minimum SDK versions: iOS 18 SDK mandatory (Xcode 16+, April 2025), Android API 34 minimum
- Release signing configured (not debug)
- `.aab` format for Google Play (APK no longer accepted for new apps)

**Step 2 — Generate metadata**
From README and app config, generate: app title (30 chars max), subtitle (30 chars), description (4000 chars), keywords (100 chars), category selection, age rating questionnaire answers, and screenshot specifications per device size.

Current required screenshot sizes:
- iPhone 6.9" (1320×2868) — iPhone 16 Pro Max (NEW, required for new apps)
- iPhone 6.7" (1290×2796)
- iPhone 6.5" (1242×2688)
- iPad 12.9" (2048×2732) — if app supports iPad
- Android: feature graphic 1024×500

**Step 3 — Emit submission checklist**
Output structured checklist covering both platforms with platform-specific gotchas.

#### Example

```markdown
## App Store Submission Checklist

### iOS (Apple App Store Connect)
- [ ] App icon: 1024x1024 PNG, no alpha, no rounded corners
- [ ] Screenshots: 6.9" (1320x2868), 6.7" (1290x2796), 6.5" (1242x2688)
- [ ] Privacy policy URL: https://example.com/privacy
- [ ] `PrivacyInfo.xcprivacy` included (MANDATORY since April 2025)
- [ ] NSCameraUsageDescription: "Used to scan QR codes for quick login" (SPECIFIC, not generic)
- [ ] NSLocationWhenInUseUsageDescription: "Used to show nearby stores on the map"
- [ ] TestFlight build uploaded and tested on physical device
- [ ] Export compliance: Uses HTTPS only (no custom encryption) → select "No"
- [ ] Built with Xcode 16+ / iOS 18 SDK (mandatory since April 2025)

### Android (Google Play Console)
- [ ] Adaptive icon: foreground (108dp) + background layer
- [ ] Feature graphic: 1024x500 PNG
- [ ] App Bundle format (.aab, NOT .apk)
- [ ] Target API 34+ (Android 14)
- [ ] 64-bit native libraries included (32-bit only = rejection)
- [ ] Data safety form: accurately declare ALL collected data (analytics SDKs collect device IDs)
- [ ] `SCHEDULE_EXACT_ALARM` justified if using scheduled notifications
- [ ] Content rating: IARC questionnaire completed
- [ ] Internal testing track: at least 1 build tested
- [ ] Signing: upload key + app signing by Google Play enabled
```

---

### native-bridge

Native bridge patterns — platform-specific code, Expo Modules API, TurboModules, Swift/Kotlin interop, background tasks.

#### Workflow

**Step 1 — Detect bridge requirements**
Use Grep to find platform-specific code: `Platform.OS`, `Platform.select`, `NativeModules` (legacy), `TurboModuleRegistry` (new), `MethodChannel` (Flutter), Expo modules (`expo-modules-core`). Read existing native code in `ios/` and `android/` directories.

**Step 2 — Audit bridge safety**
Check for:
- `NativeModules.X` direct access: returns `undefined` silently in bridgeless mode (New Architecture). Must use TurboModule codegen or Expo Modules API instead
- Type mismatches between JS/Dart and native (string expected, int sent): crashes app instead of returning error
- Synchronous bridge calls blocking UI thread
- Missing null checks on platform-specific returns
- Mixing old Bridge modules + new TurboModules: possible during migration but causes subtle memory leaks
- Missing codegen step (`generateCodegenArtifacts`): intermittent "module not found" errors only in release builds

**Step 3 — Emit type-safe bridge**
For React Native: emit Expo Module with TypeScript interface (preferred) or TurboModule with codegen types. For Flutter: emit MethodChannel with proper error handling, type-safe serialization, and platform-specific implementations for both iOS (Swift) and Android (Kotlin).

#### Example

```typescript
// React Native — Expo Module (type-safe, New Architecture compatible)
// modules/haptics/index.ts
import { NativeModule, requireNativeModule } from 'expo-modules-core';

interface HapticsModule extends NativeModule {
  impact(style: 'light' | 'medium' | 'heavy'): void;
  notification(type: 'success' | 'warning' | 'error'): void;
}

const HapticsNative = requireNativeModule<HapticsModule>('Haptics');

export function impact(style: 'light' | 'medium' | 'heavy' = 'medium') {
  HapticsNative.impact(style);
}

// modules/haptics/ios/HapticsModule.swift
import ExpoModulesCore
import UIKit

public class HapticsModule: Module {
  public func definition() -> ModuleDefinition {
    Name("Haptics")
    Function("impact") { (style: String) in
      let generator: UIImpactFeedbackGenerator
      switch style {
      case "light": generator = UIImpactFeedbackGenerator(style: .light)
      case "heavy": generator = UIImpactFeedbackGenerator(style: .heavy)
      default: generator = UIImpactFeedbackGenerator(style: .medium)
      }
      generator.impactOccurred()
    }
  }
}
```

---

### ios-build-pipeline

End-to-end iOS build pipeline — certificate generation, provisioning profiles, Xcode archive, IPA export, TestFlight upload, build polling. Covers both React Native and native Swift projects.

#### Workflow

**Step 1 — Detect project type and signing state**
Use Glob to find: `.xcworkspace` or `.xcodeproj` (check `ios/`, `macos/`, `apple/` for RN projects), `Podfile` (needs `pod install` if workspace missing), `project.pbxproj` for current signing config (`DEVELOPMENT_TEAM`, `CODE_SIGN_STYLE`, `PRODUCT_BUNDLE_IDENTIFIER`). Check for existing signing state file (`.rune/signing-state.json`) from previous pipeline runs — if exists, skip completed steps (idempotent pipeline).

**Step 2 — Bundle ID registration**
Check if bundle ID exists on Apple Developer portal. If not:
- Register via App Store Connect API: `POST /v1/bundleIds` with `identifier`, `name`, `platform: IOS`
- Common failure: bundle ID already taken by another team → suggest alternative namespace
- Store `bundleIdResourceId` in signing state for later use

**Step 3 — Distribution certificate**
Generate Apple Distribution certificate:
```bash
# Generate RSA 2048-bit CSR via OpenSSL
openssl req -new -newkey rsa:2048 -nodes \
  -keyout distribution.key \
  -out distribution.csr \
  -subj "/CN=Apple Distribution/O=YourTeam"

# Upload CSR to App Store Connect API
# Download signed certificate (.cer)

# Create .p12 bundle — OpenSSL 3.x requires -legacy flag
openssl pkcs12 -export -legacy \
  -inkey distribution.key \
  -in distribution.cer \
  -out distribution.p12 \
  -passout pass:""

# Import to login keychain
security import distribution.p12 -k ~/Library/Keychains/login.keychain-db -T /usr/bin/codesign
```

Sharp edges:
- OpenSSL 3.x (macOS 14+) changed default encryption — `.p12` without `-legacy` flag silently fails import
- Must import to `login.keychain-db` specifically, not `System.keychain`
- `codesign` needs explicit trust via `-T /usr/bin/codesign` flag

**Step 4 — Provisioning profile**
Create App Store distribution profile via ASC API → download → install:
```bash
# Decode base64 profile content from API response
base64 -d profile_content.b64 > profile.mobileprovision

# Install to standard location
cp profile.mobileprovision ~/Library/MobileDevice/Provisioning\ Profiles/<UUID>.mobileprovision
```

**Step 5 — Patch project.pbxproj**
Update Xcode project build settings:
- `DEVELOPMENT_TEAM` = team ID from ASC
- `CODE_SIGN_STYLE` = `Automatic` for dev, `Manual` for distribution
- `PRODUCT_BUNDLE_IDENTIFIER` = registered bundle ID
- For React Native: detect workspace in `ios/`, run `pod install` if Podfile exists without workspace

**Step 6 — Archive and export**
```bash
# Archive
xcodebuild archive \
  -workspace App.xcworkspace \
  -scheme App \
  -archivePath build/App.xcarchive \
  -destination "generic/platform=iOS" \
  CODE_SIGN_STYLE=Manual \
  CODE_SIGN_IDENTITY="Apple Distribution" \
  PROVISIONING_PROFILE_SPECIFIER="<profile-name>"

# Export IPA
xcodebuild -exportArchive \
  -archivePath build/App.xcarchive \
  -exportPath build/export \
  -exportOptionsPlist ExportOptions.plist
```

Sharp edges:
- Archive fails silently if CocoaPods not installed → check for `Pods/` directory
- Export failure diagnostics hidden in `IDEDistribution.standard-log.txt` inside archive — always check this file on failure
- `ExportOptions.plist` must specify `method: app-store`, `teamID`, `signingStyle: manual`, `provisioningProfiles` dict

**Step 7 — Upload to TestFlight**
```bash
# Upload via xcrun altool with API key auth (.p8 file)
xcrun altool --upload-app \
  -f build/export/App.ipa \
  --type ios \
  --apiKey <key-id> \
  --apiIssuer <issuer-id>
```

**Step 8 — Poll build processing**
After upload, poll ASC API every 30s (up to 30 min) for build to transition from `PROCESSING` → `VALID` or `INVALID`. On `VALID`: auto-attach build to pending App Store version. On `INVALID`: fetch `betaBuildLocalizations` for error details.

#### Example

```json
// .rune/signing-state.json — idempotent pipeline state
{
  "bundleId": "com.example.myapp",
  "bundleIdResourceId": "ABC123",
  "certificateId": "DEF456",
  "provisioningProfileUUID": "GHI-789-...",
  "provisioningProfileName": "MyApp Distribution",
  "teamId": "TEAM123",
  "lastArchivePath": "build/App.xcarchive",
  "lastUploadBuildNumber": "42",
  "completedSteps": ["bundleId", "certificate", "profile", "patch"]
}
```

```xml
<!-- ExportOptions.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>app-store</string>
  <key>teamID</key>
  <string>TEAM123</string>
  <key>signingStyle</key>
  <string>manual</string>
  <key>provisioningProfiles</key>
  <dict>
    <key>com.example.myapp</key>
    <string>MyApp Distribution</string>
  </dict>
</dict>
</plist>
```

---

### app-store-connect

App Store Connect API automation — version management, localized store listings, screenshot upload, IAP/subscription creation, review submission, customer review monitoring.

#### Workflow

**Step 1 — Authenticate with ASC API**
App Store Connect uses JWT (ES256) with 20-minute expiry:
```typescript
import jwt from 'jsonwebtoken';
import fs from 'fs';

function generateASCToken(keyId: string, issuerId: string, privateKeyPath: string): string {
  const privateKey = fs.readFileSync(privateKeyPath, 'utf8');
  return jwt.sign({}, privateKey, {
    algorithm: 'ES256',
    expiresIn: '20m',
    issuer: issuerId,
    header: {
      alg: 'ES256',
      kid: keyId,
      typ: 'JWT',
    },
    audience: 'appstoreconnect-v1',
  });
}
```
Sharp edge: Token expires in 20 min — must auto-refresh when within 60s of expiry. Rate limit: 200 requests/minute, 429 response requires exponential backoff.

**Step 2 — Version management**
Create new App Store version:
```
POST /v1/appStoreVersions
{
  "data": {
    "type": "appStoreVersions",
    "attributes": {
      "platform": "IOS",
      "versionString": "1.2.0"
    },
    "relationships": {
      "app": { "data": { "type": "apps", "id": "<app-id>" } }
    }
  }
}
```
- Only ONE editable version allowed per platform at a time
- Cannot create version if existing version is "Pending Developer Release"
- Version string must be higher than current live version (semver)

**Step 3 — Localized store listing**
For each locale (`en-US`, `ja`, `de-DE`, etc.):
- `description` (4000 chars max)
- `keywords` (100 chars max, comma-separated)
- `whatsNew` (4000 chars, release notes)
- `promotionalText` (170 chars, can be updated without new version)

**Step 4 — Screenshot upload (chunked reservation)**
ASC uses a 3-step upload process:
1. Reserve upload: `POST /v1/appScreenshots` with `fileName`, `fileSize` → get `uploadOperations` array
2. Upload chunks: PUT each chunk to the returned URLs with correct `Content-Length` and offset headers
3. Commit: `PATCH /v1/appScreenshots/{id}` with `uploaded: true` and SHA-256 `sourceFileChecksum`

Sharp edges:
- Chunk size dictated by API response, NOT configurable client-side
- Must send ALL chunks before commit or upload silently fails
- Screenshot dimensions must EXACTLY match device class (e.g., 1320×2868 for 6.9")
- Maximum 10 screenshots per locale per device class

**Step 5 — In-App Purchase & subscription management**
Create IAP:
```
POST /v1/inAppPurchases
{ "type": "inAppPurchases", "attributes": { "name": "Pro Upgrade", "productId": "com.example.pro", "inAppPurchaseType": "NON_CONSUMABLE" } }
```
For subscriptions: create subscription group first, then subscription within group, then set pricing per territory. Territory pricing requires concurrent requests with retry — ASC rate limits per-territory pricing endpoints aggressively.

**Step 6 — Submission readiness check**
Before submitting for review, verify completeness:
- [ ] App Store version exists with build attached
- [ ] All required locales have description, keywords, screenshots
- [ ] Screenshots uploaded for ALL required device classes (6.9", 6.7", 6.5")
- [ ] Age rating questionnaire completed
- [ ] App review contact info set (first name, last name, phone, email)
- [ ] Privacy policy URL set
- [ ] Export compliance answered
- [ ] Content rights declaration completed (if app has third-party content)

**Step 7 — Submit and monitor**
```
POST /v1/appStoreVersionSubmissions
{ "data": { "relationships": { "appStoreVersion": { "data": { "type": "appStoreVersions", "id": "<version-id>" } } } } }
```
Poll `GET /v1/appStoreVersions/{id}` for `appStoreState` transitions: `WAITING_FOR_REVIEW` → `IN_REVIEW` → `READY_FOR_SALE` (or `REJECTED`). On rejection: fetch `appStoreVersionSubmissions` for reviewer notes.

#### Example

```typescript
// Complete ASC API client pattern
interface ASCClient {
  // Auth
  refreshToken(): string;

  // Apps
  listApps(): Promise<ASCApp[]>;
  getApp(id: string): Promise<ASCApp>;

  // Versions
  createVersion(appId: string, version: string): Promise<ASCVersion>;
  attachBuild(versionId: string, buildId: string): Promise<void>;

  // Localization
  updateLocalization(versionId: string, locale: string, data: LocalizationData): Promise<void>;

  // Screenshots (3-step)
  reserveScreenshot(setId: string, fileName: string, fileSize: number): Promise<UploadOps>;
  uploadChunks(ops: UploadOps, fileBuffer: Buffer): Promise<void>;
  commitScreenshot(screenshotId: string, checksum: string): Promise<void>;

  // IAP
  createIAP(appId: string, name: string, productId: string, type: IAPType): Promise<ASCIAP>;

  // Submission
  checkReadiness(versionId: string): Promise<ReadinessReport>;
  submitForReview(versionId: string): Promise<void>;
  pollReviewStatus(versionId: string, intervalMs?: number): AsyncGenerator<ReviewStatus>;

  // Reviews
  listCustomerReviews(appId: string): Promise<CustomerReview[]>;
  respondToReview(reviewId: string, body: string): Promise<void>;
}

// Pagination helper — ASC uses cursor-based pagination via `next` links
async function* paginate<T>(client: ASCClient, url: string): AsyncGenerator<T> {
  let nextUrl: string | null = url;
  while (nextUrl) {
    const response = await client.request(nextUrl);
    for (const item of response.data) {
      yield item as T;
    }
    nextUrl = response.links?.next ?? null;
  }
}
```

---

## Connections

```
Calls → browser-pilot (L3): device testing and screenshot automation
Calls → asset-creator (L3): generate app icons and splash screens
Calls → sentinel (L2): audit push notification security, deep link validation
Calls → verification (L3): run mobile-specific checks (build, lint, type-check)
Calls → @rune/ui (L4): design system tokens, palette, typography for mobile UI consistency
Calls → @rune/backend (L4): API patterns for mobile backend integration (auth, push server)
Calls → @rune/security (L4): code signing audit, API key management, certificate validation
Called By ← cook (L1): when mobile task detected
Called By ← team (L1): when porting web to mobile
Called By ← launch (L1): app store submission flow
Called By ← deploy (L2): mobile-specific deployment (EAS Build, Fastlane)
Inter-skill: ios-build-pipeline → app-store-prep (pipeline feeds into submission checklist)
Inter-skill: app-store-connect → app-store-prep (API automation completes manual checklist items)
Inter-skill: ios-build-pipeline → app-store-connect (upload build → attach to version → submit)
```

## Tech Stack Support

| Framework | State Management | Navigation | Build | OTA |
|-----------|-----------------|------------|-------|-----|
| React Native (bare) | Zustand / Redux | React Navigation v7 | Metro + Gradle/Xcode | CodePush |
| Expo (managed) | Zustand | Expo Router v4 | EAS Build | EAS Update |
| Flutter | Riverpod / BLoC | GoRouter | Flutter CLI | Shorebird |
| Native iOS (Swift) | SwiftUI @Observable | NavigationStack | xcodebuild | — |

## Constraints

1. MUST test on both iOS and Android physical devices — simulators cannot test deep links, push notifications, or biometric auth.
2. MUST NOT ship with debug configurations (Flipper, dev menu, debug signing) in production builds.
3. MUST include specific (not generic) usage descriptions for every permission requested — vague descriptions cause instant rejection.
4. MUST include `PrivacyInfo.xcprivacy` for iOS builds targeting App Store (mandatory since April 2025).
5. MUST handle offline gracefully — mobile apps lose connectivity; show cached data or clear offline state.
6. MUST use platform-adaptive components (Material on Android, Cupertino on iOS) or declare a unified design system.
7. MUST verify Hermes bytecode compatibility before deploying OTA updates across RN version boundaries.
8. MUST use FCM v1 HTTP API with service account auth — FCM Legacy API is shut down.

## Sharp Edges

| Failure Mode | Severity | Mitigation |
|---|---|---|
| New Architecture breaks third-party libraries silently (shim fallback, no error) | CRITICAL | Audit all native dependencies against `reactnative.directory` before SDK upgrade |
| `setNativeProps` removed in New Architecture — blank screen, no error | CRITICAL | Search codebase for `setNativeProps`, migrate to Reanimated or Animated API |
| Reanimated < 3.8 causes Android animation stutter on New Architecture | HIGH | Pin `react-native-reanimated` >= 3.8, test animations on Android physical device |
| OTA update built against different RN version crashes on launch (Hermes bytecode mismatch) | CRITICAL | Never OTA across RN version boundaries; use `fingerprintExperimental` runtime version |
| Universal Links silently break when AASA endpoint redirects (HTTP→HTTPS) | HIGH | Serve AASA at exact path, no redirects, verify with `curl -I` |
| `getDevicePushTokenAsync()` never resolves on SDK 53+ (race condition) | HIGH | Call after app fully initialized, not in root layout mount; add timeout fallback |
| Push notifications removed from Expo Go on Android (SDK 53+) | MEDIUM | Use development builds (`eas build --profile development`) for all notification testing |
| App Store rejection for missing `PrivacyInfo.xcprivacy` (auto-rejection, no human review) | CRITICAL | Add privacy manifest to iOS build; audit all APIs that require declaration |
| Firebase Dynamic Links shut down August 2025 — all `page.link` URLs dead | CRITICAL | Migrate to Branch.io, custom deep link server, or standard App Links/Universal Links |
| React Navigation v7 `navigate()` pushes duplicate screens (changed from v6) | MEDIUM | Audit all `navigate()` calls; use `popTo()` or check current route before navigating |
| `react-native-fast-image` not compatible with New Architecture | HIGH | Migrate to `expo-image` (supports blurhash, transitions, caching) |
| FlatList `removeClippedSubviews` causes blank cells on fast scroll | MEDIUM | Test with 1000+ items; prefer FlashList with accurate `estimatedItemSize` |
| iOS 18 requires explicit permission before scheduling local notifications | MEDIUM | Always call `requestPermissionsAsync()` before any `scheduleNotificationAsync()` |
| EAS Build queue: free tier 30-60 min wait during peak hours | LOW | Use `--local` flag for faster iteration; upgrade to priority queue for production |
| OpenSSL 3.x `.p12` export silently fails without `-legacy` flag (macOS 14+) | CRITICAL | Always use `openssl pkcs12 -export -legacy` for certificate bundles |
| `xcodebuild` export failure diagnostics hidden in archive log | HIGH | On export failure, read `IDEDistribution.standard-log.txt` inside `.xcarchive` |
| ASC API rate limit: 200 req/min, 429 with no Retry-After header | HIGH | Implement exponential backoff; batch territory pricing operations |
| ASC JWT expires in 20 min — stale token returns 401 with misleading error | MEDIUM | Auto-refresh token when within 60s of expiry |
| Screenshot upload: chunks must ALL complete before commit or upload silently corrupts | HIGH | Verify all chunk PUT responses before PATCH commit |
| TestFlight build processing takes 5-30 min — no webhook, must poll | MEDIUM | Poll every 30s with 30-min timeout; store build number for resume |

## Done When

- React Native/Flutter codebase audited for New Architecture compatibility with migration plan
- Deep links working on both platforms with authentication integration and real device verification
- Push notifications delivering reliably via FCM v1 with proper permission handling
- OTA update strategy configured with runtime version management and rollback procedure
- FlatList/FlashList optimized with memoization, key extraction, and window sizing
- App store metadata generated with correct dimensions, privacy manifest, and platform-specific requirements
- Native bridges typed and error-handled for both platforms using modern APIs (Expo Modules or TurboModules)
- iOS build pipeline producing signed IPA with idempotent signing state
- App Store Connect operations automated — version, localization, screenshots, IAP, submission
- Structured report emitted for each skill invoked

## Cost Profile

~16,000–32,000 tokens per full pack run (all 9 skills). Individual skill: ~2,000–5,000 tokens. Sonnet default. Use haiku for config detection; escalate to sonnet for code generation, build pipeline, and ASC API patterns.
