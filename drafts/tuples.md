# PHP RFC: Sealed Classes

- Date: 2021-06-01 
- Author: Saif Eddin Gmati <azjezz@protonmail.com>, Joe Watkins
- Status: Draft
- First Published at: <https://wiki.php.net/rfc/tuples>

## Introduction


Most PHP projects have started to adopt static analysis tools, mostly
because they protect against bugs, but also because they allow type
hinting parameters and return types more specifically.

One of the most widely syntax's used nowadays with static analysis tools in PHP
is [`object-like array`](https://psalm.dev/docs/annotating_code/type_syntax/array_types/#object-like-arrays), which in fact is a combinations of two types:

1. tuples
2. shapes/structures

This RFC proposes adding support for tuple types natively in PHP, allowing people to also benefit from runtime type checking.

This feature is inspired by [`Hack`](https://docs.hhvm.com/hack/built-in-types/tuples) and [`Rust`](https://doc.rust-lang.org/std/primitive.tuple.html).

## Proposal

Support for tuple return/parameter types, using the following syntax.

```php
function reverse(
  (int, bool) $pair
): (bool, int) {
  list($integer, $boolean) = $pair;

  return ($boolean, $integer);
}
```

A `tuple` is *not* subtype of `array`, as it has some unique properties.

1. Tuples are fixed size, after a tuple is created, elements cannot be added or removed.

```php
function get_pair(): (string, int) {
  return ("foo", 1);
}


$tuple = get_pair();

assert($tuple[0] === "foo");
assert($tuple[1] === 1);

$tuple[2] = "baz"; // Error: Field "2" of tuple(string, int) is not found. 
```

However, tuples are mutable, therefore their values can change:

```php
function get_pair(): (string, int) {
  return ("foo", 1);
}


$tuple = get_pair();

assert($tuple[0] === "foo");
assert($tuple[1] === 1);

$tuple[0] = "baz"; // Ok

assert($tuple[0] === "baz");
```

2. Tuples hold type information about their elements, and the type cannot change:

```php
function get_pair(): (string, int) {
  return ("foo", 1);
}


$tuple = get_pair();

assert($tuple[0] === "foo");
assert($tuple[1] === 1);

$tuple[0] = 1; // Error: Field "0" of tuple(string, int) must contain a string.
```

---

However, tuple elements support both union types, and support intersection types.

```php
interface A {
  public function getPair(): (mixed, mixed);
}

interface B extends A {
  public function getPair(): (string|int, mixed, A);
}

interface C extends B {
  public function getPair(): (string, bool, B&C);
}
```

The type information held internally about tuple elements changes whenever a tuple is passed through type checking, an example:

```php
function foo(): (string|int, bool) {
   // Here, $foo is (string, bool)
   $foo = ("foo", false);
   
   return $foo;
}

// Here, $foo is (string|int, bool), allowing int values for the first field.
$foo = foo();

$foo[0] = 1;
```

A tuple is required to have at least 2 elements.

```php
function bar(): (string) // Error: A Tuple is required to have at least 2 elements.
{
  ...
}
```

This is done because of BC reason, since the following is currently valid PHP code:

```php
$a = ("foo");
```

While this is not:

```php
$a = ("foo", "bar"); // syntax error, unexpected token ","
```

## Backward Incompatible Changes

None

## Proposed PHP Version(s)

PHP 8.2

## RFC Impact

### To Opcache

> TBD

### To Reflection

None

## Proposed Voting Choices

As this is a language change, a 2/3 majority is required.

## Patches and Tests

> TBA

## References

- [Tuples in Rust](https://doc.rust-lang.org/std/primitive.tuple.html)
- [Tuples in HackLang](https://docs.hhvm.com/hack/built-in-types/tuples)
- [Tuples in Python](https://docs.python.org/3/tutorial/datastructures.html#tuples-and-sequences)
