---
name: mobile-developer
description: Cross-platform mobile development: React Native and Flutter patterns, native bridge design, offline-first architecture, push notifications, and app store deployment.
---

# Mobile Developer

Production-grade cross-platform mobile development skill covering React Native and Flutter in depth. Addresses the full lifecycle of a mobile app: framework selection, native bridge design, offline-first data architecture, push notification plumbing, performance optimization, testing on real devices, and shipping through app store review. Every section includes working code, rationale for design choices, and anti-patterns to avoid.

## Table of Contents

1. [React Native](#1-react-native)
2. [Flutter](#2-flutter)
3. [Offline-First Architecture](#3-offline-first-architecture)
4. [Push Notifications](#4-push-notifications)
5. [Performance Optimization](#5-performance-optimization)
6. [Native Bridge Design](#6-native-bridge-design)
7. [Testing Mobile Apps](#7-testing-mobile-apps)
8. [App Store Deployment](#8-app-store-deployment)

---

## 1. React Native

### Expo vs Bare Workflow

Expo provides a managed build pipeline, over-the-air updates, and a curated set of native modules. Use Expo when the project does not require custom native code that falls outside the Expo SDK. Use the bare workflow (or a dev client) when you need full control over the native layer.

| Criteria | Expo Managed | Bare / Dev Client |
|---|---|---|
| Custom native modules | Via config plugins or dev client | Full native access |
| Build pipeline | EAS Build (cloud) | Xcode / Gradle locally |
| OTA updates | Built-in (EAS Update) | Manual (CodePush or EAS) |
| App size | Slightly larger (includes Expo runtime) | Minimal — only what you link |
| Upgrade path | `npx expo install` handles compat | Manual RN upgrade, pod install |

Starting a new project:

```bash
# Expo managed (recommended default)
npx create-expo-app@latest my-app --template blank-typescript

# Bare workflow with Expo modules
npx react-native@latest init MyApp
npx install-expo-modules@latest
```

### New Architecture: Turbo Modules and Fabric

React Native's New Architecture replaces the old bridge with a synchronous JSI (JavaScript Interface) layer. Turbo Modules replace Native Modules, and Fabric replaces the old renderer.

```typescript
// turbo module spec — src/specs/NativeDeviceInfo.ts
import type { TurboModule } from "react-native";
import { TurboModuleRegistry } from "react-native";

export interface Spec extends TurboModule {
  getDeviceName(): string;
  getBatteryLevel(): Promise<number>;
}

export default TurboModuleRegistry.getEnforcing<Spec>("DeviceInfo");
```

Enable the New Architecture in `react-native.config.js`:

```javascript
module.exports = {
  project: {
    android: { unstable_reactLegacyComponentNames: ["MapView"] },
    ios: {},
  },
};
```

And in `android/gradle.properties`:

```properties
newArchEnabled=true
```

### Navigation with React Navigation

```typescript
// app/navigation/types.ts
export type RootStackParamList = {
  Home: undefined;
  Profile: { userId: string };
  Settings: undefined;
};

// app/navigation/RootStack.tsx
import { createNativeStackNavigator } from "@react-navigation/native-stack";
import type { RootStackParamList } from "./types";

const Stack = createNativeStackNavigator<RootStackParamList>();

export function RootStack() {
  return (
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen
        name="Profile"
        component={ProfileScreen}
        options={{ headerBackTitle: "Back" }}
      />
      <Stack.Screen name="Settings" component={SettingsScreen} />
    </Stack.Navigator>
  );
}
```

Type-safe navigation hooks:

```typescript
import { useNavigation } from "@react-navigation/native";
import type { NativeStackNavigationProp } from "@react-navigation/native-stack";
import type { RootStackParamList } from "./types";

type NavigationProp = NativeStackNavigationProp<RootStackParamList>;

function HomeScreen() {
  const navigation = useNavigation<NavigationProp>();

  return (
    <Button
      title="View Profile"
      onPress={() => navigation.navigate("Profile", { userId: "u-123" })}
    />
  );
}
```

### Hermes Engine

Hermes is the default JavaScript engine for React Native. It compiles JS to bytecode at build time, reducing startup time and memory usage.

```json
// app.json (Expo)
{
  "expo": {
    "jsEngine": "hermes"
  }
}
```

Verify Hermes is active at runtime:

```typescript
const isHermes = () => !!global.HermesInternal;
console.log("Hermes enabled:", isHermes());
```

Hermes supports Chrome DevTools Protocol for debugging — connect via `npx react-native start` and open Chrome DevTools.

### Anti-patterns

- Ejecting from Expo prematurely — use config plugins and dev clients first; ejecting creates a maintenance burden.
- Ignoring the New Architecture — the old bridge is deprecated; new libraries target Turbo Modules and Fabric.
- Storing navigation state in Redux/Zustand — React Navigation manages its own state; duplicating it causes sync bugs.
- Disabling Hermes for "compatibility" — Hermes handles 99% of JS syntax; the rare incompatibility is better fixed than worked around.

---

## 2. Flutter

### Widget Tree and Composition

Everything in Flutter is a widget. Build UIs by composing small, focused widgets rather than creating deep inheritance hierarchies.

```dart
class ProfileCard extends StatelessWidget {
  final String name;
  final String avatarUrl;
  final VoidCallback onTap;

  const ProfileCard({
    super.key,
    required this.name,
    required this.avatarUrl,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Card(
        elevation: 2,
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              CircleAvatar(backgroundImage: NetworkImage(avatarUrl)),
              const SizedBox(width: 12),
              Text(name, style: Theme.of(context).textTheme.titleMedium),
            ],
          ),
        ),
      ),
    );
  }
}
```

### State Management with Riverpod

Riverpod provides compile-safe, testable state management without relying on BuildContext.

```dart
// providers.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'providers.g.dart';

@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
  void decrement() => state--;
}

@riverpod
Future<List<Product>> products(ProductsRef ref) async {
  final response = await ref.read(apiClientProvider).get('/api/v1/products');
  return (response.data as List).map((e) => Product.fromJson(e)).toList();
}
```

Consuming providers in a widget:

```dart
class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);

    return Scaffold(
      body: Center(child: Text('Count: $count')),
      floatingActionButton: FloatingActionButton(
        onPressed: () => ref.read(counterProvider.notifier).increment(),
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

### State Management with Bloc

```dart
// events
sealed class AuthEvent {}
class LoginRequested extends AuthEvent {
  final String email;
  final String password;
  LoginRequested({required this.email, required this.password});
}
class LogoutRequested extends AuthEvent {}

// states
sealed class AuthState {}
class AuthInitial extends AuthState {}
class AuthLoading extends AuthState {}
class AuthAuthenticated extends AuthState {
  final User user;
  AuthAuthenticated(this.user);
}
class AuthError extends AuthState {
  final String message;
  AuthError(this.message);
}

// bloc
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final AuthRepository _repo;

  AuthBloc(this._repo) : super(AuthInitial()) {
    on<LoginRequested>((event, emit) async {
      emit(AuthLoading());
      try {
        final user = await _repo.login(event.email, event.password);
        emit(AuthAuthenticated(user));
      } catch (e) {
        emit(AuthError(e.toString()));
      }
    });

    on<LogoutRequested>((event, emit) async {
      await _repo.logout();
      emit(AuthInitial());
    });
  }
}
```

### Platform Channels

Communicate between Dart and native code (iOS/Android) using method channels.

```dart
// Dart side
import 'package:flutter/services.dart';

class BatteryService {
  static const _channel = MethodChannel('com.example.app/battery');

  Future<int> getBatteryLevel() async {
    final level = await _channel.invokeMethod<int>('getBatteryLevel');
    return level ?? -1;
  }
}
```

```kotlin
// Android — MainActivity.kt
class MainActivity : FlutterActivity() {
    private val CHANNEL = "com.example.app/battery"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                if (call.method == "getBatteryLevel") {
                    val level = getBatteryLevel()
                    result.success(level)
                } else {
                    result.notImplemented()
                }
            }
    }

    private fun getBatteryLevel(): Int {
        val manager = getSystemService(BATTERY_SERVICE) as BatteryManager
        return manager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
    }
}
```

### Custom Painting

```dart
class ChartPainter extends CustomPainter {
  final List<double> data;
  ChartPainter(this.data);

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue
      ..strokeWidth = 2
      ..style = PaintingStyle.stroke;

    final path = Path();
    final stepX = size.width / (data.length - 1);
    final maxVal = data.reduce((a, b) => a > b ? a : b);

    for (var i = 0; i < data.length; i++) {
      final x = i * stepX;
      final y = size.height - (data[i] / maxVal) * size.height;
      if (i == 0) {
        path.moveTo(x, y);
      } else {
        path.lineTo(x, y);
      }
    }
    canvas.drawPath(path, paint);
  }

  @override
  bool shouldRepaint(ChartPainter old) => old.data != data;
}

// Usage
CustomPaint(
  size: const Size(double.infinity, 200),
  painter: ChartPainter([10, 25, 18, 40, 32, 55]),
)
```

### Anti-patterns

- Putting business logic inside `build()` methods — extract it into providers, blocs, or service classes.
- Using `setState()` for app-wide state — it only works within a single widget; use Riverpod or Bloc for shared state.
- Creating deeply nested widget trees in a single file — extract widgets into separate classes for readability and rebuild efficiency.
- Ignoring `const` constructors — Flutter skips rebuild for `const` widgets, so omitting `const` wastes frames.

---

## 3. Offline-First Architecture

### Local Database Options

| Database | Best For | Platform |
|---|---|---|
| SQLite (via `sqflite` / `expo-sqlite`) | Structured relational data, complex queries | Both |
| WatermelonDB | Large datasets with lazy loading, sync built-in | React Native |
| Realm | Object-oriented data, real-time listeners | Both |
| Hive / Isar | Fast key-value and object storage (Flutter) | Flutter |
| MMKV | Small key-value pairs, preferences | Both |

### SQLite with Type-Safe Queries (React Native)

```typescript
import * as SQLite from "expo-sqlite";

const db = await SQLite.openDatabaseAsync("app.db");

// Create tables with migrations
await db.execAsync(`
  CREATE TABLE IF NOT EXISTS tasks (
    id TEXT PRIMARY KEY NOT NULL,
    title TEXT NOT NULL,
    completed INTEGER NOT NULL DEFAULT 0,
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    synced INTEGER NOT NULL DEFAULT 0
  );
`);

// Typed insert
async function createTask(task: { id: string; title: string }): Promise<void> {
  await db.runAsync(
    "INSERT INTO tasks (id, title) VALUES (?, ?)",
    [task.id, task.title]
  );
}

// Typed query
interface TaskRow {
  id: string;
  title: string;
  completed: number;
  updated_at: string;
  synced: number;
}

async function getPendingTasks(): Promise<TaskRow[]> {
  return db.getAllAsync<TaskRow>(
    "SELECT * FROM tasks WHERE completed = 0 ORDER BY updated_at DESC"
  );
}
```

### WatermelonDB Sync Strategy

```typescript
// model/Task.ts
import { Model } from "@nozbe/watermelondb";
import { field, date, readonly } from "@nozbe/watermelondb/decorators";

class Task extends Model {
  static table = "tasks";

  @field("title") title!: string;
  @field("completed") completed!: boolean;
  @readonly @date("created_at") createdAt!: Date;
  @date("updated_at") updatedAt!: Date;
}

// sync.ts
import { synchronize } from "@nozbe/watermelondb/sync";

async function syncWithServer(database: Database) {
  await synchronize({
    database,
    pullChanges: async ({ lastPulledAt }) => {
      const response = await fetch(
        `/api/v1/sync?last_pulled_at=${lastPulledAt ?? 0}`
      );
      const { changes, timestamp } = await response.json();
      return { changes, timestamp };
    },
    pushChanges: async ({ changes, lastPulledAt }) => {
      await fetch("/api/v1/sync", {
        method: "POST",
        body: JSON.stringify({ changes, lastPulledAt }),
        headers: { "Content-Type": "application/json" },
      });
    },
  });
}
```

### Conflict Resolution Strategies

| Strategy | How It Works | Best For |
|---|---|---|
| Last-write-wins | Highest timestamp wins | Simple apps, low conflict risk |
| Field-level merge | Merge non-conflicting fields, flag conflicts | Collaborative editing |
| Operational transform | Transform concurrent operations | Real-time collab (Google Docs style) |
| CRDT | Conflict-free replicated data types | Distributed, peer-to-peer sync |

Field-level merge implementation:

```typescript
interface SyncRecord {
  id: string;
  fields: Record<string, { value: unknown; updatedAt: number }>;
}

function mergeRecords(local: SyncRecord, remote: SyncRecord): SyncRecord {
  const merged: SyncRecord = { id: local.id, fields: {} };

  const allKeys = new Set([
    ...Object.keys(local.fields),
    ...Object.keys(remote.fields),
  ]);

  for (const key of allKeys) {
    const localField = local.fields[key];
    const remoteField = remote.fields[key];

    if (!localField) {
      merged.fields[key] = remoteField;
    } else if (!remoteField) {
      merged.fields[key] = localField;
    } else {
      // Per-field last-write-wins
      merged.fields[key] =
        remoteField.updatedAt > localField.updatedAt ? remoteField : localField;
    }
  }

  return merged;
}
```

### Optimistic Updates

Apply changes locally before the server confirms, then reconcile.

```typescript
async function toggleTask(taskId: string): Promise<void> {
  // 1. Optimistic update — flip immediately in local DB
  const task = await db.getFirstAsync<TaskRow>(
    "SELECT * FROM tasks WHERE id = ?", [taskId]
  );
  if (!task) return;

  const newCompleted = task.completed ? 0 : 1;
  await db.runAsync(
    "UPDATE tasks SET completed = ?, synced = 0, updated_at = datetime('now') WHERE id = ?",
    [newCompleted, taskId]
  );

  // 2. Notify UI (listeners or state update)
  emitter.emit("task-updated", taskId);

  // 3. Push to server — retry on failure
  try {
    await fetch(`/api/v1/tasks/${taskId}`, {
      method: "PATCH",
      body: JSON.stringify({ completed: !!newCompleted }),
      headers: { "Content-Type": "application/json" },
    });
    await db.runAsync("UPDATE tasks SET synced = 1 WHERE id = ?", [taskId]);
  } catch {
    // Stays unsynced — background sync will retry
    enqueueRetry({ type: "toggle-task", taskId });
  }
}
```

### Anti-patterns

- Treating the server as the source of truth for UI state — in offline-first, the local database is the source of truth; the server is a replication target.
- Not tracking sync status per record — without a `synced` flag or `dirty` queue, you cannot efficiently push only changed records.
- Using `AsyncStorage` / `SharedPreferences` for structured data — these are key-value stores designed for preferences, not queryable datasets.
- Ignoring conflict resolution — "last write wins everywhere" silently drops user data in multi-device scenarios.

---

## 4. Push Notifications

### APNs and FCM Overview

| | APNs (Apple) | FCM (Google) |
|---|---|---|
| Transport | HTTP/2 to Apple servers | HTTP v1 API to Google servers |
| Auth | JWT or certificate | Service account or OAuth token |
| Payload limit | 4 KB | 4 KB (data), 4 KB (notification) |
| Priority | 5 (low) or 10 (immediate) | normal or high |
| Token format | 64-byte hex device token | ~150 char registration token |

### Unified Setup (React Native — Expo Notifications)

```typescript
import * as Notifications from "expo-notifications";
import * as Device from "expo-device";
import { Platform } from "react-native";

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

async function registerForPushNotifications(): Promise<string | null> {
  if (!Device.isDevice) {
    console.warn("Push notifications require a physical device");
    return null;
  }

  const { status: existing } = await Notifications.getPermissionsAsync();
  let finalStatus = existing;

  if (existing !== "granted") {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== "granted") return null;

  // Android notification channel
  if (Platform.OS === "android") {
    await Notifications.setNotificationChannelAsync("default", {
      name: "Default",
      importance: Notifications.AndroidImportance.MAX,
      vibrationPattern: [0, 250, 250, 250],
    });
  }

  const token = await Notifications.getExpoPushTokenAsync({
    projectId: "your-expo-project-id",
  });

  return token.data;
}
```

### Flutter Push Notifications (Firebase Messaging)

```dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

// Top-level handler for background messages
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  // Process data payload — no UI access here
  print('Background message: ${message.messageId}');
}

class PushNotificationService {
  final _messaging = FirebaseMessaging.instance;
  final _localNotifications = FlutterLocalNotificationsPlugin();

  Future<void> initialize() async {
    // Request permission (iOS)
    await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    // Background handler
    FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

    // Foreground handler
    FirebaseMessaging.onMessage.listen((message) {
      _showLocalNotification(message);
    });

    // Setup local notification display
    await _localNotifications.initialize(
      const InitializationSettings(
        android: AndroidInitializationSettings('@mipmap/ic_launcher'),
        iOS: DarwinInitializationSettings(),
      ),
      onDidReceiveNotificationResponse: _handleNotificationTap,
    );
  }

  Future<String?> getToken() => _messaging.getToken();

  void _showLocalNotification(RemoteMessage message) {
    final notification = message.notification;
    if (notification == null) return;

    _localNotifications.show(
      notification.hashCode,
      notification.title,
      notification.body,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'high_importance',
          'High Importance',
          importance: Importance.max,
        ),
      ),
      payload: message.data['deepLink'],
    );
  }

  void _handleNotificationTap(NotificationResponse response) {
    final deepLink = response.payload;
    if (deepLink != null) {
      // Navigate to deep link
      navigatorKey.currentState?.pushNamed(deepLink);
    }
  }
}
```

### Deep Linking from Notifications

```typescript
// React Native — handle notification tap
import * as Notifications from "expo-notifications";
import * as Linking from "expo-linking";
import { useEffect } from "react";

function useNotificationDeepLink() {
  useEffect(() => {
    // App was opened by tapping a notification
    const subscription = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const deepLink = response.notification.request.content.data?.deepLink;
        if (typeof deepLink === "string") {
          Linking.openURL(deepLink);
        }
      }
    );

    return () => subscription.remove();
  }, []);
}
```

### Silent Push and Background Processing

Silent pushes wake the app without displaying a notification. Use them to trigger background data sync.

```typescript
// Server-side payload for silent push (APNs)
const silentPayload = {
  aps: {
    "content-available": 1, // silent push flag
  },
  syncType: "new-messages",
};

