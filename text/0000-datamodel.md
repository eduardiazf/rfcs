- Start Date: 2019-03-23
- RFC PR: (leave this empty)
- Prisma Issue: (leave this empty)

# Motivation

This RFC proposes a new syntax for the Prisma Datamodel. Main focus areas:

- Break from the existing GraphQL SDL syntax where it makes sense
- Clearly separate responsibilities into two categories: Core Prisma primitives and Connector specific primitives

## Basic example

This example illustrate many aspects of the proposed syntax:

```groovy
@db(name: "user")
model User {
  id: ID @id
  createdAt: DateTime @createdAt
  email: String @unique
  name: String?
  role: Role = USER
  posts: [Post]
  profile: Profile? @relation(link: INLINE)
}

enum Role {
  USER
  ADMIN
}

@db(name: "profile")
model Profile {
  id: ID @id
  user: User
  bio: String
}

@db(name: "post")
model Post {
  id: ID @id
  createdAt: DateTime @createdAt
  updatedAt: DateTime @updatedAt
  title: String
  author: User
  published: Boolean = false
  categories: [Category]? @relation(link: TABLE, name: "PostToCategory")
}

@db(name: "category")
model Category {
  id: ID @id
  name: String
  posts: [Post] @relation(name: "PostToCategory")
}

@db(name: "post_to_category")
@linkTable
model PostToCategory {
  post: Post
  category: Category
}
```

## Motivation for core/connector split

Prisma core provides a set of data modelling primitives that are supported by most connectors.

Connectors enable Prisma to work with a specific datastore. This can be a database, a file format or even a web service.

- The primary job of a connector is to translate highher level Prisma concepts to lower level operations on the storage engine.
- Secondary, connectors enable the Prisma user to take full advantage of the underlying storage engine by exposing performance tuning options and extra functionality that can not be accessed through the core Prisma primitives.

**Core Prisma primitives**

Prisma provides a set of core primitives:

- Types (String, Int, Float etc)
- Type Declarations (model, embedded, enum)
- Type Alias (`type MyInt = Int`)
- Relations
- Generators
- Constraints
- ID / Primary Key

**Connector specific primitives**

Connectors can additionally provide primitives specific to their underlying datastore

- Type Specification
- Custom Primitive Types
- Custom Complex Types
- Connector Specific Constraints
- Indexes
- Connector specific generators

The Prisma Datamodel provides primitives that describe the structure of your databases. Core Primitives are so fundamental that they map to most database types. Some primitives are there only to express some special feature in a single database. We call them Connector specific primitives. The following sections describe why each primitive is either core or connector specific.

### Types (String, Int, Float etc)

Prisma specifies a common set of primitive types. Connectors have some flexibility in how they implement the type, but there are minimum requirements that must be satisfied.

This makes it possible to use diferent connectors in different environments.

### Type Declarations (model, embedded, enum)

`model` declares a top level type that can be referenced. `embedded` declares a complex structure that is embedded in a top level type. `enum` declares a set of possible values.

Prisma includes these 3 type declarations. It is not possible for a conenctor to introduce a new type declaration.

### Type Alias (`type MyInt = Int`)

The concept of a type alias is provided by Prisma core. Connectors can use type alias to introduce custom types

### Relations

The concept of relations is provided by Prisma core. Relations between models is fundamental to databases and web services dealing with data. Much of Prismas value proposition is around making working with data relations easier (nested mutations, relational filters, sub-selection), so requiring connectors to adhere to a specific style of relations is worth it.

### Generators

Prisma core provides a set of connector agnostic generators. Additionally, conenctors can provide generators that take advantage of database specific capabilities such as Sequences in Postgres. 

### Constraints

Constraints such as uniqueness or value ranges are a data modelling concern. As such, they are provided by Prisma core. Most connectors implement all the core constraints, making it easy to use different conenctors in different environments.

Constraints that are bound to the data record (for example `age > 18`) are implemented in Prisma Core. Constraints that must access other records (for example uniqueness) must be implemented by the connectors as they can take advantage of lower level primitives such as indexes and database triggers.

### ID / Primary Key

Prismas relations rely on a primary key to uniquely identify a single record. Therefore, primary keys are considered a data modelling concern and provided by Prisma core.

### Generators

Prisma has a number of built in generators that produce one of the built in primitive types without knowledge of the underlying datastore. Examples include `now()`, `cuid()` and any literal value such as `42`, `"answer"` or `[1, 2, 3]`

Connectors can provide additional generators that take advantage of low level storage engine primitives such as `SEQUENCE` in a relational database.

### Type Specification

Connectors can provide extra hooks to configure how the underlying storage engine treats generic Prisma types. For example, the MySQL connector enables you to specify that a `String` field is stored in a `varchar(100)` column.

Storage engines vary wildly, so it is not possible to have a generic interface for deciding low level configuration.

### Custom Primitive Types and Custom Complex Types

Connectors can introduce primitive or complex types. These types can be used the same way as a type alias or complex type (model, embedded or enum) declared directly in the datamodel file.

### Indexes

Indexes are storage engine specific and mostly relevant for performance configuration. Therefore, index configuration is provided by connectors.

There are two exceptions where indexes intersect with data modelling:

- Most storage engines implement the _unique constraint_ as an index. Unique constraint is provided by Prisma core, and the connector can choose to create only a single index if both a index and unique constraint is present on a single field in the datamodel.
- The concept of a _primary key_ is provided by Prisma Core (`@id`). Many storage engines implement the primary key using a special index (sometimes called clustered index or primary index) that organises the data on disk by that field, even if no index is specified separately. These connectors will allow you to configure the index used for the primary key separately using the connector specific index configuration.

# Detailed design

## Model

Use the keyword `model` instead of `type`

```groovy
type User {
  name: String!
}
```

