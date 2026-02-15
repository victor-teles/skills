# Performance Profiling with Integration Tests

Timeline recording is NOT supported on web.

## Tracing Actions

Use `traceAction()` to record performance timelines:

```dart
void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('scroll performance', (tester) async {
    await tester.pumpWidget(
      MyApp(items: List<String>.generate(10000, (i) => 'Item $i')),
    );

    await binding.traceAction(() async {
      await tester.scrollUntilVisible(
        find.byKey(const ValueKey('item_50_text')),
        500.0,
        scrollable: find.byType(Scrollable),
      );
    }, reportKey: 'scrolling_timeline');
  });
}
```

Use unique `reportKey` values when tracing multiple operations in the same test.

## Performance Driver

Create `test_driver/perf_driver.dart` to capture timeline output:

```dart
import 'package:flutter_driver/flutter_driver.dart' as driver;
import 'package:integration_test/integration_test_driver.dart';

Future<void> main() {
  return integrationDriver(
    responseDataCallback: (data) async {
      if (data != null) {
        final timeline = driver.Timeline.fromJson(
          data['scrolling_timeline'] as Map<String, dynamic>,
        );
        final summary = driver.TimelineSummary.summarize(timeline);
        await summary.writeTimelineToFile(
          'scrolling_timeline',
          pretty: true,
          includeSummary: true,
        );
      }
    },
  );
}
```

## Running Performance Tests

Always use `--profile` mode:

```bash
flutter drive \
  --driver=test_driver/perf_driver.dart \
  --target=integration_test/scrolling_test.dart \
  --profile
```

Add `--no-dds` when running on a mobile device or emulator.

## Output

Two files are generated in the `build/` directory:

1. **`scrolling_timeline.timeline_summary.json`** -- summary with key metrics:
   - `average_frame_build_time_millis`
   - `worst_frame_build_time_millis`
   - `missed_frame_build_budget_count`
   - `average_frame_rasterizer_time_millis`
   - `worst_frame_rasterizer_time_millis`
   - `missed_frame_rasterizer_budget_count`

2. **`scrolling_timeline.timeline.json`** -- full timeline, viewable at `chrome://tracing`.

Key metrics: `missed_frame_build_budget_count` and `missed_frame_rasterizer_budget_count` indicate frames exceeding the 16.67ms budget for 60fps.

## watchPerformance

For continuous performance monitoring during a test action:

```dart
await binding.watchPerformance(
  () async {
    // actions to monitor
    await tester.pumpAndSettle();
  },
  reportKey: 'performance',
);
```

## Timeline Streams

Control which VM timeline streams to record:

```dart
await binding.traceAction(
  () async { /* ... */ },
  streams: <String>['all'],  // default; or specific like ['GC']
  reportKey: 'timeline',
);
```

## Extended Driver with Screenshots

For tests that combine screenshots with performance profiling:

```dart
// test_driver/extended_integration_test.dart
import 'package:flutter_driver/flutter_driver.dart';
import 'package:integration_test/integration_test_driver.dart';

Future<void> main() async {
  final driver = await FlutterDriver.connect();
  await integrationDriver(
    driver: driver,
    onScreenshot: (String name, List<int> bytes, [Map<String, Object?>? args]) async {
      // Save bytes to file or validate
      return true;
    },
  );
}
```

## Multiple Timeline Outputs

Write each timeline entry to a separate file:

```dart
responseDataCallback: (data) async {
  if (data != null) {
    for (final entry in data.entries) {
      await writeResponseData(
        entry.value as Map<String, dynamic>,
        testOutputFilename: entry.key,
      );
    }
  }
},
```
