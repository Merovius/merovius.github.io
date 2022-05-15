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
NP[^2]. But it is [a famously open question](https://en.wikipedia.org/wiki/P_versus_NP_problem)
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

<div style="display: none">
\[
\def\t{{\mathrm{true}}}
\def\f{{\mathrm{false}}}
\def\coloneqq\mathrel{:=}
\]
</div>

Imagine I give you a formula. This formula has a bunch of variables, which can
all be \\(\t\\) or \\(\f\\), joined by logical operators. For example

<div>
\[
((\lnot X \land Y) \lor Z) \land (X \lor \lnot Y)
\]
</div>

If you not familiar with this mathematical notation, in Go this would be
written as

```go
((!x && y) || z) && (x || !y)
```

- \\(\lnot\\) is the logical negation, in Go written as `!`
- \\(\land\\) is the logical “and” operator, in Go written as `&&`. Mathematicians sometimes call this “conjunction”.
- \\(\lor\\) is the logical “or” operator, in Go written as `||`. Mathematicians sometimes call this “disjunction”

If I give you an assignment to these variables, you can efficiently tell me, if
the formula evaluates to \\(\t\\) or \\(\f\\), by substituting them in and
evaluating the formula. For example

<div>
\[
\begin{aligned}
X &\leftarrow \t \\
Y &\leftarrow \t \\
Z &\leftarrow \f \\
((\lnot X \land Y) \lor Z) \land (X \lor\lnot Y) &= ((\lnot\t\land\t)\lor\f)\land(\t\lor\lnot\t) \\
  &= ((\f\land\t)\lor\f)\land(\t\lor\lnot\t) \\
  &= ((\f\land\t)\lor\f)\land(\t\lor\f) \\
  &= ((\f\land\t)\lor\f)\land\t \\
  &= (\f\land\t)\lor\f \\
  &= \f\land\t \\
  &= \f
\end{aligned}
\]
</div>

This takes at most one step per operator in the formula. So it takes a *linear*
number of steps in the length of the input, which is very efficient.

But if I *only* give you the formula and ask you to *find* an assignment which
makes it \\(\t\\) - or even to find out whether such an assignment exists - you
probably have to try out all possible assignments to see if any of them does.
That's easy for three variables, but for \\(n\\) variables there are \\(2^n\\)
possible assignments, so it takes *exponential* time in the number of
variables.

The problem of finding an assignment that makes a formula \\(\t\\) (or proving
that no such assignment exists) is called "boolean satisfiability" and it is
NP-complete.

It turns out to be extremely important *in what form* the formula is given,
though. Some forms make it pretty easy to solve, while others make it hard.

For example, every formula can be rewritten into what is called a [“Disjunctive Normal Form” (DNF)](https://en.wikipedia.org/wiki/Disjunctive_normal_form),
so called because it consists of a series of “and” (“conjunction”,
\\(\land\\)) terms, joined together by “or” (“disjunction”, \\(\lor\\))
operators[^3]:

<div>
\[
(X\land Z)\lor(\lnot Y\land Z)
\]
</div>

(This is the DNF of our example formula above)

Each term has a subset of the variables, possibly negated, joined by
\\(\land\\). The terms are then joined together using \\(\lor\\).

Solving the satisfiability problem for a formula in DNF is easy:

1. Go through the individual terms. \\(\lor\\) is \\(\t\\) if and only if
   either of its operands is \\(\t\\), so for each term:
   * If it contains both a variable and its negation (\\(X\land\lnot X\\))
     it can never be \\(\t\\), continue to the next term.
   * Otherwise, you can infer an assignment from the term: If the formula
     contains \\(X\\), then \\(X\\) must be \\(\t\\). If it contains \\(\lnot
     X\\), then \\(X\\) must be \\(\f\\). If it contains neither, then the
     value of \\(X\\) does not matter and either value works.

     The term is then \\(\t\\), so the entire formula is.
2. If none of the terms can be made \\(\t\\), the formula can't be made \\(\t\\)
   and there is no solution.

On the other hand, there is also a [“Conjunctive Normal Form”
(CNF)](https://en.wikipedia.org/wiki/Conjunctive_normal_form). Here, the
formula is a series of “or” (“disjunction”, \\(\lor\\)) terms, joined together
with “and” (“conjunction”, \\(\land\\)) operators:

<div>
\[
(\lnot X\lor Z)\land(Y\lor Z)\land(X\lor\lnot Y)
\]
</div>

(You can compare this with the DNF above. Both describe the same formula)

For this, the idea of our algorithm does not work. To find a solution, you have
to take *all terms* into account simultaneously. You can't just tackle them one
by one. In fact, solving satisfiability on CNF (often abbreviated as “CNFSAT”)
is NP-complete[^4].

Very often, when we want to prove a problem is hard, we do so by reducing it to
CNFSAT. That's what we will do for the problem of calculating type sets. But
there is one more preamble we need.

## Sets and Satisfiability

There is an important relationship between
[sets](https://en.wikipedia.org/wiki/Set_(mathematics)) and boolean functions
(functions which return \\(\t\\) or \\(\t\\)).

Say we have some universe of objects \\(U\\). And let's introduce \\(B\\) as an
abbreviation of the set \\(\\{\t,\f\\}\\). If we have a boolean function
\\(f\colon U\to B\\) (a function which takes an element of \\(U\\) and maps it
to an element of \\(B\\)) we can create a set from that, by looking at all
objects for which \\(f\\) is \\(\t\\):

<div>
\[
S_f \coloneqq \{ x\in U\colon f(x) \}
\]
</div>

You read this as “\\(S\\) index \\(f\\) is defined as the set of all \\(x\\) in
\\(U\\), for which \\(f\\) of \\(x\\)”. We use the index-notation to indicate
that the set depends on the function \\(f\\).

Likewise, if we have any \\(S\subseteqq U\\) (\\(S\\) is a subset of
\\(U\\)), we can create a boolean function from it:

<div>
\[
\begin{align}
f_S\colon U &\to B \\
x &\mapsto x\in S
\end{align}
\]
</div>

That is, the function returns \\(\t\\) if an element is in \\(S\\) and \\(\f\\)
otherwise[^5].

An interesting aspect of this is that we can translate between boolean
operators and set operators this way:

- The logical “or” \\(\lor\\) becomes set *union*: \\(S\cup T\\) contains
  any element which is in \\(S\\) **or** which is in \\(T\\).
- The logical “and” \\(\land\\) becomes set *intersection*: \\(S\cap T\\)
  contains any element which is in \\(S\\) **and** which is in \\(T\\).
- The logical negation \\(\lnot\\) becomes set *complement*: \\(S^c\\) contains
  any element which is **not** in \\(S\\).

The boolean formulas we looked at in the previous section are a form of these
boolean functions. They take some boolean arguments and return a boolean, e.g.

<div>
\[
\begin{align}
f\colon B\times B\times B &\to B \\
(X,Y,Z)&\mapsto ((\lnot X\land Y)\lor Z)\land(X\lor\lnot Y)
\end{align}
\]
</div>

[It turns out](https://en.wikipedia.org/wiki/Functional_completeness) that
*every* boolean function with boolean parameters can be written as a logical
formula only using \\(\land\\), \\(\lor\\) and \\(\lnot\\) - in particular,
every such function has a CNF and a DNF.

If we want to do that for other sets (apart from `B^n`, that is), we first have
to be able to translate it into a set of boolean variables. As we will see,
this can be easily done in the case we are interested in.

All of this allows us to talk about what we mean by “computing a set is hard”.
With this equivalency, checking if a given object is in a set becomes the
*decision* problem of SAT - deciding if `x∈S` is equivalent to deciding if
`f[S](x)` evaluates to `true`. And checking if a set is empty becomes the
*solution* problem of SAT, as it is equivalent to verifying that `f[S]` is not
satisfiable.

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

is “the set of all types which have a method `X()` and a method `Y()` and a method `Z()`”.

We can also express set intersection, using [interface embedding](https://go.dev/ref/spec#Embedded_interfaces):


```go
type S interface { X() }
type T interface { Y() }
type U interface {
    S
    T
}
```

This expresses `U=S∩T` as an interface. Or we can view the predicate “has a
method `X()`” as a boolean variable and think of this as the formula `X∧Y`.

Surprisingly, there is also a simple form of negation. It happens implicitly,
because a type can not have two different methods with the same name. So,
implicitly, if a type has a method `X()` it does *not* have a method `X()
int` for example[^6]:

```go
type X interface { X() }
type NotX interface{ X() int }
```

This means that we can express interfaces which could never be satisfied by
any type (i.e. which describe an empty type set):

```go
interface{ X; NotX }
```

[The compiler rejects such interfaces](https://go.dev/play/p/r4kpXNynscX).

So we have a language to build boolean formulas, where every formula has the
form

    X∧Y∧¬Z

Checking if a type implements an interface means checking if it is in the set
described by that interfaces. Which means checking if the corresponding formula
is satisfied.

And checking if an interface is invalid means checking if the set it describes
is empty. Which means checking if the formula is *satisfiable*. This is easy,
though, as these formulas are in DNF. They only have a single term and that
term is a conjunction of variables or their negations. And as we noted above,
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
`A` and `B` - that is, it is the set of all types which are in `A` *or* in `B`
(or both).

This means our language of expressible formulas now also includes a
`∨`-operator - we have added set unions (`∪`) and set unions are equivalent to
`∨` in the language of formulas. What's more, the form of our formula is now a
*conjunctive* normal form - every line is a term of `∨` and the lines are
connected by `∧`:

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
these new type sets is NP complete, we need to fix that. We need a type set to
be empty *exactly* if its formula is not satisfiable. Boolean variables are
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

So, checking that an interface has an empty type set is equivalent to checking
that a boolean formula in CNF has no solution. As described above, that is
NP-complete.

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
the original discussion, before the limitations were introduced and so far, I
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
    extremely inefficient. \\(n^{1000}\\) still grows extremely fast, but is
    polynomial. And for many practical problems, even \\(n^3\\) is
    intolerably slow. But for complicated reasons, there is a qualitatively
    important difference between “polynomial” and “exponential”[^7] run time, so
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
    and that no one can remember which of these means “and” and which means “or”
    anyways.

    You would be correct.

[^4]: You might wonder why we can't just solve CNFSAT by transforming the formula
    into DNF and solving that.

    The answer is that the transformation can make the formula exponentially
    larger. So even though solving the problem on DNF is linear in the size the
    DNF formula, that size is *exponential* in the size of the CNF formula. So
    we still use exponential time in the size of the CNF formula.

[^5]: To explain this notation:

    <div>
    \[
    \begin{align}
    f_S\colon U &\to B \\
    x &\mapsto x\in S
    \end{align}
    \]
    </div>

    - \\(f_S\\) is the name of the function. The index
    again means that the function depends on the set \\(S\\)).
    - \\(U\\) is the “input type”
    - \\(B\\) is the “output type”
    - On the left of the \\(\mapsto\\) arrow is the name of the parameter
    - On the right is the “body” of the function

    So in Go this function could be written as

    ```go
    var s Set[U]
    f := func(x U) bool {
      return s.Contains(x)
    }
    ```

[^6]: You might notice that this isn't *really* negation. After all, a type
    might have *neither* the method `X()` *nor* the method `X() int`. So, in the
    logic language we are building, we don't have
    [the law of the excluded middle](https://en.wikipedia.org/wiki/Law_of_excluded_middle).
    That's a problem we'll encounter again in a bit, but for this section it does
    not matter.

[^7]: Yes, I know that there are complexity classes between polynomial and
    exponential. Allow me the simplification.