```groovy
model User {
  name: string
}
```

## Required and optional fields

Default to required and use the modifier `?` for optional fields instead of defaulting to optional and using the modifier `!` for required fields.

```groovy
type User {
  requiredString: String!
  optionalString: String
}
```

```groovy
model User {
  requiredString: String
  optionalString: String?
}
```

## Embedded

Use the keyword `embedded` instead of the directive `@embedded`

```groovy
type UserSettings @embedded {
  receiveNewsletter: Boolean!
}
```

```groovy
embedded UserSettings {
  receiveNewsletter: Boolean
}
```

Embbeded models can also be defined inline:

```groovy
model User {
  settings: UserSettings {
    receiveNewsletter: Boolean
  }
}
```

> TODO:
>
> - must inlined embeddeds be named?
> - can named inlined embeddeds be used as normal embeddeds in otherr model definitions?

## Primitive types

Prisma defines a set of core types that are awailable accross multiple conenctors. The primary purpose of primitive types is to make it easier to use a single datamodel with different connectors (For example Postgres in production and SQLlite for local testing.) Most connectors will map all primitive types directly to a type supported by the underlying storage engine, enabling fast queries and indexing.

> We considered making primitive types lowercase, but don't find this distinction useful

```groovy
model Primitives {
  string: String
  optionalString: String?
  int: Int
  float: Float
  boolean: Boolean
  datetime: DateTime
  enum: SomeEnum
  json: Json
  id: Id
}

enum SomeEnum {
  SomeEnumValue
}
```

## New primitive types

Prisma 1.x lacks some primitive types. Other types are mapping to the wrong storage engine type.

See https://github.com/prisma/prisma/issues/1753

### Float

Our `float` primitive type is implemented as `NUMERIC(precision = 65, scale = 30)` ([doc](https://github.com/prisma/prisma/issues/2934#issuecomment-451545681)). There is no way to actually get a float.

We will change float to use the largest available floating point primitive type:

| GraphQL | MySQL             | Elastic Search                              | MongoDB    | PostgreS           |
| :------ | ----------------- | ------------------------------------------- | ---------- | ------------------ |
| Float   | FLOAT, **DOUBLE** | float, **double**, half_float, scaled_float | **Double** | float4, **float8** |

Most storage engines will provide a double/float8. It is possible to use type specification to change to float/float4.

### Decimal

The decimal type should map to a predefined configuration of the exact precision type. We have used `NUMERIC(precision = 65, scale = 30)` since the beginning of Graphcool and has never received a complaint, so we will use that as the default on SQL storage engines. It is possible to use type specification to change this.

Elastic seems uninterested in supporting Decimal (BigDecimal): https://github.com/elastic/elasticsearch/issues/17006

Mongo supports decimal (128-bit decimal floating point, not configurable)

### String

Prisma has only one `String` type that maps to the largest available text representation supported by the storage engine. We make no effort to unify size constraints across connectors.

Type specification can be used to specify a smaller storage type for performance. On SQL it is common to use `varchar(128)`.

### Binary

> Task:
>
> Valid oprations and filters need to be specified

| GraphQL | MySQL                   | Elastic Search | MongoDB | PostgreS |
| :------ | ----------------------- | -------------- | ------- | -------- |
| Binary  | Binary, VarBinary, Blob | Binary         | binData | bytea    |

In practice, a binary type is a string type without collation.

### JSON

> Note: SQL connectors will implement Embedded types using JSON columns. Embedded types are different from JSON fields in that they have a schema that is enforced by Prisma on write

JSON is treated as a schema-less JSON value. Prisma validates that inserted values are well-formed JSON.

> TASK:
>
> We should support generic JSON manipulation, ideally similar to the Mongo API. It should work the same across all connectors.
> We should support indexing
> Consider how explicit null vs not even in the document is handled

| GraphQL | MySQL | Elastic Search | MongoDB | PostgreS |
| :------ | ----- | -------------- | ------- | -------- |
| JSON    | JSON  | Object         | Object  | JSON     |

### Datetime Types

Prisma date and time types are always following ISO 8601. DateTimes are always stored with timezones.

Prisma support the 3 DateTime types: `DateTime`, `Date`, `Time`

| Prisma              | MySQL     | Elastic | Mongo | Postgres  |
| ------------------- | --------- | ------- | ----- | --------- |
| DateTime            | TIMESTAMP | -       | Date  | TIMESTAMP |
| Date                | DATE      | -       | -     | DATE      |
| Time                | TIME      | -       | -     | TIME      |
| Interval (from-to)  | -         | -       | -     | -         |
| Duration (timespan) | -         | -       | -     | INTERVAL  |

Elastic does not support `DataTime`. Instead DateTime is stored as a long number representing milliseconds-since-the-epoch. DateTimes are returned by Elastic as rendered strings. Prisma will convert them to be consistent with other connectors.

Mongo and Elastic does not natively support `Date`. Prisma will simply map to DateTime and set the time component to 0.

Mongo and Elastic does not support `Time`. Prisma will simply map to Int and store a millisecond offset from midnight.

> Note: While Interval and Duration might be useful, Prisma does not specify these and individual connectors are free to implement this as needed

### Spatial Types

Sptial data types only make sense if they are augmented with proper operations, like intersection tests or area calculation. PostGIS has some [nice documentation](https://postgis.net/docs/manual-2.5/using_postgis_dbmanagement.html#PostGIS_GeographyVSGeometry) which can serve as a starting point.

For spatial types, two conventions are meaningful:

- Geographic coordinates (lat, lon)
- Geometric coordinates (x, y)

| Prisma  | MySQL      | Elastic Search      | MongoDB    | PostgreS   |
| :------ | ---------- | ------------------- | ---------- | ---------- |
| Point   | POINT      | geo_shape/geo_point | Point      | POINT      |
| Line    | LINESTRING | geo_shape           | LineString | LINESTRING |
| Polygon | POLYGON    | geo_shape           | Polygon    | POLYGON    |

> Task:
>
> Decide what spatial primitive types Prisma should support
>
> Valid operations and filters need to be specified

## Enum

declaring and using an enum:

```groovy
model Primitives {
  enum: SomeEnum
}

enum SomeEnum {
  SomeEnumValue
  SomeOtherEnumValue
}
```

The following table specifies how connectors will implement enums. Note that Prisma 1.x implements enums as a string, even when a dedicated ENUM type is available.

| Prisma | MySQL | Elastic Search | MongoDB | PostgreS |
| :----- | ----- | -------------- | ------- | -------- |
| Enum   | ENUM  | text           | String  | ENUM     |

### Ordered Enum values

Connectors for databases without native support for enums will store enums as strings containing the name of the enum value. In the future we could add a feature to specify an int representing the enum value similar to how protobuf specifies the order of fields. This minimises space use and simplifies renaming of values. This new feature will be backwards compatible:

```groovy
enum SomeEnum {
  SomeEnumValue = 1
  SomeOtherEnumValue = 2
}
```

## Type specification

Prismas primitive types are implemented by all connectors. As such, they are often too coarce to express the full power of a connectors type system. It is possible to specify the exact type of a storage engine using type specification.
A type specification is always scoped to a specific connector. If the datamodel is used with any other connector, it is ignored. It is possible to provide type specification for multiple different connectors in a single datamodel.

```groovy
# Traditional SDL
model User {
  age: Int @postgres(type: "smallint")
  name: String @postgres(type: "varchar(128)")
  id: ID @postgres(type: "char(100)")
  height: Float @postgres(type: "float4")
  cashBalance: Decimal @postgres(type: "numeric(precision = 30, scale = 60)")
  props: Json @postgres(type: "mediumtext")
}

# namespaced and unnamed argument
model User {
  age: Int @postgres.type("smallint")
  name: String @postgres.type("varchar(128)")
  id: ID @postgres.type("char(100)")
  height: Float @postgres.type("float4")
  cashBalance: Decimal @postgres.type("numeric(precision = 30, scale = 60)")
  props: Json @postgres.type("mediumtext")
}

# not a string
model User {
  age: Int @postgres.type(smallint)
  name: String @postgres.type(varchar(128))
  id: ID @postgres.type(char(100))
  height: Float @postgres.type(float4)
  cashBalance: Decimal @postgres.type(numeric(precision = 30, scale = 60))
  props: Json @postgres.type(mediumtext)
}

# without .type
model User {
  age: Int @postgres.smallint
  name: String @postgres.varchar(128)
  id: ID @postgres.char(100)
  height: Float @postgres.float4
  cashBalance: Decimal @postgres.numeric(precision = 30, scale = 60)
  props: Json @postgres.mediumtext
}
```

Type specifications for multiple connectors:

```groovy
model User {
  age: Int @postgres.type(smallint) @mysql.type(smallint)
}
```

> TODO:
>
> - decide on one of the syntax proposals above

## Custom primitive types

There are two distinct uses for custom primitive types. Users of Prisma can create a custom type to encapsulate constraints or other configuration in a reusable type. Implementers of connectors can declare primitive types that work only in the context of that connector.

### User-defined primitive types

If you have a certain field configuration that is used in multiple places, it can be convenient to create a custom type instead of repeating the configuration. This also ensures that all uses are in sync.

```groovy
type Email = String @constraint(regex: ".*.com")

# Without custom type
model User {
  email: String @constraint(regex: ".*.com")
}

# With custom type
model User {
  email: Email
}

# With additional field config
model User {
  email: Email @postgres.type(varchar(250))
}
```

> A user-defined primitive type is a collection of field configurations that can be extended at place of use.

### Connector-defined primitive types

When implementing a connector it might be necessary to augment Prisma with types that are not already part of the muilt in primitive types. For example, a legacy database might have a special string type that support emoji, but does not support indexing. The connector could introduce a new primitive type to expose this type:

```groovy
type EmojiString = String
```

Prisma users can then use it in their datamodel like this:

```groovy
model User {
  displayName: EmojiString
}
```

Connector implementors can rely on Prisma for certain validations if they don't need custom error messages. Any extra field configuration will work exactly the same as if the Prisma user added it in their datamodel. In this example, if a EmojiString is longer than 1000 characters, Prisma will reject it with a standard error message without calling the connector:

```groovy
type EmojiString = String @constraint(maxLength: 1000)
```

### Connector-defined complex types

A conenctor might also want to introduce a complex type. For example a custom connector for a legacy SOAP API could introduce a complex type that is transparently mapped to a bitmap:

```groovy
type UserSettingBitmap = model @embeded {
	sendEmail: Boolean
	showVideos: Boolean
}
```

> Task:
>
> This needs to be mapped out in much greater detail

## Directives

### Directive List

[Datamodel v1.1](https://github.com/prisma/prisma/issues/3408) specifies the following directives. Some of them might become obsolete with the changes proposed in this document

#### Type Level

1. `@db` - to map fields or types to underlying db objects with different names. **@db is used only for this purpose. We would like to find a broader construct. Maybe it is connector specific: `@postgres(column: "myName")`. This would make datamodels that rely on renaming less portable, but it is a edge case feature anyway**
2. `@plural` - to force a certain plural for client schema generation. **We are moving this to client generator configuration**
3. `@linkTable` - to mark a table as intermediate table for relations. **This is superseded by explicit join model**
4. `@embedded` - to mark a type as embedded, e.g. embedded into a field of another type when stored. **This is superseded by the embedded keyword**
5. `@indexes` - to declare indexes on a type. **This is moved out of Prisma Core and is a responsibility of connectors**
6. `@discriminator` - to declare a discriminator on a polymorphic type, e.g. a value to distinguish it from other types. **TODO: do we need this?**

#### Field Level

1. `@id` - marks the primary key/id **We keep this**
2. `@createdAt` - marks a field as the special createdAt field. **This is superseded by the generator concept**
3. `@updatedAt` - marks a field as the special updatedAt field. **This feature is being sunset**
4. `@default` - sets the default value of a field. **This is superseded by the generator concept**
5. `@db` - see above
6. `@scalarList` **TODO: We should probably have a default behavior. Connectors can then optionally allow to customise implementation strategies**
7. `@constraint` - for single field db constraints. **We will keep this**
8. `@sequence` **This is superseeded by the generator concept**
9. `@immutable` **TODO: Research if we still need this**
10. `@relation` **We keep this**
11. `@unique` - for single-field unique constraints. **TODO: We probably keep this**

#### Interface Level

1. `@inheritance`

### Placement

SDL specifies that a directive must always be placed after the element it describes. We have the option to change this

```groovy
model User @plural("users") {
  name: string
}
```

```groovy
@plural("users")
model User {
  name: string
}
```

> Task:
>
> 1. Map out all directives, and where it would be most intuitive to place it.
>
> 2. Consider if some directives would be better described with alternative syntax

## Generators and default values

Default values can be specified as follows:

```groovy
model User {
  name: String = "Default name"
}
```

When required fields have a default value, they become optional in creates.

### Literal generators

The `"Default name"` part on the right side above is called a literal generator. It is a generator that always returns the same literal value. Literal generators are supported for all core primitive types:

| Primitive type | Literal generator |
| -------------- | ----------------- |
| Int            | 1                 |
| Float          | 1.1               |
| Decimal        | 1.1               |
| String         | "some text"       |
| boolean        | true              |
| datetime       | "2018"            |
| enum           | SomeEnum          |
| json           | '{"a":3}'         |

> TODO:
> - extract literals to separate section
> - spec out literals for remaining primitive types such as binary and spatial

### Dynamic generators

Prisma core provides a set of dynamic generators that can be used as default values:

- `uuid()` - generates a fresh UUID
- `cuid()` - generates a fresh cuid
- `randomInt(min, max)` - generates a random int in the specified range
- `randomFloat(min, max)` - generates a random Float in the specified range
- `now()` - current time

Default values using a dynamic generator can be specified as follows:

```groovy
model User {
  createdAt: DateTime = now()
}
```

### Connector specific generators

Connectors can provide additional generators that rely on specific database primitives to work. For example, MySQL and Postgres connectors could provide the `autoIncrement` generator that uses the underlying `AUTO_INCREMENT` and `SERIAL` column modifiers respectively:

```groovy
model User {
  id: Int = autoIncrement
}
```

`SERIAL` In Postgres is a simplified version of the underlying `Sequence` construct. As such, a Postgress connector could additionally provide a `sequence` generator:

> Note that MariaDB introduced support for `SEQUENCE` in 10.3 in 2018: https://jira.mariadb.org/browse/MDEV-10139

```groovy
sequence MyCustomSequence = Postgres.Sequence(start: 100, increment: 10)

model User {
  id: Int = MyCustomSequence
}
```

> Note: Sequences are selfcontained objects in Postgres that can be referenced from multiple tables. As such, the Postgres connector needs to introduce a new typedeclaration `sequence`. Connectors should take extreme care when introducing a typedeclaration like that as it is not scoped to Postgres. Practical experience implementing the core connectors will show us how well this eorks in practice


### Expression generators

> Note: this section is experimental and might never be implemented in Prisma

Prisma could support expressions in generators:

```groovy
model User {
  name: String
  age: Int
  someRandomField: String = `${this.name} is ${this.age} years old`
  ageInDays: Int = ${this.age * 365}
}
```

## Relations

There are three kinds of relations: `1-1`, `1-m` and `m-n`. In relational databases `1-1` and `1-m` is modeled the same way, and there is no built-in support for `m-n` relations.

Prisma core provides explicit support for all 3 relation types and connectors must ensure that their guarantees are upheld:

- `1-1` The return value on both sides is a nullable single value. Prisma prevents accidentally storing multiple records in the relation. This is an improvement over the standard implementation in relational databases that model 1-1 and 1-m relations the same, relying on application code to uphold this constraint.
- `1-m` The return value on one side is a nullable single value, on the other side a list that might be empty.
- `m-n` The return value on both sides is a list that might be empty. This is an improvement over the standard implementation in relational databases that require the application developer to deal with implementation details such as an intermediate table / join table. In Prisma, each connector will implement this concept in the way that is most efficient on the given storage engine and expose an API that hides the implementation details.

### 1-1

A writer can have exactly 1 blog and a blog must have a single author:

```groovy
model Blog {
  id: ID @id
  author: Writer?
}

model Writer {
  id: ID @id
  blog: Blog?
}
```

Connectors for relational databases will implement this as two tables with a single relation column:

| **Blog** |          |
| -------- | -------- |
| id       | authorId |

| **Writer** |
| ---------- |
| id         |

**Specifying Relation id side**

> TODO: we need to pick a syntax for deciding on what table should have the relation id. Below are some options.

```groovy
model Blog {
  id: ID @id
  author: Writer? @relation(storeId: true)
}

model Writer {
  id: ID @id
  blog: Blog?
}
```

```groovy
model Blog {
  id: ID @id
  author: Writer? @id
}

model Writer {
  id: ID @id
  blog: Blog?
}
```

```groovy
model Blog {
  id: ID @id
  author: Writer?
}

model Writer {
  id: ID @id
  blog: Blog? @relation(virtual: true)
}
```

**Required relations**

Either side in a 1-1 relation can be made required:

```groovy
model Blog {
  id: ID @id
  author: Writer
}

model Writer {
  id: ID @id
  blog: Blog?
}
```

```groovy
model Blog {
  id: ID @id
  author: Writer?
}

model Writer {
  id: ID @id
  blog: Blog
}
```

```groovy
model Blog {
  id: ID @id
  author: Writer
}

model Writer {
  id: ID @id
  blog: Blog
}
```

When you are modeling a parent-child relationship it is comon for the parent to be required and the child to be optional. In this example a Blog might be required to have a Writer while a Writer can exist without a Blog.

**Implicit relation field**

It is possible to specify only one relation field:

```groovy
model Blog {
  id: ID @id
  author: Writer
}

model Writer {
  id: ID @id
}
```

This will be interpreted as if there was an implicit optional relation field on `Writer` named after the `Blog` model:

```groovy
model Blog {
  id: ID @id
  author: Writer
}

model Writer {
  id: ID @id
  blog: Blog?
}
```

### 1-m

A writer can have multiple blogs

```groovy
model Blog {
  id: ID @id
  author: Writer?
}

model Writer {
  id: ID @id
  blogs: [Blog]
}
```

Connectors for relational databases will implement this as two tables with a single relation column, exactly like the 1-1 relation:

| **Blog** |          |
| -------- | -------- |
| id       | authorId |

| **Writer** |
| ---------- |
| id         |

The implementation in the relational database matches the 1-m semantics, and these are reflected in the exposed API.

**Required relations**

The single element side in a 1-m relation can be marked required:

```groovy
model Blog {
  id: ID @id
  author: Writer
}

model Writer {
  id: ID @id
  blog: [Blog]
}
```

The many field will either be an empty list or a non-empty list. There is no distinction between an optional and required many field.

**Implicit relation field**

It is possible to specify only one relation field:

```groovy
model Blog {
  id: ID @id
}

model Writer {
  id: ID @id
  blogs: [Blog]
}
```

This will be interpreted as if there was an implicit optional single item relation field on `Blog` named after the `Writer` model:

```groovy
model Blog {
  id: ID @id
  author: Writer?
}

model Writer {
  id: ID @id
  blogs: [Blog]
}
```

Doing it the other way would not work as it would result in a 1-1 relation.

### m-n

Blogs can have multiple writers

```groovy
model Blog {
  id: ID! @id
  authors: [Writer]
}

model Writer {
  id: ID! @id
  blogs: [Blog]
}
```

Connectors for relational databases will implement this as two data tables and a single join table:

| **Blog** |
| -------- |
| id       |

| **Writer** |
| ---------- |
| id         |

| **\_BlogToWriter** |          |                |
| ------------------ | -------- | -------------- |
| blogId             | writerId | becameWriterOn |

Relations using a join table feel exactly like any other relation. The generated client API is identical to that for the many side of a 1-m relation.

**Required relations**

m-n relations makes no distinction between required or optional

**Implicit relation field**

m-n relations do not support an implicit relation field

### Explicit join model

The `1-1`, `1-m` and `m-n` relations are high-level constructs provided by Prisma and implemented by most connectors. Especially the `m-n` relation construct provided by Prisma selects a set of compromises that might not be appropriate for all cases. You can gain more control over the implementation by using a concept familiar to users of relational databases: The join table. A flexible version of the `m-n` relations can be implemented using an extra model that acts as the join table, and two `1-m` relations.

This data model:

```groovy
model Blog {
  id: ID @id
  authors: [Writer]
}

model Writer {
  id: ID @id
  blogs: [Blog]
}
```

Can be represented like this:

```groovy
model Blog {
  id: ID @id
  authors: [_BlogToWriter]
}

model Writer {
  id: ID @id
  blogs: [_BlogToWriter]
}

model _BlogToWriter {
  author: Author
  blog: Blog
}
```

These Datamodels will map to the same database schema

| **Blog** |
| -------- |
| id       |

| **Writer** |
| ---------- |
| id         |

| **\_BlogToWriter** |          |
| ------------------ | -------- |
| blogId             | writerId |

But the generated client will be different in the following ways:

- There will be a new top-level model `_BlogToWriter`, which can be accessed in all the normal ways, including a query like this: `prisma._BlogToWriters.findAll()`
- Nested mutations will include an extra level of nesting: `prisma.writers.create({ id: 'a', blogs: { create: [{ blog: { id: "b" } }] } })`
- Following a relation will require traversing an extra step: `prisma.writers.findAll().blogs().blog()`
- Filtering by a related value will require an extra step: `prisma.writers.findAll({ where: { blogs_all: { blog: { id_ne: "abba" }}}})`

**Extra data on the join model**

When the `m-n` relation is modeled with a explicit join model, it is possible to add extra fields on that model:

```groovy
model Blog {
  id: ID @id
  authors: [_BlogToWriter]
}

model Writer {
  id: ID @id
  blogs: [_BlogToWriter]
}

model _BlogToWriter {
  becameWriterOn: DateTime
  author: Author
  blog: Blog
}
```

| **Blog** |
| -------- |
| id       |

| **Writer** |
| ---------- |
| id         |

| **\_BlogToWriter** |          |                |
| ------------------ | -------- | -------------- |
| blogId             | writerId | becameWriterOn |

The extra field can be accessed as you would expect:

- On the top-level model `_BlogToWriter`: `(await prisma._BlogToWriters.findAll())[0].becameWriterOn`
- In nested mutations: `prisma.writers.create({ id: 'a', blogs: { create: [{ blog: { id: "b" }, becameWriterOn: "2018" }] } })`
- In relation filters: `prisma.writers.findAll({ where: { blogs_all: { blog: { id_ne: "abba" }, becameWriterOn_gt: "2017"}}})`

### Ambiguous Relations

If there are more than one relation between two types, the relation must be named:

```groovy
model Blog {
  id: ID @id
  author: Writer @relation(name: "blogAuthor")
  sunscribers: [Writer] @relation(name: "blogSubscribers")
}

model Writer {
  id: ID @id
  authorOf: [Blog] @relation(name: "blogAuthor")
  subscribedTo: [Blog] @relation(name: "blogSubscribers")

}
```

### Cascading Deletes

With Cascading Deletes you can ensure that related data is automatically cleaned up when a record is deleted. In general, when there is a parent-child relationship, Cascading Deletes can be used to automatically delete the child.

There are 3 options:

- `CASCADE`: Delete all child records
- `RESTRICT`: Prevent deleting a parent record when there are child records
- `SET_NULL` (default): Break the relation, but leave child records alone

**1-1**

Cascading can be enabled on either side, but not both:

```groovy
model Blog {
  id: ID @id
  author: Writer @relation(onDelete: CASCADE)
}

model Writer {
  id: ID @id
  blog: Blog?
}
```

```groovy
model Blog {
  id: ID @id
  author: Writer
}

model Writer {
  id: ID @id
  blog: Blog? @relation(onDelete: CASCADE)
}
```

This would return an error:

```groovy
model Blog {
  id: ID @id
  author: Writer @relation(onDelete: CASCADE)
}

model Writer {
  id: ID @id
  blog: Blog? @relation(onDelete: CASCADE)
}
```

**1-m**

Cascading can be enabled on the parent side only:

```groovy
model Blog {
  id: ID @id
  author: Writer
}

model Writer {
  id: ID @id
  blog: Blog? @relation(onDelete: CASCADE)
}
```

This would return an error:

```groovy
model Blog {
  id: ID @id
  author: Writer @relation(onDelete: CASCADE)
}

model Writer {
  id: ID @id
  blog: Blog?
}
```

**m-n**

In a `m-n` relation there is no natural parent-child relationship. Therefore, cascading deletes are not supported.

### Edge Relation

> Note: The Edge Relation part of the spec is additive and will not be part of the initial release of Prisma 2
> Note: This is only a draft

The edge relation concept comes from property graphs, implemented in systems such as Neo4j and ArangoDB. An edge connects two nodes in the graph and can have extra properties. The Edge Relation extends the Prisma API to handle this concept without the use of a full model to represent the edge.

**Implementation in wire protocol**

```groovy
# Inserting relation data

mutation createWriter(data: {
  id: "a"
  blogs: { create: { id: "b" } relationData: { becameWriterOn: "${now()}" }}
})

# Reading relation data

writers {
  blogsConnection {
    node { id }
    relationData { becameWriterOn }
  }
}

# Filtering by relation data

writers(where: { blogs_all: { id_ne: "abba" _relation_becameWriterOn_gt: "2018" } })
```

**Implementation in TS client**

```typescript
// Inserting relation data

prisma.writers.create({ id: 'a', blogs: { create: [{ id: "b", _relationData: { becameWriterOn: "${now()}" } }] } })
prisma.writers.create({ id: 'a', blogs: { createWithRelationData: [{ data: { id: "b" }, relation: { becameWriterOn: "${now()}" } }] } })
prisma.writers.create({ id: 'a', blogs: { create: [{ data: { id: "b" }, relation: { becameWriterOn: "${now()}" } }] } }) // can discriminated union types handle this?
prisma.writers.create({ id: 'a'})
              .createBlogs({id: "b"}, )
              .withRelationData({ becameWriterOn: "${now()}" })
              .end()


// Reading relation data

const writersWithBlogsRelationData: WriterWithBlogsIncludingRelationData[] = await prisma.writers // can we avoid this extreme type explosion?
  .findAll()
  .blogsWithRelationData()

// Filtering by relation data

const writers: Writer[] = await prisma.writers
  .findAll({ where: { blogs_all: { id_ne: "abba" _relation_becameWriterOn_gt: "2018" } } })

const writers: Writer[] = await prisma.writers
  .findAll({ where: { blogs_all: { data: {id_ne: "abba" } relation: { becameWriterOn_gt: "2018" } } } }) // assume this is possible with discriminated union
```

### Aggregations

> NOTE: this section is an aside examining the appliccability of the above API design to aggregations
> We will use the same datamodel

```typescript

# Reading aggregate data

// aggregate record data
const writersWithBlogsRelationData: DynamicType[] = await prisma.writers
  .findAll()
  .blogsWithRelationData({select: {data: {$aggregate: { id: { avg: true } } }}})

// aggregate relation data
const writersWithBlogsRelationData: DynamicType[] = await prisma.writers
  .findAll()
  .blogsWithRelationData({select: {relation: {$aggregate: { becameWriterOn: { avg: true } } }}})

// or the same result relying on top level select
const writersWithBlogsRelationData: DynamicType[] = await prisma.writers
  .findAll({select: {id: true, blogs: {id: true, $aggregate: { id: { avg: true } }}}}) // note that we don't need the {data, relation} intermediate type

const writersWithBlogsRelationData: DynamicType[] = await prisma.writers
  .findAll({select: {id: true, blogs: {data: {id: true}, relation: { $aggregate: { becameWriterOn: { avg: true } }}}}}) // The {data, relation} intermediate type is the only way to access relation


# Filtering by aggregation data

const writers: Writer[] = await prisma.writers
  .findAll({ where: { blogs_all: { id_ne: "abba" }, blogs_aggregate: { id_avg_gt: "2018"} } })

const writers: Writer[] = await prisma.writers
  .findAll({ where: { blogs_all: { id_ne: "abba" }, blogs_relation_aggregate: { becameWriterOn_avg_gt: "2018"} } })
```

## Index

SDL limits us to 1 instance of a specific directive, forcing us to add extra syntax:

```groovy
type Post @indexes(value: [
  { fields: ["published"] name: "Post_published_idx" }
]) {
  id: ID @id
  title: String
  published: DateTime
  viewCount: Int
  author: User
}
```

### Alternatives

For MDL, we can express indices via other meachnism:

### Field Groups

Could introduce the concept of column groups, and allow indices on them:

```groovy
type Post {
  id: ID @id
  {
     title: String
     published: DateTime
  } @index // An index.
  viewCount: Int
  author: User
}
```

or easier to read:

```groovy
type Post {
  id: ID @id
  @index {
     title: String
     published: DateTime
  }
  viewCount: Int
  author: User
}
```

Pro:

Con: Counter-Intuitive to read (is that an embedded type?), overlapping indices not possible, need to group fields in index together.

### Index via interface

We could allow index declarations only on interfaces. These indices would be then applied to the inheriting type:

```groovy
indexed interface PostSearchable @index {
  title: String
  published: DateTime
}

type Post extends PostSearchable {
  id: ID @id
  viewCount: Int
  author: User
}
```

Pro: Allows overlapping indices

Con: Pollutes interfaces

### Named Indics

We could make indices named, so we can assign multiple fields to an index

```groovy
type Post extends PostSearchable {
  id: ID @id
  title: String @index(name: 'SearchIndex')
  published: DateTime @index(name: 'SearchIndex')
  viewCount: Int
  author: User
}
```

Pro: Allows overlapping indices

Con: Hard to read/write, especially for large types. Typos in Index name would be critical.

### Index declaration on type

Like before

```groovy
type Post {
  id: ID @id
  title: String
  published: DateTime
  viewCount: Int
  author: User
}
@index (fields: ["title", "published"] name: "Post_published_idx")
```

Or, dropping GraphQL restrictions (e.g. adding typecheck):

```groovy
type Post {
  id: ID @id
  title: String
  published: DateTime
  viewCount: Int
  author: User
}
@index (fields: [.title, .published] name: "Post_published_idx")
```

Pro: Allows overlapping indices, allows typecheck

Con: Declaration not "inside" model

### Special Search Index

Full text/phrase/spatial search in its simplest form can be reduced to providing an index. For this, an optional `type` argument could be added to indices, to represent this indices.

```groovy
type Post {
  id: ID @id
  title: String
  published: DateTime
  text: String
  author: User
}
@index (fields: ["text", "title"] name: "Fuzzy_Text_Index", type: FuzzyFullText)
@index (fields: ["text", "title"] name: "Text_Index", type: FullText)
```

An alterative would be a distinct `textIndex` directive which allows additional tuning params.

```groovy
type Post {
  id: ID @id
  title: String
  published: DateTime
  text: String
  author: User
}
@fuzzyTextIndex(fields: ["text", "title"] name: "Text_Index", weight: [0.4, 0.6])
```

These indices would, when created, add extra filter fields to the schema.

### Proposed Special Indices

| Description                                     | DirectiveName proposals                     | Filter fields                                           |
| ----------------------------------------------- | ------------------------------------------- | ------------------------------------------------------- |
| Spatial Geometry Contains                       | `@spatialIndex`, `@geoContainsIndex`        | `field_contains:Gemoetry`, `field_intersects: Geometry` |
| Full Text Trigram index, for contains queries   | `@fullTextIndex`, `@fullTextContainsIndex`  | `field_contains: String`                                |
| Full Text index with stemming for phrase search | `@fuzzyFullTextIndex`, `@phraseSearchIndex` | `field_matches: String`                                 |

For the fuzzy text index, it would be useful to also expose an order by field which would allow to order by rank of the match.

> Task:
>
> Find further special inidices
> Map out all index tuning settings and create common capability groupings between DBs

## Ineritance

Inheritance in this context describes the concept of sharing common fields between types that are conceptually related. While inheritance is well discussed and researched on a language level, we have to tie these concepts closely to database models and to introspection.

> Polymorphic relations are a powerful concept and should be discussed here seperately.

This concept is not to be confused with polymorphic relations, which is described in the [datamodel v2 specification](https://github.com/prisma/prisma/issues/3407). The polymorphic relation discussion is recommended reading for this topic as well.

Also, inheritance has to be distinguished from **interfaces**. The concept is similar, but interfaces are not backed by the databases, and any model can implement multiple interfaces.

> Tasks:
>
> What about union types?
>
> What are the precise implications on data layout when inheritance is used?
>
> Do we include discriminators or do we rely on native mechanisms, exposed by the client (like instanceof)?

### Inheritance in Prisma

In the concept of prisma **inheritance** allows extending a type that is backed by the database by **inheriting** from it.

Conventional **abstract** types behave like conventional types, but cannot be created. We have to take care of existing data in the database correctly.

Inheritance in prisma respects **all properties** of base fields, including:

- Default Values
- Indices
- Types
- Field Names
- Constraints

Inheritance in the datamodel is declared by an `extends` clause:

```groovy
type LivingBeing {
    dateOfBirth: DateTime @createdAt = now()
}

type Human extends LivingBeing {
    firstName: String
    lastName: String
}

type Pet extends LivingBeing {
    nickname: String
    owner: Human
}

type Cat extends Pet {
    likesFish: Boolean
}

type Dog extends Pet {
    likesFrisbee: Boolean
}
```

In the example above, `Dog` would inherit all fields from `Pet` and `LivingBeing`without explicitly declaring them.

When a prisma query for base type happens, all super types are taken into consideration. In other words, when quering all Pets, all cats and dogs are returned as well.

### Inheritance in Relational DBs

Via **single table inheritance**: We simply have all base field and all fields from superclasses in the same table.

Drawbacks: Impossible to enforce not null, field names collide. A `type` collumn will be mandatory.

Via **concrete table inheritance**: We have a seperate table for each subtype, copying base fields.

Drawbacks: No clear distinction between base and sub fields. It is hard to query all for the base type. Auto incrementing PKs on the base type are hard to achieve.

Via **class table inheritance**/**join table inhertiance**: We create a base table for the base class and specific tables for subtypes, which are joined.

Drawback: Performance (Feedback from Marcus)

### Implementation in Prisma

> This point should be discussion. Marcus mentioned that join table forms can lead to poor performance . Maybe single table is better for prisma - prisma could hide the not null shortcoming in the application layer.

Prisma always uses the **join table form** for relational DBs, as it poses the least drawbacks. Optionally, prisma could offer support for the other inheritance concepts to allow easier adoption of existing databases.

When introspecting, inheritance is never discovered, as there are no hints we could salvage for detecting inheritance. However, a user can always declare an inheritance in an existing datamodel to match the database.

> Marcus pointed out that it might make sense to limit or at least discourage to deep inheritance, since it can lead to poor performance.

> Task:
>
> Map out the MDL syntax for supporting all inheritance types

### Inheritance in Document DBs

On Top Level, Document Databases can theoretically leverage the same approaches as Relational Databases. For embedded types, only **single table inheritance** is really feasible. In the context of document DBs, this means mixing all base and sub types inside the same collection or array. This will require a type tag on each object to function properly.

### Implementation in Prisma

Prisma always stores super and sub types in the same collection, with a type tag.

Introspection does not identify inheritance (in theory, it could with heuristics), but allows a user to declare an existing inheritance relationship in the datamodel. For that, a type tag needs to be added, which can be done using provided tooling.

### Migration Considerations

For any form of inheritance, migrating away from a super/subtype relationship will move (and potentially duplicate!) a lot of data.

Migrating towards a class/subclass relationship is can be a difficult task if it's allowed to create a base types for two types simultaneously because of conflicts. Splitting a single type into super/subtype is less of a problem.

### Client considerations

The prisma client needs to expose a way to distinguish between different subclasses for superclass queries. This can be done in a language-native way or with type tags.

> How does that work with querying?

### Impact on filters

Filters on supertypes also include an is operator to check for a specific subtype. This is needed for relations that point to a supertype.

When querying a specific subtype on top level, the appropriate sub type should be queried directly.

## Constraints

Constraints restrict updating data when the update would violate a certain condition. Constraints can be defined with multiple scopes:

|                        | Single Field | Multi Field |
| ---------------------- | ------------ | ----------- |
| Only Updated Values    | x            | x           |
| All Values in Row      | x            | x           |
| All Values in Table    | x            | x           |
| All Values in Database | x            | x           |

### Single field constraints

The following is an excellent reading on [single field constraints](https://github.com/prisma/prisma/issues/728). Countless extensions, or even using a simple expression language is thinkable.

Example:

```groovy
type Employee {
    salary: Int
    bonus: Int
    firstName: String
    lastName: String
  	email: String @constraint(regex: "(^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$)")
}
```

### Multi field Constraints

For declaring multi field constraints, a similar structure as with indexes (on type level) could be used.

Example:

```groovy
type Employee {
    salary: Int
    bonus: Int
    firstName: String
    lastName: String
  	email: String
}
@constraint(expression: salary < bonus) // Custom constraint
```

### Constraints that respect other values

**Caution**: Constraints that respect other values in tables or the database are generally not supported natively by all databases.

Example:

```
type Employee {
    salary: Int! @constraint(salary < AVG(salary) * 1.5) // Cap salary
    bonus: Int!
    firstName: String!
    lastName: String!
  	email: String!
}
```

> Task:
>
> Map out what it would look like to have reusable named constraints. Maybe custom scalars?

## Interfaces

Interfaces operate similar to inheritance, although interfaces ONLY transfer

- The field name
- The field type
- Field constraints

To a base type.

Other properties (indices, directives, default) cannot be declared on interfaces.

Interfaces are not backed by the database and they do not change the API schema per se. However, they are exposed in the generated client's type system and are also included in the client API. A type can inherit multiple interfaces, as long as single fields are not conflicting. The type still has to explicitly declare all interface fields.

In other words, Interfaces offer a guarantee that a subset of a certain type follows a certain schema.

Interfaces can be declared using the `Interface` keyword and used using the `implements` clause:

```groovy
interface IDatabase {
    storageSize: Int
}

interface IMessageQueue {
    capacity: Int
}

type Kafka implements IMessageQueue {
    capacity: Int
    serverName: String
}

type PostGres implements IDatabase {
    storageSize: Int
    serverName: String
}

type Prisma implements IDatabase, IMessageQueue {
    storageSize: Int
    capacity: Int
}
```

# Drawbacks

# Alternatives

# Adoption strategy

# How we teach this

# Unresolved questions

# Open Questions

- [ ] Support for models spread across multiple datasources ("compound models")
- [ ] Uppercase vs lowercase for (scalar) type names
- [x] New primitive types (JSON, spatial)
- [ ] @updatedAt vs DateTime(behavior: UPDATED_AT)
- [x] embed strategy: JSON, JSONB, multi-columns
- [x] share sequences across types
- [ ] Rethink polymorphic relations (interfaces/unions)
- [x] Rethink inheritance - do we need both inheritance and interfaces?
- [ ] Maybe rethink the "link table" concept
- [ ] Values for enum types?
- [x] Indices/Unique
- [ ] Unnamed embeds
- [ ] If we were a little more radical with the syntax, could we create something much better?
- [ ] How to handle fields with arguments? (also related to connectors and clients)

# Notes

- https://edgedb.com/docs/datamodel

> Some things to think about:
>
> // Top level things
>
> // - sequence
>
> // - view
>
> // - function
