# Assert: a cross platform drop-in + self-contained C++ assertion library

## Why?

It all started with the need to provide a meaningful message when assertions
fire. There is a well-known hack with standard `assert` to inject a message next
to the expression being tested:

    assert(expression && "message");

But it's limited to string literals. I wanted improve on `assert()` by
providing the following features:

- being able to format a message that would also contain the values for
  different variables around the point of failure
- having different levels of severity
- being able to selectively ignore assertions while debugging
- being able to break into the debugger at the exact point an assertion fires
  (that is in your own source code, instead of somewhere deep inside `assert`
  implementation)
- no memory allocation
- no unused variables warning when assertions are disabled

--------------------------------------------------------------------------------

## What?

The library is designed to be lightweight would you dedice to keep assertions
enabled even in release builds.

### Message Formatting

The library provides `printf` like formatting:

    PEMPEK_ASSERT(expression);
    PEMPEK_ASSERT(expression, message, ...);

E.g:

    PEMPEK_ASSERT(validate(v, min, max), "invalid value: %f, must be between %f and %f", v, min, max);

### Levels Of Severity

This library defines different levels of severity:

- `PEMPEK_ASSERT_WARNING`
- `PEKPEK_ASSERT_DEBUG`
- `PEMPEK_ASSERT_ERROR`
- `PEMPEK_ASSERT_FATAL`

When you use `PEMPEK_ASSERT`, the severity level is determined by the
`PEMPEK_ASSERT_DEFAULT_LEVEL` preprocessor token.

You can also add your own additional severity levels by using:

    PEMPEK_ASSERT_CUSTOM(level, expression);
    PEMPEK_ASSERT_CUSTOM(level, expression, message, ...);

### Default Assertion Handler

The default handler associates a predefined behavior to each of the different
levels:

- `WARNING <= level < DEBUG`: print the assertion message to `stderr`
- `DEBUG <= level < ERROR`: print the assertion message to `stderr` and prompt
   the user for action
- `ERROR <= level < FATAL`: throw an `AssertionException`
- `FATAL < level`: abort the program

When prompting for user action, the default handler prints the following
message on `stderr`:

    `Press (I)gnore / Ignore (F)orever / Ignore (A)ll / (D)ebug / A(b)ort:`

And waits for input on `stdin` (except on iOS and Android platforms):

- Ignore: ignore the current assertion
- Ignore Forever: remember the file and line where the assertion fired and
  ignore it for the remaining execution of the program
- Ignore All: ignore all remaining assertions (all files and lines)
- Debug: break into the debugger if attached, otherwise `abort()` (on Windows,
  the system will prompt the user to attach a debugger)
- Abort: call `abort()` immediately

If you know you're going to launch your program from within a login shell
session on iOS or Android (e.g. through SSH), define the
`PEMPEK_ASSERT_DEFAULT_HANDLER_STDIN` preprocessor token.

### Providing Your Own Handler

If you want to change the default behavior, e.g. by opening a dialog box or
logging assertions to a database, you can provide a custom handler with the
following signature:

    typedef AssertAction::AssertAction (*AssertHandler)(const char* file,
                                                        int line,
                                                        const char* function,
                                                        const char* expression,
                                                        int level,
                                                        const char* message);

Your handler will be called with the proper information filled and needs to
return the action to be performed: 

    PEMPEK_ASSERT_ACTION_NONE,
    PEMPEK_ASSERT_ACTION_ABORT,
    PEMPEK_ASSERT_ACTION_BREAK,
    PEMPEK_ASSERT_ACTION_IGNORE,
    PEMPEK_ASSERT_ACTION_IGNORE_LINE,
    PEMPEK_ASSERT_ACTION_IGNORE_ALL,
    PEMPEK_ASSERT_ACTION_THROW

To install your custom handler, call:

    pempek::assert::implementation::setAssertHandler(customHandler);

### Unused Return Values

The library provides `PEMPEK_ASSERT_USED` that fires an assertion when an unused
return value reaches end of scope:

    PEMPEK_ASSERT_USED(int) foo();

When calling `foo()`,

    {
      foo();

      // ...

      bar();

      // ...

      baz();
    } <- assertion fires, caused by unused `foo()` return value reaching end of scope

Just like `PEMPEK_ASSERT`, `PEMPEK_ASSERT_USED` uses
`PEMPEK_ASSERT_DEFAULT_LEVEL`. If you want more control on the severity, use one
of:

    PEMPEK_ASSERT_USED_WARNING(type)
    PEMPEK_ASSERT_USED_DEBUG(type)
    PEMPEK_ASSERT_USED_ERROR(type)
    PEMPEK_ASSERT_USED_FATAL(type)
    PEMPEK_ASSERT_USED_CUSTOM(level, type)

Arguably, unused return values are better of detected by the compiler. For
instance GCC and Clang allow you to mark function with attributes:

    __attribute__((warn_unused_result)) int foo();

Which will emit the following warning in case the return value is not used:

    warning: ignoring return value of function declared with warn_unused_result attribute [-Wunused-result]

However there is no MSVC++ equivalent. Well there is `__checkReturn` but it
supposedly only have effect when running static code analysis and I failed to
make it work with Visual Studio 2013 Express. Wrapping `PEMPEK_ASSERT_USED`
around a return type is a cheap way to debug a program where you suspect a
function return value is being ignored and shouldn't have been.

