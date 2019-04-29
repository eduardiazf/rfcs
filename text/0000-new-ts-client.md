- Start Date: 2019-01-16
- RFC PR: (leave this empty)
- Prisma Issue: (leave this empty)

# Summary

# Basic example

# Motivation

# Detailed design

The example code below assumes the following datamodel:

```groovy
model Post {
  id: ID
  title: String
  body: String
  comments: [Comment]
  author: User
}

model Comment {
  id: ID
  text: String
  post: Post
  author: User
}

model User {
  id: ID
  firstName: String
  lastName: String
  email: String
  posts: [Post]
  comments: [Comment]
  friends: [User]
  profile: Profile
}

embed Profile {
  imageUrl: String
  imageSize: String
}
```

## Types

```ts
// NOTE the following types are auto-generated
type Post = {
  id: string
  title: string
  body: string
}

type Comment = {
  id: string
  text: string
}

type User = {
  id: string
  firstName: string
  lastName: string
  email: string
  profile: Profile
}

type Profile = {
  imageUrl: string
  imageSize: number
}
```

## Basic Queries

```ts
// Get single node
const alice: User = await prisma.users.findOne({ email: 'alice@prisma.io' })

// Lookup-by Multi-field indexes
const john: User = await prisma.users.findOne({
  name: { firstName: 'John', lastName: 'Doe' },
})

// Get many nodes
const allUsers: User[] = await prisma.users.findAll({ first: 100 })
const allUsersShortcut: User[] = await prisma.users({ first: 100 })

// Ordering
const usersByEmail = await prisma.users({ orderBy: { email: 'ASC' } })
const usersByEmailAndName = await prisma.users({
  orderBy: [{ email: 'ASC' }, { name: 'DESC' }],
})
const usersByProfile = await prisma.users({
  orderBy: { profile: { imageSize: 'ASC' } },
})

// Where / filtering
await prisma.users({ where: { email: { contains: '@gmail.com' } } })

await prisma.users({ where: { email: { containsInsensitive: '@gmail.com' } } })

// Exists
await prisma.users.findOne({ where: { email: { containsInsensitive: '@gmail.com' } } }).$exists()

// Raw

// Fluent API
const bobsPosts: Post[] = await prisma.users.findOne('bobs-id').posts({ first: 50 })

type DynamicResult1 = (User & {
  posts: (Post & { comments: Comment[] })[]
  friends: User[]
})[]

// Select API
const userWithPostsAndFriends: DynamicResult1 = await prisma.users.findOne({
  where: 'bobs-id',
  select: {
    posts: { select: { comments: true } },
    friends: true,
  },
})
```

## Writing Data

```ts
const newUser: User = await prisma.users.create({ firstName: 'Alice' })

// Updates
const updatedUser: User = await prisma.users.update({
  where: 'bobs-id',
  data: { firstName: 'Alice' },
})

const updatedUserByEmail: User = await prisma.users.update({
  where: { email: 'bob@prisma.io' },
  data: { firstName: 'Alice' },
})

const upsertedUser: User = await prisma.users.upsert({
  where: 'bobs-id',
  update: { firstName: 'Alice' },
  create: { id: '...', firstName: 'Alice' },
})

// NOTE has Fluent API disabled (incl. nested queries)
const deletedUser: User = await prisma.users.delete('bobs-id')
```

## `$withPageInfo`

```ts
// PageInfo
const bobsPostsWithPageInfo: PageInfo<Post> = await prisma.users
  .findOne('bobs-id')
  .posts({ first: 50 })
  .$withPageInfo()

type PageInfo<Data> = {
  data: Data[]
  hasNext: boolean
  hasPrev: boolean
}
```

- Can be applied to every paginable list and stream

## Aggregations

```ts
type DynamicResult2 = (User & { aggregate: { age: { avg: number } } })[]
const dynamicResult2: DynamicResult2 = await prisma.users({
  select: { aggregate: { age: { avg: true } } },
})

type DynamicResult3 = User & {
  posts: (Post & { aggregate: { count: number } })[]
}
const dynamicResult3: DynamicResult3 = await prisma.users.findOne({
  where: 'bobs-id',
  select: { posts: { select: { aggregate: { count: true } } } },
})

const deletedCount: number = await prisma.users.deleteMany()
```

## Optimistic Concurrency Control / Optimistic Offline Lock

```ts
const updatedUserOCC: User = await prisma.users.update({
  where: 'bobs-id',
  if: { version: 12 },
  data: { firstName: 'Alice' },
})

const upsertedUserOCC: User = await prisma.users.upsert({
  where: 'bobs-id',
  if: { version: 12 },
  update: { firstName: 'Alice' },
  create: { id: '...', firstName: 'Alice' },
})

const deletedUserOCC: User = await prisma.users.delete({
  if: { version: 12 },
  where: 'bobs-id',
})

// Ensure that user with name Bob has been created successfully
// If not, roll back the first step
await prisma.batch([
  prisma.createUser({ name: 'Bob' }),
  {
    checkCurrent: [
      {
        User: {
          name: 'Bob',
        },
      },
    ],
  },
])
```

## Group By

```ts
type DynamicResult4 = {
  lastName: string
  records: User[]
  aggregate: { age: { avg: number } }
}
const groupByResult: DynamicResult4 = await prisma.users.groupBy({
  key: 'lastName',
  having: { age: { avgGt: 10 } },
  where: { isActive: true },
  first: 100,
  orderBy: { lastName: 'ASC' },
  select: {
    records: { first: 100 },
    aggregate: { age: { avg: true } },
  },
})

type DynamicResult5 = {
  raw: any
  records: User[]
  aggregate: { age: { avg: number } }
}
const groupByResult2: DynamicResult5 = await prisma.users.groupBy({
  raw: { key: 'firstName || lastName', having: 'AVG(age) > 50' },
  select: {
    records: { $first: 100 },
    aggregate: { age: { avg: true } },
  },
})
```

