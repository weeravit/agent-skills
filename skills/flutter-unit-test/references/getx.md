# GetX testing

Every name is `Given …, When …, Then …`.

## Harness

Every GetX test — controller or widget — needs test mode on and the container reset, or state leaks into the next test.

```dart
setUp(() {
  mockUseCase = MockGetUserUseCase();
  Get.testMode = true;
  controller = UserController(useCase: mockUseCase);
});

tearDown(() => Get.reset());
```

Widget tests additionally register the mock controller so the widget's `Get.find` resolves:

```dart
Get.put<UserController>(mockController);
```

## Controller state

Assert `.value`, never the `Rx` wrapper.

```dart
test('Given the use case returns a user, '
    'When loadUser is called, '
    'Then user is populated and loading is cleared', () async {
  // Given
  when(() => mockUseCase('123')).thenAnswer((_) async => expectedUser);

  // When
  await controller.loadUser('123');

  // Then
  expect(controller.user.value, equals(expectedUser));
  expect(controller.isLoading.value, isFalse);
  verify(() => mockUseCase('123')).called(1);
});

test('Given the use case throws, '
    'When loadUser is called, '
    'Then an error message is surfaced and user stays null', () async {
  // Given
  when(() => mockUseCase('bad')).thenThrow(UserNotFoundException());

  // When
  await controller.loadUser('bad');

  // Then
  expect(controller.user.value, isNull);
  expect(controller.errorMessage.value, isNotEmpty);
  expect(controller.isLoading.value, isFalse);
});
```

## Transient loading state

`isLoading` is only observable *mid-flight* — delay the stub and assert before awaiting.

```dart
test('Given the use case is slow, '
    'When loadUser is in flight, '
    'Then isLoading is true until it completes', () async {
  // Given
  when(() => mockUseCase('123')).thenAnswer((_) async {
    await Future.delayed(const Duration(milliseconds: 100));
    return expectedUser;
  });

  // When
  final pending = controller.loadUser('123');

  // Then
  expect(controller.isLoading.value, isTrue);
  await pending;
  expect(controller.isLoading.value, isFalse);
});
```

## Workers

`debounce`/`ever`/`once` fire asynchronously — advance past the debounce window, then assert the call that survived *and* the one that got dropped.

```dart
test('Given two queries typed inside the debounce window, '
    'When the window elapses, '
    'Then only the last query is searched', () async {
  // Given
  when(() => mockUseCase.search(any())).thenAnswer((_) async => []);
  controller.onInit();

  // When
  controller.searchQuery.value = 'test';
  controller.searchQuery.value = 'testing';
  await Future.delayed(const Duration(milliseconds: 500));

  // Then
  verify(() => mockUseCase.search('testing')).called(1);
  verifyNever(() => mockUseCase.search('test'));
});
```

## Lifecycle

`onInit` and `onClose` are ordinary methods — call them directly.

```dart
test('Given a controller with a seeded id, '
    'When onInit runs, '
    'Then the user is loaded', () async {
  // Given
  when(() => mockUseCase('123')).thenAnswer((_) async => expectedUser);

  // When
  controller.onInit();
  await untilCalled(() => mockUseCase('123'));

  // Then
  expect(controller.user.value, equals(expectedUser));
});

test('Given an initialised controller, '
    'When onClose runs, '
    'Then its subscription is paused', () {
  // Given
  controller.onInit();

  // When
  controller.onClose();

  // Then
  expect(controller.subscription.isPaused, isTrue);
});
```

## Widgets

Stub every observable the widget reads — a missing stub throws inside `build` and the failure points at the wrong line.

```dart
testWidgets('Given the controller holds a loaded user, '
    'When UserScreen is pumped, '
    'Then the name is rendered', (tester) async {
  // Given
  when(() => mockController.user).thenReturn(const User(id: '123', name: 'John Doe').obs);
  when(() => mockController.isLoading).thenReturn(false.obs);
  when(() => mockController.errorMessage).thenReturn(''.obs);

  // When
  await tester.pumpWidget(GetMaterialApp(home: UserScreen()));

  // Then
  expect(find.text('John Doe'), findsOneWidget);
});

testWidgets('Given the controller is loading, '
    'When UserScreen is pumped, '
    'Then a spinner is shown', (tester) async {
  // Given
  when(() => mockController.user).thenReturn(Rx<User?>(null));
  when(() => mockController.isLoading).thenReturn(true.obs);
  when(() => mockController.errorMessage).thenReturn(''.obs);

  // When
  await tester.pumpWidget(GetMaterialApp(home: UserScreen()));

  // Then
  expect(find.byType(CircularProgressIndicator), findsOneWidget);
});

testWidgets('Given a user with a Thai name, '
    'When UserScreen is pumped, '
    'Then the Thai text is rendered', (tester) async {
  // Given
  when(() => mockController.user).thenReturn(const User(id: '123', name: 'สวัสดี ครับ').obs);
  when(() => mockController.isLoading).thenReturn(false.obs);
  when(() => mockController.errorMessage).thenReturn(''.obs);

  // When
  await tester.pumpWidget(GetMaterialApp(home: UserScreen()));

  // Then
  expect(find.text('สวัสดี ครับ'), findsOneWidget);
});
```

## Interactions

Find by `Key`, not by type — type finders break the moment a second button appears.

```dart
testWidgets('Given UserScreen is rendered, '
    'When the load button is tapped, '
    'Then loadUser is called on the controller', (tester) async {
  // Given
  when(() => mockController.loadUser(any())).thenAnswer((_) async {});
  // ...stub observables...

  // When
  await tester.pumpWidget(GetMaterialApp(home: UserScreen()));
  await tester.tap(find.byKey(const Key('load_user_button')));
  await tester.pump();

  // Then
  verify(() => mockController.loadUser(any())).called(1);
});
```

`pump()` for a single frame; `pumpAndSettle()` when an animation or async rebuild is in flight.

## Obx rebuilds

Mutate the observable the widget holds, then pump.

```dart
testWidgets('Given a rendered screen bound to an Rx user, '
    'When the observable changes, '
    'Then Obx rebuilds with the new name', (tester) async {
  // Given
  final user = const User(id: '123', name: 'John').obs;
  when(() => mockController.user).thenReturn(user);
  // ...stub observables...
  await tester.pumpWidget(GetMaterialApp(home: UserScreen()));

  // When
  user.value = const User(id: '123', name: 'Jane');
  await tester.pump();

  // Then
  expect(find.text('Jane'), findsOneWidget);
  expect(find.text('John'), findsNothing);
});
```

## Navigation

Register `getPages`, then assert `Get.currentRoute`.

```dart
testWidgets('Given a user list with routes registered, '
    'When a row is tapped, '
    'Then the detail route becomes current', (tester) async {
  // Given
  Get.testMode = true;
  await tester.pumpWidget(GetMaterialApp(
    home: UserListScreen(),
    getPages: [
      GetPage(name: '/', page: () => UserListScreen()),
      GetPage(name: '/user/:id', page: () => UserDetailScreen()),
    ],
  ));

  // When
  await tester.tap(find.text('John Doe'));
  await tester.pumpAndSettle();

  // Then
  expect(Get.currentRoute, equals('/user/123'));
});
```
