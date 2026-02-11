+++
title = "Building CRUD with Rust"
date = 2020-07-09
[taxonomies]
tags = ["rust", "crud"]
+++

This is the first article in the Rust series.

While Rust is known for its high performance, for simple CRUD we usually prefer Spring Boot or Go. But it's not impossible with Rust either.

Code available at:
https://github.com/gaxxx/rust-crud


<!-- more -->

## Dependencies
* postgresql
* rust
* vscode

## Initialization (git checkout step0)

* Create initial project
1. cargo new hero-api
2. Add dependencies to cargo.toml
```
actix-web="2.0.0"
actix-rt = "1.0"
```
3. Try the initial file
```
use actix_web::{web, App, Responder, HttpServer, get};

#[get("/{username}")]
async fn hello(user : web::Path<String> ) -> impl Responder {
    format!("Hello world {}", user.into_inner())
}


#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(move || {
        App::new()
            .service(
                web::scope("/hello")
                    .service(hello)
            )
    })
        .bind("127.0.0.1:8088")?
        .run()
        .await
}
```
4. Create an api.http file, add tests, install REST Client plugin
```
GET http://127.0.0.1:8088/hello/test
```


## Add JSON Support (git checkout json)

1. Add serde to cargo.toml
2. Define a struct - serialization and deserialization via derive:
```
#[derive(Deserialize, Debug, Clone)]
struct Info {
    username : String
}

#[get("/{username}")]
async fn hello(user : web::Path<Info> ) -> impl Responder {
    format!("Hello world {}", user.username)
}
```


## Add Database Support (git checkout db)
1. Add diesel to Cargo.toml:
```
diesel = { version = "1.0.0", features = ["postgres"] }
dotenv = "0.9.0"
r2d2-diesel = "1.0.0"
```
2. Install diesel CLI:

```
cargo install diesel_cli --no-default-features --features postgres
```
3. Configure PostgreSQL:
```
# Create database
sudo -u postgres psql
ALTER USER postgres PASSWORD 'password';
create database heroes;

# Write connection string to .env file
echo DATABASE_URL=postgres://postgres:password@127.0.0.1/heroes > .env


# Generate database tables
disel setup
// generate module tables
diesel migration generate heroes


# If it fails, add to pg_hba.conf:
host    all             all             127.0.0.1/32            md5

# Configure the database table, see rust-crud for details
# Generate diesel schema
diesel migration run

# File contents - diesel generates many macros for this structure

table! {
    users (id) {
        id -> Int4,
        name -> Varchar,
        identity -> Varchar,
        hometown -> Varchar,
        age -> Int4,
    }
}

```

5. Add DB:
```


# Get DB from request
impl FromRequest for DB {
    type Error = ();
    type Future = LocalBoxFuture<'static, Result<Self, Self::Error>>;
    type Config = DB;

    fn from_request(req: &HttpRequest, _: &mut Payload<PayloadStream>) -> Self::Future {
        let config= req.app_data::<Self::Config>().unwrap();
        futures::future::ok(config.clone()).boxed_local()
    }
}

# Add connection pool
pub type Pool = r2d2::Pool<ConnectionManager<PgConnection>>;
#[derive(Clone)]
pub struct DB(pub Pool);
```

6. Add DB to the router layer:
```
# Add DB to App
App::new()
            .app_data(db::DB::default())

# In handlers, DB is available directly since FromRequest is implemented
#[post("/")]
async fn create(hero: web::Json<hero::HeroInput>, db: db::DB) -> impl Responder {
    let created : hero::Hero = diesel::insert_into(schema::users::table)
        .values(hero.into_inner())
        .get_result(db.get().deref()).expect("Error creagint");
    println!("output is {:?}", created);
    web::Json(created)
}
```

7. Diesel provides support for all CRUD operations:

```
// create
diesel::insert_into(schema::users::table)
        .values(hero.into_inner())
// read
users.filter(id.eq(update_id.into_inner()))
        .first::<hero::Hero>(&*db.get());
// update
diesel::update(users.filter(id.eq(update_id.into_inner())))
        .set(hero.into_inner())
// delete
diesel::delete(users.filter(id.eq(update_id.into_inner())))


//list
users.load::<hero::Hero>(&*db.get()).unwrap();
```

## Add Pagination Support (git checkout master)

```
# Define a pagination trait
pub trait Paginate: Sized {
    fn paginate(self, start: i64) -> Paginated<Self>;
}

# Extend query results from returning a single row to returning a (data, total_count) tuple
impl<T: Query> Query for Paginated<T> {
    type SqlType = (T::SqlType, BigInt);
}

# Implement query

impl<T> QueryFragment<Pg> for Paginated<T>
    where
        T: QueryFragment<Pg>,
{
    fn walk_ast(&self, mut out: AstPass<Pg>) -> QueryResult<()> {
        out.push_sql("SELECT *, COUNT(*) OVER () FROM (");
        self.query.walk_ast(out.reborrow())?;
        out.push_sql(") t LIMIT ");
        out.push_bind_param::<BigInt, _>(&self.count)?;
        out.push_sql(" OFFSET ");
        let offset = self.start;
        out.push_bind_param::<BigInt, _>(&offset)?;
        Ok(())
    }
}

# Usage in the handler
#[get("/")]
async fn gets(req : web::Query<PageRequest> ,db : db::DB) -> impl Responder {
    use schema::users::dsl::*;
    use crate::db::pagination::Paginate;
    let query = users.order(id.asc());
    let query = query.paginate(req.start.unwrap_or(0).into());
    let all_users : Vec<_> = query.load::<(hero::Hero, i64)>(&*db.get()).unwrap();
    web::Json(all_users.iter().map(|v| v.0.clone()).collect::<Vec<_>>())
}
```


## Thoughts

1. Diesel is different from other ORMs - it's implemented through macros at the bottom level, with fairly complete support for various queries
2. Performance-wise, it just delivers
3. Reminds me of when I first started with Go, when beego had just come out and Bitcoin was still super cheap...


## References
https://medium.com/sean3z/building-a-restful-crud-api-with-rust-1867308352d8
