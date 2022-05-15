---
date: "2022-05-12 08:00:00"
tags:
- golang
- programming
title: "Calculating type sets is harder than you think"
summary: I try to explain why the problem of calculating the exact contents of
  a type set in Go is harder than it might seem on the surface.
---

<script defer crossorigin="anonymous" src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.2/MathJax.js?config=TeX-MML-AM_CHTML"></script>

Go 1.18 added the biggest and probably one of the most requested features of
all time to the language: [Generics](https://go.dev/doc/go1.18#generics). If
you want a comprehensive introduction to the topic, there are many out there
and I would personally [recommend this talk I gave at the Frankfurt Gopher
Meetup](https://www.youtube.com/watch?v=QP6v-Q5Foek).

This blog post is not an introduction to generics, though. It is about [this
sentence from the spec](https://go.dev/ref/spec#Operands):

> Implementation restriction: A compiler need not report an error if an
> operand's type is a type parameter with an empty type set.

This restriction might seem strange to you. After all, if a type set is empty,
it seems very helpful to report that to the user, because they obviously made a
mistake - an empty type set can never be used as a constraint, as the function
using it could never be instantiated.

I want to explain the reason that sentence is there and also go into a couple
of related design decisions of the generics design. I'm trying to be expansive
in my explanation, which means that you should not *need* any special knowledge
to understand it. It also means, some of the information might be boring to
you - feel free to skip the corresponding sections.

The reason that sentence is in the Go spec, is that it turns out to be hard to
actually determine if a type set is empty. Hard enough, that the Go team did
not want to require an implementation to solve that. Let's see why.

## P vs. NP

When we talk about whether or not a problem is hard, we often group problems
into two big classes:

1. Problems which can be *solved* reasonably efficiently. This class is called
   P.
2. Problems which can be *decided* reasonably efficiently. This class is called
   NP.

The first obvious follow up question is “what does ‘reasonably efficient’
mean?”. The answer to that is “there is an algorithm with a running time
polynomial in its input size”[^1].

The second obvious follow up question is “what's the difference between
‘solving’ and ‘deciding’?”.

*Solving* a problem means what you think it means: Finding a solution. If I
give you a number and ask you to solve the factorization problem, I'm asking
you to find a (non-trivial) factor of that number.

*Deciding* a problem means that I give you a solution and I'm asking you if the
solution is correct. For the factorization problem, I'd give you two numbers
and ask you to verify that the second is a factor of the first.

These two things are often very different in difficulty. If I ask you to give
me a factor of 297863737, you probably know no better way than to sit down and
try to divide it by a lot of numbers and see if it comes out evenly. But if I
ask you to verify that 9883 is a factor of that number, you just have to do a
bit of long division and it either divides it, or it does not.

It turns out, that every problem which is efficiently *solvable* is also
efficiently *decidable*. You can just calculate the solution and compare it to
the given one, if you want to decide it. So, every problem in P is also in
NP[^2]. But it is
[a famously open question](https://en.wikipedia.org/wiki/P_versus_NP_problem)
whether the opposite is true - that is, we don't really *know*, if there are
problems which are hard to solve, but easy to decide.

The reason this is hard to know in general, is that just because *we have not
found an efficient algorithm to solve a problem*, does not mean there is none.
In practice, though, we usually assume that there are some problems like that.

One fact that helps us talk about hard problems, is that it turns out that
there are some problems which are *as hard as possible* in NP. That means we
were able to prove that if you can solve one of these problems you can use
that to solve *any other problem in NP*. These problems are called “NP-complete”.

That is, to be frank, plain magic and explaining it is far beyond my
capabilities. But it helps us to tell if a given problem is hard, by doing it
the other way around. If solving problem X would enable us to solve one of
these NP-complete problems then solving problem X is obviously itself NP-complete
and therefore *probably very hard*. This is called a “proof by reduction”.

One such problem, which is very often used to prove a problem is hard, is
boolean satisfiability.

## SAT

Imagine I give you a boolean function. The function has a bunch of `bool`
arguments and returns `bool`, by joining its arguments with logical operators
into a single expression. For example:

```go
func F(x, y, z bool) bool {
  return ((!x && y) || z) && (x || !y)
}
```

If I give you values of these arguments, you can efficiently tell me, if
the formula evaluates to `true` or `false`, by substituting them in and
evaluating every operator. For example

```go
f(true, true, false)
  → ((!true && true) || false) && (true || !true)
  → ((false && true) || false) && (true || !true)
  → ((false && true) || false) && (true || false)
  → ((false && true) || false) && true
  →  (false && true) || false
  →   false && true
  →   false
```

This takes at most one step per operator in the expression. So it takes a
*linear* number of steps in the length of the input, which is very efficient.

But if I *only* give you the function and ask you to *find* arguments which
make it return `true` - or even to find out whether such arguments exist - you
probably have to try out all possible input combinations to see if any of them
does. That's easy for three arguments, but for \\(n\\) arguments there are
\\(2^n\\) possible assignments, so it takes *exponential* time in the number of
arguments.

The problem of finding arguments that makes such a function return `true` (or
proving that no such arguments exists) is called "boolean satisfiability" and
it is NP-complete.

It turns out to be extremely important *in what form* the expression is given,
though. Some forms make it pretty easy to solve, while others make it hard.

For example, every expression can be rewritten into what is called a
[“Disjunctive Normal Form” (DNF)](https://en.wikipedia.org/wiki/Disjunctive_normal_form),
so called because it consists of a series of *conjunction* (`&&`) terms, joined
together by *disjunction* (`||`) operators[^3]:

```go
func F_DNF(x, y, z bool) bool {
  return (x && z) || (!y && z)
}
```

(You can verify that this is the same function as above, by
[trying out all 8 input combinations](https://go.dev/play/p/dCtSs3tf91F))

Each term has a subset of the arguments, possibly negated, joined by
`&&`. The terms are then joined together using `||`.

Solving the satisfiability problem for an expression in DNF is easy:

1. Go through the individual terms. `||` is `true` if and only if
   either of its operands is `true`, so for each term:
   * If it contains both an argument and its negation (`x && !x`)
     it can never be `true`, continue to the next term.
   * Otherwise, you can infer valid arguments from the term:
       - If it contains `x`, then we must pass `true` for `x`
       - If it contains `!x`, then we must pass `false` for `x`
       - If it contains neither, then what we pass for `x` does not matter and
         either value works.
   * The term then evaluates to `true` with these arguments, so the entire
     expression does.
2. If none of the terms can be made `true`, the function can never return
   `true` and there is no valid set of arguments.

On the other hand, there is also a [“Conjunctive Normal Form”
(CNF)](https://en.wikipedia.org/wiki/Conjunctive_normal_form). Here, the
expression is a series of *disjunction* (`||`) terms, joined together with
*conjunction* (`&&`) operators:

```go
func F_CNF(x, y, z bool) bool {
  return (!x || z) && (y || z) && (x || !y)
}
```

(Again, you can [verify that this is the same function](https://go.dev/play/p/0xldLVGqu7m))

For this, the idea of our algorithm does not work. To find a solution, you have
to take *all terms* into account simultaneously. You can't just tackle them one
by one. In fact, solving satisfiability on CNF (often abbreviated as “CNFSAT”)
is NP-complete[^4].

[It turns out](https://en.wikipedia.org/wiki/Functional_completeness) that
*every* boolean function can be written as a single expression using only `||`, `&&` and `!`. In particular, every boolean has a DNF and a CNF.

Very often, when we want to prove a problem is hard, we do so by reducing it to
CNFSAT. That's what we will do for the problem of calculating type sets. But
there is one more preamble we need.

## Sets and Satisfiability

There is an important relationship between
[sets](https://en.wikipedia.org/wiki/Set_(mathematics)) and boolean functions.

Say we have a type `T` and a `Universe` which contains all possible values of
`T`. If we have a `func(T) bool`, we can create a set from that, by looking at
all objects for which the function returns `true`:

```go
var Universe Set[T]

func MakeSet(f func(T) bool) Set[T] {
  s := make(Set[T])
  for v := range Universe {
    if f(v) {
      s.Add(v)
    }
  }
  return s
}
```

This set contains exactly all elements for which `f` is `true`. So calculating
`f(v)` is equivalent to checking `s.Contains(f)`. And checking if `s` is empty
is equivalent to checking if `f` can ever return `true`.

We can also go the other way around:

```go
func MakeFunc(s Set[T]) func(T) bool {
  return func(v T) bool {
    return s.Contains(v)
  }
}
```

So in a sense `func(T) bool` and `Set[T]` are “the same thing”. We can
transform a question about one into a question about the other and back.

As with observed above it is important *how* a boolean function is given.
To take that into account we have to also convert boolean operators into set
operations:

```go
// Union(s, t) contains all elements which are in s *or* in t.
func Union(s, t Set[T]) Set[T] {
  return MakeSet(func(v T) bool {
    return s.Contains(v) || t.Contains(v)
  })
}

// Intersect(s, t) contains all elements which are in s *and* in t.
func Intersect(s, t Set[T]) Set[T] {
  return MakeSet(func(v T) bool {
    return s.Contains(v) && t.Contains(v)
  })
}

// Complement(s) contains all elements which are *not* in s.
func Complement(s Set[T]) Set[T] {
  return MakeSet(func(v T) bool {
    return !s.Contains(v)
  })
}
```

And back:

```go
// Or returns a function which returns if f or g is true.
func Or(f, g func(T) bool) func(T) bool {
  return MakeFunc(Union(MakeSet(f), MakeSet(g)))
}

// And returns a function which returns if f and g are true.
func And(f, g func(T) bool) func(T) bool {
  return MakeFunc(Intersect(MakeSet(f), MakeSet(g)))
}

// Not returns a function which returns if f is false
func Not(f func(T) bool) func(T) bool {
  return MakeFunc(Complement(MakeSet(f)))
}
```

The takeaway from all of this is that constructing a set using `Union`,
`Intersect` and `Complement` is really the same as writing a boolean function
using `||`, `&&` and `!`.

And proving that a set constructed in this way is empty is the same as proving
that a corresponding boolean function is never `true`.

With this, let us look at the specific sets we are interested in.

## Basic interfaces as type sets

Interfaces in Go are used to describe sets of types. For example, the interface

```go
type S interface {
    X()
    Y()
    Z()
}
```

is “the set of all types which have a method `X()` and a method `Y()` and a
method `Z()`”.

We can also express set intersection, using [interface embedding](https://go.dev/ref/spec#Embedded_interfaces):

```go
type S interface { X() }
type T interface { Y() }
type U interface {
    S
    T
}
```

This expresses the intersection of `S` and `T` as an interface. Or we can view
the property “has a method `X()`” as a boolean variable and think of this as
the formula `x && y`.

Surprisingly, there is also a limited form of negation. It happens implicitly,
because a type can not have two different methods with the same name. So,
implicitly, if a type has a method `X()` it does *not* have a method `X()
int` for example[^6]:

```go
type X interface { X() }
type NotX interface{ X() int }
```

There is a small snag: A type can have *neither* a method `X()` *nor* have a
method `X() int`. That's why our negation operator is limited. Real boolean
variables are always *either* `true` *or* `false`, whereas our negation also
allows them to be neither. In mathematics we say that this logic language lacks
[the law of the excluded middle](https://en.wikipedia.org/wiki/Law_of_excluded_middle)
(also called “Tertium Non Datur” - “there is no third”). For this section, that does not matter. But we have to worry about it later.

Because we have intersection and negation, we can express interfaces which
could never be satisfied by any type (i.e. which describe an empty type set):

```go
interface{ X; NotX }
```

[The compiler rejects such interfaces](https://go.dev/play/p/r4kpXNynscX). But
how can it do that? Did we not say above that checking if a set is empty is
NP-complete?

The reason this works is that we only have negation and conjunction (`&&`). So,
all the boolean expressions e can build with this language have the form

```go
x && y && !z
```

These expressions are in DNF! We have a term, which contains a couple of
variables - possibly negated - and joins them together using `&&`. We don't
have `||`, so there is only a single term.

Solving satisfiability in DNF is easy, as we said. So with the language as we
have described it so far, we can only express type sets which are easy to check
for emptiness.

## Adding unions

Go 1.18 extends the interface syntax. For our purposes, the important addition
is the `|` operator:

```go
type S interface{
    A | B
}
```

This represents the set of all types which are in the *union* of the type sets
`A` and `B` - that is, it is the set of all types which are in `A` *or* in `B`
(or both).

This means our language of expressible formulas now also includes a
`||`-operator - we have added set unions and set unions are equivalent to
`||` in the language of formulas. What's more, the form of our formula is now a
*conjunctive* normal form - every line is a term of `||` and the lines are
connected by `&&`:

```go
type X interface { X() }
type NotX interface{ X() int }
type Y interface { Y() }
type NotY interface{ Y() int }
type Z interface { Z() }
type NotZ interface{ Z() int }

// (!x || z) && (y || z) && (x || !y)
type S interface {
    NotX | Z
    Y | Z
    X | NotY
}
```

This is not *quite* enough to prove NP-completeness though, because of the snag
above. If we want to prove that it is easy, it does not matter that a type can
have neither method. But if we want to prove that it is hard, we really need an
*exact* equivalence between boolean functions and type sets. So we need to
guarantee that a type has one of our two contradictory methods.

“Luckily”, the `|` operator gives us a way to fix that:

```go
type TertiumNonDatur interface {
    X | NotX
    Y | NotY
    Z | NotZ
}

// (!x || z) && (y || z) && (x || !y)
type S interface {
    TertiumNonDatur

    NotX | Z
    Y | Z
    X | NotY
}
```

Now any type which could possibly implement `S` *must* have either an `X()` or
an `X() int` method, because it must implement `TertiumNonDatur` as well. So,
this extra interface helps us to get the law of the excluded middle into our
language of type sets.

With this, checking if a type set is empty is in general as hard as checking if
an arbitrary boolean formula in CNF has no solution. As described above, that
is NP-complete.

Even worse, we want to define which operations are allowed on a type parameter
by saying that it is allowed if every type in a type set supports it. However,
[that check is also NP-complete](https://github.com/golang/go/issues/45346#issuecomment-822330394).

The easy way to prove that is to observe that if a type set is empty, *every
operator* should be allowed on a type parameter constrained by it. Because any
statement about “every element of the empty set“ is true[^5].

But this would mean that type-checking a generic function would be NP-complete.
If an operator is used, we have to at least check if the type set of its
constraint is empty. Which is NP-complete.

## Why do we care?

A fair question is “why do we even care? Surely these cases are super exotic.
In any real program, checking this is trivial”.

That's true, but there are still reasons to care:

- Go has the goal of having a fast compiler. And importantly, one which is
  guaranteed to be fast *for any program*. If I give you a Go program, you can
  be reasonably sure that it compiles quickly, in a time frame predictable by
  the size of the input.

  If I *can* craft a program which compiles slowly - and may take longer than
  the lifetime of the universe - this is no longer true.

  This is especially important for environments like the Go playground, which
  regularly compiles untrusted code.
- NP complete problems are notoriously hard to debug if they fail.

  If you use Linux, you might have occasionally run into a problem where you
  accidentally tried installing conflicting versions of some package. And if
  so, you might have noticed that your computer first chugged along for a while
  and then gave you an unhelpful error message about the conflict. And maybe
  you had trouble figuring out which packages declared the conflicting
  dependencies.

  This is typical for NP complete problems. As an exact solution is often too
  hard to compute, they rely on heuristics and randomization and it's hard to
  work backwards from a failure.
- We generally don't want the correctness of a Go program to depend on the
  compiler used. That is, a program should not suddenly stop compiling because
  you used a different compiler or the compiler was changed in a new Go
  version.

  But NP-complete problems don't allow us to calculate an exact solution. They
  always need some heuristic (even if it is just “give up after a bit”). If we
  don't want the correctness of a program to be implementation defined, that
  heuristic must become part of the Go language specification. But these
  heuristics are very complex to describe. So we would have to spend a lot of
  room in the spec for something which does not give us a very large benefit.

Note that Go also decided to restrict the version constraints a `go.mod` file
can express, [for exactly the same reasons](https://research.swtch.com/version-sat).
Go has a clear priority, not to require too complicated algorithms in its
compilers and tooling. Not because they are hard to implement, but because the
behavior of complicated algorithms also tends to be hard to understand for
humans.

So, clearly, requiring to solve an NP-complete problem is out of the question.

## The fix

Given that there must not be an NP-complete problem in the language
specification and given that Go 1.18 was released, this problem must have
somehow been solved.

What changed is that the language for describing interfaces was limited from
what I described above. [Specifically](https://go.dev/ref/spec#General_interfaces)

> Implementation restriction: A union (with more than one term) cannot contain
> the predeclared identifier comparable or interfaces that specify methods, or
> embed comparable or interfaces that specify methods.

This disallows the main mechanism we used to map formulas to interfaces above.
We can no longer express our `TertiumNonDatur` type, or the individual `|`
terms of the formula, as the respective terms specify methods. Without
specifying methods, we can't get our “implicit negation” to work either.

The hope is that this change (among a couple of others) is sufficient to ensure
that we can always calculate type sets accurately. Which means I pulled a bit
of a bait-and-switch: I said that calculating type sets is hard. But as they
actually exist, they *might not be*.

The reason I wrote this blog post anyways is to explain the *kind of problems*
that exist in this area. It is easy to say we have solved this problem
[once and for all](https://www.youtube.com/watch?v=0SYpUSjSgFg).

But to be certain, someone should *prove* this - either by writing a proof like
this post, that the problem is still hard, or by writing an algorithm which
solves it efficiently.

There are also still discussions about changing the generics design. As one
example, the limitations we introduced to fix all of this made
[one of the use cases from the design doc](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#interface-types-in-union-elements)
impossible to express. We might want to tweak the design to allow this use
case. In these discussions, we have to look out for this, so we don't
re-introduce the problem. It took us some time to even detect it
[when the union operator was proposed](https://github.com/golang/go/issues/45346)a.

And there are other kinds of “implicit negations” in the Go language. For
example, a `struct` can not have both a field *and* a method with the same
name. Or being one type implies not being another type (so `interface{int}`
implicitly negates `interface{string}`).

All of which is to say that even if the problem might no longer be
NP-complete - I hope that I convinced you it is still more complicated than you
might have thought.

---

I want to thank my beta-readers for helping me improve this article. In
particular,
[arnehormann](https://github.com/arnehormann),
[@johanbrandhorst](https://twitter.com/johanbrandhorst),
[@mvdan_](https://twitter.com/mvdan_),
[@readcodesing](https://mobile.twitter.com/readcodesing),
[@rogpeppe](https://twitter.com/rogpeppe) and
[@zekjur](https://twitter.com/zekjur).

They took a significant chunk of time and their suggestions where invaluable.

[^1]: It should be pointed out, though, that “polynomial” can still be
    extremely inefficient. \\(n^{1000}\\) still grows extremely fast, but is
    polynomial. And for many practical problems, even \\(n^3\\) is
    intolerably slow. But for complicated reasons, there is a qualitatively
    important difference between “polynomial” and “exponential”[^6] run time, so
    you just have to trust me, that the distinction makes sense.

[^2]: These names might seem strange, by the way. P is easy to explain: It
    stands for “polynomial”.

    NP doesn't mean “not polynomial”, though. It means “non-deterministic
    polynomial”. A non-deterministic computer, in this context, is a
    hypothetical machine which can run arbitrarily many computations
    simultaneously. A program which can be *decided* efficiently by any
    computer, can be *solved* efficiently by a non-deterministic one, by just
    trying out all possible solutions at the same time and returning a correct
    one. Thus, being able to decide a problem on a normal computer, means being
    able to solve it on a non-deterministic one.

[^3]: You might complain that it is hard to remember if the “disjunctive normal
    form” is a disjunction of conjunctions, or a conjunction of disjunctions -
    and that no one can remember which of these means `&&` and which means `||`
    anyways.

    You would be correct.

[^4]: You might wonder why we can't just solve CNFSAT by transforming the formula
    into DNF and solving that.

    The answer is that the transformation can make the formula exponentially
    larger. So even though solving the problem on DNF is linear in the size the
    DNF formula, that size is *exponential* in the size of the CNF formula. So
    we still use exponential time in the size of the CNF formula.

[^5]: This is called [the principle of explosion](https://en.wikipedia.org/wiki/Principle_of_explosion)
    or “ex falso quodlibet” (“from falsehoold follows anything”).

    Many people - including many first year math students - have anxieties and
    confusion around this principle and feel that it makes no sense. So I have
    little hope that I can make it palatable to you. But it is extremely
    important for mathematics to “work” and it really *is* the most reasonable
    way to set things up.

    Sorry.

[^6]: Yes, I know that there are complexity classes between polynomial and
    exponential. Allow me the simplification.