// Server-side payload for FCM data-only message
const fcmDataMessage = {
  message: {
    token: deviceToken,
    data: {
      syncType: "new-messages",
    },
    android: {
      priority: "high",
    },
    apns: {
      headers: { "apns-priority": "5" },
      payload: { aps: { "content-available": 1 } },
    },
  },
};
```

### Android Notification Channels

Android 8+ requires notification channels. Define them at app startup.

```typescript
import * as Notifications from "expo-notifications";

async function setupChannels() {
  await Notifications.setNotificationChannelAsync("messages", {
    name: "Messages",
    description: "New message notifications",
    importance: Notifications.AndroidImportance.HIGH,
    sound: "message_tone.wav",
  });

  await Notifications.setNotificationChannelAsync("promotions", {
    name: "Promotions",
    description: "Deals and offers",
    importance: Notifications.AndroidImportance.LOW,
  });
}
```

### Anti-patterns

- Requesting notification permission on first launch without context — users deny permissions they do not understand. Show a pre-permission screen explaining the value.
- Sending all notifications as high priority — both Apple and Google throttle apps that abuse priority, eventually delivering notifications with delay.
- Not handling token refresh — device tokens change; subscribe to token refresh events and update the server.
- Ignoring notification channels on Android — all notifications fall into a default low-importance channel, and users cannot configure them.

---

## 5. Performance Optimization

### FlatList / RecyclerView Optimization (React Native)

```typescript
import { FlatList, type ListRenderItem } from "react-native";

