- Start Date: 2019-04-29
- RFC PR: (leave this empty)
- Prisma Issue: (leave this empty)

# Summary

Modifiers are a way to adjust the generated Prisma Client.

# Basic example

# Motivation

- Aliases/Rename
- Computed fields
- Custom Types
- Custom Fields
- Preselection aka Default selection
- Custom queries
- Custom camelcase
- Custom pluralization
- Result decomposition (see [Massive.js](https://massivejs.org/docs/resultset-decomposition))

# Detailed design

## Information Flow

Note, that we an unidirectional information flow. That means the higher levels of abstraction like `Client` won't be able to access lower levels like the `Connectors`.

```
┌───────────┐
│  Client   │
└─────▲─────┘
      │
┌─────┴─────┐
│ Modifiers │
└─────▲─────┘
      │
┌─────┴─────┐
│ Datamodel │
└─────▲─────┘
      │
┌─────┴─────┐
│Connectors │
└───────────┘
```

```ts
import { modifiers } from '@prisma/sdk'

export default modifiers(p => {
  // always fetch the friends for every user
  p.defaultSelection({
    type: 'User',
    select: {
      friends: true,
    },
  })

  // define a custom query and the custom type UserWithFriend, so that you can
  // import it from TypeScript
  p.query({
    name: 'userWithFriend',
    type: 'UserWithFriend',
    query: 'user',
    args: {
      select: {
        friends: true,
        whatever: true,
      },
    },
  })

  // hide specific fields
  p.getModel().forEach(model => {
    model.fields.forEach(field => {
      if (field.name === 'password') {
        p.hideField({
          model,
          field,
        })
      }
    })
  })

  // maybe we should also provide a general callback to override any field name?
  // this is interesting for general naming conventions like camelCase
  p.renameField(({ modelName, originalFieldName, prismasGeneratedFieldName }) => {
    if (modelName === 'User' && originalFieldName === 'name') {
      return 'newName'
    }
    return prismasGeneratedFieldName
  })

  // general hook to rename any model
  p.renameModel(({ orig, newModel }) => {
    if (orig === 'Post') {
      return 'User'
    }

    if (orig === 'User') {
      return 'Post'
    }

    return newModel
  })

  // rename a single model
  p.renameModel({
    oldName: 'User',
    newName: 'User2',
  })

  // define a custom field that the TS client will have
  p.field({
    model: 'User',
    select: {
      firstName: true,
      lastName: true,
    },
    name: 'fullName',
    type: 'string',
    resolve: parent => parent.firstName + ' ' + parent.lastName,
  })

  // add the *: false definition
  p.field({
    model: 'User',
    select: {
      friends: {
        '*': false,
        id: true,
      },
    },
    name: 'friendsCountTimesTwo',
    type: 'number',
    resolve: ({ friends }) => friends.length * 2,
  })

  // Definining a custom Type
  // In the datamodel this would just be an alias type
  // https://github.com/prisma/rfcs/blob/datamodel/text/0000-datamodel.md#user-defined-primitive-types

  p.type({
    name: 'Time',
    primitiveType: 'Int',
    serialize: val => val,
    deserialize: val => new Date(val).getTime(),
    fields: {
      User: ['createdAt', 'updatedAt'],
      Post: ['publishDate'],
    },
  })

  p.type({
    name: 'Point',
  })
})
```

# Drawbacks

# Alternatives

# Adoption strategy

# How we teach this

# Unresolved questions