## `raw` fallbacks

```ts
await prisma.users({
  where: { email: { contains: '@gmail.com' } },
  orderBy: {
    $raw: 'age + postsViewCount DESC',
  },
})

const someEmail = 'bob@prisma.io'
await prisma.users({
  orderBy: {
    $raw: 'age + postsViewCount DESC',
  },
  where: {
    $raw: ['email = $1', someEmail],
  },
})

// Raw: Knex & Prisma
const userWithPostsAndFriends1 = await prisma.users.findOne({
  where: knex.whereBuilderInSelecet(
    knex.fields.User.name,
    knex.queryMany.Post({ title: 'Alice' }, kx.fields.Post.title),
  ),
  select: knex.select('*').from('User'),
})

// Raw: SQL & Prisma
const userWithPostsAndFriends2 = await prisma.users.findOne({
  where: {
    $raw: 'User.name != "n/a"',
  },
  select: {
    $raw: {
      name: {
        query: 'User.firstName + User.lastName; DROP TABLE',
        type: 'string',
      },
      hobbies: {
        topLevelQuery: 'SELECT * from Hobbies where User.id = $id',
        type: {
          name: 'Hobby',
          fields: {
            id: {
              type: 'string',
            },
            name: {
              type: 'string',
            },
          },
        },
      },
    },
  },
})
```

## `$exec`

```ts
const usersQueryWithTimeout = await prisma.users.$exec({ timeout: 1000 })
```

## Batching

```ts
// Batching, don't get the results with $noData
const m1 = prisma.users.create({ firstName: 'Alice', $noData: true })
const m2 = prisma.posts.create({ title: 'Hello world', $noData: true })
const [u1, p1]: [User, Post] = await prisma.batch([m1, m2])

// Batching with "check consistent" or "check current"
const [u2, p2]: [User, Post] = await prisma.batch([
  m1,
  {
    checkCurrent: [
      {
        User: {
          id: 'bobs-id',
          name: 'Bob',
        },
      },
    ],
  },
  m2,
])

// Batching with transaction
await prisma.batch([m1, m2], { transaction: true })
```

## Top level query API

```ts
const nestedResult = await prisma.query({
  users: {
    first: 100,
    select: {
      posts: { select: { comments: true } },
      friends: true,
    },
  },
})
```

## Pagination / Streaming

```ts
for await (const post of prisma.posts().$stream()) {
  console.log(post)
}

const postStreamWithPageInfo = await prisma
  .posts()
  .$stream()
  .$withPageInfo()

for await (const posts of prisma.users
  .findOne('bobs-id')
  .posts({ first: 50 })
  .batch({ batchSize: 100 })) {
  console.log(posts) // 100 posts
}

// Configure streaming chunkSize and fetchThreshold
prisma.posts({ first: 10000 }).$stream({ chunkSize: 100, fetchThreshold: 0.5 /*, tailable: true*/ })

// Buffering
const posts = await prisma
  .posts({ first: 100000 })
  .$stream()
  .toArray()

// Shortcut for count
const userCount = await prisma.users.count({
  where: {
    age: {
      gt: 18,
    },
  },
})
```

## Life-cycle hooks

### Middleware (blocking)

```ts
function beforeUserCreate(user: UserCreateProps): UserCreateProps {
  return {
    ...user,
    email: user.email.toLowerCase(),
  }
}

type UserCreateProps = { name: string }
type UserCreateCallback = (userStuff: UserCreateProps) => Promiselike<UserCreateProps>

const beforeUserCreateCallback: UserCreateCallback = user => ({
  name: 'Tim',
})

function afterUserCreate(user) {
  datadog.log(`User Created ${JSON.stringify(user)}`)
}

const prisma = new Prisma({
  middlewares: { beforeUserCreate, afterUserCreate },
})
```

### Events (non-blocking)

```ts
const prisma = new Prisma()
prisma.on('User:beforeCreate', user => {
  stripe.createUser(user)
})
```

## Error Handling

If any error should occur, Prisma client will throw. The resulting error instance will have a `.code` property.
You can find the possible error codes that we have in Prisma 1 [here](https://github.com/prisma/prisma/blob/master/server/connectors/api-connector/src/main/scala/com/prisma/api/schema/Errors.scala)

### Where

```ts
prisma.users.deleteMany('id')
prisma.users.deleteMany(['id1', 'id2'])

prisma.users({
  where: {
    id: ['id1', 'id2'], // instead of `_in` or `OR`
    email: { endsWith: '@gmail.com' },
  },
})

prisma.users({
  where: {
    name: { contains: 'Bob' },
    email: { contains: ['prisma.io', 'gmail.com'] }, // instead of `_in` or `OR`
  },
})
```

# Drawbacks

# Alternatives

- `$nested` API

# Adoption strategy

# How we teach this

# Unresolved questions

- [ ] Connection management when used with embedded query engine

# Future topics

- [ ] Add support for revisioning API

- [ ] Type mapping and static field preselection (see [comment in #4](https://github.com/prisma/rfcs/pull/4#issuecomment-471202364))
- [ ] Rails-like scopes (see [Sequelize](http://docs.sequelizejs.com/manual/tutorial/scopes.html))

- [ ] Datomic-style API

* [ ] Non-CRUD API operations
* [ ] Real-time API (subscriptions/live queries)
* [ ] Operation Expressions
  - [ ] API for atomic operations
  - [ ] Update(many) API to use existing values
* [ ] Silent mutations [prisma/prisma#4075](https://github.com/prisma/prisma/issues/4075)