interface Item {
  id: string;
  title: string;
  thumbnail: string;
}

const renderItem: ListRenderItem<Item> = ({ item }) => (
  <ItemCard item={item} />
);

const keyExtractor = (item: Item) => item.id;

function FeedList({ items }: { items: Item[] }) {
  return (
    <FlatList
      data={items}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      // Performance tuning
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      windowSize={5}
      initialNumToRender={10}
      getItemLayout={(_, index) => ({
        length: 80,
        offset: 80 * index,
        index,
      })}
    />
  );
}
```

For very large lists, use `@shopify/flash-list` as a drop-in replacement:

```typescript
import { FlashList } from "@shopify/flash-list";

<FlashList
  data={items}
  renderItem={renderItem}
  keyExtractor={keyExtractor}
  estimatedItemSize={80}
/>
```

### Image Caching

```typescript
// React Native — expo-image (fast, disk-cached, blurhash placeholders)
import { Image } from "expo-image";

<Image
  source={{ uri: "https://example.com/photo.jpg" }}
  placeholder={{ blurhash: "LGF5]+Yk^6#M@-5c,1J5@[or[Q6." }}
  contentFit="cover"
  transition={200}
  cachePolicy="memory-disk"
  style={{ width: 300, height: 200 }}
/>
```

```dart
// Flutter — cached_network_image
import 'package:cached_network_image/cached_network_image.dart';

