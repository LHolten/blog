+++
title = "Anouncing rust-query"
date = "2024-11-24"
description = "New database library for rust"
[taxonomies]
tags = [ "database", "rust" ]
+++
# Safe relational database queries using the Rust type system

Do you want to persist your data safely without migration issues and easily write complicated queries? All of this without writing a single line of SQL? If so, then I am making `rust-query` for you!

> This is my first blog post about `rust-query`, a project I've been working on for many months. I hope you like it!

### Rust and databases

There is only one reason why I made this library and it is because I don't like the current options for interacting with a database from rust. The existing libraries don't provide the compile time guarantees that I want and are verbose or awkward like SQL.

The reason I care so much is that databases are really cool. They solve a huge problem of making crash resistant software with support for atomic transactions.

## Structured query language (SQL) is a protocol

For those who don't know, SQL is **the** standard when it comes to interacting with databases. So much so that almost all databases only accept queries in some dialect of SQL.

My opinion is that SQL should be for computers to write. This would put it firmly in the same category as LLVM IR. The fact that it is human-readable is useful for debugging and testing, but I don't think it's how you want to write queries.

It seems like there are two schools of thought in the design of database libraries:
- The wrappers that require you to write SQL directly. Especially `sqlx` is an interesting one, because it can ask the database to validate SQL and give back type information.
- Wrappers that don't require you to write SQL. There is so much potential here, but I think it has been under-explored. Diesel is a bit of an edge case that tries to stay close to SQL syntax, even adopting SQL's column scoping rules – which in my opinion is one of the worst parts of SQL.


# Introducing `rust-query`

`rust-query`[^crate] is my answer to relational database queries in Rust. It's an opinionated library that deeply integrates with Rust's type system to make database operations feel Rust native.

## Key Features and Design Decisions

I could write a blog post about each one of these, but lets keep it short for now:

- **Explicit table aliasing**: Joining a table gives back a dummy representing that table `let user = User::join(rows);`.
- **Null safety**: Optional values in queries have `Option` type, requiring special care to handle.
- **Row references tied to transaction lifetime**: References can only be used while the row is guaranteed to exist.
- **Encapsulated typed row IDs**: The actual row numbers are never exposed from the library API. Application logic should not need to know about them.
- **Intuitive aggregates**: Our aggregates are guaranteed to give a single result for every row they're joined on. After trying it, you'll see this is much more intuitive than traditional GROUP BY operations.
- **Type-safe foreign key navigation**: Database constraints are like type signatures, so you can rely on them for your queries with easy-to-use implicit joins by foreign key (e.g., `track.album().artist().name()`).
- **Type-safe unique lookups**: For example, you can get a an `Option<Rating>` dummy with `Rating::unique(my_user, my_story)`.
- **Multi-versioned schema**: It's declarative and you can see the differences between all past versions of the schema at once!
- **Type-safe migrations**: Migrations have all the power of queries and can use arbitrary Rust code to process rows. Ever had to consult something outside the database for use in a migration? Now you can!
- **Type-safe unique conflicts**: Inserting and updating rows in tables with unique constraints results in specialized error types.

## Lets see it!

You always start by defining a schema. With `rust-query` it's easy to migrate to a different schema later.

```rust
#[schema]
enum Schema {
    User {
        name: String,
    },
    Story {
        author: User,
        title: String,
        content: String
    },
    #[unique(user, story)]
    Rating {
        user: User,
        story: Story,
        stars: i64
    },
}
use v0::*;
```

Schema defintions in `rust-query` use enum syntax, but no actual enum is defined here.
This schema defines three tables with specified columns and relationships:
- Using another table name as a column type creates a foreign key constraint.
- The `#[unique]` attribute creates named unique constraints.
- The `#[schema]` macro generates a module `v0` that contains the database API.

### Writing Queries

First, let's see how to insert some data into our schema:

```rust
fn insert_data(txn: &mut TransactionMut<Schema>) {
    // Insert users
    let alice = txn.insert(User {
        name: "alice",
    });
    let bob = txn.insert(User {
        name: "bob",
    });
    
    // Insert a story
    let dream = txn.insert(Story {
        author: alice,
        title: "My crazy dream",
        content: "A dinosaur and a bird..."
    });
    
    // Insert a rating - note the try_insert due to the unique constraint
    let rating = txn.try_insert(Rating {
        user: bob,
        story: dream,
        stars: 5,
    }).expect("no rating for this user and story exists yet");
}
```

A few important points about insertions:
- We need a mutable transaction (`TransactionMut`) to modify the database.
- Insert operations return references to the newly inserted rows.
- When inserting into tables with unique constraints, use `try_insert` to handle potential conflicts.
- The error type of `try_insert` is based on how many unique constraints the table has.

Now let's query this data:

```rust
fn query_data(txn: &Transaction<Schema>) {
    let results = txn.query(|rows| {
        let story = Story::join(rows);
        let avg_rating = aggregate(|rows| {
            let rating = Rating::join(rows);
            rows.filter_on(rating.story(), &story);
            rows.avg(rating.stars().as_float())
        });
        rows.into_vec((story.title(), avg_rating))
    });

    for (title, avg_rating) in results {
        println!("story '{title}' has avg rating {avg_rating:?}");
    }
}
```

Key points about queries:
- `rows` represents the current set of rows in the query.
- Joins can add rows and filters can remove rows. <details>By joining a table like `Story`, the `rows` set is mutated to be the Cartesian product of itself and the rows from the joined table. The query above only has a single `join`, so we know it will give exactly one result for each row in the `Story` table.</details> 
- Column dummies like `story` refer to joined tables.
- Results can be collected into vectors of tuples or structs.
- Using `aggregate` to calculate an aggregate, does not change the number of rows in the query.
- `rows.filter_on` can be used to filter rows in the aggregate to match a value from the outer scope.
- The `rows.avg` method returns the average of the rows in the aggregate, if there are no rows then the average will evaluate to `None`.

### Schema Evolution and Migrations

Let's say you want to add an email address to each user. Here's how you'd create the new schema version:

```rust
#[schema]
#[version(0..=1)]
enum Schema {
    User {
        name: String,
        #[version(1..)]
        email: String,
    },
    // ... rest of schema ...
}
use v1::*;
```

And here's how you'd migrate the data:

```rust
let m = m.migrate(v1::update::Schema {
    user: Box::new(|old_user| {
        Alter::new(v1::update::UserMigration {
            email: old_user
                .name()
                .map_dummy(|name| format!("{name}@example.com")),
        })
    }),
});
```

- The `v1::update` module contains `struct`s defining the difference between schema `v0` and schema `v1`.
- We use these structs to implement the migration. This way the migration is type checked against both the old and new schemas.
- Note that inside migrations we can execute all single-row queries we want: aggregates, unique constraint lookups etc!
- We can also use `map_dummy` with arbitrary rust to process rows further.

## Conclusion

`rust-query` represents a fresh approach to database interactions in Rust, prioritizing:
- Checking everything possible at compile time.
- Making it possible to compose queries with each other and arbitrary Rust.
- Enabling schema evolution with type-checked migrations.

While still in development, the library already allows building experimental database-backed applications in Rust. I encourage you to try it out and provide feedback through GitHub[^github] issues!

> The library currently uses SQLite as its only backend, chosen for its embedded nature. This will not change anytime soon, as one backend is most practical while `rust-query` is in development.

[^crate]: <https://crates.io/crates/rust-query>

[^github]: <https://github.com/LHolten/rust-query>