# How to Build a REST API with Prisma and PostgreSQL

### Introduction

[Prisma](https://www.prisma.io) is an open source database toolkit. It consists of three main tools:

- **Prisma Client**: Auto-generated and type-safe query builder for Node.js & TypeScript
- **Prisma Migrate**: Declarative data modeling & migration system
- **Prisma Studio**: GUI to view and edit data in your database

Prisma makes working with databases easy for application developers who want to focus on implementing value-adding features instead of spending time figuring out complex database workflows (such as schema migrations or writing SQL queries).

In this tutorial, you will learn how to build a REST API for a small _blogging application_ in TypeScript using Prisma and a PostgreSQL database. At the end of the tutorial, you will have a web server running locally on your machine that can respond to various HTTP requests and read and write data in the database.

## Prerequisites

This tutorial assumes the following:

- [Node.js v10](https://nodejs.org/en/) or higher installed on your machine
- [Docker](https://www.docker.com/) installed on your machine (to run the PostgreSQL database)

Basic familiarity with TypeScript and REST APIs are helpful but not required for this tutorial.

## Step 1 —  Setting up Your TypeScript Project

In this step, you will set up a plain TypeScript project using npm. This project will be the foundation for the REST API you're going to build throughout the course of this tutorial.

First, create a new directory for your project:

```command
mkdir my-blog
```

Next, navigate into the directory and initialize an empty npm project:

```command
cd my-blog
npm init -y
```

This command creates a minimal `package.json` file that is used as the configuration file for your npm project. You're now ready to configure TypeScript in your project.

Execute the following command for a plain TypeScript setup:

```command
npm install typescript ts-node @types/node --save-dev
```

This installs three packages as _development dependencies_ in your project:

- [`typescript`](https://www.npmjs.com/package/typescript): The TypeScript toolchain
- [`ts-node`](https://www.npmjs.com/package/ts-node): A package to easily run TypeScript applications without prior compilation to JavaScript
- [`@types/node`](https://www.npmjs.com/package/@types/node): The TypeScript type definitions for Node.js

The last thing to do in this step is to add a [`tsconfig.json`](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html) file to ensure TypeScript is properly configured for the purposes of the application you are going to build.

First, run the following command to create the file:

```command
touch tsconfig.json
```

Now open the newly created file and paste the following code into it:

```json
[label my-blog/tsconfig.json]
{
  "compilerOptions": {
    "sourceMap": true,
    "outDir": "dist",
    "strict": true,
    "lib": ["esnext"],
    "esModuleInterop": true
  }
}
```

This is a standard and minimal configuration for a TypeScript project. If you want to learn about the individual properties of the configuration file, you can look them up in the [TypeScript documentation](https://www.typescriptlang.org/docs/handbook/compiler-options.html).

## Step 2 — Setting up Prisma with PostgreSQL

In this step, you will install the [Prisma CLI](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-cli), create your initial [Prisma schema](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-schema) file, set up PostgreSQL with Docker and connect Prisma to it. The Prisma schema is the main configuration file for your Prisma setup and contains your database schema.

Start by installing the Prisma CLI with the following command:

```command
npm install @prisma/cli --save-dev
```

As a best practice, it is recommended to [install the Prisma CLI locally](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-cli/installation#local-installation-recommended) in your project (as opposed to a global installation). This helps avoid version conflicts in case you have more than one Prisma project on your machine.

Next, you'll set up your PostgreSQL database using Docker. Create a new Docker Compose file with the following command:

```command
touch docker-compose.yml
```

Now add the following code to the newly created file:

```yml
[label my-blog/docker-compose.yml]
version: '3.8'
services:
  postgres:
    image: postgres:10.3
    restart: always
    environment:
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=mypassword42
    volumes:
      - postgres:/var/lib/postgresql/data
    ports:
      - '5432:5432'
volumes:
  postgres:
```

With this setup in place, go ahead and launch the PostgreSQL database server with the following command:

```command
docker-compose up -d
```

You can verify that the database server is running with the following command:

```command
docker ps
```

This should output something similar to this:

```
[secondary_label Output]
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
8547f8e007ba        postgres:10.3       "docker-entrypoint.s…"   3 seconds ago       Up 2 seconds        0.0.0.0:5432->5432/tcp   my-blog_postgres_1
```

<!-- > **Note**: At this point you should be able to connect to the database using a GUI like [TablePlus](https://tableplus.com/) or [Postico](https://eggerapps.at/postico/). -->

With the database server running, you can now create your Prisma setup. Run the following command from the Prisma CLI:

```command
npx prisma init
```

Note that as a best practice, all invokations of the Prisma CLI should be prefixed with `npx`. This ensures your local installation is being used.

After running the command, the Prisma CLI created a new folder called `prisma` in your project. It contains the following two files:

- `schema.prisma`: The main configuration file for your Prisma project (will include your data model)
- `.env`: A [dotenv](https://github.com/motdotla/dotenv) file to define your database connection URL

To make sure Prisma knows about the location of your database, open the `.env` file and adjust the `DATABASE_URL` environment variable to look as follows:

```
[label my-blog/prisma/.env]
DATABASE_URL="postgresql://myuser:mypassword42@localhost:5432/my-blog?schema=public"
```

Note that you're using the database credentials `myuser` and `mypassword42` which are specified in the Docker Compose file. To learn more about the format of the connection URL, visit the [Prisma docs](https://www.prisma.io/docs/reference/database-connectors/postgresql/#connection-url).

## Step 3 - Defining Your Data Model and Create Database Tables

In this step, you will define your [data model](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-schema/data-model) in the Prisma schema file. This data model will then be mapped to the database with [Prisma Migrate](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-migrate) which will generate and send the SQL statements for creating the tables that correspond to your data model. Since you're building a _blogging application_, the main entities of the application will be _users_ and _posts_.

Prisma uses its own [data modeling language](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-schema#syntax) to define the shape of your application data.

Open your `schema.prisma` file and add the following model definitions to it:

```prisma
[label my-blog/prisma/schema.prisma]
model User {
  id    Int     @default(autoincrement()) @id
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int     @default(autoincrement()) @id
  title     String
  content   String?
  published Boolean @default(false)
  author    User?   @relation(fields: [authorId], references: [id])
  authorId  Int?
}
```

There's a lot going on, so let's try understand the individual pieces.

You are defining two [models](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-schema/models), called `User` and `Post`. Each of these has a number of [fields](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-schema/models#fields). The models will be mapped to database _tables_, the fields represent the individual _columns_.

Also note that there's a one-to-many [relation](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-schema/relations) between the two models, specified by the `posts` and `author` relation fields on `User` and `Post`.

With these models in place, you can now create the corresponding tables in the database using Prisma Migrate. Open up your terminal again and run the following command:

```command
npx prisma migrate save --experimental --create-db --name "init"
```

This command creates a new _migration_ on your file system. Here's a quick overview of the three options that are provided to the command:

- `--experimental`: Required because Prisma Migrate is currently in an _experimental_ state
- `--create-db`: Enables Prisma Migrate to create the database named `my-blog` that's specified in the connection URL
- `--name "init"`": Specifies the name of the migration (will be used to name the migration folder that's created on your file system)

Your `prisma` directory should now contain files that look similar to this:

```
[secondary_label Output]
prisma
├── migrations
│   ├── 20200730153912-init
│   │   ├── README.md
│   │   ├── schema.prisma
│   │   └── steps.json
│   └── migrate.lock
└── schema.prisma
```

Feel free to explore the migration files that are being created.

To actually run the migration against your database and create the tables for your Prisma models, run the following command in your terminal:

```command
npx prisma migrate up --experimental
```

Prisma Migrate now generates the SQL statements that are required for the migration and sends them to the database. Concretely, it created the following tables:

```sql
CREATE TABLE "public"."User" (
  "id" SERIAL,
  "email" text  NOT NULL ,
  "name" text   ,
  PRIMARY KEY ("id")
)

CREATE TABLE "public"."Post" (
  "id" SERIAL,
  "title" text  NOT NULL ,
  "content" text   ,
  "published" boolean  NOT NULL DEFAULT false,
  "authorId" integer   ,
  PRIMARY KEY ("id")
)

CREATE UNIQUE INDEX "User.email" ON "public"."User"("email")

ALTER TABLE "public"."Post" ADD FOREIGN KEY ("authorId")REFERENCES "public"."User"("id") ON DELETE SET NULL ON UPDATE CASCADE
```

Congratulations, you are now ready to query your database with Prisma Client!

## Step 4 – Installing Prisma Client in Your Project

Prisma Client is an auto-generated and type-safe query builder that you can use to programmatically read and write data in a database from a Node.js or TypeScript application. In this step, you will learn how to install Prisma Client in your project.

Open up your terminal again and install the Prisma Client npm package:

```command
npm install @prisma/client
```

## Step 5 – Exploring Prisma Client Queries in a Plain Script

In this step, you will get familiar with the queries you can send with Prisma Client. Before diving into implementing the routes for your REST API in the next steps, you will first explore some of the Prisma Client queries in a plain, executable script.

To do so, first create a new directory called `src` that will contain your source files:

```command
mkdir src
```

Now create a TypeScript file inside of the new directory:

```command
touch src/index.ts
```

All of the Prisma Client queries return [promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that you can `await` in your code. This requires you to send the queries inside of an `async` function.

Add the following boilerplate with an `async` function that's executed in your script:

```ts
[label my-blog/src/index.ts]
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
  // ... your Prisma Client queries will go here
}

main()
  .catch((e) => console.error(e))
  .finally(async () => await prisma.disconnect())
```

With the `main` function in place, you can start adding Prisma Client queries to the script. Adjust `index.ts` to look as follows:

```ts
[label my-blog/src/index.ts]
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
  const newUser = await prisma.user.create({
    data: {
      name: 'Alice',
      email: 'alice@prisma.io',
      posts: {
        create: {
          title: 'Hello World',
        },
      },
    },
  })
  console.log('Created new user: ', newUser)

  const allUsers = await prisma.user.findMany({
    include: { posts: true },
  })
  console.log('All users: ')
  console.dir(allUsers, { depth: null })
}

main()
  .catch((e) => console.error(e))
  .finally(async () => await prisma.disconnect())
```

In the code above, you're using two Prisma Client queries:

- `create`: Creates a new `User` record. Notice that you're actually using a [_nested write_](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-client/relation-queries#nested-writes), meaning you're in fact creating both a `User` and `Post` record in the same query.
- `findMany`: Reads all existing `User` records from the database. Notice that you're providing the [`include`](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-client/field-selection#include) option which additionally loads the related `Post` records for each `User` record.

Now that you understand the code, go ahead and run the script with the following command:

```command
npx ts-node src/index.ts
```

You should see the following output in your terminal:

```
[secondary_label Output]
Created new user:  { id: 1, email: 'alice@prisma.io', name: 'Alice' }
[
  {
    id: 1,
    email: 'alice@prisma.io',
    name: 'Alice',
    posts: [
      {
        id: 1,
        title: 'Hello World',
        content: null,
        published: false,
        authorId: 1
      }
    ]
  }
```

<$>[note]
**Note:** If you are using a database GUI you can validate that the data was created by looking at the `User` and `Post` tables. Alternatively you can explore the data in Prisma Studio by running `npx prisma studio --experimental`.
<$>

Great job so far! You have now used Prisma Client to read and write data in your database. In the remaining steps, you'll apply that new knowledge to implement the routes for a sample REST API.

## Step 6 – Setting up Express.js and Implementing Your First REST API Route

In this step, you will install [Express.js](https://expressjs.com/) in your application. Express.js is a popular web framework for Node.js that you will use to implement your REST API routes in this project. The first route you will implement will allow you to fetch all _users_ from the API using a `GET` request.

Go ahead and install Express.js with the following command:

```command
npm install express
```

Since you're using TypeScript, you'll also want to install the respective types as development dependencies. Run the following command to do so:

```command
npm install @types/express --save-dev
```

With the dependencies in place, you can set up your Express.js application. Delete all the current code in `index.ts` and replace it with the following "sceleton" for your REST API:

```ts
[label my-blog/src/index.ts]
import { PrismaClient } from '@prisma/client'
import express from 'express'

const prisma = new PrismaClient()
const app = express()

app.use(express.json())

// ... your REST API routes will go here

app.listen(3000, () =>
  console.log('REST API server ready at: http://localhost:3000'),
)
```

Now you can implement your first route. Between the calls to `app.use` and `app.listen`, add the following code:

```ts
[label my-blog/src/index.ts]
app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany()
  res.json(users)
})
```

Once added, you can start your local web server using the following command:

```command
npx ts-node src/index.ts
```

You should see the following output:

```
[secondary_label Output]
REST API server ready at: http://localhost:3000
```

To access the `/users` route you can point your browser to [`http://localhost:3000/users`](http://localhost:3000/users). However, because this approach only allows to send `GET` requests, but you'll be implementing `POST`, `PUT` and `DELETE` requests later on, it is recommended that you use an HTTP client such as [Postwoman](https://github.com/liyasthomas/postwoman) or the [Advanced REST Client](https://install.advancedrestclient.com/install). In this tutorial, these HTTP requests will be made [`curl`](https://curl.haxx.se/), a terminal-based HTTP client.

To test your route, execute the following command in your terminal:

```command
curl http://localhost:3000/users
```

You should see the `User` data that you created in the previous step:

```
[secondary_label Output]
[{"id":1,"email":"alice@prisma.io","name":"Alice"}]
```

Note that the `posts` array is not included this time. This is because you're not passing the `include` option to the `findMany` call in the implementation of the `/users` route.

## Step 7 – Implementing the Remaining REST API Routes

In this step, you will implement the remaining REST API routes for your _blogging application_. At the end, your web server will serve various `GET`, `POST`, `PUT` and `DELETE` requests.

Here is an overview of the different routes you will implement:

| HTTP Method | Route               | Description                                     |
| ----------- | ------------------- | ----------------------------------------------- |
| `GET`       | `/feed`             | Fetches all _published_ posts.                  |
| `GET`       | `/post/:id`         | Fetches a specific post by its ID.              |
| `POST`      | `/user`             | Creates a new user.                             |
| `POST`      | `/post`             | Creates a new post (as a _draft_).              |
| `PUT`       | `/post/publish/:id` | Sets the `published` field of a post to `true`. |
| `DELETE`    | `post/:id`          | Deletes a post by its ID.                       |

Go ahead and implement the remaining `GET` routes first. Add the following code below the implementation of the `/users` route:

```ts
[label my-blog/src/index.ts]
app.get('/feed', async (req, res) => {
  const posts = await prisma.post.findMany({
    where: { published: true },
    include: { author: true }
  })
  res.json(posts)
})

app.get(`/post/:id`, async (req, res) => {
  const { id } = req.params
  const post = await prisma.post.findOne({
    where: { id: Number(id) },
  })
  res.json(post)
})
```

You can stop the server hitting `CTRL+C` on your keyboard. Then, restart the server using:

```command
npx ts-node src/index.ts
```

To test the `/feed` route, you can use the following `curl` command:

```command
curl http://localhost:3000/feed
```

Since no posts have been _published_ yet, the response is an empty array:

```
[secondary_label Output]
[]
```

To test the `/post/:id` route, you can use the following `curl` command:

```command
curl http://localhost:3000/post/1
```

This will return the post you initially created:

```
[secondary_label Output]
{"id":1,"title":"Hello World","content":null,"published":false,"authorId":1}
```

Next, implement the two `POST` routes. Add the following code to `index.ts` below the implementations of the three `GET` routes:

```ts
[label my-blog/src/index.ts]
app.post(`/user`, async (req, res) => {
  const result = await prisma.user.create({
    data: { ...req.body },
  })
  res.json(result)
})

app.post(`/post`, async (req, res) => {
  const { title, content, authorEmail } = req.body
  const result = await prisma.post.create({
    data: {
      title,
      content,
      published: false,
      author: { connect: { email: authorEmail } },
    },
  })
  res.json(result)
})
```

Like before, you can test the new routes by stopping the server with `CTRL+C` on your keyboard. Then, restart the server using:

```command
npx ts-node src/index.ts
```

To create a new user via the `/user` route, you can send the following `POST` request with `curl`:

```command
curl -X POST -H "Content-Type: application/json" -d '{"name":"Bob", "email":"bob@prisma.io"}' http://localhost:3000/user 
```

This will create a new user in the database print the following output:

```
[secondary_label Output]
{"id":2,"email":"bob@prisma.io","name":"Bob"}
```

To create a new post via the `/post` route, you can send the following `POST` request with `curl`:

```command
curl -X POST -H "Content-Type: application/json" -d '{"title":"I am Bob", "authorEmail":"bob@prisma.io"}' http://localhost:3000/post 
```

This will create. a new post in the database and _connect_ it to the user with the amil `bob@prisma.io`. It prints the following output:

```
[secondary_label Output]
{"id":2,"title":"I am Bob","content":null,"published":false,"authorId":2}
```

Finally, you can implement the `PUT` and `DELETE` routes. Inside `index.ts`, right below the implementation of the two `POST` routes, add the following code:

```ts
[label my-blog/src/index.ts]
app.put('/post/publish/:id', async (req, res) => {
  const { id } = req.params
  const post = await prisma.post.update({
    where: { id: Number(id) },
    data: { published: true },
  })
  res.json(post)
})

app.delete(`/post/:id`, async (req, res) => {
  const { id } = req.params
  const post = await prisma.post.delete({
    where: { id: Number(id) },
  })
  res.json(post)
})
```

Again, stop the server with `CTRL+C` on your keyboard. Then, restart the server using:

```command
npx ts-node src/index.ts
```

You can test the `PUT` route with the following `curl` command:

```command
curl -X PUT http://localhost:3000/post/publish/2
```

This is going to _publish_ the post with an ID value of `2`. If you re-send the `/feed` request, this post will now be included in the response.

Finally, you can test the `DELETE` route with the following `curl` command:

```command
curl -X DELETE http://localhost:3000/post/1
```

This is going to delete the post with an ID value of `1`.

## Conclusion

In this article, you created a REST API server with a number of different routes to create, read, update and delete user and post data for a sample blogging application. Inside of the API routes, you are using Prisma Client to send the respective queries to your database.

As next steps, you can implement additional API routes or extend your database schema using Prisma Migrate. Be sure to visit the [Prisma documentation](https://www.prisma.io/docs) to learn about different aspects of Prisma and explore some ready-to-run example projects in the [`prisma-examples`](https://github.com/prisma/prisma-examples/) repository (e.g. for GraphQL or grPC APIs). 
