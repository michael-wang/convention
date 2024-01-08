This is coding convention I used for both hobby and job (if possible). These are best practices I distilled from years of experience. The criteria I used:

- Necessary: Only when convention is really needed (or readability, maintainability would drop quickly).
- Simple: Easy to read, implement, and verify. Which means short, concise doc in plain words.
- Opinionated: Best practices that accepted by public may NOT work on my cases. In these cases, I pick conventions that works best for me. Try-catch is one typical example, never get used to it, and unable to produce quality programs (in terms of fewer bugs) than explicit return values for me.
- Middleway: There is no silver bullet to 'fix it all'. Compromise is necessary. I tried to provide options if suitable, and let programmer to judge when/where to use them.

This is a *work in progress* (not even first draft), and I shall revise as I gain more experiences.

# Return result and error (instead of using throw/catch)

```dart
(int, Object?) getInt() {
    // OK
    return (42, null);
    // Failed
    return (0, ErrorObject());
}
```

Why:
1. Throw/catch leading to multiple code path instead of single (some exceptions are handled on caller, some keep going down the call stack and handled somewhere. Hard to know if all exceptions are handled.)
2. Of course we can avoid missing exception by always try-catch ALL, but that usually leads to pool exception handling, for exception should be handled case by case. (Catch all usually leads to return/throw a generic error to caller, and caller usually handle generic exception by 1. log it, 2. tell user that something went wrong and the App can do nothing about it).
3. *Runtime exception* is bad, because *surprise* is bad for UX. How can we be sure that all exceptions are tested if we use throw/catch model?
4. Conclusion: Golang's design is better here, by explicitly return error(s) and force caller to either: deal with it or, explicitly return to it's own caller. Either way the error has to be handled at some point *explicitly*, which makes it easier to cover by testing.

# Value initialization and invalid values

Consider a bool value: `bool foo`, how to know if it's initialized or it's a invalid value?

1. Use `late bool foo`:
    - This delay the using of un-init value to runtime exception, which is NOT good.
    - Still unknown if it's a invalid value.
2. Use `enum`:
```dart
enum Foo {
  uninitialized(0),
  invalid(-1),
  on(1),
  off(2);

  const Foo(this._v);
  final int _v;
}
```
    This fixed both init and invalid issue.
3. Use custom class:
```dart
class Foo {
    bool foo = false;
    bool _inited = false;
    bool get inited => _inited;
    bool _invalid = false;

    init() {
        bool ok = initFoo();
        _inited = true;
        if (!ok) {
            _invalid = true;
        }
    }
}
```

Case 2 is best for simple state like `bool foo`, but in most cases the state is combination of many types:
```dart
class SomeState {

}
```

Why:
1. What is `int? foo; if (foo == null) ...` mean? Does it means `foo` is not initalized? or it means invalid value? We can't tell from that line of code, but trace code to where it's initialized, and validated.
2. Dart already entered [null safety](https://dart.dev/null-safety) age, null should be avoid when possible.
3. Golang already 'reduce' this issue by using [zero value](https://go.dev/tour/basics/12) to represent both uninitialized and invalid values, which is a wise compromise to me.

The first case `bool`

```dart
class Foo {
    int intVal;
    String strVal;
}
```
