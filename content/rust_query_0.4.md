+++
title = "What is new in rust-query 0.4?"
draft = true
date = "2025-3-7"
description = "rust-query 0.4 has many improvements described in this post"
[taxonomies]
tags = [ "database", "rust" ]
+++

# Version 0.4 of rust-query is released!

A lot has changed.


# Type Directed Select

There is a new trait called `FromExpr` which allows doing type directed selects.
Implementating this trait can be done in three different ways"
- Manual
- With derive macro
- Using the new structural types

# Structural Types

For every table type like `User` there now is also a macro with the same name.
This allows you to select a subset of the columns in that table.
so for example it is possible to write `User!{name, age}` and that will specify the
`User` type with only those columns filled in.

How it works is that `User` is actually generic over the type of each field and the macro
expands to specify those generics. Fields that are not used will get the `()` type.

# Migration Refactor

The new structural types and type directed selects make it possible to redesign migrations to be much
more ergonomic.
....
There is now a reduced number of closures requires because selecting the columns required for the migration
is done by specifying the argument type of the closure.

## Panic Free Migration

It has been made impossible for the migration functions to panic due to foreign key constraint and unique constraint violations.
When there is the possibility for one of these violations to occur, the user has to handle the error in some way.
(note that when foreign key violations can occur, they can only be resolved by explicitly diverging for now.)

# New Schema Syntax.

One of the commenters on my first post suggested that I change the syntax for schemas to a module of structs.
That is exactly what I did. It is more verbose, but also a lot more clear what the output of the macro approximately looks like.

# Optional Combinator

The `optional` combinator is similar to `aggregate`, but instead of allowing joins, it only allows adding more optional values.
This is useful if you want to join a table on an optional column for example.


# Lots of Renaming

Rarely do I get the name of a type or method correct on the first try.

## Types
- `Column` -> `Expr`
- `Dummy` -> `Select`

## methods

# Some Additional Operators
