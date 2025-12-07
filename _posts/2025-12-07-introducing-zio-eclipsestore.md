---
layout: post
title: "🛰️ Introducing zio-eclipsestore: Type-Safe Persistence with ZIO"
date: 2025-12-07
tags: [scala, zio, storage, persistence, streaming]
---

Meet **zio-eclipsestore**: a ZIO-first wrapper around [EclipseStore](https://github.com/eclipse-store/store) that gives you type-safe persistence, automatic query batching, streaming reads, and batteries-included lifecycle operations (checkpoints, backups, imports/exports). It leans on **zio-schema** for typed codecs and **zio-streams** for memory-safe processing, so you can keep everything functional and testable.

## Why zio-eclipsestore

- **Type-safe API**: compile-time safety via ZIO and zio-schema
- **Automatic batching + parallelism**: batchable queries are optimized; non-batchable run in parallel with controlled concurrency
- **Streaming persistence**: stream keys/values, batch writes (`putAll`, `persistAll`)
- **Typed roots**: declaratively describe aggregates with `RootDescriptor`
- **Lifecycle management**: checkpoints, backups, status, import/export
- **GigaMap module**: indexed maps with predicates, CRUD, persistence
- **Storage targets**: filesystem, mmap, in-memory, SQLite, S3/FTP backups

## Install

```scala
libraryDependencies ++= Seq(
  "io.github.riccardomerolla" %% "zio-eclipsestore" % "1.0.5",
  "io.github.riccardomerolla" %% "zio-eclipsestore-gigamap" % "1.0.5",      // optional: GigaMap
  "io.github.riccardomerolla" %% "zio-eclipsestore-storage-sqlite" % "1.0.5" // optional: SQLite backend/backup
)
```

## Quick Start (put/get + streaming)

```scala
import io.github.riccardomerolla.zio.eclipsestore.config.EclipseStoreConfig
import io.github.riccardomerolla.zio.eclipsestore.domain.{Query, RootDescriptor}
import io.github.riccardomerolla.zio.eclipsestore.service.{EclipseStoreService, LifecycleCommand}
import io.github.riccardomerolla.zio.eclipsestore.ToonValue
import zio._
import scala.collection.mutable.ListBuffer

def app = for {
  // Batch store values
  _ <- EclipseStoreService.putAll(List(
    "user:1" -> "Alice",
    "user:2" -> "Bob",
    "user:3" -> "Charlie"
  ))

  // Retrieve one
  user1 <- EclipseStoreService.get[String, String]("user:1")
  _     <- ZIO.logInfo(s"User1: ${user1.getOrElse("n/a")}")

  // Stream all values (zio-streams)
  all <- EclipseStoreService.streamValues[String].runCollect
  _   <- ZIO.logInfo(s"All users: ${all.mkString(", ")}")

  // Typed root (mutable aggregate managed by EclipseStore)
  favoritesDescriptor = RootDescriptor(
    id = "favorite-users",
    initializer = () => ListBuffer.empty[String]
  )
  favorites <- EclipseStoreService.root(favoritesDescriptor)
  _         <- ZIO.succeed(favorites.addOne("user:1"))
  _         <- EclipseStoreService.maintenance(LifecycleCommand.Checkpoint)

  // Batched queries
  queries = List(
    Query.Get[String, String]("user:1"),
    Query.Get[String, String]("user:2"),
    Query.Get[String, String]("user:3")
  )
  results <- EclipseStoreService.executeMany(queries)
  _       <- ZIO.logInfo(s"Batch results: ${results.mkString(", ")}")
} yield ()

object MyApp extends ZIOAppDefault:
  def run = app.provide(
    EclipseStoreConfig.temporaryLayer,  // in-memory; great for tests
    EclipseStoreService.live
  )
```

### Real scenario: Bookstore backend (HTTP + persistence)

The repo ships a full sample (`BookstoreServer`) using `zio-http` + `zio-json` + zio-eclipsestore.

```bash
sbt bookstore/run   # serves on :8080
```

Routes:
- `GET /books` list
- `POST /books` create
- `GET /books/{id}` fetch
- `PUT /books/{id}` update
- `DELETE /books/{id}` delete

All writes persist the `BookstoreRoot`; you can also trigger checkpoints/backups via `EclipseStoreService`.

## zio-schema: typed codecs, zero boilerplate

```scala
import zio.schema._
import io.github.riccardomerolla.zio.eclipsestore.service._
import io.github.riccardomerolla.zio.eclipsestore.domain.Query

case class UserProfile(id: String, email: String, active: Boolean)
object UserProfile { given Schema[UserProfile] = DeriveSchema.gen }

val profilesRoot = RootDescriptor.concurrentMap[String, UserProfile]("profiles")

def profileProgram = for {
  root <- EclipseStoreService.root(profilesRoot)
  _    <- EclipseStoreService.persistAll(root, Map(
            "u-1" -> UserProfile("u-1", "a@example.com", active = true),
            "u-2" -> UserProfile("u-2", "b@example.com", active = false)
          ))

  fetched <- EclipseStoreService.execute(
    Query.Custom[List[UserProfile]](
      operation = "active-profiles",
      run = ctx => ctx.container.ensure(profilesRoot).values().filter(_.active).toList
    )
  )
  _ <- ZIO.logInfo(s"Active: ${fetched.map(_.id).mkString(",")}")
} yield ()
```

Benefits:
- Compile-time schemas; no hand-written codecs
- Type-safe queries and roots
- Serializable across storage targets

## zio-streams: large-scale processing

```scala
import zio.stream._

case class Metric(ts: Long, name: String, value: Double)
object Metric { given Schema[Metric] = DeriveSchema.gen }

val metricsStream: ZStream[Any, Nothing, (String, Metric)] =
  ZStream.fromIterable(1 to 1000000).map { i =>
    val m = Metric(System.currentTimeMillis() + i, s"metric-$i", math.sin(i) * 100)
    s"metric:$i" -> m
  }

def ingestMetrics = for {
  // Persist in streaming batches without loading all into memory
  _ <- EclipseStoreService.persistAll(metricsStream)

  // Stream them back for analytics
  top <- EclipseStoreService.streamValues[Metric]
           .filter(_.value > 50)
           .take(10)
           .runCollect
  _ <- ZIO.logInfo(s"Hot metrics: ${top.map(_.name).mkString(", ")}")
} yield ()
```

Why it matters:
- Backpressure-aware ingestion
- Constant memory even for millions of records
- Works with any storage target

## Storage targets, backups, and lifecycle

```scala
import java.nio.file.Paths
import io.github.riccardomerolla.zio.eclipsestore.config._
import io.github.riccardomerolla.zio.eclipsestore.service.{EclipseStoreService, LifecycleCommand}

val config = EclipseStoreConfig(
  storageTarget = StorageTarget.FileSystem(Paths.get("./data/store")),
  backupTarget  = Some(BackupTarget.S3Backup(
    accessKeyId     = "key",
    secretAccessKey = "secret",
    region          = "us-east-1"
  )),
  backupDirectory = Some(Paths.get("./data/backup"))
)

val maintenance = for {
  _      <- EclipseStoreService.maintenance(LifecycleCommand.Checkpoint)
  _      <- EclipseStoreService.maintenance(LifecycleCommand.Backup(Paths.get("./data/backup")))
  status <- EclipseStoreService.status
  _      <- ZIO.logInfo(s"Status: $status")
} yield ()
```

Optional SQLite backend:

```scala
import io.github.riccardomerolla.zio.eclipsestore.sqlite.SQLiteAdapter
val sqliteLayer = SQLiteAdapter.live(basePath = Paths.get("./data"), storageName = "my-store.db")
```

## GigaMap: indexed, queryable maps

```scala
import io.github.riccardomerolla.zio.eclipsestore.gigamap._
import zio._

case class Book(id: Long, title: String, authors: List[String], year: Int)
object Book { given Schema[Book] = DeriveSchema.gen }

val definition = GigaMapDefinition[Long, Book](
  id = "books",
  indexes = Chunk(
    GigaMapIndex.byField("byTitle", _.title),
    GigaMapIndex.byField("byYear", _.year.toString)
  )
)

def gigamapProgram = for {
  map <- GigaMap.make(definition)
  _   <- map.put(1L, Book(1, "ZIO In Action", List("Alice"), 2024))
  _   <- map.put(2L, Book(2, "FP for Builders", List("Bob", "Cara"), 2023))

  recent <- map.query(GigaMapQuery.byIndex("byYear", "2024"))
  titled <- map.query(GigaMapQuery.byIndex("byTitle", "ZIO In Action"))

  _ <- ZIO.logInfo(s"Recent: ${recent.map(_.title)}")
  _ <- ZIO.logInfo(s"Match: ${titled.map(_.title)}")
} yield ()
```

## Config with zio-config (HOCON/resources)

```scala
import io.github.riccardomerolla.zio.eclipsestore.config.EclipseStoreConfigZIO
import zio.ConfigProvider

val configLayer = EclipseStoreConfigZIO
  .fromResource("application.conf")
  .map(_.toLayer)
  .orDie
```

`application.conf` example:

```hocon
eclipsestore {
  storage-target = "filesystem:/data/store"
  performance.channel-count = 8
  backup.directory = "/data/backup"
}
```

## Takeaways

- **ZIO everywhere**: effect-oriented API, typed errors, ZLayer wiring, resource safety
- **zio-schema**: compile-time codecs for roots, queries, and payloads
- **zio-streams**: backpressure and constant-memory ingestion/extraction
- **Ops ready**: checkpoints, backups, import/export, performance tuning (channels, cache, compression, encryption)
- **Real apps**: shipped bookstore backend and gigamap CLI REPL to learn from

Give it a spin:

```bash
sbt "runMain io.github.riccardomerolla.zio.eclipsestore.app.EclipseStoreApp"
```

Repo: [github.com/riccardomerolla/zio-eclipsestore](https://github.com/riccardomerolla/zio-eclipsestore)

Built with ❤️ for the Scala and ZIO communities.
