# flutter_firebase_crashlytics_app

A new Flutter project.

## Getting Started

This project is a starting point for a Flutter application.

A few resources to get you started if this is your first Flutter project:

- [Lab: Write your first Flutter app](https://docs.flutter.dev/get-started/codelab)
- [Cookbook: Useful Flutter samples](https://docs.flutter.dev/cookbook)

For help getting started with Flutter development, view the
[online documentation](https://docs.flutter.dev/), which offers tutorials,
samples, guidance on mobile development, and a full API reference.

## Setting up Sentry

Follow these steps to add Sentry error monitoring to a Flutter app.

### 1. Create a Sentry project
1. Sign up or log in at [sentry.io](https://sentry.io).
2. Create a new project and choose **Flutter** as the platform.
3. Copy the **DSN** shown on the project page (`https://<key>@<org>.ingest.sentry.io/<project>`).

### 2. Add the dependency
```bash
flutter pub add sentry_flutter
```

### 3. Initialize Sentry in `lib/main.dart`
```dart
import 'package:flutter/widgets.dart';
import 'package:sentry_flutter/sentry_flutter.dart';

Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = 'YOUR_SENTRY_DSN';
      options.tracesSampleRate = 1.0; // lower this in production
    },
    appRunner: () => runApp(const MyApp()),
  );
}
```
> Tip: avoid hardcoding the DSN. Pass it with
> `flutter run --dart-define=SENTRY_DSN=your_dsn` and read it using
> `String.fromEnvironment('SENTRY_DSN')`.

### 4. Test the integration
Trigger a test error, for example from a button:
```dart
ElevatedButton(
  onPressed: () => throw Exception('Sentry test error'),
  child: const Text('Break the app'),
),
```
Or capture manually:
```dart
try {
  throw StateError('Manual test');
} catch (e, stackTrace) {
  await Sentry.captureException(e, stackTrace: stackTrace);
}
```
Open your project's **Issues** page in Sentry; the error should appear within a minute.

### 5. (Optional) Upload debug symbols
Readable stack traces for release builds need symbols. Add to `pubspec.yaml`:
```yaml
dev_dependencies:
  sentry_dart_plugin: ^2.0.0

sentry:
  upload_debug_symbols: true
  upload_source_maps: true
  project: your-project-slug
  org: your-org-slug
```
Then build with `flutter build apk --obfuscate --split-debug-info=build/symbols`
and run:
```bash
export SENTRY_AUTH_TOKEN=your_token
dart run sentry_dart_plugin
```

More info: [Sentry Flutter docs](https://docs.sentry.io/platforms/dart/guides/flutter/)


## Setting up Firebase Crashlytics

Follow these steps to add Firebase Crashlytics to a Flutter app.

### 1. Create a Firebase project
1. Go to the [Firebase Console](https://console.firebase.google.com) and click **Add project**.
2. Follow the wizard (Google Analytics is recommended, since Crashlytics uses it for breadcrumbs).

### 2. Install the tools
```bash
npm install -g firebase-tools
firebase login
dart pub global activate flutterfire_cli
```

### 3. Add the dependencies
```bash
flutter pub add firebase_core firebase_crashlytics firebase_analytics
```

### 4. Connect the app to Firebase
From the project root run:
```bash
flutterfire configure
```
Select your Firebase project and the platforms (Android / iOS). This generates
`lib/firebase_options.dart` and the platform config files.

> Android: the FlutterFire CLI also applies the `google-services` and
> `firebase-crashlytics` Gradle plugins. If not, add them manually to
> `android/settings.gradle(.kts)` and `android/app/build.gradle(.kts)`.

### 5. Initialize Firebase and Crashlytics in `lib/main.dart`
```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'firebase_options.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);

  // Catch Flutter framework errors
  FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

  // Catch async errors outside the Flutter framework
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };

  runApp(const MyApp());
}
```

### 6. Test the integration
Add a button that forces a crash:
```dart
TextButton(
  onPressed: () => FirebaseCrashlytics.instance.crash(),
  child: const Text('Crash'),
),
```
Or log a non-fatal error:
```dart
FirebaseCrashlytics.instance.recordError(
  Exception('Test non-fatal error'),
  StackTrace.current,
);
```

1. Run the app in **release or profile mode** (`flutter run --release`); do not attach a debugger.
2. Trigger the crash, then **re-open the app** so the report is sent.
3. Open **Firebase Console → Crashlytics**. The first report may take a few minutes.

### 7. (Optional) Useful extras
```dart
FirebaseCrashlytics.instance.setUserIdentifier('user_123');
FirebaseCrashlytics.instance.setCustomKey('screen', 'home');
FirebaseCrashlytics.instance.log('User tapped checkout');
// Disable collection in debug builds:
await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(!kDebugMode);
```

More info: [Firebase Crashlytics docs](https://firebase.google.com/docs/crashlytics/get-started?platform=flutter)
