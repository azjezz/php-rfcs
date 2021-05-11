# PHP RFC: Sealed Classes

- Date: 2021-04-24 
- Author: Saif Eddin Gmati <azjezz@protonmail.com>, Joe Watkins
- Status: Draft
- First Published at: <https://wiki.php.net/rfc/sealed_classes>

## Introduction

The purpose of inheritance is code reuse, for when you have a class that
shares common functionality, and you want others to be able to extend it
and make use of this functionality in their own class.

However, when you have a class in your code base that shares some
implementation detail between 2 or more other objects, your only
protection against others making use of this class is to add
`@internal` annotation, which doesn't offer any runtime guarantee that
no one is extending this object.

Internally, PHP has the `Throwable` interface, which defines common
functionality between `Error` and `Exception` and is implemented by
both, however, end users are not allowed to implement `Throwable`.

Currently, PHP has a special case for `Throwable`, and what this RFC is
proposing is to make this kind of functionally possible to end users as
well, so that `Throwable` is not a spacial case anymore.

## Proposal

Support for sealed classes is added through a new modifier `sealed`,
and a new `permits` clause that takes place after `extends`, and
`implements`.

``` php
sealed class Shape permits Circle, Square, Rectangle {}

class Circle extends Shape {} // ok
class Square extends Shape {} // ok
class Rectangle extends Shape {} // ok

class Triangle extends Shape {} // Fatal error: Class Triangle cannot extend sealed class Shape.
```

An interface that is sealed can be implemented directly only by the
classes named in the `permits` clause.

``` php
namespace Psl\Result {
    sealed interface ResultInterface permits Success, Failure { ... }

    class Success implements ResultInterface { ... }
    class Failure implements ResultInterface { ... }

    function wrap(callable $callback): ResultInterface { ... }

    function unwrap(ResultInterface $result): mixed
    {    
        return match(true) {
            $result instanceof Success => $result->value(),
            $result instanceof Failures => throw $result->error(),
        }; // no need for default, it's not possible.
    }
}

namespace App {
    use Psl\Result;

    // ok
    class MySuccess implements Result\Success { ... }

    // ok
    class MyFailure implements Result\Failure { ... }

    // Fatal error: Class App\Maybe cannot implement sealed interface Psl\Result\ResultInterface.
    class Maybe implements Result\ResultInterface {}
}
```

Similarly, a trait that is sealed can only be used by the classes named
in the `permits` clause.

> This is an example taken from the [Symfony Cache component](https://github.com/symfony/symfony/blob/bb1e1e58aea5318e96d1c22cc8a91668ed7baaaa/src/Symfony/Component/Cache)

``` php
namespace Symfony\Component\Cache\Traits {
  use Symfony\Component\Cache\Adapter\FilesystemAdapter;
  use Symfony\Component\Cache\Adapter\FilesystemTagAwareAdapter;
  use Symfony\Component\Cache\Adapter\PhpFilesAdapter;

    sealed trait FilesystemCommonTrait permits FilesystemTrait, PhpFilesAdapter { ... }
    sealed trait FilesystemTrait permits FilesystemAdapter, FilesystemTagAwareAdapter {
        use FilesystemCommonTrait; // ok
        ...
    }
}

namespace Symfony\Component\Cache\Adapter {
    use Symfony\Component\Cache\Traits\FilesystemTrait;
    
    final class FilesystemAdapter {
        use FilesystemTrait; // ok
        ...
    }

    final class FilesystemTagAwareAdapter {
        use FilesystemTrait; // ok
        ...
    }
}

namespace App\Cache {
    use Symfony\Component\Cache\Traits\FilesystemTrait;

    // Error: Class App\Cache\MyFilesystemCache may not use sealed trait (Symfony\Component\Cache\Traits\FilesystemTrait)
    final class MyFilesystemAdapter {
      use FilesystemTrait;
    }

    // Error: Trait App\Cache\MyFilesystemTrait may not use sealed trait (Symfony\Component\Cache\Traits\FilesystemTrait)
    trait MyFilesystemTrait {
      use FilesystemTrait;
    }
}
```

## Interfaces behavior

Interfaces have a different behavior to classes and traits due to backward compability reasons.

An interfaces is allowed to extend a sealed interface that does not permit it, however, a class cannot implement the newly introduced interface unless it extends one of the permitted classes, or implements one of the permitted interfaces.

This is the same as the current behavior for `Throwable` and `DateTimeInterface`.

Example:
```php
sealed interface OperatingSystem permits Linux, MacOS, Windows { ... }

interface UnixLikeOperatingSystem extends OperatingSystem {} // ok

class Linux implements UnixLikeOperatingSystem { ... } // ok, permitted to implement `OperatingSystem`
class MacOS implements UnixLikeOperatingSystem { ... } // ok, permitted to implement `OperatingSystem`
class Windows implements OperatingSystem { ... } // ok, permitted to implement `OperatingSystem`

class WindowsSubsystemLinux extends Windows implements UnixLikeOperatingSystem { ... } // ok, Windows is permitted to implement `OperatingSystem`, and `WindowsSubsystemLinux` is a sub-class of `Windows`

class DummyOS implements UnixLikeOperatingSystem { ... } // Fatal error: Class DummyOS cannot implement sealed interface OperatingSystem implicitly.
```

This behavior doesn't not break the promise sealing provides, as any instance of `OperatingSystem` will be either `Linux`, `MacOS`, `Windows`, or a sub-type of these three.

## Syntax

Some people might be against introducing a new keyword into the
language, which will lead to `sealed` and `permits` not being a
valid class names anymore, therefore, a second vote will take place to
decide which syntax should be used.

The available options are the following:

1. using `sealed`+`permits`:

``` php
sealed class Foo permits Bar, Baz {}

sealed interface Qux permits Quux, Quuz {}

sealed trait Corge permits Grault, Garply {}
```

2. using `permits` only:

``` php
class Foo permits Bar, Baz {}

interface Qux permits Quux, Quuz {}

trait Corge permits Grault, Garply {}
```

3. using pre-reserved `for` keyword:

``` php
class Foo for Bar, Baz {}

interface Qux for Quux, Quuz {}

trait Corge for Grault, Garply {}
```

## Backward Incompatible Changes

`sealed` and `permits` become reserved keywords in PHP 8.1

## Proposed PHP Version(s)

PHP 8.1

## RFC Impact

### To Opcache

> TBD

### To Reflection

The following additions will be made to expose the new flag via
reflection:

- New constant `ReflectionClass::IS_SEALED` to expose the bit flag used for sealed classes
- The return value of `ReflectionClass::getModifiers()` will have this bit set if the class being reflected is sealed 
- `Reflection::getModifierNames()` will include the string `"sealed"` if this bit is set
- A new `ReflectionClass::isSealed()` method will allow directly checking if a class is sealed 
- A new `ReflectionClass::getPermittedClasses()` method will return the list of class names allowed in the `permits` clause.

## Proposed Voting Choices

As this is a language change, a 2/3 majority is required.

## Patches and Tests

> TBA

## References

- [Sealed class and interface in Java](https://docs.oracle.com/en/java/javase/15/language/sealed-classes-and-interfaces.html)
- [Sealed attribute in HackLang](https://docs.hhvm.com/hack/attributes/predefined-attributes#__sealed)
- [Sealed classes in Kotlin](https://kotlinlang.org/docs/sealed-classes.html)
