---
sidebar_position: 1
---

# Basic concepts

Since the `win32` package primarily focuses on providing a lightweight wrapper
for the underlying Windows API primitives, you can use the same API calls as
described in Microsoft documentation to create and manipulate objects (e.g.
`CoCreateInstance` and `IUnknown->QueryInterface`). However, since this
introduces a certain amount of boilerplate and non-idiomatic Dart code, the
library also provides some helper functions that reduce the labor compared to a
pure C-style calling convention.

## Initializing the COM library

Before you call any COM functions, first initialize the COM library by calling
the `CoInitializeEx` function. Details of the threading models are outside the
scope of this document, but typically you should write something like:

```dart
final hr = CoInitializeEx(
    nullptr, COINIT_APARTMENTTHREADED | COINIT_DISABLE_OLE1DDE);
if (FAILED(hr)) throw WindowsException(hr);
```

## Creating a COM object

You can create COM objects using the [C library]:

```dart
hr = CoCreateInstance(clsid, nullptr, CLSCTX_INPROC_SERVER, iid, ppv);
```

However, rather than manually allocate GUID structs for the `clsid` and `iid`
values, checking the `hr` result code and deal with casting the `ppv` return
object, it is easier to use the `createFromID` static helper function:

```dart
final fileDialog2 = IFileDialog2(
    COMObject.createFromID(CLSID_FileOpenDialog, IID_IFileDialog2));
```

`createFromID` returns a `Pointer<COMObject>` containing the requested object,
which can then be cast into the appropriate interface as shown above.

## Asking a COM object for an interface

COM allows objects to implement multiple interfaces, but it does not let you
merely cast an object to a different interface. Instead, returned pointers are
to a specific interface. However, every COM interface in the `win32` package
derives from `IUnknown`, so as in other language implementations of COM, you
may call `queryInterface` on any object to retrieve a pointer to a different
supported interface.

More information on COM interfaces may be found in the
[Microsoft documentation].

COM interfaces supply a method that wraps `queryInterface`. If you
have an existing COM object, you can call it as follows:

```dart
  final modalWindow = IModalWindow(fileDialog2.toInterface(IID_IModalWindow));
```

or, you can use the `from` constructor that wraps the `toInterface` for you:

```dart
  final modalWindow = IModalWindow.from(fileDialog2);
```

Where `createFromID` creates a new COM object, `toInterface` casts an existing
COM object to a new interface.

Attempting to cast a COM object to an interface it does not support will fail,
of course. A `WindowsException` will be thrown with an hr of `E_NOINTERFACE`.

## Calling a method on a COM object

No special considerations are needed here; however, it is wise to assign the
return value to a variable and test it for success or failure. You can use the
`SUCCEEDED()` or `FAILED()` top-level functions to do this, for example:

```dart
final hr = fileOpenDialog.show(NULL);
if (SUCCEEDED(hr)) {
  // Do something with the returned dialog box values
}
```

Failures are reported as `HRESULT` values (e.g. `E_ACCESSDENIED`). Sometimes a
Win32 error code is converted to an `HRESULT`, as in the case where a user
cancels a common dialog box:

```dart
final hr = fileOpenDialog.show(NULL);
if (FAILED(hr) && hr == HRESULT_FROM_WIN32(ERROR_CANCELLED)) {
  // User clicked cancel
}
```

## Releasing COM objects

In general, releasing COM objects isn't something you need to worry about,
because when the object becomes inaccessible to the program, the [Finalizer]
automatically releases it for you.

::::caution

If you are manually managing the lifetime of an object, such as by calling the
`.detach()` method, then it is important to ensure that you release it properly
by calling the `.release()` method. Additionally, you should free up the memory
that was allocated for the object by calling the `free()` helper function as
follows:

```dart
fileOpenDialog.release(); // Release the COM object
free(fileOpenDialog.ptr); // Release the allocated memory for the object
```

This is necessary to prevent memory leaks and ensure that the memory used by
the object is properly released.

:::tip

It is important to include this code as part of a `try` / `finally` block to
ensure that the object is released properly, even if an exception is thrown
during the execution of your code.

:::

::::

[C library]: https://docs.microsoft.com/en-us/windows/win32/learnwin32/creating-an-object-in-com
[Finalizer]: https://api.dart.dev/stable/dart-core/Finalizer-class.html
[Microsoft documentation]: https://docs.microsoft.com/en-us/windows/win32/learnwin32/asking-an-object-for-an-interface
