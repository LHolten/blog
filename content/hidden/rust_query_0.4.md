+++
title = "[Draft Post] Version 0.4 of rust-query is released!"
date = "2025-4-13"
description = "rust-query 0.4 has many improvements described in this post"
[taxonomies]
tags = [ "database", "rust" ]
[extra]
hidden = true
+++

It has been more than 4 months since I first [wrote about](/rust-query-announcement) `rust-query`, the query builder for SQLite. Since then there have been a lot of changes. I am really excited about the state of things and I am looking forward to feedback from the Rust community!

To show the new features, I will set the stage by defining a schema:
```rust
#[schema(Schema)]
#[version(0..=0)]
pub mod vN {
    pub struct Measurement {
        pub score: i64,
        pub duration: i64,
        pub confidence: f64,
        pub timestamp: i64,
        pub location: Location,
    }
    pub struct Location {
        pub name: String,
    }
}
use v0::*;
```
This syntax is slightly different from `rust-query 0.3`; the module of structs is much closer to the actual output of the `schema` macro. You can think of the whole `vN` module as a template for the modules that the macro generates. The generated modules have names `v0`, `v1` etc, depending on the version range specified.

Now that we have a schema to play with, let us compare the old and new approach to retrieving some data.
First, this is how you would retrieve data in `rust-query 0.3`.
```rust
#[derive(Select)]
struct Score {
    score: i64,
    timestamp: i64,
}

fn read_scores(txn: &Transaction<Schema>) -> Vec<Score> {
    txn.query(|rows| {
        let m = Measurement::join(rows);
        rows.into_vec(ScoreSelect {
            score: m.score(),
            timestamp: m.timestamp(),
        })
    })
}
```
As you can see, this is very flexible, but for a simple query like this one it is too verbose.

## Type Driven Select
That is why I introduce a new trait called `FromExpr`. This trait is intended to create `Select`s from single `Expr`essions. What is more interesting is that it can be derived on structs as well. With this derive macro the previous example now looks like this:
```rust,hl_lines=1-2 11
#[derive(FromExpr)]
#[rust_query(From = Measurement)]
struct Score {
    score: i64,
    timestamp: i64,
}

fn read_scores(txn: &Transaction<Schema>) -> Vec<Score> {
    txn.query(|rows| {
        let m = Measurement::join(rows);
        rows.into_vec(FromExpr::from_expr(m))
    })
}
```
The derive macro will fill each field from the column with identical name. So no more unnecessary selection like `score: m.score()` and `timestamp: m.timestamp()`.

Defining the `Score` type is still quite verbose though.
Which is why I added another option that is even more concise.
```rust,hl_lines=1
fn read_scores(txn: &Transaction<Schema>) -> Vec<Measurement!(score, timestamp)> {
    txn.query(|rows| {
        let m = Measurement::join(rows);
        rows.into_vec(FromExpr::from_expr(m))
    })
}
```
The `Score` struct is completely gone! All that is left is the return type of the function, which now specifies the columns that will be retrieved from the database. This features is what I will call "structural types".

## Structural Types

For every table type like `Measurement` there is a macro with the same name.
This allows you to create a type to select a subset of columns in that table.
For example, it is possible to write `Measurement!(score, timestamp)` and that will specify the
`Measurement` type with only those two columns filled in.
How it works is that `Measurement` struct is actually generic over the type of each field and the macro
expands to specify those generics. Fields that are not used will get the `()` type.

It is also possible to override the type of a field with a different type that implements `FromExpr`.
So for example it is possible to use `Measurement!(score, location as Location!(name))` to retrieve
scores with the name of the location.
The syntax is a bit weird because I wanted to make it concise and still let `rustfmt` format it.

## Migration Refactor

The new structural types and `FromExpr` trait made it possible to redesign migrations to be much
more ergonomic.
To show of the new migrations we need to add a new version to the schema.
As an example, let us rename the `score` column to `value` and change the datatype from `i64` to `f64`.

```rust,hl_lines=2 5 7-8 18
#[schema(Schema)]
#[version(0..=1)]
pub mod vN {
    pub struct Measurement {
        #[version(..1)]
        pub score: i64,
        #[version(1..)]
        pub value: f64,
        pub duration: i64,
        pub confidence: f64,
        pub timestamp: i64,
        pub location: Location,
    }
    pub struct Location {
        pub name: String,
    }
}
use v1::*;
```
This is quite straightforward, we add the new column and specify when these columns start and stop to exist.