CachedNetworkImage(
  imageUrl: 'https://example.com/photo.jpg',
  placeholder: (context, url) => const CircularProgressIndicator(),
  errorWidget: (context, url, error) => const Icon(Icons.error),
  memCacheWidth: 300,  // decode at display size, not full resolution
)
```

### Bundle Size Reduction

React Native:

```javascript
// metro.config.js — tree shaking with re.pack or Metro minifier
const { getDefaultConfig } = require("expo/metro-config");
const config = getDefaultConfig(__dirname);

config.transformer.minifierPath = "metro-minify-terser";
config.transformer.minifierConfig = {
  compress: { drop_console: true },
};

module.exports = config;
```

```bash
# Analyze bundle contents
npx react-native-bundle-visualizer
```

Flutter:

```bash
# Build with size analysis
flutter build apk --analyze-size
flutter build ios --analyze-size

# Deferred components (lazy load features)
flutter build appbundle --deferred-components
```

### Startup Time Optimization

| Technique | Platform | Impact |
|---|---|---|
| Hermes bytecode precompilation | React Native | 30-50% faster JS init |
| Reduce auto-linked native modules | React Native | Fewer modules = faster init |
| `flutter build --split-debug-info` | Flutter | Smaller binary, faster load |
| Lazy-load heavy screens | Both | Defer non-critical code |
| Splash screen bridging | Both | Perceived perf while loading |

### Memory Profiling

```bash
# React Native — Flipper memory profiler
npx react-native start
# Open Flipper > Memory tab > Take heap snapshot

