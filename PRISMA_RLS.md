## Adding row level security in PRISMA
- Use this syntax on column on which RLS needs to be implemented `userId String @default(dbgenerated("(current_setting('app.current_user_id'::text))::uuid")) @db.Uuid`
- while creating tables that require row level security use `npx prisma migrate dev --create-only`
- In the newly created migration file add the following to enable RLS for the table
```
ALTER TABLE "table_name" ENABLE ROW LEVEL SECURITY;

ALTER TABLE "table_name" FORCE ROW LEVEL SECURITY;

CREATE POLICY user_select_own on "table_name" using (
    "userId" = current_setting('app.current_user_id'::text)::uuid
)
WITH
    CHECK (
        "userId" = current_setting('app.current_user_id'::text)::uuid
    );
```

- Now run `npx prisma migrate dev` to complete the migration
- Create prisma extension definitions
```
export function extensionDefinition(userId: string) {
  return Prisma.defineExtension((prisma) =>
    prisma.$extends({
      query: {
        $allModels: {
          async $allOperations({ args, query}) {
            const [, result] = await prisma.$transaction([
              prisma.$executeRaw`SELECT set_config('app.current_user_id', ${userId}, TRUE)`,
              query(args),
            ]);
            return result;
          },
        },
      },
    })
  );
}
```
- Use this extension definition to create a prisma extension using `prisma.$extends(extensionDefinition(user_id))`
- Use this newly created extension definition to query the DB.


### Common issues
- MAKE SURE TO NOT USE ADMIN USER TO CONNTECT PRISMA WITH DB
- Create a separate user (say `prisma`) and give it all required PERMISSIONS AND PRIVILEGES
```
CREATE DATABASE EXAMPLE_DB;
CREATE USER EXAMPLE_USER WITH ENCRYPTED PASSWORD 'Sup3rS3cret';
GRANT ALL PRIVILEGES ON DATABASE EXAMPLE_DB TO EXAMPLE_USER;
\c EXAMPLE_DB postgres
GRANT ALL ON SCHEMA public TO EXAMPLE_USER;
ALTER USER prisma CREATEDB;
```
- Check if RLS is implemented properly. (SQL query to db)
