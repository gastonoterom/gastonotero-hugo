---
id: never-throw-exceptions
title: "Never Throw Exceptions: Make the Implicit Explicit Instead"
description: Type-safe error handling with monads
author: Gaston Otero
date: "2026-01-27"
---

## Table of Contents

- [Most languages have incomplete type declarations](#most-languages-have-incomplete-type-declarations)
- [Error throwing is unpredictable](#error-throwing-is-unpredictable)
- [Typing errors with Result<E, T>](#typing-errors-with-resulte-t)
- [Addressing composition](#addressing-composition)
- [Introducing map and flatMap](#introducing-map-and-flatmap)
- [Representing absence of values with the Option wrapper](#representing-absence-of-values-with-the-option-wrapper)
- [Abstracting the wrapper type](#abstracting-the-wrapper-type)
- [Real-world implementations](#real-world-implementations)
- [On the title](#on-the-title)

## Most languages have incomplete type declarations

Consider this innocent-looking function type declaration:

```typescript
type WarmGreeting = (name: string) => string;
```

Now, let's read the implementation:

```typescript
const warmGreeting: WarmGreeting = (name) => {
  if (name.startsWith("Dr.")) {
    throw new Error("Formal titles require different greeting format");
  }

  return "Hello " + name + " ☺️!";
};
```

How in the world are we supposed to know that from the type signature alone!?

Do you really expect me to predict these things? Am I supposed to know every possible scenario for every different domain?

This type of code really frustrates me. I've found myself in many situations where calling seemingly innocent functions ends really badly.

Let's analyze a more common situation:

```typescript
type Division = (a: number) => (b: number) => number;

const division: Division = (a) => (b) => {
  if (b === 0) {
    throw new Error("Division by zero is not allowed");
  }

  return a / b;
};
```

Yes, yes, you probably already knew this limitation, but extrapolate these scenarios to domains you are not very familiar with and things can go wrong very quickly.

## Error throwing is unpredictable

When you throw an exception, your code suddenly jumps to an unknown location somewhere up the call stack, bypassing everything in between. You can't see where you'll end up just by reading the function's type signature. Unpredictability makes code harder to reason about, harder to test, and harder to maintain.

We'll try to find a way to make errors explicit in the type system.

## Typing errors with `Result<E, T>`

Let's make failure explicit in the type system using the Result type:

```typescript
type Result<E, T> = { value: E; __tag: "error" } | { value: T; __tag: "ok" };

const ok = <E, T>(val: T): Result<E, T> => ({ value: val, __tag: "ok" });

const err = <E, T>(val: E): Result<E, T> => ({ value: val, __tag: "error" });
```

We can define `Result<E, T>` as a computation that can either fail with a value of type `E` or succeed with a value of type `T`.

Now we can define error-aware functions:

```typescript
type OperationError =
  | { type: "DIVISION_BY_ZERO"; dividend: number }
  | { type: "INDETERMINATE" }
  | { type: "NEGATIVE_INPUT"; value: number };

type SafeDivision = (
  a: number
) => (b: number) => Result<OperationError, number>;

const safeDivision: SafeDivision = (a) => (b) => {
  if (b === 0 && a !== 0) {
    return err({ type: "DIVISION_BY_ZERO", dividend: a });
  }

  if (b === 0 && a === 0) {
    return err({ type: "INDETERMINATE" });
  }

  return ok(a / b);
};

type RealSqrt = (a: number) => Result<OperationError, number>;

const realSqrt: RealSqrt = (a) => {
  if (a < 0) {
    return err({ type: "NEGATIVE_INPUT", value: a });
  }
  return ok(Math.sqrt(a));
};

type AddTwo = (a: number) => number;

const addTwo: AddTwo = (a) => {
  return a + 2;
};
```

Notice how operations that can fail have an explicit error interface while pure functions like `addTwo` just have a simple unwrapped return type?

## Addressing composition

Suppose we want to compute: √(a/b) + 2

The naive approach is verbose:

```typescript
type SqrtOfDivisionPlusTwo = (
  a: number
) => (b: number) => Result<OperationError, number>;

const sqrtOfDivisionPlusTwo: SqrtOfDivisionPlusTwo = (a) => (b) => {
  const divResult = safeDivision(a)(b);

  if (divResult.__tag === "error") {
    return divResult;
  }

  const sqrtResult = realSqrt(divResult.value);

  if (sqrtResult.__tag === "error") {
    return sqrtResult;
  }

  const plusTwo = addTwo(sqrtResult.value);

  return ok(plusTwo);
};
```

This pattern of "unwrap, check, short-circuit if error, continue if success" repeats constantly. We can abstract this.

## Introducing map and flatMap

We can abstract the repetitive pattern into two fundamental operations: `map` and `flatMap`.

### The map operation

Given a pure function `f: A → B` and a computation `ma: Result<E, A>`, the `map` operation applies `f` to the value contained in `ma` if it represents a success, otherwise it propagates the error:

```typescript
type Map = <A, B>(func: (a: A) => B) => <E>(ma: Result<E, A>) => Result<E, B>;

const map: Map = (func) => (ma) =>
  ma.__tag === "error" ? ma : ok(func(ma.value));
```

The type signature `Map: (A → B) → Result<E, A> → Result<E, B>` indicates that `map` lifts a pure function into the `Result` context, transforming the success type while preserving the error type.

### The nested Result problem

When composing computations that may fail, we encounter a structural issue. Consider applying a function `f: A → Result<E, B>` to a value `ma: Result<E, A>` using `map`:

```typescript
// Applying map with a function that returns Result yields nested Results:
const result = map(realSqrt)(safeDivision(10)(2));
// Type: Result<OperationError, Result<OperationError, number>>
//        ^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//        Outer Result           Inner Result
```

This yields the following nested structure: `Result<E, Result<E, T>>`.

### The flatMap operation

`flatMap` (also known as `bind` or `>>=`) resolves this by applying a function `f: A → Result<E, B>` and flattening the resulting nested structure. The operation propagates errors and applies the function only when the input represents a success:

```typescript
type FlatMap = <E, A, B>(
  func: (a: A) => Result<E, B>
) => (ma: Result<E, A>) => Result<E, B>;

const flatMap: FlatMap = (func) => (ma) =>
  ma.__tag === "error" ? ma : func(ma.value);
```

The type signature `FlatMap: (A → Result<E, B>) → Result<E, A> → Result<E, B>` ensures that the result remains a single `Result` layer, enabling sequential composition of computations that may fail.

Can you spot the implementation difference between map and flatMap, besides the types?

**Distinction:**

- `map`: Lifts pure functions `(A → B)` into the `Result` context
- `flatMap`: Composes computations that may fail `(A → Result<E, B>)` while preserving a flat structure

Now our composition becomes a bit more elegant:

```typescript
type Pipe = <A, B, C, D>(
  initial: A,
  f1: (a: A) => B,
  f2: (b: B) => C,
  f3: (c: C) => D
) => D;

const pipe: Pipe = (initial, f1, f2, f3) => f3(f2(f1(initial)));

const sqrtOfDivisionPlusTwo: SqrtOfDivisionPlusTwo = (a) => (b) =>
  pipe(b, safeDivision(a), flatMap(realSqrt), map(addTwo));
```

**Exercise for the reader**: create a more generic `pipe` function with non-fixed parameter arguments.

## Representing absence of values with the Option wrapper

Beyond errors, another common scenario is representing the _absence_ of a value. Consider a function that finds an element in an array:

```typescript
type FindUser = (users: User[]) => (id: string) => User | undefined;
```

The `undefined` return type doesn't compose well. We can apply the same wrapper pattern we used for errors:

```typescript
type Option<A> = { value: A; __tag: "some" } | { __tag: "none" };

const some = <A>(value: A): Option<A> => ({ value, __tag: "some" });

const none = <A>(): Option<A> => ({ __tag: "none" });
```

`Option<A>` represents a computation that may or may not produce a value of type `A`.

Notice the structural similarity to `Result<E, A>`.

**Exercise for the reader:** Implement `map` and `flatMap` for `Option<A>` following the same pattern we used for `Result`.

## Abstracting the wrapper type

You may have noticed that `Result` and `Option` seem similar. Both are wrapper types that:

1. Wrap a value (`ok`/`some`)
2. Support `flatMap` for sequential composition

This pattern has a name in category theory: **Monad**.

A Monad is any type constructor `M<A>` equipped with two operations:

```typescript
type Of = <A>(a: A) => M<A>;

type FlatMap = <A, B>(func: (a: A) => M<B>) => (ma: M<A>) => M<B>;
```

Note that map can be derived from `flatMap` & `of`. An equivalent formulation uses `join`: `M<M<A>> → M<A>` instead of flatMap, they're interderivable.
We won't go into details here, and if you know this already, this article is probably well beneath you.

### The three monad laws

To qualify as a proper Monad, these operations must satisfy three laws:

**1. Left Identity**

```typescript
flatMap(f)(of(a)) <===> f(a);
```

**2. Right Identity**

```typescript
flatMap(of)(ma) <===> ma;
```

**3. Associativity**

```typescript
flatMap(g)(flatMap(f)(ma)) <===> flatMap((a) => flatMap(g)(f(a)))(ma);
```

These laws ensure that monadic composition behaves predictably and that the operations are consistent.

**Exercise for the reader:** Do our `Result` and `Option` types satisfy the three laws?

**Exercise for the reader:** What about the `Promise<T>` type in TypeScript? It appears similar, but does it really satisfy the three laws?

## Real-world implementations

Throughout this article, we've built `Result` and `Option` from scratch to understand the underlying concepts. In practice, mature implementations already exist at language level or in well-tested libraries.

**Rust** has `Result<T, E>` and `Option<T>` as first-class citizens in the language, with syntactic sugar like the `?` operator for ergonomic error propagation:

```rust
fn sqrt_of_division(a: f64, b: f64) -> Result<f64, MathError> {
    let div = safe_division(a, b)?;
    let sqrt = real_sqrt(div)?;

    Ok(sqrt + 2.0)
}
```

**Haskell** has the `Either` type (equivalent to our `Result`) and `Maybe` (equivalent to our `Option`), along with do-notation for monadic composition:

```haskell
sqrtOfDivision :: Double -> Double -> Either MathError Double

sqrtOfDivision a b = do
    div <- safeDivision a b
    sqrt <- realSqrt div
    return (sqrt + 2)
```

**TypeScript** doesn't have native support, but libraries like [fp-ts](https://gcanti.github.io/fp-ts/) provide battle-tested implementations of `Either`, `Option`, `IO`, and many other functional abstractions with full type safety:

```typescript
import { pipe } from "fp-ts/function";
import { Either, chain, map } from "fp-ts/Either";

const sqrtOfDivisionPlusTwo = (a: number) => (b: number) =>
  pipe(b, safeDivision(a), chain(realSqrt), map(addTwo));
```

These implementations handle edge cases, provide utility functions, and integrate with the broader ecosystem. Use them.

## On the title

Thanks for reaching so far, and sorry for the _click-bait_ title.

The "Never" in the title is deliberately provocative, there are legitimate use cases for exception throwing.

The goal here was to present a new pattern and to make you aware that every `throw` creates an invisible contract that your type system can't enforce.

When you want predictable, composable error handling the Result pattern is very useful.
