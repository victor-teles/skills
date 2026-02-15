---
name: flutter-integration-tests
description: Best practices and patterns for writing Flutter integration tests using the integration_test package. Use when creating, reviewing, or improving integration tests in Flutter projects. Triggers on tasks involving integration_test/ directory, IntegrationTestWidgetsFlutterBinding, flutter drive commands, end-to-end testing, screenshot testing, golden file testing, or performance profiling in Flutter apps.
---

# Flutter Integration Tests

## Setup

Add `integration_test` as a dev dependency:

```bash
flutter pub add 'dev:integration_test:{"sdk":"flutter"}'
```

Required `pubspec.yaml` entries:

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
```

## Project Structure

```
my_app/
  lib/
    main.dart
  integration_test/
    app_test.dart
    feature_a_test.dart
    feature_b_test.dart
  test_driver/                # Only for web testing and performance profiling
    integration_test.dart
    perf_driver.dart
```

- Place all integration tests in `integration_test/`.
- Only create `test_driver/` when targeting web browsers or capturing performance timelines.

## Writing Tests

### Binding Initialization

Always initialize the binding before any tests:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_app/main.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('description', (tester) async {
    await tester.pumpWidget(const MyApp());
    await tester.pumpAndSettle();
    // assertions
  });
}
```

### Widget Identification

Add `Key` or `ValueKey` to widgets that tests need to find:

```dart
// In production code
FloatingActionButton(
  key: const ValueKey('increment'),
  onPressed: _incrementCounter,
  child: const Icon(Icons.add),
)

// In test code
final fab = find.byKey(const ValueKey('increment'));
await tester.tap(fab);
```

Prefer `find.byKey()` over `find.text()` for stability across localization changes.

### Core Test Pattern

```dart
testWidgets('tap increment, verify counter updates', (tester) async {
  await tester.pumpWidget(const MyApp());

  // 1. Verify initial state
  expect(find.text('0'), findsOneWidget);

  // 2. Perform action
  await tester.tap(find.byKey(const ValueKey('increment')));

  // 3. Wait for UI to settle
  await tester.pumpAndSettle();

  // 4. Verify result
  expect(find.text('1'), findsOneWidget);
});
```

### Grouping Tests

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('counter', () {
    testWidgets('starts at zero', (tester) async { /* ... */ });
    testWidgets('increments on tap', (tester) async { /* ... */ });
  });

  group('navigation', () {
    testWidgets('opens settings page', (tester) async { /* ... */ });
  });
}
```

## Platform-Specific Patterns

### Platform-Conditional Imports

When tests differ between web and native platforms:

```dart
// example_test.dart
import '_example_test_io.dart'
    if (dart.library.js_interop) '_example_test_web.dart' as tests;

void main() => tests.main();
```

Create `_example_test_io.dart` and `_example_test_web.dart` with platform-specific implementations.

### Screenshots

```dart
void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('capture screenshot', (tester) async {
    await tester.pumpWidget(const MyApp());

    // Required on Android before taking screenshots
    await binding.convertFlutterSurfaceToImage();
    await tester.pumpAndSettle();

    final bytes = await binding.takeScreenshot('home_screen');
    expect(bytes.isNotEmpty, isTrue);
  });
}
```

- Call `convertFlutterSurfaceToImage()` before screenshots on Android/iOS.
- Web screenshots return empty bytes; handle accordingly.

### Golden File Testing

```dart
testWidgets('matches golden', (tester) async {
  await tester.pumpWidget(const MyApp());
  await tester.pumpAndSettle();

  await expectLater(
    find.byType(MaterialApp),
    matchesGoldenFile('goldens/home_screen.png'),
  );
});
```

## Performance Profiling

For detailed guidance on performance profiling with `traceAction()`, timeline recording, and driver setup, see [references/performance-profiling.md](references/performance-profiling.md).

## Running Tests

### Mobile and Desktop

```bash
flutter test integration_test/app_test.dart
```

### Web

1. Create `test_driver/integration_test.dart`:

```dart
import 'package:integration_test/integration_test_driver.dart';

Future<void> main() => integrationDriver();
```

2. Launch ChromeDriver and run:

```bash
chromedriver --port=4444

flutter drive \
  --driver=test_driver/integration_test.dart \
  --target=integration_test/app_test.dart \
  -d chrome
```

Use `-d web-server` for headless mode.

### CI/CD (Linux)

Use `xvfb-run` for headless display:

```bash
xvfb-run flutter test integration_test -d linux
```

## Best Practices

1. **Initialize binding once** -- call `IntegrationTestWidgetsFlutterBinding.ensureInitialized()` at the top of `main()`, before any `testWidgets`.
2. **Verify initial state** -- always assert starting conditions before performing actions.
3. **Use `pumpAndSettle()` after every action** -- wait for animations and async operations to complete.
4. **Prefer `find.byKey()` over `find.text()`** -- keys are stable across locales and text changes.
5. **Add `ValueKey` to testable widgets** -- plan for testability in production code.
6. **Group related tests** -- use `group()` for logical organization.
7. **One concern per test** -- each `testWidgets` should verify a single user flow or behavior.
8. **Use `--profile` for performance tests** -- never measure performance in debug mode.
9. **Use `--no-dds` on mobile devices** -- avoids Dart Development Service conflicts during profiling.
10. **Separate platform implementations** -- use conditional imports (`_io.dart` / `_web.dart`) when behavior diverges.
11. **Call `convertFlutterSurfaceToImage()` on Android** -- required before taking screenshots.
12. **Clean up test apps on device** -- leftover apps can cause subsequent test failures.
