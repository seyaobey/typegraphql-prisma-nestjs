# Fork of [typegraphql-prisma](https://www.npmjs.com/package/typegraphql-prisma)

This project is a fork of another with minor changes, created for personal use.

Long term support is not guaranteed, use of this copy is at your own risk.

## Installation

```
npm install typegraphql-prisma-nestjs --save-dev
```

## Differences from the original project:

Functions and classes used in generated files from scheme of [prisma](https://github.com/prisma/prisma), imports from [nestjs](https://nestjs.com)

Original:
```typescript
import { Ctx, Query, Resolver } from "type-graphql";
```

In fork:
```typescript
import { Context, Query, Resolver } from "@nestjs/graphql";
```

## Example use NestJS + Prisma2 + Typegraphql

https://github.com/EndyKaufman/typegraphql-prisma-nestjs-example

---

![integration logo](https://raw.githubusercontent.com/EndyKaufman/typegraphql-prisma-nestjs/prisma/img/integration.png)

# TypeGraphQL & Prisma 2 integration

Prisma 2 generator to emit TypeGraphQL type classes and resolvers

## Installation

Fist of all, you have to install the generator, as a dev dependency:

```sh
npm i -D typegraphql-prisma-nestjs
```

Futhermore, `typegraphql-prisma` to work properly requires `prisma` to be, so please install prisma dependencies if you don't have it already installed:

```sh
npm i -D @prisma/cli
npm i @prisma/client
```

> `typegraphql-prisma` is designed to work with a selected version of `prisma` (or newer), so please make sure you use `@prisma/cli` and `@prisma/client` of version `~2.6.0`!

You also need to install the GraphQL JSON scalar library (to support the Prisma `Json` scalar):

```sh
npm i graphql-type-json
```

as well as the `graphql-fields` that is used to properly support the aggregations queries:

```sh
npm i graphql-fields @types/graphql-fields
```

Also, be aware that due to usage of some ES2019 and newer Node.js features, you also have to use **Node.js v12.4.0 or newer**.

## Configuration

After installation, you need to update your `schema.prisma` file and then add a new generator section below the `client` one:

```prisma
generator client {
  // ...
}

generator typegraphql {
  provider = "typegraphql-prisma-nestjs"
}
```

Then after running `npx prisma generate`, this will emit the generated TypeGraphQL classes to `@generated/typegraphql-prisma-nestjs` in `node_modules` folder.

You can also configure the default output folder, e.g.:

```prisma
generator typegraphql {
  provider = "typegraphql-prisma-nestjs"
  output   = "../prisma/generated/type-graphql"
}
```

By default, when the output path contains `node_modules`, the generated code is transpiled - consist of `*.js` and `*.d.ts` files that are ready to use (import) in your code.
However, if you explicitly choose some other folder in `output` config, the generated code will be emitted as a raw TS files which you can use and import as your other source code files.

You can overwrite that by explicitly setting `emitTranspiledCode` config option:

```prisma
generator typegraphql {
  provider           = "typegraphql-prisma"
  output             = "../prisma/generated/type-graphql"
  emitTranspiledCode = true
}
```

## Usage

Given that you have this part of datamodel definitions:

```prisma
enum PostKind {
  BLOG
  ADVERT
}

model User {
  id    String  @default(cuid()) @id @unique
  email String  @unique
  name  String?
  posts Post[]
}
````

It will generate a `User` class in the output folder, with TypeGraphQL decorators, and an enum - you can import them and use normally as a type or an explicit type in your resolvers:

```ts
export enum PostKind {
  BLOG = "BLOG",
  ADVERT = "ADVERT",
}
TypeGraphQL.registerEnumType(PostKind, {
  name: "PostKind",
  description: undefined,
});

@TypeGraphQL.ObjectType({
  isAbstract: true,
  description: undefined,
})
export class User {
  @TypeGraphQL.Field(_type => String, {
    nullable: false,
    description: undefined,
  })
  id!: string;

  @TypeGraphQL.Field(_type => String, {
    nullable: false,
    description: undefined,
  })
  email!: string;

  @TypeGraphQL.Field(_type => String, {
    nullable: true,
    description: undefined,
  })
  name?: string | null;

  posts?: Post[] | null;
}
```

It will also generates a whole bunch of stuffs based on your `schema.prisma` file - models classes, enums, as well as CRUD resolvers and relations resolver.

CRUD resolvers supports this following methods with args that are 1:1 matching with the `PrismaClient` API:

- findOne
- create
- update
- delete
- findMany
- updateMany
- deleteMany
- upsert

By default, the method names will be mapped to a GraphQL idiomatic ones (like `findManyUser` -> `users`).
You can opt-in to use original names by providing `useOriginalMapping = true` generator option.

Also, if you want to have relations like `User -> posts` emitted in schema, you need to import the relations resolvers and register them in your `buildSchema` call:

```ts
import {
  User,
  UserRelationsResolver,
  UserCrudResolver,
} from "@generated/type-graphql";

const schema = await buildSchema({
  resolvers: [CustomUserResolver, UserRelationsResolver, UserCrudResolver],
  validate: false,
});
```

When using the generated resolvers, you have to first provide the `PrismaClient` instance into the context under `prisma` key, to make it available for the crud and relations resolvers:

```ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();
const server = new ApolloServer({
  schema,
  playground: true,
  context: (): Context => ({ prisma }),
});
```

### Nest JS

In order to use generated types and resolvers classes in NestJS, you need to use the [official `typegraphql-nestjs` package](https://github.com/MichalLytek/typegraphql-nestjs). This module allows for basic integration of TypeGraphQL with NestJS. You can find an example in the [`examples/3-nest-js` folder](https://github.com/MichalLytek/type-graphql/tree/prisma/examples/3-nest-js).

Due to difference between TypeGraphQL and NestJS decorators, `typegraphql-prisma` doesn't work anymore with `@nestjs/graphql` from version 7.0.

### Advanced usage

#### Custom operations

You can also add custom queries and mutations to the schema as always, using the generated `PrismaClient` client:

```ts
@Resolver()
export class CustomUserResolver {
  @Query(returns => User, { nullable: true })
  async bestUser(@Ctx() { prisma }: Context): Promise<User | null> {
    return await prisma.user.findOne({
      where: { email: "bob@prisma.io" },
    });
  }
}
```

#### Adding fields to model type

If you want to add a field to the generated type like `User`, you have to add a proper `@FieldResolver` for that:

```ts
@Resolver(of => User)
export class CustomUserResolver {
  @FieldResolver(type => Post, { nullable: true })
  async favoritePost(
    @Root() user: User,
    @Ctx() { prisma }: Context,
  ): Promise<Post | undefined> {
    const [favoritePost] = await prisma.user
      .findOne({ where: { id: user.id } })
      .posts({ first: 1 });

    return favoritePost;
  }
}
```

#### Exposing selected Prisma actions

If you want to expose only certain Prisma actions, like `findManyUser` or `createOneUser`, you can import resolver classes only for them, instead of the whole model `CrudResolver`.
Then you just have to put them into the `buildSchema`:

```ts
import {
  User,
  UserRelationsResolver,
  FindManyUserResolver,
  CreateUserResolver,
} from "@generated/type-graphql";

const schema = await buildSchema({
  resolvers: [
    CustomUserResolver,
    UserRelationsResolver,
    FindManyUserResolver,
    CreateUserResolver,
  ],
  validate: false,
});
```

#### Changing exposed model type name

You can also change the name of the model types exposed in GraphQL Schema.
To achieve this, just put the `@@TypeGraphQL.type` doc line above the model definition in `schema.prisma` file, e.g:

```prisma
/// @@TypeGraphQL.type("Client")
model User {
  id     Int     @default(autoincrement()) @id
  email  String  @unique
  posts  Post[]
}
```

Be aware that this feature changes the name everywhere in the schema, so you can import `FindManyClientResolver` (not `FindManyUserResolver`), as well as only `ClientUpdateInput` is available (not `UserUpdateInput`), which means that the GraphQL queries/mutations will also be renamed, e.g.:

```graphql
type Mutation {
  createClient(data: ClientCreateInput!): Client!
}
```

#### Changing exposed model type field name

You can also change the name of the model type fields exposed in GraphQL Schema.
To achieve this, just put the `@TypeGraphQL.field` doc line above the model field definition in `schema.prisma` file, e.g:

```prisma
model User {
  id     Int     @default(autoincrement()) @id
  /// @TypeGraphQL.field("emailAddress")
  email  String  @unique
  posts  Post[]
}
```

This will result in the following GraphQL schema representation:

```graphql
type User {
  id: Int!
  emailAddress: String!
  posts: [Post!]!
}
```

All generated CRUD and relations resolvers fully support this feature and they map under the hood the original prisma property to the renamed field exposed in schema.

The same goes to the resolvers input types - they will also be emitted with changed field name, e.g.:

```graphql
input UserCreateInput {
  emailAddress: String!
  posts: PostCreateManyWithoutAuthorInput
}
```

The emitted input type classes automatically map the provided renamed field values from GraphQL query into proper Prisma input properties out of the box.

## Examples

You can check out some integration examples on this repo:

https://github.com/EndyKaufman/typegraphql-prisma-nestjs-example

## Feedback

Currently released version `0.x` is just a preview of the upcoming integration. For now it lacks customization option - picking/omitting fields of object types to expose in the schema, as well as picking exposed args fields.

However, the base functionality is working well, so I strongly encourage you to give it a try and play with it. Any feedback about the developers experience, bug reports or ideas about new features or enhancements are very welcome - please feel free to put your two cents into [discussion in the issue](https://github.com/EndyKaufman/typegraphql-prisma-nestjs/issues/476).

In near feature, when Prisma SDK will be ready, the `typegraphql-prisma-nestjs` integration will also allow to use a code-first approach to build a `schema.prisma` and GraphQL schema at once, using classes with decorators as a single source of truth. Stay tuned! :muscle:
