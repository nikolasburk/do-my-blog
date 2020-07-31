# REST API with Prisma, TypeScript and PostgreSQL

## Usage

```
git clone git@github.com:nikolasburk/do-my-blog.git
cd do-my-blog
npm install
docker-compose up -d
npx prisma migrate save --experimental --create-db --name "init"
npx prisma migrate up --experimental
npx ts-node src/index.ts
```