# React Native — Hermes memory timeline
npx react-native start
# Chrome DevTools > Memory > Allocation timeline

# Flutter — DevTools memory view
flutter run --profile
# Open DevTools > Memory tab > Monitor allocations
```

Watch for:
- Image objects not released after scrolling off screen
- Stale subscriptions in `useEffect` or `StreamSubscription` without cleanup
- Large JSON responses parsed into memory and never garbage collected

### Anti-patterns

- Using `ScrollView` for long lists — it renders all children at once; use `FlatList` or `FlashList` for virtualization.
- Loading full-resolution images for thumbnails — decode at display size using `memCacheWidth` / resize on server.
- Skipping `getItemLayout` on fixed-height lists — without it, FlatList measures every item, causing jank on scroll.
- Running performance tests in debug mode — always profile with `--release` (React Native) or `--profile` (Flutter) builds.

---

## 6. Native Bridge Design

### React Native: Turbo Modules (New Architecture)

Turbo Modules provide synchronous, type-safe access to native code via JSI (JavaScript Interface).

```typescript
// specs/NativeStorage.ts — codegen spec
import type { TurboModule } from "react-native";
import { TurboModuleRegistry } from "react-native";

export interface Spec extends TurboModule {
  setItem(key: string, value: string): Promise<void>;
  getItem(key: string): Promise<string | null>;
  removeItem(key: string): Promise<void>;
  getAllKeys(): Promise<string[]>;
}

