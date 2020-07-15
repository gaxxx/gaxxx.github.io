+++
title = "用rust写crud"
date = 2020-07-09
[taxonomies]
tags = ["rust", "crud"]
+++

这是rust系列的第一篇

虽然rust以其高性能著称，但是写简单crud的时候，我们还是喜欢用spring boot或者golang，
但是要用rust写，也不是不可以

具体代码在:
https://github.com/gaxxx/rust-crud


<!-- more -->

## 依赖
* postgresql
* rust
* vscode

## 初始化 (git checkout step0)

* 创建初始化项目
1. cargo new hero-api
2. 在cargo.toml添加dependencies
```
actix-web="2.0.0"
actix-rt = "1.0"
```
3. 试试初始化文件
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
4. 创建一个api.http,添加测试, 安装rest client插件
```
GET http://127.0.0.1:8088/hello/test
```


## 增加json支持 (git checkout json)

1. 在cargo.toml中添加serde
2. 定义一个结构,通过derive可以序列化和反序列化
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


## 增加数据库支持 (git checkout db)
1. 添加diesel,在Cargo.toml里添加
```
diesel = { version = "1.0.0", features = ["postgres"] }
dotenv = "0.9.0"
r2d2-diesel = "1.0.0"
```
2. 安装diesel客户端

```
cargo install diesel_cli --no-default-features --features postgres
```
3. 配置postgresql
```
# 创建数据库
sudo -u postgres psql
ALTER USER postgres PASSWORD 'password';
create database heroes;

# 将连接串写入.env文件
echo DATABASE_URL=postgres://postgres:password@127.0.0.1/heroes > .env


# 生成数据库表
disel setup 
// generage module tables
diesel migration generate heroes


# 如果失败需要在pg_hba.conf里面添加
host    all             all             127.0.0.1/32            md5

# 配置对应的数据库表，具体参考rust-crud的配置
# 生成diesel schema
diesel migration run

# 文件内容如下，diesel为这个结构生成了很多宏

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

5. 增加DB
```


# 从request拿到db
impl FromRequest for DB {
    type Error = ();
    type Future = LocalBoxFuture<'static, Result<Self, Self::Error>>;
    type Config = DB;

    fn from_request(req: &HttpRequest, _: &mut Payload<PayloadStream>) -> Self::Future {
        let config= req.app_data::<Self::Config>().unwrap();
        futures::future::ok(config.clone()).boxed_local()
    }
}

# 增加连接池
pub type Pool = r2d2::Pool<ConnectionManager<PgConnection>>;
#[derive(Clone)]
pub struct DB(pub Pool);
```

6. 在路由层添加db
```
# App对象加上db
App::new()
            .app_data(db::DB::default())

# 具体handler里面因为实现了FromRequest,可以直接拿到db
#[post("/")]
async fn create(hero: web::Json<hero::HeroInput>, db: db::DB) -> impl Responder {
    let created : hero::Hero = diesel::insert_into(schema::users::table)
        .values(hero.into_inner())
        .get_result(db.get().deref()).expect("Error creagint");
    println!("output is {:?}", created);
    web::Json(created)
}
```

7. diesel对CRUD都提供了相应的支持

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

## 提供分页支持 (git checkout master)

```
# 定义一个分页的trait
pub trait Paginate: Sized {
    fn paginate(self, start: i64) -> Paginated<Self>;
}

# 扩展查询结果，从返回一行数据，到返回一个（数据， 总数量)的tuple
impl<T: Query> Query for Paginated<T> {
    type SqlType = (T::SqlType, BigInt);
}

# 实现query

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

# 在函数处理里面这样使用
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


## 感想

1. diesel跟其他的orm不太一样，底层通过宏来实现的，对于各种query也比较完备
2. 性能没啥说的，就是干
3. 想起了刚开始撸golang的时候，那时候beego也刚出来,比特币还特别便宜...


## 参考
https://medium.com/sean3z/building-a-restful-crud-api-with-rust-1867308352d8








