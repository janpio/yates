<div align="center">
  <img width="200" height="200" src="https://raw.githubusercontent.com/cerebruminc/yates/master/images/yates-icon.png">

[![npm version](https://img.shields.io/npm/v/@cerebruminc/yates)](https://www.npmjs.com/package/@cerebruminc/yates)

  <h1>Yates = Prisma + RLS</h1>

  <p>
    A module for implementing role based access control with Prisma when using Postgres
  </p>
  <br>
</div>

> English: from Middle English _yates_ ‘gates’ plural of _yate_ Old English _geat_ ‘gate’ hence a topographic or occupational name for someone who lived by the gates of a town or castle and who probably acted as the gatekeeper or porter.

<br>

Yates is a module for implementing role based access control with Prisma. It is designed to be used with the [Prisma Client](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-client) and [PostgreSQL](https://www.postgresql.org/).
It uses the [Row Level Security](https://www.postgresql.org/docs/9.5/ddl-rowsecurity.html) feature of PostgreSQL to provide a simple and secure way to implement role based access control that allows you to define complex access control rules and have them apply to all of your Prisma queries automatically.

## Prerequisites

Yates requires the `prisma` package ate version 4.9.0 or greater and the `@prisma/client` package at version 4.0.0 or greater. Additionally it makes use of the [Prisma Client extensions](https://www.prisma.io/docs/concepts/components/prisma-client/client-extensions) preview feature to generate rules, so you will need to enable this feature in your Prisma schema.

````prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["clientExtensions"]
}
```

## Installation

```bash
npm i @cerebruminc/yates
````

## Usage

Once you've installed Yates, you can use it in your Prisma project by importing it and calling the `setup` function. This function takes a Prisma Client instance and a configuration object as arguments. Yates uses prisma middleware to intercept all queries and apply the appropriate row level security policies to them, so it's important that it is setup _after_ all other middleware has been applied to the Prisma client.

The `setup` function will generate CRUD abilities for each model in your Prisma schema, as well as any additional abilities that you have defined in your configuration. It will then create a new PG role for each ability and apply the appropriate row level security policies to each role. Finally, it will create a new PG role for each user role you specify and grant them the appropriate abilities.
For Yates to be able to set the correct user role for each request, you must pass a function called `getContext` in the `setup` configuration that will return the user role for the current request. This function will be called for each request and the user role returned will be used to set the `role` in the current session. If you want to bypass RLS completely for a specific role, you can return `null` from the `getContext` function for that role.
For accessing the context of a Prisma query, we recommend using a package like [cls-hooked](https://www.npmjs.com/package/cls-hooked) to store the context in the current session.

```ts
import { setup } from "@cerebruminc/yates";
import { PrismaClient } from "@prisma/client";


const prisma = new PrismaClient();

await setup({
    prisma,
    // Define any custom abilities that you want to add to the system.
    customAbilities: () => ({
        USER: {
            Post: {
                insertOwnPost: {
                    description: "Insert own post",
                    // You can express the rule as a Prisma `where` clause.
                    expression: (client, row, context) => {
                      return {
                        // This expression uses a context setting returned by the getContext function
                        authorId: context('user.id')
                      }
                    },
                    operation: "INSERT",
                },
            },
            Comment: {
                deleteOnOwnPost: {
                    description: "Delete comment on own post",
                    // You can also express the rule as a conventional Prisma query.
                    expression: (client, row, context) => {
                      return client.post.findFirst({
                        where: {
                          id: row('postId'),
                          authorId: context('user.id')
                        }
                      })
                    },
                    operation: "DELETE",
                },
            }
            User: {
                updateOwnUser: {
                    description: "Update own user",
                    // For low-level control you can also write expressions as a raw SQL string.
                    expression: `current_setting('user.id') = "id"`,
                    operation: "UPDATE",
                },
            }
        }

    })
    // Return a mapping of user roles and abilities.
    // This function is paramaterised with a list of all CRUD abilities that have been
    // automatically generated by Yates, as well as any customAbilities that have been defined.
    getRoles: (abilities) => {
      return {
        SUPER_ADMIN: "*",
        USER: [
            abilities.User.read,
            abilities.Comment.read
        ],
      };
    },
    getContext: () => {
      // Here we are using cls-hooked to access the context in the current session.
      const ctx = clsSession.get("ctx");
      if (!ctx) {
        return null;
      }
      const { user } = ctx;

      let role = user.role;

      const role = user.role

      return {
        role,
        context: {
            // This context setting will be available in ability expressions using `current_setting('user.id')`
          'user.id': user.id,
        },
      };
    },
```

## Configuration

### Abilities

When defining an ability you need to provide the following properties:

- `description`: A description of the ability.
- `expression`: A boolean SQL expression that will be used to filter the results of the query. This expression can use any of the columns in the table that the ability is being applied to, as well as any context settings that have been defined in the `getContext` function.

  - For `INSERT`, `UPDATE` and `DELETE` operations, the expression uses the values from the row being inserted. If the expression returns `false` for a row, that row will not be inserted, updated or deleted.
  - For `SELECT` operations, the expression uses the values from the row being returned. If the expression returns `false` for a row, that row will not be returned.

- `operation`: The operation that the ability is being applied to. This can be one of `CREATE`, `READ`, `UPDATE` or `DELETE`.

## License

The project is licensed under the MIT license.

  <br>
  <br>

<div align="center">

![Cerebrum](./images/powered-by-cerebrum-lm.png#gh-light-mode-only)
![Cerebrum](./images/powered-by-cerebrum-dm.svg#gh-dark-mode-only)

</div>
