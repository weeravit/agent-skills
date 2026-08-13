# Integration testing

Integration tests drive the real app on a device or emulator: real navigation, real database, real (or staged) network. They live in `integration_test/`, not `test/`, and run under a different command. Every name is `Given …, When …, Then …`.

## Harness

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('User login flow', () {
    testWidgets('Given the login screen and valid credentials, '
        'When login is submitted, '
        'Then the home screen is reached', (tester) async {
      // Given
      await tester.pumpWidget(MyApp());
      await tester.pumpAndSettle();

      // When
      await tester.enterText(
          find.byKey(const Key('email_field')), 'test@example.com');
      await tester.enterText(
          find.byKey(const Key('password_field')), 'password123');
      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle();

      // Then
      expect(find.byKey(const Key('home_screen')), findsOneWidget);
    });

    testWidgets('Given the login screen and invalid credentials, '
        'When login is submitted, '
        'Then an error message is shown', (tester) async {
      // Given
      await tester.pumpWidget(MyApp());
      await tester.pumpAndSettle();

      // When
      await tester.enterText(
          find.byKey(const Key('email_field')), 'invalid@example.com');
      await tester.enterText(
          find.byKey(const Key('password_field')), 'wrongpassword');
      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle();

      // Then
      expect(find.text('Invalid credentials'), findsOneWidget);
    });
  });
}
```

## Rules

- `pumpAndSettle()` after every action — real network and animations need more than one frame. A bare `pump()` here is the usual cause of a flaky integration test.
- Find by `Key` only. Text finders break under localisation, and this suite runs in Thai too.
- Cover the failure path alongside the happy path; a flow test that only passes proves nothing about error handling.
- Leave no state behind: reset the database or sign out in `tearDown`, or the second run fails on the first run's leftovers.

## Scrolling and long lists

```dart
// When
await tester.dragUntilVisible(
  find.byKey(const Key('user_42')),
  find.byType(Scrollable),
  const Offset(0, -300),
);
await tester.pumpAndSettle();

// Then
expect(find.byKey(const Key('user_42')), findsOneWidget);
```

## Commands

```bash
fvm flutter test integration_test/                              # all flows
fvm flutter test integration_test/login_flow_test.dart          # one flow
fvm flutter test integration_test/ -d <device-id>               # pick device
```

Integration tests do **not** count toward the 100% coverage gate — that gate is on `test/`. They exist to catch what mocked unit tests cannot: wiring, navigation, and real persistence.
