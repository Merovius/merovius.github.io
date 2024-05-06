---
date: "2024-05-06 06:47:46 +0200"
title: "Constraining Go type parameter pointers"
tags:
- "programming"
- "golang"
summary: A quick note on an FAQ about Go generics
---

Sometimes, you need to be able to constrain a type-parameter with a method, but
that method should be defined on the pointer type. For example, say you want to
parse some bytes using JSON and pass the result to a handler. You might try
to write this as

```go
func Handle[M json.Unmarshaler](b []byte, handler func(M) error) error {
	var m M
	if err := m.UnmarshalJSON(b); err != nil {
		return err
	}
	return handler(m)
}
```

However, this code does not work. Say you have a type `Message`, which
implements `json.Unmarshaler` with a pointer receiver (it needs to use a
pointer receiver, as it needs to be able to modify data):

1. If you try to call `Handle[Message]`, you get a compiler error
   ([playground](https://go.dev/play/p/Iu6rvJ4Ff_K)).
   That is because `Message` does not implement `json.Unmarshal`, only
   `*Message` does.
2. If you try to call `Handle[*Message]`, the code panics
   ([playground](https://go.dev/play/p/Y8xn9CU11nd)),
   because `var m M` creates a `*Message` and initializes that to `nil`. You
   then call `UnmarshalJSON` with a `nil` receiver.

Neither of these options work. You really want to rewrite `Handle`, so that it
says that the *pointer* to its type parameter implements `json.Unmarshaler`.
And this is how to do that ([playground](https://go.dev/play/p/YQuGAc4erQh)):

```go
type Unmarshaler[M any] interface {
	*M
	json.Unmarshaler
}

func Handle[M any, PM Unmarshaler[M]](b []byte, handler func(M) error) error {
	var m M
	// note: you need PM(&m), as the compiler can not infer (yet) that you can
	// call the method of PM on a pointer to M.
	if err := PM(&m).UnmarshalJSON(b); err != nil {
		return err
	}
	return handler(m)
}
```

I should mention that I recommend avoiding this pattern, if you can. For the
simple reason that it is a bit of machinery and extra API surface that makes
the code harder to understand. And often, this is completely unnecessary.

For example, a slightly different version of this example (which is more
commonly used to demonstrate the question) might look roughly like this:

```go
type Unmarshaler[M any] interface {
	*M
	json.Unmarshaler
}

func UnmarshalAndLog[M any, PM Unmarshaler[M]](b []byte) (M, error) {
    var m M
    if err := PM(&m).UnmarshalJSON(b); err != nil {
        log.Println("Error unmarshaling:", err)
        return m, err
    }
    log.Println("Got message:", m)
    return m, nil
}

// called as: m, err := UnmarshalAndLog[Message](b)
```

In this case I would prefer to instead write

```go
func UnmarshalAndLog(b []byte, m json.Unmarshaler) error {
    if err := m.UnmarshalJSON(b); err != nil {
        log.Println("Error unmarshaling:", err)
        return err
    }
    log.Println("Got message:", m)
    return nil
}

// called as: m := new(Message); err := UnmarshalAndLog(b, m)
```

While this requires a little bit more code at the call site (especially as you
need a separate variable declaration), it avoids all the extra machinery and
introducing a (usually bad) name for the constraint.