## Customizing compilation

In order to use `PEMPEK_ASSERT` in your own project, you just have to bring in
the two `pempek_assert.h` and `pempek_assert.cpp` files. **It's that simple**.

You can customize the library's behavior by defining the following macros:

- `#define PEMPEK_ASSERT_ENABLED 1` or `#define PEMPEK_ASSERT_ENABLED 0`: enable
  or disable assertions, otherwise enabled state is based on `NDEBUG`
  preprocessor token being defined
- `PEMPEK_ASSERT_DEFAULT_LEVEL`: default level to use when using the
  `PEMPEK_ASSERT` macro
- `PEMPEK_ASSERT_DISABLE_STL`: `AssertionException` won't inherit from
  `std::exception`
- `PEMPEK_ASSERT_DISABLE_EXCEPTIONS`: the library won't throw exceptions on
  `ERROR` level but instead rely on a user provided `throwException` function
  that will likely `abort()` the program
- `PEMPEK_ASSERT_MESSAGE_BUFFER_SIZE`

If you want to use a different prefix, provide your own header that includes
`pempek_assert.h` and define the following:

    // custom prefix
    #define ASSERT                PEMPEK_ASSERT
    #define ASSERT_WARNING        PEMPEK_ASSERT_WARNING
    #define ASSERT_DEBUG          PEMPEK_ASSERT_DEBUG
    #define ASSERT_ERROR          PEMPEK_ASSERT_ERROR
    #define ASSERT_FATAL          PEMPEK_ASSERT_FATAL
    #define ASSERT_CUSTOM         PEMPEK_ASSERT_CUSTOM
    #define ASSERT_USED           PEMPEK_ASSERT_USED
    #define ASSERT_USED_WARNING   PEMPEK_ASSERT_USED_WARNING
    #define ASSERT_USED_DEBUG     PEMPEK_ASSERT_USED_DEBUG
    #define ASSERT_USED_ERROR     PEMPEK_ASSERT_USED_ERROR
    #define ASSERT_USED_FATAL     PEMPEK_ASSERT_USED_FATAL
    #define ASSERT_USED_CUSTOM    PEMPEK_ASSERT_USED_CUSTOM


### Compiling for Windows

There is a Visual Studio 2012 solution in the `_win-vs11/` folder.

### Compiling for Linux or Mac

There is a GNU Make 3.81 `MakeFile` in the `_gnu-make/` folder:

    $ make -C _gnu-make/

### Compiling for Mac

See above if you want to compile from command line. Otherwise there is an Xcode
project located in the `_mac-xcode/` folder.

### Compiling for iOS

There is an Xcode project located in the `_ios-xcode/` folder.

If you prefer compiling from command line and deploying to a jailbroken device
through SSH, use:

    $ make -C _gnu-make/ binsubdir=ios CXX="$(xcrun --sdk iphoneos --find clang++) -isysroot $(xcrun --sdk iphoneos --show-sdk-path) -arch armv7 -arch armv7s -arch arm64" CXXFLAGS=-DPEMPEK_ASSERT_DEFAULT_HANDLER_STDIN postbuild="codesign -s 'iPhone Developer'"

### Compiling for Android

You will have to install the Android NDK, and point the `$NDK_ROOT` environment
variable to the NDK path: e.g. `export NDK_ROOT = /opt/android-ndk` (without a
trailing `/` character).

Next, the easy way is to make a standalone Android toolchain with the following
command:

    $ $NDK_ROOT/build/tools/make-standalone-toolchain.sh --system=$(uname -s | tr [A-Z] [a-z])-$(uname -m) --platform=android-3 --toolchain=arm-linux-androideabi-clang3.3 --install-dir=/tmp/android-clang

Now you can compile the self test and self benchmark programs by running:

    $ make -C _gnu-make/ binsubdir=android CXX=/tmp/android-clang/bin/clang++ CXXFLAGS='-march=armv7-a -mfloat-abi=softfp -O2'

--------------------------------------------------------------------------------

## Credits Where It's Due:

This assertion library has been lingering in my pet codebase for years. It has
greatly been inspired by [Andrei Alexandrescu][@incomputable]'s CUJ articles:

- [Assertions][assertions]
- [Enhancing Assertions][enhancing-assertions]

[assertions]: http://www.drdobbs.com/assertions/184403861
[enhancing-assertions]: http://www.drdobbs.com/cpp/enhancing-assertions/184403745
[@incomputable]: https://twitter.com/incomputable

I learnt the `PEMPEK_UNUSED` trick from [Branimir Karadžić][@bkaradzic].

Finally, [`__VA_NARG__` has been invented by Laurent Deniau][__VA_NARG__].

[@bkaradzic]: https://twitter.com/bkaradzic
[__VA_NARG__]: https://groups.google.com/d/msg/comp.std.c/d-6Mj5Lko_s/5R6bMWTEbzQJ


--------------------------------------------------------------------------------

If you find this library useful and decide to use it in your own projects please
drop me a line [@gpakosz].

If you use it in a commercial project, consider using [Gittip].

[@gpakosz]: https://twitter.com/gpakosz
[Gittip]: https://www.gittip.com/gpakosz/
