- Start Date: 2019-04-29
- RFC PR: (leave this empty)
- Prisma Issue: (leave this empty)

# Summary

# Basic example

# Motivation

# Detailed design

## Point in time

```ts
await prisma.users.findOne({
  where: { id: 5 },
  when: { version: 3 },
})

await prisma.users({
  when: '2019-04-26T15:34:39.545Z',
})
```

## History

```ts
const userWithHistory = await prisma.users.findOne({
  where: { id: 5 },
  select: {
    $history: {
      id: true,
    },
  },
})

// or chaining

interface User {
  id: string
  version: string
  updatedAt: string
}

const history: User[] = await prisma.users
  .findOne({
    where: { id: 5 },
  })
  .$history({ orderBy: { version: 'DESC' } })
```

# Drawbacks

# Alternatives

# Adoption strategy

# How we teach this

# Unresolved questions