export default TurboModuleRegistry.getEnforcing<Spec>("SecureStorage");
```

```objc
// ios/SecureStorage.mm — native implementation
#import <SecureStorageSpec/SecureStorageSpec.h>
#import <Security/Security.h>

@interface SecureStorage : NSObject <NativeSecureStorageSpec>
@end

@implementation SecureStorage

RCT_EXPORT_MODULE()

- (void)setItem:(NSString *)key value:(NSString *)value
        resolve:(RCTPromiseResolveBlock)resolve
         reject:(RCTPromiseRejectBlock)reject {
  NSData *data = [value dataUsingEncoding:NSUTF8StringEncoding];
  NSDictionary *query = @{
    (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
    (__bridge id)kSecAttrAccount: key,
    (__bridge id)kSecValueData: data,
  };
  SecItemDelete((__bridge CFDictionaryRef)query);
  OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
  status == errSecSuccess ? resolve(nil) : reject(@"ERR", @"Keychain write failed", nil);
}

// ... getItem, removeItem, getAllKeys implementations

- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params {
  return std::make_shared<facebook::react::NativeSecureStorageSpecJSI>(params);
}

@end
```

### React Native: Fabric View Components

Custom native views using the new Fabric renderer:

```typescript
// specs/MapViewNativeComponent.ts
import type { ViewProps } from "react-native";
import codegenNativeComponent from "react-native/Libraries/Utilities/codegenNativeComponent";
import type { Double } from "react-native/Libraries/Types/CodegenTypes";

interface NativeProps extends ViewProps {
  latitude: Double;
  longitude: Double;
  zoomLevel: Double;
}

export default codegenNativeComponent<NativeProps>("MapView");
```

### Flutter: Method Channels vs Event Channels

| Channel Type | Direction | Use Case |
|---|---|---|
| MethodChannel | Dart <-> Native (request/response) | One-off calls (get battery, open camera) |
| EventChannel | Native -> Dart (stream) | Continuous data (location updates, sensor data) |
| BasicMessageChannel | Bidirectional (low-level) | Custom codecs, binary data |

Event channel example (streaming accelerometer data):

```dart
// Dart side
class AccelerometerService {
  static const _channel = EventChannel('com.example.app/accelerometer');
  Stream<AccelData>? _stream;

  Stream<AccelData> get stream {
    _stream ??= _channel
        .receiveBroadcastStream()
        .map((event) => AccelData.fromMap(event as Map));
    return _stream!;
  }
}
```

```kotlin
// Android — AccelerometerPlugin.kt
class AccelerometerPlugin : EventChannel.StreamHandler {
    private var sensorManager: SensorManager? = null
    private var listener: SensorEventListener? = null

    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                events?.success(mapOf(
                    "x" to event.values[0],
                    "y" to event.values[1],
                    "z" to event.values[2],
                ))
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }
        sensorManager?.registerListener(
            listener,
            sensorManager?.getDefaultSensor(Sensor.TYPE_ACCELEROMETER),
            SensorManager.SENSOR_DELAY_UI
        )
    }

    override fun onCancel(arguments: Any?) {
        sensorManager?.unregisterListener(listener)
    }
}
```

### Platform-Specific Code (React Native)

```typescript
// Using Platform.select
import { Platform, StyleSheet } from "react-native";

const styles = StyleSheet.create({
  shadow: Platform.select({
    ios: {
      shadowColor: "#000",
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.15,
      shadowRadius: 4,
    },
    android: {
      elevation: 4,
    },
  }),
});

// Using platform-specific file extensions
// components/Header.ios.tsx — iOS-specific implementation
// components/Header.android.tsx — Android-specific implementation
// components/Header.tsx — fallback / shared
```

### Anti-patterns

- Using the old bridge (`NativeModules`) in new projects — Turbo Modules are faster and type-safe; the old bridge is deprecated.
- Sending large binary data through method channels — use shared memory, file URIs, or `BasicMessageChannel` with binary codec instead.
- Not handling platform differences in channel implementations — always implement both iOS and Android, or throw `MissingPluginException` with a clear message.
- Creating one massive channel for all native calls — split channels by domain (`/battery`, `/storage`, `/sensors`) for maintainability.

---

## 7. Testing Mobile Apps

### Detox (End-to-End for React Native)

```typescript
// e2e/login.test.ts
import { device, element, by, expect } from "detox";

