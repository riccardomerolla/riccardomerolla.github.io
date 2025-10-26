---
layout: post
title: "ZIO – Pure Functional Programming"
date: 2022-05-10
categories: [Scala, Functional Programming, ZIO]
description: "Exploring performance improvements and pure functional programming with ZIO, fibers, and composable effects in Scala."
---

> “How to generate 1K entity records in 1 second and 10K in less than 4 seconds…”

When debugging some performance issues, I ran into the classic mistakes of the past, present, and future: homegrown frameworks, Hibernate’s mysterious N+1000 queries, 3K-line classes, dependency spaghetti, SRP and SOLID violations, and cyclomatic complexity off the charts (the eighth nested `if` makes you want to rub wasabi in your eyes).

But that’s legacy code — we’ve all been there. A bit of `Ctrl+C` and `Ctrl+V`, a few extra `if`s, and voilà, the new requirement “works.” Problem “solved.” Of course, we’ve also just planted a ticking time bomb for future disasters.

So I started from testing — not out of some dogmatic TDD spirit, but because **you can’t fix performance without numbers**. You need measurements, benchmarks, and confidence that your changes help, not hurt. The challenge with legacy systems (by definition, untested) is isolating the behavior you want to improve.

This time, my tool of choice was **ZIO**, a pure functional library for Scala focused on **asynchronous, concurrent, and composable programming**.

---

## What is ZIO?

ZIO is a library for **pure functional programming in Scala** that makes async and concurrent code type-safe, testable, and composable. It’s inspired by Haskell’s `IO` monad and represents complex effects with a simple data type:

```scala
ZIO[R, E, A]
```

Where:
- `R` = environment required for execution  
- `E` = error type  
- `A` = success type  

Conceptually, it’s like a function `R => Either[E, A]`, but instead of being executed immediately, a `ZIO` value *describes* a computation. You can compose and transform these values using familiar functional methods like `map`, `flatMap`, and `zip`, and the program only runs once the environment is provided.

ZIO also introduces **fibers** — lightweight, composable units of concurrency that make parallelism simple and efficient.

---

## From Legacy Pain to Functional Gain

In one legacy system I was optimizing, the function responsible for generating entities took *minutes* to process less than 1K records — and sometimes timed out completely. After investigating, we found that Hibernate was generating several queries per record.

I decided to rewrite the logic in ZIO, using **functional composition** and a **clean test harness** to measure progress. The stack included ZIO, Tapir, Doobie, and Http4s.

Here’s the core repository definition:

```scala
trait SerialKitRepository:
  def createIncomplete(bomId: Long, expirationDays: Int, divcod: Int, azcod: Int = 1): Task[Long]

object SerialKitRepository extends zio.Accessible[SerialKitRepository]
```

And a simple mock implementation for testing:

```scala
case object SerialKitRepositoryMock extends SerialKitRepository:
  override def createIncomplete(bomId: Long, expirationDays: Int, divcod: Int, azcod: Int = 1): Task[Long] =
    Task.succeed(1)
```

---

## First Steps: Sequential Tests

Before optimizing, I created tests to validate correctness and timing:

```scala
test("should create 1K of new serial kits in few seconds") {
  val program = SerialKitRepository(_.createIncomplete(16441870, 60, 9, 1))
  val qty = 1000
  val testcase = for {
    l  <- ZIO.foreach(1 to qty)(_ => program)
  } yield l

  testcase.map(assert(_)(anything)).provideCustomLayer(testLayer)
}
```

Result: around **3 seconds** to generate 1K records — a huge improvement already thanks to simpler queries and a cleaner architecture.

---

## Parallelism: The One-Letter Difference

Then came the magic of ZIO’s fibers. I changed just one line:

```scala
l  <- ZIO.foreachPar(1 to qty)(_ => program)
```

That small `Par` changed everything.

- **1K records** → ~1 second  
- **10K records** → ~3.5 seconds  

That’s **true performance through functional design** — achieved with clarity, composability, and pure functions.

---

## Conclusion

Functional programming isn’t about elegance for its own sake. It’s about **building systems you can reason about**, **test in isolation**, and **improve safely**.

ZIO gives Scala developers the right abstractions to handle concurrency without fear, to express intent without side effects, and to scale performance with confidence.

---

> 10K entity records generated in 3.5 seconds — this is performance, but more importantly, this is the beauty of composable code.
