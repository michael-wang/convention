This is coding convention I used for both hobby and job (if possible). These are best practices I distilled from years of experience. The criteria I used:

- Necessary: Only when convention is really needed (or readability, maintainability would drop quickly).
- Simple: Easy to read, implement, and verify. Which means short, concise doc in plain words.
- Opinionated: Best practices that accepted by public may NOT work on my cases. In these cases, I pick conventions that works best for me. Try-catch is one typical example, never get used to it, and unable to produce quality programs (in terms of fewer bugs) than explicit return values for me.
- Middleway: There is no silver bullet to 'fix it all'. Compromise is necessary. I tried to provide options if suitable, and let programmer to judge when/where to use them.

This is a *work in progress* (not even first draft), and I shall revise as I gain more experiences.

# Zero value

Use [zero value](https://go.dev/tour/basics/12) when appropriate, to avoid `null`. E.g. [TextFormField](https://api.flutter.dev/flutter/material/TextFormField-class.html)'s validator is a function returns `String?`, which by our convention, should return `String` where empty string means no error.

- `String`: `""` (when empty string is not valid).
- `int`, `double`, `num`: `0` (if only positive number is valid)

## Global Zero Values

|   Type   |  Value    |
| ---      | ---       |
| `bool?`  | `null`    |
| `int`    | `0`       |
| `double` | `0.0`     |
| `num`    | `0`       |
| `String` | `""`      |
| `List`   | `<T>[]`   |
| `Map`    | `<S,T>{}` |

PS: `bool` has no proper zero value, so pointer is required.

One can implement this with extension:
```dart
extension Int on int {
    static const zero = 0;
}
```

or simply global constant:
```dart
const int zeroInt = 0;
```


# Value init and invalid values

Simple cases:
- Init: when declared.
- Invalid:
    - No need: some cases just don't need invalid value.
        ```dart
        bool loading = false;
        for (int i = 0; i < list.length; i++) {...}
        ```
    - Constant:
        - [Global zero value](#global-zero-values): If possible.
        - Custom values: When above is not possible (int where +,0,- values are possible)


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

# Package Awareness

Dart has no *global variable* like other languages, everything in Dart is package scoped. This leads to both useful cases and pitfalls.

## Namespace

In stead of wrapping everything in class (in order to NOT pollute global namespace), use package when appropriate.

E.g. in 'storage.dart' file, this 'global variable/function' are encouraged.
```dart
late SharedPreferences _pref;
bool inited = false;

Future<void> init() async {
    _pref = await SharedPreferences.getInstance();
    inited = true;
}
```

Because caller can use it like this:
```dart
import 'package:myapp/storage.dart' as storage;

foo() async {
    if (!storage.inited) {
        await storage.init();
    }
    final bar = await storage.getInt("bar");
}
```

## Pitfall

A common pitfall of Dart's package scoped namespace is when there are same class/variable/function in different imported package:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
// import 'package:provider/provider.dart';

foo() {
    final p = Provider(); // This is riverpod Provider, NOT provider package's.
}
```

# Error Handling

Avoid throw/catch if possible, return error explicitly.

```dart
(int, Object?) getInt() {
    // OK
    return (42, null);
    // Failed
    return (0, ErrorObject());
}
```

Use `Object?` for both recoverable exception and unrecoverable error. Let caller to judge what can/cannot be recovered.

## Why

1. Throw/catch leading to multiple code path instead of single (some exceptions are handled on caller, some keep going down the call stack and handled somewhere. Hard to know if all exceptions are handled.)
2. Of course we can avoid missing exception by always try-catch ALL, but that usually leads to pool exception handling, for exception should be handled case by case. (Catch all usually leads to return/throw a generic error to caller, and caller usually handle generic exception by 1. log it, 2. tell user that something went wrong and the App can do nothing about it).
3. *Runtime exception* is bad, because *surprise* is bad for UX. How can we be sure that all exceptions are tested if we use throw/catch model?
4. Conclusion: Golang's design is better here, by explicitly return error(s) and force caller to either: deal with it or, explicitly return to it's own caller. Either way the error has to be handled at some point *explicitly*, which makes it easier to cover by testing.