describe("Login flow", () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  beforeEach(async () => {
    await device.reloadReactNative();
  });

  it("should login with valid credentials", async () => {
    await element(by.id("email-input")).typeText("user@test.com");
    await element(by.id("password-input")).typeText("secret123");
    await element(by.id("login-button")).tap();

    await expect(element(by.text("Welcome back"))).toBeVisible();
  });

  it("should show error for invalid credentials", async () => {
    await element(by.id("email-input")).typeText("user@test.com");
    await element(by.id("password-input")).typeText("wrong");
    await element(by.id("login-button")).tap();

    await expect(element(by.id("error-message"))).toBeVisible();
  });
});
```

Detox configuration:

```javascript
// .detoxrc.js
module.exports = {
  testRunner: { args: { config: "e2e/jest.config.js" } },
  apps: {
    "ios.release": {
      type: "ios.app",
      binaryPath: "ios/build/MyApp.app",
      build: "xcodebuild -workspace ios/MyApp.xcworkspace -scheme MyApp -configuration Release -sdk iphonesimulator -derivedDataPath ios/build",
    },
    "android.release": {
      type: "android.apk",
      binaryPath: "android/app/build/outputs/apk/release/app-release.apk",
      build: "cd android && ./gradlew assembleRelease",
    },
  },
  devices: {
    simulator: { type: "ios.simulator", device: { type: "iPhone 15" } },
    emulator: { type: "android.emulator", device: { avdName: "Pixel_7" } },
  },
  configurations: {
    "ios.release": { device: "simulator", app: "ios.release" },
    "android.release": { device: "emulator", app: "android.release" },
  },
};
```

### Maestro (Cross-Platform E2E)

Maestro uses YAML flows — no code, no language lock-in, works with React Native, Flutter, and native apps.

```yaml
# .maestro/login-flow.yaml
appId: com.example.myapp
---
- launchApp
- tapOn: "Email"
- inputText: "user@test.com"
- tapOn: "Password"
- inputText: "secret123"
- tapOn: "Sign In"
- assertVisible: "Welcome back"

# Scroll and verify list
- scrollUntilVisible:
    element: "Item 50"
    direction: DOWN
    timeout: 10000
- assertVisible: "Item 50"
```

```bash
# Run flows
maestro test .maestro/login-flow.yaml

# Run all flows in directory
maestro test .maestro/

# Record flow interactively
maestro record
```

### Unit Testing (React Native)

```typescript
// __tests__/utils/formatCurrency.test.ts
import { formatCurrency } from "../../src/utils/formatCurrency";

describe("formatCurrency", () => {
  it("formats USD with two decimal places", () => {
    expect(formatCurrency(1234.5, "USD")).toBe("$1,234.50");
  });

  it("handles zero", () => {
    expect(formatCurrency(0, "USD")).toBe("$0.00");
  });

  it("handles negative values", () => {
    expect(formatCurrency(-42.1, "EUR")).toBe("-€42.10");
  });
});
```

### Flutter Widget Testing

```dart
// test/widget/counter_page_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/counter_page.dart';

void main() {
  testWidgets('Counter increments when FAB is tapped', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: CounterPage()));

    expect(find.text('Count: 0'), findsOneWidget);

    await tester.tap(find.byIcon(Icons.add));
    await tester.pump();

    expect(find.text('Count: 1'), findsOneWidget);
  });

  testWidgets('Counter shows error for negative input', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: CounterPage()));

    await tester.tap(find.byIcon(Icons.remove));
    await tester.pump();

    expect(find.text('Cannot go below zero'), findsOneWidget);
  });
}
```

### Flutter Integration Testing

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('Full login flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(const Key('email')), 'user@test.com');
    await tester.enterText(find.byKey(const Key('password')), 'secret123');
    await tester.tap(find.byKey(const Key('login-button')));
    await tester.pumpAndSettle(const Duration(seconds: 3));

    expect(find.text('Welcome back'), findsOneWidget);
  });
}
```

### Device Farms and Visual Regression

| Service | Strengths |
|---|---|
| BrowserStack App Automate | Real devices, Detox/Appium support, CI integration |
| AWS Device Farm | Large device matrix, pay-per-minute |
| Firebase Test Lab | Free tier for Android, Robo tests |
| Maestro Cloud | Native Maestro support, fast feedback |

Visual regression with screenshots:

```typescript
// Detox screenshot comparison
it("matches the profile screen snapshot", async () => {
  await element(by.id("profile-tab")).tap();
  const screenshot = await device.takeScreenshot("profile-screen");
  // Compare against baseline using pixelmatch or Percy
});
```

### Anti-patterns

- Only testing on simulators — real devices have different performance characteristics, GPS, camera, and push notification behavior.
- Skipping E2E tests for "just a simple app" — navigation flows, deep links, and permission dialogs break silently without integration coverage.
- Hardcoding wait times in tests — use `waitFor` / `pumpAndSettle` / `assertVisible` with timeouts instead of fixed delays.
- Testing UI details like pixel positions — test behavior and content; visual regression tools handle pixel-level checks.

---

## 8. App Store Deployment

### iOS Signing and Provisioning

| Concept | Purpose |
|---|---|
| Certificate (`.p12`) | Identifies the developer or team |
| Provisioning Profile | Links app ID + certificate + device list (dev) or distribution method |
| App ID | Bundle identifier registered in Apple Developer Portal |
| Entitlements | Capabilities (push, iCloud, Sign in with Apple) |

