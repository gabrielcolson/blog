---
title: "Workshop: Prisma and Graphql"
date: 2020-01-19T03:10:32+01:00
draft: false
images: ["prisma-workshop/prisma.jpg"]
summary: "A workshop where we build a GraphQL API using the Prisma Framework"
---

# Introduction

This Workshop aims to introduce you to [Prisma 2](https://github.com/prisma/prisma2)
aka the Prisma Framework.

Prisma is divided in 3 components:

1. [Client](https://github.com/prisma/prisma-client-js): a type-safe and auto-generated database client.
3. [Migrate](https://github.com/prisma/migrate): a declarative data modeling language and a migration system.
3. [Studio](https://github.com/prisma/studio): an admin UI to support various database workflows.

If you don't understand everything don't worry: you will by the end
of this workshop!

Just in case, open the [Prisma2 doc](https://github.com/prisma/prisma2/tree/master/docs),
it may help!


# Getting started

Prisma is a nice tool to communicate with your database so you will obviously need...
a database! install [docker](https://docs.docker.com/install/) if you haven't already
and run the following command:

```bash
$ docker run --name db -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -p 5432:5432 -d postgres
```

Next, make sure you have Node.js installed.

```bash
$ node -v
v12.14.1
```

Nice! We're ready to start! We can now create our project with the Prisma CLI.

````bash
$ npx prisma2 init prisma-workshop
$ cd prisma-workshop
````

Delete the content of `prisma/schema.prisma` and replace it with
```
datasource db {
  provider = "postgresql"
  url      = "postgresql://postgres:postgres@localhost:5432/postgres?schema=public"
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id String @id @default(cuid())
}
```

It tells prisma that we are using a PostgreSQL database, on which URL it is located and
that we use the JavaScript client.

We also added an example model called `User` which have an `id` field.

Now we will have to setup a Node.js project and to install some dependencies.

```bash
$ npm init -y
$ npm install --save-dev prisma2 typescript ts-node
$ npx tsc --init --target es2018
$ npm install --save @prisma/client
```

Open a new terminal in the same directory and run the prisma watcher:

```bash
$ npx prisma2 generate --watch
```

This will watch our `schema.prisma`. When it changes, prisma regenerates the Client.

Add a `start` script to the `package.json`:

```json
{
  "scripts": {
    "start": "ts-node src/index.ts"
  }
}
```

Create our source file:

```typescript
// src/index.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  await prisma.connect();

  await prisma.disconnect();
}

main();
```

You can type `npm start`. If nothing happens, it works! If something does happen,
you probably missed something and should step back to see what.


# Introducing the Prisma Client

Time to put you at work! I want you to familiarize yourself with the Prisma Client
and the Prisma Language.

Go through the [docs](https://github.com/prisma/prisma2/tree/master/docs)
and find out how to add models to your `schema.prisma`. We need to update the User model
which have an id (cuid string), a unique email and a password.

Once you wrote your model, you have to apply the changes to the database. To do so run
the following commands:

```bash
$ npx prisma2 migrate save --name add_user_model --experimental
$ npx prisma2 migrate up --experimental
```

We just created a migration an applied it to our database.

Your script (`src/index.ts`) must create 3 users, update the email of one, delete
another, fetch them all and display them.

*⚠️: if you need to clear your database, run `npx prisma2 studio --experimental`, go to
the [Prisma Studio](http://localhost:5555/) and delete the users.*


# Introducing GraphQL

Now that you played a bit with the Client, we can create our first GraphQL server. Before we
proceed, I invite you to read what is [GraphQL](https://graphql.org/).

Theory is cool, but practice is cooler! Go to the
[Apollo Server docs](https://www.apollographql.com/docs/apollo-server/) and read the
*getting started* page.

Once you're done, install `apollo-server` and `graphql`, remove the `main` function from
`index.ts` and create a new `ApolloServer`.

## Queries

Now that you have read `ApolloServer` getting started you have to setup a schema, create
the `User` type and add a `users` query. We need to be able to fetch the users like this:

```graphql
query {
    users {
        email
        password
    }
}
```

Make the resolver return this global array, we will link it to the database later:

```typescript
const users = [
  { email: 'toto@demo.com', password: 'toto' },
  { email: 'tata@demo.com', password: 'tata' },
];
```

You should now have your first GraphQL API up and running!

## Mutations and parameters

We can read some data from our API but we still can't write anything to it. Let's fix
that!

Read the next page of the `ApolloServer`'s docs.

Add a `Mutation` type and a mutation called `createUser` which takes an email and a
password as arguments. It must return the newly created user after it pushed it in our
`users` array.

```graphql
mutation {
    createUser(email: "tutu@demo.com", password: "tutu") {
        email
        password
    }
}
```

## Make your life easier...

You may have noticed that you have to reload your server each time you were making any
changes. We can use `ts-node-dev` to avoid this and make our life easier! Just add this
script to your `package.json` and use this from now on to start your server during
development.

```json
{
    "dev": "ts-node-dev --no-notify --respawn src/index.ts"
}
```

```bash
$ npm install --save-dev ts-node-dev
$ npm run dev
```


# Linking Prisma and GraphQL

To access Prisma in our GraphQL API we need to pass down the prisma instance to our
resolvers. Apollo have a great way to do so through the `Context` object.

As you saw earlier, a resolver can take at least 2 arguments:

1. `parent`: The result of the previous resolver in the middleware pipeline.
2. `args`: the arguments of your resolver.

But there is a third which is `context` (that I like to call `ctx`).

As you can see [here](https://www.apollographql.com/docs/apollo-server/data/data/#context-argument),
the `context` object is where we can store various information about the current query and our
global connection to other services, like a database. To use our prisma instance in our resolvers
we just have to tell `ApolloServer` to add it to the context:

```typescript
// src/index.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

interface Context {
  prisma: PrismaClient;
}

function createContext(): Context {
  return {
    prisma,
  };
}

// ...

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: createContext,
});

// ...
```

Great! Now our resolvers should look like this:
```typescript
function(parent, args, ctx)
```

Or, if you just want your prisma instance, like this:
```typescript
function(parent, args, { prisma })
```

Now you can use your previously acquired prisma knowledge to read and write from the database
in the `users` and `createUser` resolvers.

To see if it works, try to create a user from your GraphQL API and see if it appeared in
Prisma Studio.


# Nexus

Even if writing a GraphQL schema looks simple for now, I can tell you it doesn't scale well for
several reasons:

1. Having the whole schema in a string is hard to maintain.
2. No type safety in the resolvers.
3. No auto-completion for the schema and the resolvers.

The Prisma team saw that and created a tool called [GraphQL Nexus](https://nexus.js.org).
This library allows you to define your GraphQL schema in TypeScript and generate the correct
types for your resolvers.

Go through the Nexus [Getting Started](https://nexus.js.org/docs/getting-started) and install it
with `npm install nexus`. We will now rewrite our schema with the Nexus types.


## Generate the GraphQL schema with Nexus

First, define a `User` type with the `objectType` function. Give it an `email` and
`password` field. Then, recreate the `Query` and `Mutation` types with the `queryType`
 and `mutationType` functions.

Move the resolvers to their respective Nexus type on the resolve attribute. Here is how I
did it with the `Query` type.

```typescript
// src/index.ts

// ...

const User = objectType({
  // ...
});

const Query = queryType({
  definition(t) {
    t.list.field('users', {
      type: User,
      resolve: (parent, args, ctx) => ctx.prisma.user.findMany(),
    });
    },
});

const Mutation = mutationType({
  // ...
});

const schema = makeSchema({
  types: [User, Query, Mutation],
});

const server = new ApolloServer({
  schema,
  context: createContext,
});

// ...
```

If you go to the GraphQL Playground (ensure first your server is running), in the schema tab,
you should see the exact same schema we have defined without Nexus:

```graphql
type Mutation {
  createUser(email: String!, password: String!): User!
}
​
type Query {
  users: [User!]!
}
​
type User {
  email: String!
  password: String!
}
```


## Nexus output

If you look at the logs of your server, you might see this warning:

```
You should specify a configuration value for outputs in Nexus' makeSchema.
Provide one to remove this warning.
```

Having Nexus generate files for the GraphQL schema and the generated types can be very useful
when developing. Let's setup this:

```typescript
const schema = makeSchema({
  types: [User, Query, Mutation],
  outputs: {
    schema: `${__dirname}/../schema.graphql`,
    typegen: `${__dirname}/generated/nexus.ts`
  },
});
```

Nexus author advise that we commit the `schema.graphql` file even if it is generated
because it could be useful to see how the resulting API is evolving through the versions
of our application.

The `nexus.ts` file containing the TypeScript typegen can however be added to the `.gitignore`.


## Nexus-Prisma

As you see, there is a bit of redundancy between our model definition and the GraphQL API. The
Prisma team developed a plugin to bring high level synergy between Nexus and Prisma:
[Nexus-Prisma](https://github.com/prisma-labs/nexus-prisma).

```bash
$ npm install nexus-prisma
```

Add the plugin to Nexus:

```typescript
import { nexusPrismaPlugin } from 'nexus-prisma';

const schema = makeSchema({
  types: [...],
  outputs: { ... },
  plugins: [nexusPrismaPlugin()],
});
```

Now, go through the docs and see how you can improve the `User` Nexus type implementation with
the `t.model` feature.

Next, find a way to remove our resolvers implementation by only using `t.crud`. We should be
able to search a user with the `users` query but only by email.


# The end.

We are now arriving at the end of this workshop. You should now know how to build great GraphQL
API with TypeScript, Prisma, Apollo and Nexus. I strongly encourage you to read about the best
practices (that I did not cover in this workshop) in the docs of the different tools we used.

If you have some time left, a great exercise could be to deploy your API on
[Heroku](https://heroku.com).

Thank you for following this workshop,

Gabriel Colson.
