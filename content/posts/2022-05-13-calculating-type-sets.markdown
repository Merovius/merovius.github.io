---
date: "2022-05-12 08:00:00"
tags:
- golang
- programming
title: "Calculating type sets is harder than you think"
summary: I try to explain why the problem of calculating the exact contents of
  a type set in Go is harder than it might seem on the surface.
---

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

*Deciding* a problem means that I give you solution and I'm asking you if the
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
NP[^2]. But it is [a famously open question](https://en.wikipedia.org/wiki/P_versus_NP_problem)
whether the opposite is true - that is, we don't really *know*, if there are
problems which are hard to solve, but easy to decide.

The reason this is hard to know in general, is that just because *we have not
found an efficient algorithm to solve a problem*, does not mean there is none.
In practice, though, we usually assume that there are some problems like that.

One fact that helps us talk about hard problems, is that it turns out that
there are some problems which are *as hard as possible* in NP. That means we
were able to prove, that if you can solve one of these problems, you can use
that to solve *any other problem in NP*. These problems are called “NP-complete”.

That is, to be frank, plain magic and explaining it is far beyond my
capabilities. But it helps us to tell if a given problem is hard, by doing it
the other way around. If solving problem X would enable us to solve one of
these NP-complete problems, then solving problem X is obviously itself NP-complete
and therefore *probably very hard*. This is called a “proof by reduction”.

One such problem, which is very often used to prove a problem is hard, is
boolean satisfiability.

## SAT

Imagine I give you a formula. This formula has a bunch of variables, which can
all be `true` or `false`, joined by logical operators. For example

    ((¬X∧Y)∨Z)∧(X∨¬Y)

Or, if you are more familiar with programming syntax:

    ((!x && y) || z) && (x || !y)

If I give you an assignment to these variable, you can reasonably efficiently
tell me, if the formula evaluates to `true` or `false`, by substituting them in
and evaluating the formula. For example

    X = true
    Y = false
    Z = true
    ((¬X∧Y)∨Z)∧(X∨¬Y) = ((¬true ∧ false) ∨ true)∧(true ∨ ¬false)
                      = ((false ∧ false) ∨ true)∧(true ∨ true)
                      = (false ∨ true) ∧ true
                      = true ∧ true
                      = true

But if I *only* give you the formula and ask you to *find* an assignment which
makes it `true` - or if such an assignment exist - you probably have to try out
all possible assignments to see if any of them does. That's easy for three
variables, but there are 2<sup>n</sup> possible assignments, so it takes
*exponential* time in the number of variables.

This problem is called "boolean satisfiability" and it is NP-complete.

It turns out to be extremely important *in what form* the formula is given,
though. Some forms make it pretty easy to solve, while others make it hard.

For example, every formula has what is called a [“Disjunctive Normal Form”
(DNF)](https://en.wikipedia.org/wiki/Disjunctive_normal_form), so called
because it consists of a series of conjunction (“and”) terms, joined together
by disjunction (“or”) operators[^3]:

    (X ∧ Z) ∨ (¬Y ∧ Z)

Each term has a subset of the variables, possibly negated, joined by ∧. The
terms are then joined together using `∨`.

Solving the satisfiability problem for a formula in DNF is obviously very easy:

1. Go through the individual terms. `∨` is true if and only if either of its
   operands is true, so for each term:
   * If it contains both a variable and its negation (i.e. `X∧¬X`) it can
     never be true, continue to the next term.
   * Otherwise, you can read an assignment from the term - assign `true`, if
     the term contains `X` and assign `false`, if it contains `¬X`. The term
     is then true, so the entire formula is.
2. If none of the terms can be made `true`, the formula can't be made `true`
   and there is no solution.

On the other hand, there is also a [“Conjunctive Normal
Form”](https://en.wikipedia.org/wiki/Conjunctive_normal_form). Here, the
formula is a series of *disjunction* terms, joined together with a
*conjunction*:

    (¬X ∨ Z) ∧ (Y ∨ Z) ∧ (X ∨ ¬Y)

For this, the same idea does not work - you have to take *all terms* into
account simultaneously, to find an assignment and can't just tackle them one by
one. In fact, solving satisfiability on CNF (often abbreviated as “CNFSAT”) is
NP-complete[^4].

Very often, when we want to prove a problem is hard, we do so by reducing it to
CNFSAT. That's what we will do for the problem of calculating type sets. But
there is one more preamble we need.

## Sets and Satisfiability

There is an important relationship between
[sets](https://en.wikipedia.org/wiki/Set_(mathematics)) and boolean functions
(functions which return `true` or `false`).

Say we have some universe of objects `U`. And say we have a boolean function
`f:U→B` with `B:={true,false}`. We can then create a set from that, by looking
at all objects for which `f` is `true`:

    S[f] := { x∈U | f(x) }

We use the index form `S[f]` to indicate, that `S` depends on the function `f`.

Likewise, if we have any set `S⊆U`, we can create a boolean function from it:

    f[S]: U → B
          x ↦ x∈S

That is, the function returns `true` if an element is in `S` and `false`
otherwise.

An interesting aspect of this, is that we can translate between boolean
operators and set operators this way. For example, set union `∪` becomes
equivalent to logical disjunction `∨`:

    S[f] ∪ S[g] = { x∈U | f(x) } ∪ { x∈U | g(x) }
                = { x∈U | f(x) ∨ g(x) }

    f[S](x) ∨ f[T](x) = x∈S ∨ x∈T = x∈(S∪T)

Similarly, `∩` becomes equivalent to `∧`, `⊆` becomes equivalent to `⇒` and `¬`
becomes equivalent to set complement `U\S := {x∈U | x∉S}`.

The boolean formulas we looked at in the previous section are a form of these
boolean functions. They take some boolean arguments and return a boolean, e.g.

      f: B³ → B
    (X,Y,Z) ↦ ((¬X∧Y)∨Z)∧(X∨¬Y)

It turns out that *every* boolean function with boolean parameters can be
written as a logical formula - in particular, every such function has a CNF and
a DNF.

If we want to do that for other sets (apart from `B^n`, that is), we first have
to be able to translate it into a set of boolean variables. As we will see,
this can be easily done in the case we are interested in.

All of this allows us to talk about what we mean by “computing a set is hard”.
With this equivalency, checking if a given object is in a set becomes the
*decision* problem of SAT - `x∈S`, exactly if `f[S](x)` evaluates to `true`.
And checking if a set is empty becomes the *solution* problem of SAT - `S=∅`,
exactly if `f[S]` is not satisfiable.

With this, let us look at the specific sets we are interested in.

## Basic interfaces as type sets

Interfaces in Go have always described sets of types. For example, the interface

```go
type S interface {
    X()
    Y()
    Z()
}
```

is “the set of all types which have a method `X()` and a method `Y()` and a method `Z()`”.

We also had a way to express set intersection, using [interface embedding](https://go.dev/ref/spec#Embedded_interfaces):


```go
type S interface { X() }
type T interface { Y() }
type U interface {
    S
    T
}
```

This expresses `T=S∩T` as an interface. We can also view the predicate “has a
method `X()`” as a boolean variable and think of this as the formula `X∧Y`.

Surprisingly, there is also a simple form of negation. It happens implicitly,
because a type can not have two methods with the same name, but different
types. So, implicitly, if a type has a method `X()`, it does *not* have a
method `X() int`, for example[^5]:

```go
type X interface { X() }
type NotX interface{ X() int }
```

This meant that you could express interfaces which could never be satisfied by
any type (i.e. which described an empty type set):

```go
interface{ X; NotX }
```

[The compiler rejects such interfaces](https://go.dev/play/p/r4kpXNynscX).

So, we pretty much had a language to build boolean formulas, where every
formula had the form

    X∧Y∧¬Z

Checking if a type implements an interface meant to check if it is in the set
described by that interfaces. Which meant checking if the corresponding formula
is satisfied.

And checking if an interface is invalid meant checking if the set it describes
is empty. Which meant checking if the formula is *satisfiable*. This was easy,
though. If you look closely, those formulas are in DNF and as we noted above,
solving SAT on a formula given in DNF is easy.

## Adding unions

Go 1.18 extends the interface syntax. For our purposes, the important addition
is the `|` operator:

```go
type S interface{
    A | B
}
```

This represents the set of all types which are in the *union* of the type sets
`A` and `B` - that is, the set of all types which are in `A` *or* in `B` (or
both).

This means our language of expressible formulas now also includes a
`∨`-operator. What's more, the form of our formula is now a *Conjunctive*
normal form - every line is a term of `∨` and the lines are connected by `∧`:

```go
type X interface { X() }
type NotX interface{ X() int }
type Y interface { Y() }
type NotY interface{ Y() int }
type Z interface { Z() }
type NotZ interface{ Z() int }

// (¬X ∨ Z) ∧ (Y ∨ Z) ∧ (X ∨ ¬Y)
type S interface {
    NotX | Z
    Y | Z
    X | NotY
}
```

There is one detail which becomes important here. There are types which
*neither* implement `X` *nor* do they implement `NotX` - if they don't have an
`X` method at all, for example. If we want to really prove that calculating
these new type sets is NP complete, we need to fix that. We want that a type
set is empty *exactly* if its formula is not satisfiable. Boolean variables are
always either `true` or `false`, though.

“Luckily”, our new operator gives us a way to fix that:

```go
type TertiumNonDatur interface {
    X | NotX
    Y | NotY
    Z | NotZ
}

// (¬X ∨ Z) ∧ (Y ∨ Z) ∧ (X ∨ ¬Y)
type S interface {
    TertiumNonDatur

    NotX | Z
    Y | Z
    X | NotY
}
```

Now, any type we consider necessarily has either an `X()` or an `X() int`
method, so either implements `X`, or implements `NotX` (and never both).

So, checking if these general type sets are empty is NP complete.

Even worse, as the definition of what operations you can do on a variable of
type parameter type, [we can show that *even type-checking a generic function*
is NP-complete](https://github.com/golang/go/issues/45346#issuecomment-822330394)
with this language.

## Why do we care?

A fair question is “why do we even care? Surely these cases are super exotic
and in any real program, checking this is trivial”.

That's true, but there are still reasons to care:

- We still have to at least *talk about* type sets, because they also define
  the set of valid operations on a type parameter constrained by it. We still
  have to be sure that we can do all the necessary computations in the type
  checker.
- Go has the goal of having a fast compiler. And importantly, one which is
  guaranteed to be fast *for any program*. If I give you a Go program, you can
  be reasonably sure that it compiles quickly, in a time frame predictable by
  the size of the input.

  If I *can* craft a program which compiles slowly, this is no longer true.

  This is especially important for environments like the Go playground, which
  regularly compiles untrusted code.
- NP complete problems are notoriously hard to debug, if they fail.

  If you use Linux, you might have occasionally run into a problem where you
  accidentally tried installing conflicting versions of some package. And if
  so, you might have noticed that your computer first chugged along for a while
  and then gave you an unhelpful error message about the conflict. And maybe
  you had trouble figuring out which packages declared the conflicting
  dependencies.

  This is something very typical for NP complete problems. As an exact solution
  is often too hard to compute, they rely on heuristics and randomization and
  it's hard to work backwards from a failure.
- It would require to somehow specify how a compiler should implement this
  check.
  As an exact solution is not possible, there needs to be some heuristic (even
  if it is just “give up after a bit”). Which either needs to be put into the
  language specification, which is very complex, for little benefit. Or
  we leave it up to the implementation, in which case a program might stop
  compiling when changing Go compilers.

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

There are a couple of other restrictions which are arguably at least touching
on these problem, but I won't got into them here.

Ultimately, the fix was to restrict the language so that hard to manage
interfaces could no longer be expressed to begin with.

## What now?

Maybe you noticed a bit of a bait-and-switch I pulled. Originally, I said that
I'm going to explain why the Go compiler does not complain about empty type
sets. I claimed that checking that is NP-complete. But in the last section, I
told you that the NP-completeness problem was fixed. Could the compiler
complain after all?

To the best of my knowledge, the answer is “we don't know yet”. It is entirely
possible that with these restrictions, we would efficiently implement that
check. My intuition is, that it should be possible. But I am not aware that
anyone sat down and either

1. Wrote an algorithm to prove that a type set is empty, thus proving that it
   can be done efficiently. Or
2. Wrote a proof that the problem is still NP-complete.

The issue is, that doing either of these takes time. I did invest that time in
the original discussion, before the limitations where introduced and so far, I
did not get around to doing it with them in place. Maybe someone else has, or
will.

Personally, I'm not sure there is a huge incentive, though. If you write a
constraint with an accidentally empty type set, even the most rudimentary
testing will expose that, as a function using it can not be instantiated and
thus not used. And even *if* you made such a mistake, fixing it would then not
even be a breaking change - as no one could ever use that function anyways.

I also feel that all of this hints at a deeper problem with the concepts of
type sets. There are a couple of more or less arbitrary restrictions in place.
Arguably, too many: The
[design doc contains an example use case](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#interface-types-in-union-elements)
which [no longer works](https://go.dev/play/p/mwdmdgKg6pj).

To me, such restrictions indicate a wrong approach from the start. For example,
perhaps we could have instead made the extension to the interface syntax in a
way that type sets are given in DNF, instead of CNF. Or perhaps we should not
have introduced `|` in the first place.

If you do have an algorithm, or a proof, feel free to let me know. I'd be happy
to provide feedback on either.

[^1]: It should be pointed out, though, that “polynomial” can still be
  extremely inefficient. n<sup>1000</sup> still grows extremely fast, but is
  polynomial. And for many practical problems, even n<sup>3</sup> is
  intolerably slow. But for complicated reasons, there is a qualitatively
  important difference between “polynomial” and “exponential”[^10] run time, so
  just trust me, that the distinction makes sense.

[^2]: These names might seem strange, by the way. P is easy to explain: It
  stands for “polynomial”. NP doesn't mean “not polynomial”, though. It means
  “non-deterministic polynomial”. A non-deterministic computer, in this
  context, is a hypothetical machine which can run arbitrarily many
  computations simultaneously. A program which can be *decided* efficiently by
  any computer, can be *solved* efficiently by a non-deterministic one, by just
  trying out all possible solutions at the same time and returning a correct
  one. Thus, being able to decide a problem on a normal computer, means being
  able to solve it on a non-deterministic one.

[^3]: You might complain that it is hard to remember if the “disjunctive normal
  form” is a disjunction of conjunctions, or a conjunction of disjunctions -
  and that no one can remember which of these means “and” and which means “or”
  anyways. You would be correct.

[^4]: You might wonder why we can't just solve CNFSAT by transforming the formula
  into DNF and solving that. The answer is that the transformation can make the
  formula exponentially larger. So even though solving the problem on DNF is
  linear in the size the DNF formula, that size is *exponential* in the size of
  the CNF formula. So we still use exponential time in the size of the CNF formula.

[^5]: You might notice that this isn't *really* negation. After all, a type
  might have *neither* the method `X()` *nor* the method `X() int`. So, in the
  logic language we are building, we don't have
  [the law of the excluded middle](https://en.wikipedia.org/wiki/Law_of_excluded_middle).
  That's a problem we'll encounter again in a bit, but for this section it does
  not matter.

[^10]: Yes, I know that there are complexity classes between polynomial and
  exponential. Allow me the simplification.
