# DB setup notes

For setting up multi-environment Postgres database connections, using `docker-compose` for local test db

## Setting up local db with docker-compose

https://levelup.gitconnected.com/creating-and-filling-a-postgres-db-with-docker-compose-e1607f6f882f

code snippets (archived): https://web.archive.org/web/20220303160004/https://github.com/jdaarevalo/docker_postgres_with_data

> The official PostgreSQL Docker image https://hub.docker.com/_/postgres/ allows us to place SQL files in the `/docker-entrypoint-initb.d` folder, and the first time the service starts, it will import and execute those SQL files.

> In our Postgres container, we will find this bash script /usr/local/bin/`docker-entrypoint.sh` where each _.sh, \*\*.sql and _.\*sql.gz file will be executed

docker-compose.yaml

```yaml
version: "3.7"
services:
  postgres:
    image: postgres:10.5
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    logging:
      options:
        max-size: 10m
        max-file: "3"
    ports:
      - "5438:5432"
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
      # copy the sql script to create tables
      - ./sql/create_tables.sql:/docker-entrypoint-initdb.d/create_tables.sql
      # copy the sql script to fill tables
      - ./sql/fill_tables.sql:/docker-entrypoint-initdb.d/fill_tables.sql
```

**Note**: may not need the table creation/seed sql scripts if using ORM like knex to create/seed database

excerpt from `create_tables.sql`

```sql
-- Creation of country table
CREATE TABLE IF NOT EXISTS country (
  country_id INT NOT NULL,
  country_name varchar(450) NOT NULL,
  PRIMARY KEY (country_id)
);

-- Creation of city table
CREATE TABLE IF NOT EXISTS city (
  city_id INT NOT NULL,
  city_name varchar(450) NOT NULL,
  country_id INT NOT NULL,
  PRIMARY KEY (city_id),
  CONSTRAINT fk_country
      FOREIGN KEY(country_id)
	  REFERENCES country(country_id)
);
```

run `docker-compose up`

## Example of setting up multiple envs using knex ORM

Source: https://mherman.org/blog/test-driven-development-with-node/

knexfile.js

```javascript
module.exports = {
  test: {
    client: "pg",
    connection: "postgres://localhost/mocha_chai_tv_shows_test",
    migrations: {
      directory: __dirname + "/db/migrations",
    },
    seeds: {
      directory: __dirname + "/db/seeds/test",
    },
  },
  development: {
    client: "pg",
    connection: "postgres://localhost/mocha_chai_tv_shows",
    migrations: {
      directory: __dirname + "/db/migrations",
    },
    seeds: {
      directory: __dirname + "/db/seeds/development",
    },
  },
  production: {
    client: "pg",
    connection: process.env.DATABASE_URL,
    migrations: {
      directory: __dirname + "/db/migrations",
    },
    seeds: {
      directory: __dirname + "/db/seeds/production",
    },
  },
};
```

Make migrations

`knex migrate:make tv_shows`

db/migrations

```js
exports.up = function (knex, Promise) {
  return knex.schema.createTable("shows", function (table) {
    table.increments();
    table.string("name").notNullable().unique();
    table.string("channel").notNullable();
    table.string("genre").notNullable();
    table.integer("rating").notNullable();
    table.boolean("explicit").notNullable();
  });
};

exports.down = function (knex, Promise) {
  return knex.schema.dropTable("shows");
};
```

knex.js

```js
var environment = process.env.NODE_ENV || "development";
var config = require("../knexfile.js")[environment];

module.exports = require("knex")(config);
```

apply migrations to local dbs

```bash
knex migrate:latest --env development
knex migrate:latest --env test
```

Make seeds

`knex seed:make shows_seed --env development`

seeds/development

```js
exports.seed = function (knex, Promise) {
  return knex("shows")
    .del() // Deletes ALL existing entries
    .then(function () {
      // Inserts seed entries one by one in series
      return knex("shows").insert({
        name: "Suits",
        channel: "USA Network",
        genre: "Drama",
        rating: 3,
        explicit: false,
      });
    })
    .then(function () {
      return knex("shows").insert({
        name: "Game of Thrones",
        channel: "HBO",
        genre: "Fantasy",
        rating: 5,
        explicit: true,
      });
    })
    .then(function () {
      return knex("shows").insert({
        name: "South Park",
        channel: "Comedy Central",
        genre: "Comedy",
        rating: 4,
        explicit: true,
      });
    })
    .then(function () {
      return knex("shows").insert({
        name: "Mad Men",
        channel: "AMC",
        genre: "Drama",
        rating: 3,
        explicit: false,
      });
    });
};
```

apply seeds to dev db

```bash
knex seed:run --env development
```

## TDD

Do it.
