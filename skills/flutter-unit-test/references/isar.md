# Isar testing

Isar is not mocked — tests run against a real database in a temp directory, one fresh instance per test. Every name is `Given …, When …, Then …`.

## Harness

The unique `name` is what keeps parallel tests from sharing a file. `close(deleteFromDisk: true)` stops temp files piling up between runs.

```dart
import 'package:isar/isar.dart';
import 'package:path_provider/path_provider.dart';

late Isar isar;
var dbCounter = 0;

setUp(() async {
  final dir = await getTemporaryDirectory();
  isar = await Isar.open(
    [UserSchema, MessageSchema],
    directory: dir.path,
    name: 'test_${dbCounter++}',
  );
});

tearDown(() async => isar.close(deleteFromDisk: true));
```

Isar collections are mutable, so instances are built with cascades and cannot be `const`:

```dart
final user = User()
  ..id = '123'
  ..name = 'John Doe';
```

## CRUD

Every write goes inside `writeTxn`; reads do not.

```dart
test('Given an empty database, '
    'When a user is put, '
    'Then it can be read back by id', () async {
  // Given
  final user = User()..id = '123'..name = 'John Doe';

  // When
  await isar.writeTxn(() => isar.users.put(user));

  // Then
  final saved = await isar.users.get('123');
  expect(saved?.name, equals('John Doe'));
});

test('Given a stored user, '
    'When another user is put with the same id, '
    'Then the stored record is overwritten', () async {
  // Given
  await isar.writeTxn(() => isar.users.put(User()..id = '123'..name = 'John Doe'));

  // When
  await isar.writeTxn(() => isar.users.put(User()..id = '123'..name = 'Jane Doe'));

  // Then
  expect((await isar.users.get('123'))?.name, equals('Jane Doe'));
});

test('Given a stored user, '
    'When it is deleted, '
    'Then reading it returns null', () async {
  // Given
  await isar.writeTxn(() => isar.users.put(User()..id = '123'..name = 'John Doe'));

  // When
  await isar.writeTxn(() => isar.users.delete('123'));

  // Then
  expect(await isar.users.get('123'), isNull);
});
```

## Queries

Seed with `putAll`, then assert the filter, the sort, and the empty case.

```dart
test('Given three users of whom two share a surname, '
    'When filtering by that fragment, '
    'Then only those two are returned', () async {
  // Given
  await isar.writeTxn(() => isar.users.putAll([
        User()..id = '1'..name = 'John Doe',
        User()..id = '2'..name = 'Jane Doe',
        User()..id = '3'..name = 'Bob Smith',
      ]));

  // When
  final found = await isar.users.filter().nameContains('Doe').findAll();

  // Then
  expect(found.length, equals(2));
});

test('Given users stored out of order, '
    'When sorting by name, '
    'Then they come back alphabetically', () async {
  // Given
  await isar.writeTxn(() => isar.users.putAll([
        User()..id = '1'..name = 'Charlie',
        User()..id = '2'..name = 'Alice',
        User()..id = '3'..name = 'Bob',
      ]));

  // When
  final sorted = await isar.users.where().sortByName().findAll();

  // Then
  expect(sorted.map((u) => u.name), equals(['Alice', 'Bob', 'Charlie']));
});

test('Given an empty database, '
    'When filtering for any name, '
    'Then the result is empty', () async {
  // Given — nothing written

  // When
  final found = await isar.users.filter().nameContains('Nobody').findAll();

  // Then
  expect(found, isEmpty);
});
```

`.where()` uses an index; `.filter()` scans. Match whichever the production query uses so the test exercises the same path.

## Thai text

Isar stores UTF-8, but `nameContains` is case- and form-sensitive — assert Thai round-trips and matches.

```dart
test('Given a user with a Thai name, '
    'When filtering by a Thai fragment, '
    'Then the record is found intact', () async {
  // Given
  await isar.writeTxn(() => isar.users.put(User()..id = '1'..name = 'สวัสดี ครับ'));

  // When
  final found = await isar.users.filter().nameContains('สวัสดี').findAll();

  // Then
  expect(found.single.name, equals('สวัสดี ครับ'));
});
```

## Testing a local data source over Isar

The data source takes the real `Isar` instance from the harness — no mock. Assert through the database, not through a stub.

```dart
setUp(() async {
  // ...open isar as above...
  dataSource = UserLocalDataSourceImpl(isar);
});

test('Given a model to cache, '
    'When cacheUser is called, '
    'Then the record lands in the database', () async {
  // Given
  const model = UserModel(id: '123', name: 'John');

  // When
  await dataSource.cacheUser(model);

  // Then
  expect((await isar.users.get('123'))?.name, equals('John'));
});
```
