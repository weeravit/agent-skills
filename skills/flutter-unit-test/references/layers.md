# Layer recipes

One recipe per layer. Setup boilerplate (`setUp`, `late` fields, `Mock` classes) follows the template in SKILL.md — only the distinctive parts appear below. Every name is `Given …, When …, Then …`.

## Entity

Constructor, equality, and `props` when Equatable-backed.

```dart
group('User', () {
  test('Given an id and a name, '
      'When a User is constructed, '
      'Then it exposes those values', () {
    // Given
    const id = '123';
    const name = 'John Doe';

    // When
    const user = User(id: id, name: name);

    // Then
    expect(user.id, equals(id));
    expect(user.name, equals(name));
  });

  test('Given two users with identical fields, '
      'When they are compared, '
      'Then they are equal and share props', () {
    // Given
    const a = User(id: '123', name: 'John');
    const b = User(id: '123', name: 'John');

    // When & Then
    expect(a, equals(b));
    expect(a.props, equals(b.props));
  });

  test('Given two users with different ids, '
      'When they are compared, '
      'Then they are not equal', () {
    // Given
    const a = User(id: '123', name: 'John');
    const c = User(id: '456', name: 'Jane');

    // When & Then
    expect(a, isNot(equals(c)));
  });
});
```

## Use case

The template in SKILL.md is a use case test — success and failure. Beyond those two, cover whatever caching or short-circuit the use case owns, asserting it through the call count:

```dart
test('Given the user was already fetched once, '
    'When the use case is called again, '
    'Then the repository is hit only once', () async {
  // Given
  when(() => mockRepository.getUser('123')).thenReturn(expectedUser);

  // When
  await useCase('123');
  final cached = await useCase('123');

  // Then
  expect(cached, equals(expectedUser));
  verify(() => mockRepository.getUser('123')).called(1);
});
```

Either-returning use cases assert the branch, not the unwrapped value:

```dart
when(() => mockUseCase('123')).thenReturn(Right(expectedUser));
when(() => mockUseCase('bad')).thenReturn(Left(UserNotFoundFailure()));
```

## Repository implementation

Mock both data sources. The distinctive assertion is the *orchestration*: what got called, in what order, with what fallback.

```dart
setUp(() {
  mockRemote = MockUserRemoteDataSource();
  mockLocal = MockUserLocalDataSource();
  repository = UserRepositoryImpl(
    remoteDataSource: mockRemote,
    localDataSource: mockLocal,
  );
});

test('Given the remote source returns a model, '
    'When getUser is called, '
    'Then the model is cached locally', () async {
  // Given
  const model = UserModel(id: '123', name: 'John');
  when(() => mockRemote.fetchUser('123')).thenAnswer((_) async => model);
  when(() => mockLocal.cacheUser(model)).thenAnswer((_) async {});

  // When
  await repository.getUser('123');

  // Then
  verify(() => mockLocal.cacheUser(model)).called(1);
});

test('Given the remote source throws, '
    'When getUser is called, '
    'Then the locally cached user is returned', () async {
  // Given
  const cached = UserModel(id: '123', name: 'John');
  when(() => mockRemote.fetchUser('123')).thenThrow(ServerException());
  when(() => mockLocal.getUser('123')).thenAnswer((_) async => cached);

  // When
  final result = await repository.getUser('123');

  // Then
  expect(result, equals(cached.toEntity()));
});
```

## Data source

Mock the HTTP client. One test per status code the source branches on.

```dart
class MockHttpClient extends Mock implements http.Client {}

test('Given the endpoint responds 200 with a user payload, '
    'When fetchUser is called, '
    'Then it returns the parsed UserModel', () async {
  // Given
  when(() => mockHttpClient.get(any()))
      .thenAnswer((_) async => http.Response('{"id":"123","name":"John"}', 200));

  // When
  final result = await dataSource.fetchUser('123');

  // Then
  expect(result.id, equals('123'));
});

test('Given the endpoint responds 404, '
    'When fetchUser is called, '
    'Then it throws ServerException', () async {
  // Given
  when(() => mockHttpClient.get(any()))
      .thenAnswer((_) async => http.Response('Not Found', 404));

  // When & Then
  expect(() => dataSource.fetchUser('bad'), throwsA(isA<ServerException>()));
});
```

Repeat for every non-200 branch (401, 500, timeout) — each is a separate line in the coverage report.

## Model

`fromJson`, `toJson`, `toEntity`, and Thai text.

```dart
test('Given a well-formed JSON map, '
    'When fromJson is called, '
    'Then every field is populated', () {
  // Given
  final json = {'id': '123', 'name': 'John Doe', 'email': 'john@example.com'};

  // When
  final model = UserModel.fromJson(json);

  // Then
  expect(model.id, equals('123'));
  expect(model.email, equals('john@example.com'));
});

test('Given a model, '
    'When it is serialised and parsed back, '
    'Then the result equals the original', () {
  // Given
  const model = UserModel(id: '123', name: 'John Doe', email: 'john@example.com');

  // When
  final json = model.toJson();

  // Then
  expect(UserModel.fromJson(json), equals(model));
});

test('Given JSON containing Thai characters, '
    'When fromJson is called, '
    'Then the characters survive intact', () {
  // Given
  final json = {'id': '123', 'name': 'สวัสดี ครับ', 'email': 't@example.com'};

  // When
  final model = UserModel.fromJson(json);

  // Then
  expect(model.name, equals('สวัสดี ครับ'));
});
```

`toEntity` asserts the mapping, field by field, including any rename or default the model applies.

## API client

Same shape as a data source, but the assertion is on the raw response and the exception translation.

```dart
test('Given the HTTP client throws ClientException, '
    'When get is called, '
    'Then it surfaces as NetworkException', () async {
  // Given
  when(() => mockHttpClient.get(any()))
      .thenThrow(http.ClientException('Network error'));

  // When & Then
  expect(() => apiClient.get('/users/123'), throwsA(isA<NetworkException>()));
});
```
