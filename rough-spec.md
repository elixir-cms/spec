# Rough spec

The initial target of this project is to build a headless CMS in and for Elixir that can be used either as an embeddable library in your application or as a stand-alone service.

Why headless? Long term there may well be a case for building out the website generation part and integrating that more. But there is a lot of bang for the buck in terms of time and a very wide use-case for headless CMS:es. Most Elixir enthusiasts are developers. Targeting headless first means we constrain the scope significantly.

We are targeting Phoenix and Ecto which shouldn't be particularly controversial. We are likely not committing to a single database but would at least want to support Postgres and Sqlite which likely means MySQL will work fine too.

## Part 1: The Ecto Entity library

Look at almost any popular CMS and they will have runtime (not compile-time) editing of "content types" such as "pages" or "posts". You create them, you can manage what fields they have, all without redeploying code. This presents a bit of a challenge with how Ecto does compile time schema generation. Fortunately Ecto supports schemaless everything ([queries](https://hexdocs.pm/ecto/schemaless-queries.html), likely using [dynamic fragments](https://hexdocs.pm/ecto/dynamic-queries.html#dynamic-fragments) a lot, sometimes [raw queries](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.html#query/4) and schemaless [changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html#module-schemaless-changesets)). All the tools around Repo, Query and Migrator can be called at runtime with some alternative data structures. This library should make some opinionated decisions around building "entities", our conception of content types (they aren't just for content O_o). The library should make working with Ecto in this manner as convenient as possible.

This is the necessary foundation for enabling a runtime CMS system on top of Ecto while retaining most of the niceties of Ecto and making programming against it familiar to anyone familiar with Ecto.

It needs to provide:

- Runtime entity definitions
- Runtime migrations based on definition
- Runtime changesets based on definition
- Make entity definition import/export easy
- Tools to make data import/export feasible
- No opinions on persisting definitions or data

## Part 2: The Ecto Entity Storage library

Picking up where the Ecto Entity library leaves off this should provide the best practices for working with entities. It should provide a library API that helps you deal with persisting definitions and running migrations. Much like Ecto is useful without SQL, Ecto Entity may be useful without this Storage system. But as with Ecto SQL, for the 90% use case you want both.

- Provides a behaviour for implementing storage modules
- Implements FileStorage module for managing definitions and migrations via simple files
- Implements EctoRepoStorage module for managing definitions and migrations via an Ecto Repo
- Builds on Ecto Entity APIs to provide further APIs for creating and updating definitions by hooking into storage modules according to configuration
- Build on Ecto Entity APIs to provide further APIs for creating tables via migrations, running partial migrations and keeping the actual database table up to date
- Should not require compile time config, you should be able to run multiple instances with different configurations both for testability and unpredicted usage (this is a whole thing that sometimes libraries screw up)
- Import/export of data and definitions
- Provide default functions for List, Filter and Create/Read/Update/Delete operations that are pre-instrumented with hooks and events
- Provide facilities for implementing hooks (sync) and events (async) for integrating with the entity types and entity items from Elixir code, use Phoenix Pub.Sub for events

## Part 3: $CMS_NAME Web UI - The UI library

The thing you think about when using a CMS. This library should provide a UI that maps to the underlying Ecto Entity and Ecto Entity Storage libraries. It implements a number of Routes much like Live Dashboard or Oban Web and you decide your need for auth, the path, etc. via the Phoenix router and Plug pipelines.

- Provide routes you can Integrate into an existing Phoenix router
- Built in LiveView for optimum hype (and realtime snappiness)
- Drives the Ecto Entity library and Ecto Entity Storage library from a human-friendly UI
- List, create and alter entities, typically content types
- List, Create, Edit and Delete entity items, typically content

## Part 4: $CMS_NAME Web API - The API library

I don't know how feasible building an Absinthe GraphQL API is at runtime or whether we can nicely model these dynamic entities.

A REST API plus webhooks would be straight-forward but a GraphQL one would probably be cooler and give. more convenience out of the box (self-documenting, discoverable).

- Provide means of accessing and driving the Ecto Entity Storage library via API calls
- Provide a way for an external system to receive a notice when a change happens
- Provide routes that can plug into an existing Phoenix router



## Safe migration operations

### Only during the initial set of migrations

Database side constraints such as being required or unique can only be set for the initial set of fields added via migrations to an entity. If we would allow them after that point you could have a situation where the migration fails to apply because the existing data does not conform to the new constraint.

Required could arguably be solved by requiring that a default value is also set and have that default applied to all existing data. We should explore whether this is worthwhile.

### Operation: Add field

Always includes an `id` field that is a UUID string field. This avoids a number of common problems with entity data and auto-incrementing integer IDs. Especially around importing and exporting data between environments.

Field (example)
- Identifier (body)
- Schema type (string)
- Storage type (text)
- Persistence options
    - Accept empty values (nullable), only for initial fields
    - Unique, only for initial fields
    - Indexed
- Validation options
    - Required -> validate_required
    - Exclude values -> validate_exclusion
    - Format (regex, string only) -> validate_format
    - Include values -> validate_inclusion
    - Length (string only) -> validate_length
    - Number (integer only) -> validate_number
- Filters
    - slugify (string only) - Turn string into slug before storing
- Meta (stores any additional data, planned for use with the UI stuff)
    - Display
        - Type (textarea|richtext|text)
        - Label (Body text)
        - Position (or weight, placement in any UI listing of the fields)
        - Output format (markdown)
        - Filterable (requires Indexed = true)

### Operation: Change field

- Validation options
    - All the same ones
- Meta
    - All the same ones


### Operation: Archive field

Removes a field from active duty. It will not be part of the field definition but not actually removed from the database.

## Rough data format for migration sets

Given that the migration sets are part of a definition and the real source of truth. This is what the user puts into the UI and what we use to produce everything else.

```elixir
[
    {
        migration_id: <UUID>,
        created_at: <datetime, just for convenience>,
        migration_set: [
            # Add primary key, always the same, always UUID-based probably
            {
                type: "add_primary_key",
                identifier: "id",
                key_type: "string"
            },
            {
                type: "add_field",
                identifier: "title",
                field_type: "string",
                storage_type: "text",
                persistence_options: {
                    nullable: false,
                    unique: true,
                    indexed: true
                },
                validation_options: {
                    required: true,
                    ..
                },
                filters: {}
                meta: {..}
            },
            {
                type: "add_field",
                identifier: "body",
                field_type: "string",
                storage_type: "text",
                persistence_options: {
                    nullable: false
                },
                validation_options: {
                    required: true,
                    ..
                },
                filters: {},
                meta: {..}
            }
        ]
    },
    ..
]
```

## Rough data generated from migrations sets

### fields

These are used directly by Ecto for schemaless changesets and not needing to derive them from migrations every time we load the definition as well as the readability advantage for the definition will both contribute to a better experience working with the definitions. It has a potential drawback of needing to update two places if editing definitions by hand, similarly to working with Ecto normally and needing to update both migration and schema.

```elixir
{
    title: string,
    body: string
}
```

### changesets

Generated from the migrations. Every entity should provide a few changesets by default and like with fields I think it is helpful for the definition to provide them pre-chewed to some extent. This could arguably be generated at parse-time though.

```elixir
{
    create: [
        {
            operation: "cast",
            field_names: [
                "title",
                "body"
            ]
        },
        {
            operation: "validate_required",
            field_names: [
                "title",
                "body"
            ]
        }
    ],
    update: [..],
    import: [..]
    # import Would not have validate_required for nullable fields for example
}
```

# Ecto Entity Library API examples

For field names, source name and types you can use strings or atoms interchangably. For options we will likely use Keywords so they need to be atoms. They will all be strings in the definition.

## Create and update an Entity type

```elixir
alias EctoEntity.Type

# Create new one with initial migrations
definition = "posts"
|> Type.new(name: "Post", singular: "post", plural: "posts")
|> Type.migration_set(fn set ->
    set
    |> Type.add_field("title", "string", "text", required: true, null: false, unique: true, indexed: true, meta: %{..})
    |> Type.add_field("body", "string", "text", null: true)
    |> Type.add_timestamps()  # Add updated_at and created_at
end)

# Alter things, without a migration_set every migration operation will be
# a new migration set which is fine for single-change operations
definition = definition
|> Type.alter_field("body", required: true)

# Bad alter, will raise error due to attempting to set new constraint
definition = definition
|> Type.alter_field("body", null: false)

```

## Use a type to support list/get/create/update/delete data

Ecto has this weird thing where it only allows building the shape of a query at compile time (excludes the possibility to create SQL injection holes) when using Ecto.Query. This works great when coding against this library with known field names and stuff. For our library though we will likely need to use the escape hatch of [raw queries](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.html#query/4) to some extent to provide filtering.

```elixir
alias EctoEntity.Entity
alias EctoEntity.Type
import Ecto.Query
alias MyApp.Repo

definition = %Type{..our definition, loaded from somewhere..}

# Listing
# Use the Type struct as an Ecto.Queryable, it needs to implement that protocol
# This does not express opinions about the storage beyond the "source" name, that is, the table name, which is pulled from the Type's type field.

definition
|> from()  # normal Ecto.Query.from
|> Repo.all()

# get, uses Ecto.Queryable
definition
|> Repo.get(my_id)

# get
Entity.get(definition, my_item_id)

# changeset for creating
definition
|> Entity.changeset_create(%{"title" => "foo", "body" => "bar"})
|> Repo.insert()

# changeset for updating
definition
|> Entity.get(my_item_id)
|> Entity.changeset_update(%{"title" => "new-title", "body" => "new-body"})
|> Repo.update()

```

## Ecto Entity Storage for managing types

```elixir
alias EctoEntity.Type
alias EctoEntityStorage.Store

# The store config includes reference to which
# EctoEntityStorage.Storage backend to use and how it is configured
# It also knows the Repo where the data tables live
config = %{..my store config, not necessarily compile-time config..}

store = Store.init(config)

definition = <generate a definition as described previously>

# Load a stored definition
store
|> Store.get_type("posts")

# Save a definition, the key/name is pulled from the definition type key
# Triggers event {"entity:created", "posts"} for all subscribers
# OR
# Triggers event {"entity:updated", "posts"} for all subscribers
store
|> Store.put_type(definition)

# Get migration status from Storage backend compared to definition
store
|> Store.migration_status(definition)

# Apply any missing migrations for a definition
store
|> Store.migrate(definition)

# Remove type data
store
|> Store.remove_all_data(definition)

# Remove type definition
store
|> Store.remove_type(definition)

OR

store
|> Store.remove_type("posts")

```

## Ecto Entity Storage for managing entities

Building on top of EctoEntity we can create some more involved abstractions in EctoEntityStorage with powerful instrumentation for making an extensible CMS without tying the events and hooks to the UI. If you want a CLI-based CMS this would be a very useful foundation.

```elixir
alias EctoEntityStorage.Store
alias EctoEntity.Entity
alias MyApp.Repo

config = %{..my store config, not necessarily compile-time config..}

store = Store.init(config)
# We might want to allow people to set up a module with a use macro
# that creates a MyApp.Store module that compiles in the config
# so one can use to similar to how Repo is commonly used
# I just don't want to make that the default by design
# as dynamic repos are quite a bit less convenient in Ecto

# As a definition is loaded from a store it remembers where it is from and
# how it is configured to be stored (what Repo to use), this all goes under
# the meta.store key
definition = %Type{meta: %{store: ^store, ..} = Store.get_type(store, "posts")

# Listing
# calls any registered before_store_all function before run to modify query
# calls any registered after_store_all function after run to modify returned
# triggers {"entity:posts", :list, filters} event message to subscribers
# result

definition
|> Store.list()

# Filtering with list
# Requires the use of raw queries from Ecto because of runtime field names

definition
|> Store.list(
    where: %{"published" => true},
    order_by: [{"title", :desc}],
    limit: 100,
    offset: 200
)

my_id = 5
# get
# calls any registered before_store_get function before run to modify query
# calls any registered after_store_get function to modify result
# triggers {"entity:posts", :get, %{"id" => 5}} event message to subscribers

definition
|> Store.get(my_id)

# create and create!
# calls any registered before_store_create function before run to modify
# calls any registered after_store_create function after to modify result
# calls any registered fail_store_create function after to modify result
# triggers {"entity:posts", :created, %{"id" => 6}} event message if ok
# triggers {"entity:posts", :create_failed, changeset} event message if error

definition
|> Store.create(%{"title" => "foo", "body" => "bar"})

# update and update!
# calls any registered before_store_update function before run to modify
# calls any registered after_store_update function after to modify result
# calls any registered fail_store_update function after to modify result
# triggers {"entity:posts", :updated, %{"id" => 6}} event message to subscribers
# triggers {"entity:posts", :update_failed, changeset} event message if error

definition
|> Store.get(my_id)
|> Store.update(%{"title" => "foo", "body" => "bar"})

```


# Notes for later

Tailwind CSS and Theme:able library styles?
utility classes might cause trouble

UI should restrict filtering to only filter/order on fields that have indexes