```bash
# Fastlane match — manage certificates via Git repo
fastlane match init
fastlane match development
fastlane match appstore

# EAS Build handles signing automatically
eas credentials   # manage from CLI
eas build --platform ios --profile production
```

### Android Signing

```properties
# android/keystore.properties (never commit this file)
storeFile=release-key.jks
storePassword=<redacted>
keyAlias=upload
keyPassword=<redacted>
```

```groovy
// android/app/build.gradle
android {
    signingConfigs {
        release {
            def props = new Properties()
            props.load(new FileInputStream(rootProject.file("keystore.properties")))
            storeFile file(props["storeFile"])
            storePassword props["storePassword"]
            keyAlias props["keyAlias"]
            keyPassword props["keyPassword"]
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

### CI/CD Pipelines

#### EAS Build (Expo)

```json
// eas.json
{
  "cli": { "version": ">= 12.0.0" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal",
      "ios": { "simulator": true }
    },
    "production": {
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": { "appleId": "you@example.com", "ascAppId": "1234567890" },
      "android": { "serviceAccountKeyPath": "./google-sa-key.json" }
    }
  }
}
```

```bash
# Build and submit
eas build --platform all --profile production
eas submit --platform all --profile production
```

#### Fastlane

```ruby
# fastlane/Fastfile
default_platform(:ios)

platform :ios do
  desc "Push a new beta build to TestFlight"
  lane :beta do
    increment_build_number(xcodeproj: "MyApp.xcodeproj")
    build_app(workspace: "MyApp.xcworkspace", scheme: "MyApp")
    upload_to_testflight
  end

  desc "Push to App Store"
  lane :release do
    build_app(workspace: "MyApp.xcworkspace", scheme: "MyApp")
    upload_to_app_store(
      skip_metadata: false,
      skip_screenshots: false,
      submit_for_review: true,
      automatic_release: true
    )
  end
end

platform :android do
  desc "Deploy to Play Store internal track"
  lane :internal do
    gradle(task: "clean bundleRelease", project_dir: "android/")
    upload_to_play_store(
      track: "internal",
      aab: "android/app/build/outputs/bundle/release/app-release.aab"
    )
  end
end
```

#### Codemagic (Flutter)

```yaml
# codemagic.yaml
workflows:
  flutter-production:
    name: Production Build
    max_build_duration: 60
    environment:
      flutter: stable
      groups:
        - app_store_credentials
        - google_play
    scripts:
      - name: Install dependencies
        script: flutter pub get
      - name: Run tests
        script: flutter test
      - name: Build iOS
        script: |
          flutter build ipa --release \
            --export-options-plist=/Users/builder/export_options.plist
      - name: Build Android
        script: flutter build appbundle --release
    artifacts:
      - build/**/outputs/**/*.aab
      - build/ios/ipa/*.ipa
    publishing:
      app_store_connect:
        auth: integration
        submit_to_testflight: true
      google_play:
        credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
        track: internal
```

### OTA Updates

Over-the-air updates push JavaScript/Dart changes without going through app store review.

```bash
# Expo EAS Update
eas update --branch production --message "Fix cart total calculation"

# Channel-based rollout
eas update --branch staging --message "New checkout flow"
eas channel:edit production --branch staging   # promote staging to production
```

```bash
# CodePush (React Native, non-Expo)
appcenter codepush release-react -a MyOrg/MyApp-iOS -d Production
appcenter codepush release-react -a MyOrg/MyApp-Android -d Production
```

OTA update constraints:
- Can update JavaScript, assets, and Dart code
- Cannot change native code, permissions, or native module versions
- Apple requires that OTA updates do not change the app's primary purpose
- Always version-gate updates so users on old native builds do not receive incompatible JS bundles

### Store Listing Optimization

| Element | iOS (App Store) | Android (Play Store) |
|---|---|---|
| Title | 30 characters | 30 characters |
| Subtitle / Short description | 30 characters | 80 characters |
| Description | 4000 characters | 4000 characters |
| Keywords | 100 characters (hidden field) | Extracted from description |
| Screenshots | Up to 10 per size class | Up to 8 per form factor |
| Preview video | Up to 30 seconds | Up to 30 seconds |

Tips:
- Front-load keywords in the first line of the description — both stores weight the opening heavily.
- Use all 10 screenshot slots on iOS — the first three appear in search results.
- A/B test store listings on Google Play using Custom Store Listings.
- Localize metadata for your top markets — a translated title and first screenshot dramatically improve conversion.

### Anti-patterns

- Committing signing keys to the repository — use CI secret management (EAS credentials, GitHub Secrets, Fastlane Match with a private repo).
- Skipping TestFlight / internal track testing — store review catches crashes you did not; always soak a build for 24-48 hours on a real-device beta.
- Relying entirely on OTA updates — native dependency changes, permission additions, and SDK bumps require a full binary release.
- Manually incrementing build numbers — automate via `autoIncrement` (EAS), `increment_build_number` (Fastlane), or CI pipeline scripts.
