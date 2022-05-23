---
date: "2022-05-23 17:34:00"
title: "Operator constraints in Go"
tags:
- "golang"
- "programming"
summary: I describe four ways to express constraints for operators in generic
  Go code and the trade-offs involved.
---

Let's say you want to implement a sorting function in Go. Or perhaps a data
structure like a
[binary search tree](https://en.wikipedia.org/wiki/Binary_search_tree),
providing ordered access to its elements. Because you want your code to be
re-usable and type safe, you want to use type parameters. So you need a way to
order user-provided types.

There are multiple methods of doing that, with different trade-offs. Let's talk
about four in particular here:

1. `constraints.Ordered`
2. A method constraint
3. Taking a comparison function
4. Comparator types

## `constraints.Ordered`

Go 1.18 has a mechanism to constrain a type parameter to all types which have
the `<` operator defined on them. The types which have this operator are
exactly all types whose underlying type is `string` or one of the predeclared
integer and float types. So we can write a type set expressing that:

```go
type Integer interface {
  ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

type Float interface {
  ~float32 | ~float64
}

type Ordered interface {
  Integer | Float | ~string
}
```

Because that's a fairly common thing to want to do,
[there is already a package which contains these kinds of type sets](https://pkg.go.dev/golang.org/x/exp/constraints#Ordered).

With this, you can write the signature of your sorting function or the
definition of your search tree as:

```go
func Sort[T constraints.Ordered](s []T) {
  // …
}

type SearchTree[T constraints.Ordered] struct {
  // …
}
```

The main advantage of this is that it works directly with predeclared types and
simple types like `time.Duration`. It also is very clear.

The main disadvantage is that it does not allow composite types like `struct`s.
And what if a user wants a different sorting order than the one implied by `<`?
For example if they want to reverse the order or want specialized string
collation. A multimedia library might want to sort “The Expanse” under E. And
some letters sort differently depending on the language setting.

`constraints.Ordered` is simple, but it also is inflexible.

## Method constraints

We can use method constraints to allow more flexibility. This allows a user to
implement whatever sorting order they want as a method on their type.

We can write that constraint like this:

```go
type Lesser[T any] interface {
  // Less returns true if the receiver is less than v.
  Less(v T) bool
}
```

The type parameter is necessary because we have to refer to the receiver type
itself in the `Less` method. This is hopefully clearer when we look at how this
is used:

```go
func Sort[T Lesser[T]](s []T) {
  // …
}

func SearchTree[T Lesser[T]](s []T) {
  // …
}
```

This allows the user of our library to customize the sorting order by defining
a new type with a `Less` method:

```go
type ReverseInt int

func (i ReverseInt) Less(j ReverseInt) bool {
  return j < i // order is reversed
}
```

The disadvantage of this is that it requires some boiler plate on part of your
user. Using a custom sorting order always requires defining a type with a method.

They can't use your code with predeclared types like `int` or `string` but
always have to wrap it into a new type.

Likewise if a type already has a natural comparison method but it is not
called `Less`. For example `time.Time` is naturally sorted by
`time.Time.Before`. For cases like that there needs to be a wrapper to rename
the method.

Whenever one of these wrappings happens your user might have to convert back
and forth when passing data to or from your code.

It also is a little bit more confusing than `constraints.Ordered`, as your user
has to understand the purpose of the extra type parameter on `Lesser`.

## Passing a comparison function

A simple way to get flexibility is to have the user pass us a function used for
comparison directly:

```go
func Sort[T any](s []T, less func(T, T) bool) {
  // …
}

type SearchTree[T any] struct {
  Less func(T, T) bool
  // …
}

func NewSearchTree(less func(T, T) bool) *SearchTree[T] {
  // …
  return &SearchTree[T]{
    Less: less,
    // …
  }
}
```

This essentially abandons the idea of type constraints altogether. Our code
works with *any* type and we directly pass around the custom behavior as
`func`s. Type parameters are only used to ensure that the arguments to those
`func`s are compatible.

The advantage of this is maximum flexibility. Any type which already has a
`Less` method like above can simply be used with this directly by using
[method expressions](https://go.dev/ref/spec#Method_expressions). Regardless of
how the method is actually named:

```go
func main() {
  a := []time.Time{ /* … */ }
  Sort(a, time.Time.Before)
}
```

There is also no boilerplate needed to customize sorting behavior:

```go
func main() {
  a := []int{42,23,1337}
  Sort(a, func(i, j int) bool {
    return j < i // reversed order
  })
}
```

And you can provide helpers for common customizations:

```go
func Reversed[T any](less func(T, T) bool) (greater func(T, T) bool) {
  return func(a, b T) bool { return less(b, a) }
}
```

This approach is arguably also more correct than the one above because it
decouples the type from the comparison used. If I use a `SearchTree` as a set
datatype, there is no real reason why the elements in the set would be specific
to the comparison used. It should be “a set of `string`” not “a set of
`MyCustomlyOrderedString`”. This reflects the fact that with the method
constraint, we have to convert back-and-forth when putting things into the
container or taking it out again.

The main *disadvantage* of this approach is that it means you can not have
useful zero values. Your `SearchTree` type needs the `Less` field to be
populated to work. So its zero value can not be used to represent an empty set.

You cannot even lazily initialize it (which is a common trick to make types
which need initialization have a useful zero value) because *you don't know
what it should be*.

## Comparator types

There is a way to pass a function “statically”. That is, instead of passing
around a `func` value, we can pass it as a type argument. The way to do that is
to attach it as a method to a `struct{}` type:

```go
import "golang.org/x/exp/slices"

type IntComparator struct{}

func (IntComparator) Less(a, b int) bool {
  return a < b
}

func main() {
  a := []int{42,23,1337}
  less := IntComparator{}.Less // has type func(int, int) bool
  slices.SortFunc(a, less)
}
```

Based on this, we can devise a mechanism to allow custom comparisons:

```go
// Comparator is a helper type used to compare two T values.
type Comparator[T any] interface {
  ~struct{}
  Less(a, b T) bool
}

func Sort[C Comparator[T], T any](a []T) {
  var c C
  less := c.Less // has type func(T, T) bool
  // …
}

type SearchTree[C Comparator[T], T any] struct {
  // …
}
```

The `~struct{}` constraints any implementation of `Comparator[T]` to have
underlying type `struct{}`. It is not strictly necessary, but it serves two
purposes here:

1. It makes clear that `Comparator[T]` itself is not supposed to carry any
   state. It only exists to have its method called.
2. It ensures (as much as possible) that the zero value of `C` is safe to use.
   In particular, `Comparator[T]` would be a normal interface type. And it
   would have a `Less` method of the right type, so it would implement itself.
   But a zero `Comparator[T]` is `nil` and would always panic, if its method is
   called.

An implication of this is that it is not possible to have a `Comparator[T]`
which uses an arbitrary `func` value. The `Less` method can not rely on having
access to a `func` to call, for this approach to work.

But you can provide other helpers. This can also be used to combine this approach
with the above ones:

```go
type LessOperator[T constraints.Ordered] struct{}

func (LessOperator[T]) Less(a, b T) bool {
  return a < b
}

type LessMethod[T Lesser[T]] struct{}

func (LessMethod[T]) Less(a, b T) bool {
  return a.Less(b)
}

type Reversed[C Comparator[T], T any] struct{}

func (Reversed[C, T]) Less(a, b T) bool {
  var c C
  return c.Less(b, a)
}
```

The advantage of this approach is that it makes the zero value of
`SearchTree[C, T]` useful. For example, a `SearchTree[LessOperator[int], int]`
can be used directly, without extra initialization.

It also carries over the advantage of decoupling the comparison from the
element type, which we got from accepting comparison functions.

One disadvantage is that the comparator can never be inferred. It always has to
be specified in the instantiation explicitly[^1]. That's similar to how we
always had to pass a `less` function explicitly above.

[^1]: Though as we put the `Comparator[T]` type parameter first, we can infer
      `T` from the `Comparator`.

Another disadvantage is that this *always* requires defining a type for
comparisons. Where with the comparison function we could define customizations
(like reversing the order) inline with a `func` literal, this mechanism always
requires a method.

Lastly, this is arguably too clever for its own good. Understanding the purpose
and idea behind the `Comparator` type is likely to trip up your users when
reading the documentation.

## Summary

We are left with these trade-offs:

|| `constraints.Ordered` | `Lesser[T]` | `func(T,T) bool` | `Comparator[T]` |
|--------------------------|----|--------|----|----|
| Predeclared types        | ✅ | ❌     | ❌ | ❌ |
| Composite types          | ❌ | ✅     | ✅ | ✅ |
| Custom order             | ❌ | ✅     | ✅ | ✅ |
| Reversal helpers         | ✅ | ❌     | ✅ | ✅ |
| Type boilerplate         | ✅ | ❌     | ✅ | ❌ |
| Useful zero value        | ✅ | ✅     | ❌ | ✅ |
| Type inference           | ✅ | ✅     | ✅ | ❌ |
| Coupled Type/Order       | ❌ | ❌     | ✅ | ✅ |
| Clarity                  | ✅ | 🤷[^2] | ✅ | ❌ |

[^2]: It's a *little* bit worse, but probably fine.

One thing standing out in this table is that there is no way to *both* support
predeclared types *and* support user defined types.

It would be great if there was a way to support multiple of these mechanisms
using the same code. That is, it would be great if we could write something
like

```go
// Ordered is a constraint to allow a type to be sorted.
// If a Less method is present, it has precedent.
type Ordered[T any] interface {
  constraints.Ordered | Lesser[T]
}
```

Unfortunately, allowing this
[is harder than one might think](https://blog.merovius.de/posts/2022-05-16-calculating-type-sets/).

Until then, you might want to provide multiple APIs to allow your users more
flexibility. The standard library currently seems to be converging on providing
a `constraints.Ordered` version and a comparison function version.  The latter
gets a `Func` suffix to the name. See
[the experimental `slices` package](https://pkg.go.dev/golang.org/x/exp/slices)
for an example.
