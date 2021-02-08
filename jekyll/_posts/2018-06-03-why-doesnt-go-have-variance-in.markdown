---
layout: post
title: "Why doesn't Go have variance in its type system?"
tldr: "I explain what co-, contra- and invariance are and what the implications for Go's type system would be. In particular, why it's impossible to have variance in slices."
tags: ["golang", "programming"]
date: 2018-06-03 23:20:00
---

**tl;dr: I explain what co-, contra- and invariance are and what the
implications for Go's type system would be. In particular, why it's impossible
to have variance in slices.**

A question that comes up relatively often with Go newcomers is "why can't I
pass e.g. an `[]int` to a `func([]interface{})`"? In this post I want to
explore this question and its implications for Go. But the concept of
variance (which this is about) is also useful in other languages.

Variance describes what happens to subtype relationships, when they are
used in composite types. In this context, "A is a subtype of B" means
that a value of type A can always be used, where a value of type B is required.
Go doesn't have explicit subtype relationships - the closest it has is
[assignability](https://golang.org/ref/spec#Assignability) which mostly
determines whether types can be used interchangeably. Probably the most
important case of this is given by interfaces: If a type T (whether it's a
concrete type, or is itself an interface) implements an interface I, then T can be
viewed as a subtype of I. In that sense,
[`*bytes.Buffer`](https://godoc.org/bytes#Buffer) is a subtype of
[io.ReadWriter](https://godoc.org/io#ReadWriter), which is a subtype of
[io.Reader](https://godoc.org/io#Reader). And every type is a subtype of
`interface{}`.

The easiest way to understand what variance means, is to look at function
types. Let's assume, we have a type and a subtype - for example, let's look at
`*bytes.Buffer` as a subtype of `io.Reader`. Say, we have a `func()
*bytes.Buffer`. We could also use this like a `func() io.Reader` - we just
reinterpret the return value as an `io.Reader`. The reverse is not true: We
can't treat a `func() io.Reader` as a `func() *bytes.Buffer`, because not every
`io.Reader` is a `*bytes.Buffer`. So, function return values could *preserve*
the direction of subtyping relationships: If A is a subtype of B, `func() A`
could be a subtype of `func() B`. This is called *covariance*.

```go
func F() io.Reader {
	return new(bytes.Buffer)
}

func G() *bytes.Buffer {
	return new(bytes.Buffer)
}

func Use(f func() io.Reader) {
	useReader(f())
}

func main() {
	Use(F) // Works

	Use(G) // Doesn't work right now; but *could* be made equivalent to…
	Use(func() io.Reader { return G() })
}
```

On the other hand, say we have a `func(*bytes.Buffer)`. Now we can't use that
as a `func(io.Reader)`: You can't call it with an `io.Reader`. But we *can* do
the reverse. If we have a `*bytes.Buffer`, we can call a `func(io.Reader)` with
it. Thus, function arguments *reverse* the subtype relationship: If A is a
subtype of B, then `func(B)` could be a subtype of `func(A)`. This is called
*contravariance*.

```go
func F(r io.Reader) {
	useReader(r)
}

func G(r *bytes.Buffer) {
	useReader(r)
}

func Use(f func(*bytes.Buffer)) {
	b := new(bytes.Buffer)
	f(b)
}

func main() {
	Use(F) // Doesn't work right now; but *could* be made equivalent to…
	Use(func(r *bytes.Buffer) { F(r) })

	Use(G) // Works
}
```

So, `func` is contravariant for arguments and covariant for return values. Of
course, we can combine the two: If A and C are subtypes of B and D
respectively, we can make `func(B) C` a subtype of `func(A) D`, by converting
like this:

```go
// *os.PathError implements error

func F(r io.Reader) *os.PathError {
	// ...
}

func Use(f func(*bytes.Buffer) error) {
	b := new(bytes.Buffer)
	err := f(b)
	useError(err)
}

func main() {
	Use(F) // Could be made to be equivalent to
	Use(func(r *bytes.Buffer) error { return F(r) })
}
```

However, `func(A) C` and `func(B) D` are incompatible. Neither can be a subtype
of the other:

```go
func F(r *bytes.Buffer) *os.PathError {
	// ...
}

func UseF(f func(io.Reader) error) {
	b := strings.NewReader("foobar")
	err := f(b)
	useError(err)
}

func G(r io.Reader) error {
	// ...
}

func UseG(f func(*bytes.Buffer) *os.PathErorr) {
	b := new(bytes.Buffer)
	err := f()
	usePathError(err)
}

func main() {
	UseF(F) // Can't work, because:
	UseF(func(r io.Reader) error {
		return F(r) // type-error: io.Reader is not *bytes.Buffer
	})

	UseG(G) // Can't work, because:
	UseG(func(r *bytes.Buffer) *os.PathError {
		return G(r) // type-error: error is not *os.PathError
	})
}
```

So in this case, there just *is* not relationship between the composite types.
This is called *invariance*.

---

Now, we can get back to our opening question: Why can't you use `[]int` as
`[]interface{}`? This really is the question "Why are slice-types invariant"?.
The questioner assumes that because `int` is a subtype of `interface{}`, we
should also make `[]int` a subtype of `[]interface{}`. However, we can now see
a simple problem with that. Slices support (among other things) two fundamental
operations, that we can roughly translate into function calls:

```go
as := make([]A, 10)
a := as[0] 		// func Get(as []A, i int) A
as[1] = a  		// func Set(as []A, i int, a A)
```

This shows a clear problem: The type A appears *both* as an argument *and*
as a return type. So it appears both covariantly and contravariantly. So while
with functions there is a relatively clear-cut answer to how variance might
work, it just doesn't make a lot of sense for slices. Reading from it would
require covariance but writing to it would require contravariance. In other
words: If you'd make `[]int` a subtype of `[]interface{}` you'd need to explain
how this code would work:

```go
func G() {
	v := []int{1,2,3}
	F(v)
	fmt.Println(v)
}

func F(v []interface{}) {
	// string is a subtype of interface{}, so this should be valid
	v[0] = "Oops"
}
```

Channels give another interesting perspective here. The bidirectional channel
type has the same issue as slices: Receiving requires covariance, whereas
sending requires contravariance. But you can restrict the directionality of a
channel and only allow send- or receive-operations respectively. So while `chan
A` and `chan B` would not be related, we could make `<-chan A` a subtype of
`<-chan B`. And `chan<- B` a subtype of `chan<- A`.

In that sense, [read-only types](https://github.com/golang/go/issues/22876)
have the potential to at least theoretically allow variance for slices. While
`[]int` still wouldn't be a subtype of `[]interface{}`, we could make `ro
[]int` a subtype of `ro []interface{}` (borrowing the syntax from the
proposal).

---

Lastly, I want to emphasize that all of these are just the *theoretical* issues
with adding variance to Go's type system. I consider them harder, but even if
we *could* solve them we would still run into practical issues. The most
pressing of which is that subtypes have different memory representations:

```go
var (
	// super pseudo-code to illustrate
	x *bytes.Buffer // unsafe.Pointer
	y io.ReadWriter // struct{ itable *itab; value unsafe.Pointer }
					// where itable has two entries
	z io.Reader		// struct{ itable *itab; value unsafe.Pointer }
					// where itable has one entry
)
```

So even though you might think that all interfaces have the same memory
representation, they actually don't, because the method tables have a different
assumed layout. So in code like this

```go
func Do(f func() io.Reader) {
	r := f()
	r.Read(buf)
}

func F() io.Reader {
	return new(bytes.Buffer)
}

func G() io.ReadWriter {
	return new(bytes.Buffer)
}

func H() *bytes.Buffer {
	return new(bytes.Buffer)
}

func main() {
	// All of F, G, H should be subtypes of func() io.Reader
	Do(F)
	Do(G)
	Do(H)
}
```

there still needs to be a place where the return value of `H` is wrapped into
an `io.Reader` and there needs to be a place where the itable of the return
value of `G` is transformed into the correct format expected for an
`io.Reader`. This isn't a *huge* problem for `func`: The compiler can
generate the appropriate wrappers at the call site in `main`.
There is a performance overhead, but only code that actually uses this form of
subtyping needs to pay it. However, it becomes significant problem for slices.

For slices, we must either a) convert the `[]int` into an `[]interface{}` when
passing it, meaning an allocation and complete copy. Or b) delay the conversion
between `int` and `interface{}` until the access, which would mean that every
slice access now has to go through an indirect function call - just *in case*
anyone would ever pass us a subtype of what we are expecting. Both options
seem prohibitively expensive for Go's goals.