Now we need to write the actual migration:
```rust,hl_lines=5-9
fn migrate(client: &mut LocalClient) -> Database<Schema> {
    let m = client
        .migrator(Config::open("db.sqlite"))
        .expect("database should not be older than supported versions");
    let m = m.migrate(|txn| v0::migrate::Schema {   
        measurement: txn.migrate_ok(|old: v0::Measurement!(score)| v0::migrate::Measurement {
            value: old.score as f64,
        }),
    });
    m.finish()
        .expect("database should not be newer than supported versions")
}
```
The highlighted lines are all that we need to add to implement this migration.
- Types in the `v0::migrate` module are generated specifically to help with this migration.
They check that all tables are migrated and that things like unique constraint conflicts and foreign key errors are handled.
- The closure provided in `txn.migrate_ok(|old: v0::Measurement!(score)| ..)` can choose the argument type as long as it implements `FromExpr`.
This works perfectly together with the structural type macros to select whatever is necessary to perform the migration.
- Any number of changes can be bundled together in a single schema version update.
The recommended approach is to aggregate all schema changes in a single schema version until the software is released and then to remove schema versions from the code when they no longer need to be supported.
This approach keeps the size of the multi-versioned schema in check. 

### Migration "from" Attribute
You can now use the `#[from]` attribute on table definitions to facilitate migrations like table renaming and duplication/splitting of tables. These were not possible before, so this is really helpfull.
```rust,hl_lines=4 8-12 14
#[schema(Schema)]
#[version(0..=1)]
pub mod vN {
    #[version(..1)]
    pub struct User {
        pub name: String,
    }
    #[version(1..)]
    #[from(User)]
    pub struct Author {
        pub name: String,
    }
    pub struct Book {
        pub author: Author,
    }
}
```
As you can see, renaming a table is as easy as copying the definition with a new name and adding the `#[from]` attribute. All tables with a foreign key to the old `User` table, such as `Book` here, just need to use the new table name (`Author`). The macro will figure out that for `v0`, the `Book` table actually still refers to the old table `User` by following the `#[from]` specification.

## Deletion
Deleting rows has been possible since `rust-query 0.3.1`. It makes use of a new transaction type called `TransactionWeak`, which is named after the `Weak` reference counted type.

```rust
fn do_stuff(mut txn: TransactionMut<Schema>) {
    let loc: TableRow<Location> = txn.insert_ok(Location {
        name: "Amsterdam",
    });

    let mut txn: TransactionWeak<Schema> = txn.downgrade();
    
    let is_deleted = txn.delete(loc).expect("there should be no fk references to this row");
    assert!(is_deleted);

    let is_not_deleted_twice = !txn.delete(loc).expect("there should be no fk references to this row");
    assert!(is_not_deleted_twice);
}
```
To prevent use-after-delete of row references, `TransactionWeak` does not allow querying the database.
`TransactionWeak` can thus only remove rows for which a `TableRow` has already been retrieved.
Upgrading the `TransactionWeak` back to a `TransactionMut` is something I intend to add later.

## Optional Combinator

The `optional` combinator allows combining any number of optional expressions and returning a `Select` when all of the input expressions are `Some` (not null).

Let us looks at a bit of an advanced example:
```rust
#[derive(Select)]
struct Info {
    average_value: f64,
    total_duration: i64,
}

fn location_info<'t>(txn: &Transaction<'t, Schema>, loc: TableRow<'t, Location>) -> Option<Info> {
    txn.query_one(aggregate(|rows| {
        let m = Measurement::join(rows);
        rows.filter_on(m.location(), loc);

        optional(|row| {
            let average_value = row.and(rows.avg(m.value()));
            row.then(InfoSelect {
                average_value,
                total_duration: rows.sum(m.duration()),
            })
        })
    }))
}
```
First note that the aggregate filters measurements for a specific location using `rows.filter_on`.
Since not all locations have associated measurements, this list may be empty.
That is why the `rows.avg` method return an expression with `Option` type.
We only want to return `Some(Info)` if we have an average, so that is why we use the `optional` combinator. It allows us to use `row.and` with `row.then` to only construct `Info` when we have an average.

### Covariant Expressions
`Expr` (or rather `Column` before it was renamed) used to be invariant in its lifetime, this fundamental property made it easy to separate scopes.
However, for the `optional` combinator to be practical, it should be possible to use `Expr` from the outer scope inside, while expressions created inside should be prevented from leaking out.
To support this use case, `Expr` is now covariant and all APIs that relied on the invariant lifetime have been updated.

Covariant lifetimes seem to have the additional benefit that `rustc` understands them better. This means that the error messages are better on average. Sadly there is at least one case I know of where `rustc` is being a bit too smart and suggests making the transaction lifetime static. While this suggestion is technically a correct one, it is not useful because transactions should normally have a short lifetime.
Luckily, the error message is actually better than before when the transaction is not used in the same scope as where it is created.

## Other Changes
Here are some more changes that I won't go into too much detail about:

- `Query::into_vec` no longer sorts the rows.
- Updates now use the `Update` type for each column.
- Added support for `GLOB` and `LIKE` operators (contributed by @teamplayer3).
- Renamed `Dummy` to `Select`.
- Renamed `Column` to `Expr`.
- Renamed `try_insert` to `insert` and `insert` to `insert_ok`.
- Renamed `try_delete` to `delete` and `delete` to `delete_ok`.
- Renamed `try_update` to `update` and `update` to `update_ok`.

Please take a look at [the changelog](https://github.com/LHolten/rust-query/blob/main/CHANGELOG.md) for more.
