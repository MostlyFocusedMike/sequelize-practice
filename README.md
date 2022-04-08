# Sequelize Practice
## With some other notes as well
Sequelize command line things:

- sequelize db:create
  - uses the config/config.json to make a connection (default to development), and it's pretty neat

- sequelize model:generate --name Test --attributes name:string,email:string,role:string,age:integer
  - this automatically generates both a model and migration. The attributes must not have spaces. Also note that the model does not need to add all the attributes

- sequelize db:migrate
  - This will migrate and what's really cool is the migrations are saved in a table so theoretically, editing that is super easy for tight rollbacks if you *have* to. However, what's better would just be using `db:migrate:undo` to roll back migrations one by one and then migrate forward.

- sequelize seed:generate --name create-fake-users
  - this makes a new seed file

- sequelize db:seed:all
  - this will run all the seed files

## Sequelize Notes
### Default columns
With no modifications, Sequelize expects there to be `id, createdAt, updatedAt` *exactly* on the table. To undo this there are several options
- no timestamps
  - To avoid the timestamps altogether then in the model options you would put `timestamps: false` to override the default `true` value
  - https://stackoverflow.com/questions/20386402/sequelize-unknown-column-createdat-in-field-list
- Use underscores
  - If you have the table column names that use snake case, just have `underscored: true,` in the model options, which will then generate queries like: `SELECT "id", "created_at" AS "createdAt", "updated_at" AS "updatedAt" FROM "users" AS "User";` where the returned values are in the JS camelCase, but the raw column is snake_case
- Redfine them
  - In them model you can tell it what the fields are in the options object like: `{createdAt: created, updatedAt: updated}`
- Model vs sequelize
  - While you can include model by model, if you want, you can also include the whole thing in the init of sequelize

    ```js
      var sequelize = new Sequelize('sequelize_test', 'root', null, {
        host: "127.0.0.1",
        dialect: 'mysql',
        define: {
            underscored: true
        }
      });
    ```

### What is define/init
If you use the outdated define method, know that Sequelize calls the same `init` ES6+ method, so they mean the same thing. And what these methods do is create a model in Sequelize to link to tables. This DOES NOT AFFECT the actual tables in *any* way. What it does affect are the queries. Leaving things out of the init fields will mean Sequelize won't know about them. This means using Sequelize.create, the fields wouldn't be there (conversely, using sequelize.find* methods, these fields would not be returned).

The only fields that are not affected by the init fields are `id, updatedAt, createdAt`. Queries will always run with those fields. If you want to modify the timestamps, you can in init's settings object (eg set the query to look for underscored versions). Also, on create, sequelize will automatically generate timestamps itself for `updatedAt and createdAt`, it does not set a default `NOW()` in the table SQL.

So, init can be missing columns without causing *too* many problems. However, if it has fields that don't exist in the db, that will break all queries. Because it will try to read/alter a column that does not exist in the DB, in which case SQL will be the one breaking. This is why it's important that your migrations run and your tables are setup before initing your models.

### Relations
By default, if you just use things on the Sequelize init side like:

```js
    static associate({ User }) {
      // define association here
      // userId
      this.belongsTo(User, { foreignKey: 'userId', as: 'user' })
    }
```
It does not add a foreign Key restraint in SQL. Again, nothing that init/define does actually affects the structure, only the queries generated. Constants like that should be generated in the migrations

### Migrations/Table requirements
Sequelize is just a query generator, so the only kind of queries that can actually affect the tables are TABLE, ALTER TABLE etc, queries, and those are only run in the migrations. Sequelize migrations are a lot like Knex migrations, in that you have a handful of special methods on the queryInterface (see https://sequelize.org/docs/v6/other-topics/query-interface/). This is a lower level API than the standard Sequelize instance and deals primarily with table architecture and bulk data actions (list of methods https://sequelize.org/api/v6/class/src/dialects/abstract/query-interface.js~queryinterface).

However, it's still a regular JS file, so you can gain access to the niceties of Sequelize by importing the models like you would into app.js in this example.

As previously discussed, every table should have a primary id (which defaults to `id` not `tableName+id`, didn't bother looking how to change that) and some form of created and updated columns.

# Cool Links
https://www.citusdata.com/blog/2016/07/14/choosing-nosql-hstore-json-jsonb/
- That talks about the differences between the unstructured Data types which summarizes as:
  ```
  JSONB - In most cases

  JSON - If you’re just processing logs, don’t often need to query, and use as more of an audit trail

  hstore - Can work fine for text based key-value looks, but in general JSONB can still work great here
  ```
https://www.youtube.com/watch?v=3qlnR9hK-lQ
- This is a pretty good intro, and where almost all of the code in this repo comes from



## General SQL notes
### Are DB Views, triggers, and functions useful for us?
Overall, no. All of these things are super useful for DB admins wanting to restrict access or anyone using raw SQL and want to DRY out their processes. However, we are already using a Node app to interact with our DB, so we don't really need them. Functions are the only one that *might* come in useful, but that would need testing.

### What are DB Views?
Essentially they are a way to make pseudo tables.

```sql
-- create the view
CREATE VIEW DetailsView AS
SELECT NAME, ADDRESS
FROM StudentDetails
WHERE S_ID < 5;

-- load the view
SELECT * FROM DetailsView;
```
So, again, if we were interacting with raw SQL, and didn't want to keep writing out a long, complicated query, we could wrap it into a view. Views can also have different user permissions, so it's a good way to hide certain columns from queries.

But, we don't interact with our DB in this way, we just write queries into Sequelize, and don't keep typing them out. We also do not have multiple users and sensitive data where we need to literally, programmatically hide only certain columns.

### What are DB triggers
These are basically hooks that enact certain queries whenever certain actions happen:

```sql
create trigger stud_marks
before INSERT
on
Student
for each row
set Student.total = Student.subj1 + Student.subj2 + Student.subj3, Student.per = Student.total * 60 / 100;
```

Here again we wouldn't see much benefit to this. It would be cleaner if any actions like this were built in the app level, since that's where all our queries originate from anyway. I do not see any benefit to using the hooks at the SQL level with more obscure syntax, vs just using the web layer where it's obvious.

### What are SQL functions
SQL has things like AVG, COUNT, and other functions, however you can define these on your own. The main benefit of them beyond being DRY is that they *can* perform faster executions. This is because they can essentially be cached and optimized differently than standard queries. We do use some *obscenely* complex queries, however they don't actually get used all that often, so the trade off of learning how to use functions with Sequelize AND how to use SQL in *general* probably wouldn't be worth it in our case. We could probably first just write better queries and see performance improvements anyway.