# 第 16 章：API 开发实战

在前面的章节中，我们学习了 Rust 的各种核心概念和高级特性。现在，是时候将这些知识应用到实际项目中了。本章将带领你使用 Rust 构建一个完整的 RESTful API，展示如何利用 Rust 的优势开发高性能、安全可靠的 Web 服务。

**学习目标：**

- 理解 RESTful API 设计的核心原则
- 掌握 Actix-web 框架的基本用法
- 学习如何连接和操作数据库
- 实现认证和授权功能
- 编写 API 测试和生成文档

## 16.1 RESTful API 设计原则

RESTful API（Representational State Transfer）是一种流行的 Web API 设计风格，它基于 HTTP 协议，强调资源（Resources）的概念。在开始编写代码前，让我们先了解 RESTful API 的核心设计原则。

### 资源与 URI

RESTful API 将数据和功能视为资源，并使用 URI（统一资源标识符）进行标识：

- 资源通常使用名词表示（如 `/users` 而非 `/getUsers`）
- URI 应遵循层次结构（如 `/users/123/orders`）
- 使用复数名词表示集合（如 `/users` 而非 `/user`）

```
# 良好的 URI 设计
GET /api/users                # 获取所有用户
GET /api/users/42             # 获取特定用户
GET /api/users/42/orders      # 获取特定用户的所有订单
```

### HTTP 方法

RESTful API 充分利用 HTTP 方法表达对资源的操作：

| HTTP 方法 | 操作      | 描述                   | 示例                 |
| --------- | --------- | ---------------------- | -------------------- |
| GET       | 读取      | 检索资源，不应有副作用 | GET /api/users       |
| POST      | 创建      | 创建新资源             | POST /api/users      |
| PUT       | 更新/替换 | 替换现有资源           | PUT /api/users/42    |
| PATCH     | 更新/修改 | 部分更新资源           | PATCH /api/users/42  |
| DELETE    | 删除      | 删除资源               | DELETE /api/users/42 |

### 状态码与响应

使用标准 HTTP 状态码表示请求处理结果：

- 2xx：成功（如 200 OK，201 Created，204 No Content）
- 4xx：客户端错误（如 400 Bad Request，404 Not Found）
- 5xx：服务器错误（如 500 Internal Server Error）

响应格式通常为 JSON 或 XML，并应包括足够的信息以便客户端理解：

```json
// 成功响应示例
{
  "status": "success",
  "data": {
    "id": 42,
    "name": "张三",
    "email": "zhangsan@example.com"
  }
}

// 错误响应示例
{
  "status": "error",
  "message": "用户不存在",
  "code": "USER_NOT_FOUND"
}
```

### 无状态与幂等性

RESTful API 应是无状态的，即每个请求包含处理该请求所需的所有信息，服务器不依赖会话信息。

某些操作（如 GET、PUT、DELETE）应满足幂等性，即多次重复执行产生的效果与执行一次相同，这有助于提高系统可靠性。

### 版本控制

为 API 提供版本控制，可以通过 URI 路径或请求头实现：

```
# URI 路径版本控制
GET /api/v1/users

# 请求头版本控制
GET /api/users
Accept: application/vnd.myapp.v1+json
```

## 16.2 使用 Actix-web 框架

### Actix-web 简介

[Actix-web](https://actix.rs/) 是 Rust 生态系统中最流行的 Web 框架之一，它基于 actor 系统构建，具有高性能、可扩展性和低内存占用的特点。

首先，创建一个新项目并添加依赖：

```bash
cargo new rust_api_server --bin
cd rust_api_server
```

在 `Cargo.toml` 中添加依赖：

```toml
[package]
name = "rust_api_server"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4.4.0"
actix-rt = "2.9.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
env_logger = "0.10.0"
log = "0.4"
```

### 创建简单的 API 服务器

现在，我们来创建一个基本的 API 服务器。编辑 `src/main.rs`：

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};
use serde::{Deserialize, Serialize};
use log::{info, error};

// 数据模型
#[derive(Serialize, Deserialize, Debug, Clone)]
struct User {
    id: u32,
    name: String,
    email: String,
}

// 响应模型
#[derive(Serialize)]
struct ApiResponse<T> {
    status: String,
    data: Option<T>,
    message: Option<String>,
}

// 获取所有用户
async fn get_users() -> impl Responder {
    // 这里应该从数据库获取，我们先使用硬编码数据
    let users = vec![
        User { id: 1, name: "张三".to_string(), email: "zhangsan@example.com".to_string() },
        User { id: 2, name: "李四".to_string(), email: "lisi@example.com".to_string() },
    ];

    let response = ApiResponse {
        status: "success".to_string(),
        data: Some(users),
        message: None,
    };

    HttpResponse::Ok().json(response)
}

// 获取特定用户
async fn get_user(path: web::Path<(u32,)>) -> impl Responder {
    let user_id = path.into_inner().0;

    // 模拟查找用户
    let user = if user_id == 1 {
        Some(User {
            id: 1,
            name: "张三".to_string(),
            email: "zhangsan@example.com".to_string()
        })
    } else {
        None
    };

    match user {
        Some(user) => {
            let response = ApiResponse {
                status: "success".to_string(),
                data: Some(user),
                message: None,
            };
            HttpResponse::Ok().json(response)
        }
        None => {
            let response = ApiResponse::<User> {
                status: "error".to_string(),
                data: None,
                message: Some("用户不存在".to_string()),
            };
            HttpResponse::NotFound().json(response)
        }
    }
}

// 创建新用户
#[derive(Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

async fn create_user(user: web::Json<CreateUserRequest>) -> impl Responder {
    // 模拟用户创建（实际应存入数据库）
    let new_user = User {
        id: 3, // 模拟自动生成 ID
        name: user.name.clone(),
        email: user.email.clone(),
    };

    info!("创建新用户: {:?}", new_user);

    let response = ApiResponse {
        status: "success".to_string(),
        data: Some(new_user),
        message: None,
    };

    HttpResponse::Created().json(response)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // 初始化日志
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));

    info!("启动 API 服务器在 http://127.0.0.1:8080");

    // 启动 HTTP 服务器
    HttpServer::new(|| {
        App::new()
            .service(
                web::scope("/api")
                    .route("/users", web::get().to(get_users))
                    .route("/users/{id}", web::get().to(get_user))
                    .route("/users", web::post().to(create_user))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

运行服务器：

```bash
RUST_LOG=info cargo run
```

现在你可以使用 HTTP 客户端（如 curl 或 Postman）测试 API：

```bash
# 获取所有用户
curl http://localhost:8080/api/users

# 获取特定用户
curl http://localhost:8080/api/users/1

# 创建新用户
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"王五","email":"wangwu@example.com"}'
```

### 路由与资源配置

Actix-web 提供了灵活的路由配置方式：

```rust
App::new()
    .service(
        web::scope("/api/v1")  // API 版本前缀
            .service(
                web::resource("/users")
                    .route(web::get().to(get_users))
                    .route(web::post().to(create_user))
            )
            .service(
                web::resource("/users/{id}")
                    .route(web::get().to(get_user))
                    .route(web::put().to(update_user))
                    .route(web::delete().to(delete_user))
            )
    )
```

### 提取请求数据

Actix-web 提供多种提取器，从请求中获取数据：

```rust
// 路径参数
async fn get_user(path: web::Path<(u32,)>) -> impl Responder {
    let user_id = path.into_inner().0;
    // ...
}

// 查询参数
#[derive(Deserialize)]
struct UserQuery {
    limit: Option<usize>,
    offset: Option<usize>,
}

async fn get_users(query: web::Query<UserQuery>) -> impl Responder {
    let limit = query.limit.unwrap_or(10);
    let offset = query.offset.unwrap_or(0);
    // ...
}

// JSON 请求体
#[derive(Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

async fn create_user(user: web::Json<CreateUserRequest>) -> impl Responder {
    // ...
}
```

### 应用状态管理

对于需要在请求处理程序之间共享的数据（如数据库连接池），Actix-web 提供了应用状态管理：

```rust
// 定义应用状态
struct AppState {
    db_pool: DbPool,
    app_name: String,
}

// 在 main 函数中配置
let db_pool = create_db_pool().expect("Failed to create pool");
let app_state = web::Data::new(AppState {
    db_pool,
    app_name: "Rust API Server".to_string(),
});

// 在应用中注册
App::new()
    .app_data(app_state.clone())
    .service(...)

// 在处理程序中使用
async fn get_users(state: web::Data<AppState>) -> impl Responder {
    let conn = state.db_pool.get().expect("Failed to get DB connection");
    // 使用连接...
}
```

## 16.3 数据库连接与操作

实际的 API 服务器通常需要与数据库交互。在 Rust 中，有多种数据库驱动和 ORM 可供选择。本节我们将使用 Diesel ORM 与 PostgreSQL 数据库交互。

### 设置 Diesel 与 PostgreSQL

首先，在 `Cargo.toml` 中添加相关依赖：

```toml
[dependencies]
# ... 现有依赖
diesel = { version = "2.1.0", features = ["postgres", "r2d2", "chrono"] }
dotenv = "0.15.0"
chrono = { version = "0.4", features = ["serde"] }
r2d2 = "0.8"
```

你需要安装 Diesel CLI 来管理数据库迁移：

```bash
cargo install diesel_cli --no-default-features --features postgres
```

创建 `.env` 文件，配置数据库连接：

```
DATABASE_URL=postgres://username:password@localhost/rust_api_db
```

初始化 Diesel 并创建数据库：

```bash
diesel setup
```

### 创建模型和模式

使用 Diesel 创建数据库迁移：

```bash
diesel migration generate create_users
```

这将创建两个文件：`up.sql` 和 `down.sql`。编辑 `migrations/xxxx_create_users/up.sql`：

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR NOT NULL,
  email VARCHAR NOT NULL UNIQUE,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

编辑 `migrations/xxxx_create_users/down.sql`：

```sql
DROP TABLE users;
```

运行迁移：

```bash
diesel migration run
```

Diesel 会自动生成 `src/schema.rs` 文件，定义表结构：

```rust
// src/schema.rs (自动生成)
table! {
    users (id) {
        id -> Int4,
        name -> Varchar,
        email -> Varchar,
        created_at -> Timestamp,
        updated_at -> Timestamp,
    }
}
```

创建数据库模型 `src/models.rs`：

```rust
use crate::schema::users;
use chrono::NaiveDateTime;
use diesel::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Queryable, Serialize, Debug)]
pub struct User {
    pub id: i32,
    pub name: String,
    pub email: String,
    pub created_at: NaiveDateTime,
    pub updated_at: NaiveDateTime,
}

#[derive(Insertable, Deserialize)]
#[diesel(table_name = users)]
pub struct NewUser {
    pub name: String,
    pub email: String,
}

#[derive(AsChangeset, Deserialize)]
#[diesel(table_name = users)]
pub struct UpdateUser {
    pub name: Option<String>,
    pub email: Option<String>,
}
```

### 数据库连接配置

现在我们需要设置数据库连接池，这样我们可以在不同的请求处理程序中重用连接。创建 `src/db.rs`：

```rust
use diesel::pg::PgConnection;
use diesel::r2d2::{self, ConnectionManager};
use dotenv::dotenv;
use std::env;

pub type Pool = r2d2::Pool<ConnectionManager<PgConnection>>;
pub type DbConnection = r2d2::PooledConnection<ConnectionManager<PgConnection>>;

pub fn establish_connection_pool() -> Pool {
    dotenv().ok();

    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");

    let manager = ConnectionManager::<PgConnection>::new(database_url);

    r2d2::Pool::builder()
        .build(manager)
        .expect("Failed to create pool")
}
```

### 实现数据库操作

接下来，我们创建一个模块来实现对用户表的操作。创建 `src/users.rs`：

```rust
use diesel::prelude::*;
use diesel::result::Error;
use crate::models::{User, NewUser, UpdateUser};
use crate::schema::users;
use crate::db::DbConnection;

pub fn find_all(conn: &mut DbConnection) -> Result<Vec<User>, Error> {
    users::table
        .order(users::id.asc())
        .load::<User>(conn)
}

pub fn find_by_id(user_id: i32, conn: &mut DbConnection) -> Result<User, Error> {
    users::table
        .find(user_id)
        .get_result::<User>(conn)
}

pub fn create(new_user: NewUser, conn: &mut DbConnection) -> Result<User, Error> {
    diesel::insert_into(users::table)
        .values(&new_user)
        .get_result(conn)
}

pub fn update(user_id: i32, updated_user: UpdateUser, conn: &mut DbConnection) -> Result<User, Error> {
    diesel::update(users::table.find(user_id))
        .set(&updated_user)
        .get_result(conn)
}

pub fn delete(user_id: i32, conn: &mut DbConnection) -> Result<usize, Error> {
    diesel::delete(users::table.find(user_id))
        .execute(conn)
}
```

### 整合到 API 中

现在，我们需要修改 `main.rs` 以整合数据库功能：

```rust
mod schema;
mod models;
mod db;
mod users;

use actix_web::{web, App, HttpResponse, HttpServer, Responder, Result as ActixResult};
use diesel::result::Error as DieselError;
use log::{info, error};
use models::{User, NewUser, UpdateUser};
use serde::Serialize;

// 响应模型
#[derive(Serialize)]
struct ApiResponse<T> {
    status: String,
    data: Option<T>,
    message: Option<String>,
}

// 处理 Diesel 错误
fn handle_diesel_error(err: DieselError) -> HttpResponse {
    match err {
        DieselError::NotFound => {
            let response = ApiResponse::<()> {
                status: "error".to_string(),
                data: None,
                message: Some("资源不存在".to_string()),
            };
            HttpResponse::NotFound().json(response)
        },
        DieselError::DatabaseError(_, info) => {
            error!("数据库错误: {:?}", info);
            let response = ApiResponse::<()> {
                status: "error".to_string(),
                data: None,
                message: Some("数据库操作失败".to_string()),
            };
            HttpResponse::InternalServerError().json(response)
        },
        _ => {
            error!("未知数据库错误: {:?}", err);
            let response = ApiResponse::<()> {
                status: "error".to_string(),
                data: None,
                message: Some("服务器内部错误".to_string()),
            };
            HttpResponse::InternalServerError().json(response)
        },
    }
}

// 获取所有用户
async fn get_users(db_pool: web::Data<db::Pool>) -> ActixResult<HttpResponse> {
    let mut conn = db_pool.get()
        .expect("无法获取数据库连接");

    match users::find_all(&mut conn) {
        Ok(users_list) => {
            let response = ApiResponse {
                status: "success".to_string(),
                data: Some(users_list),
                message: None,
            };
            Ok(HttpResponse::Ok().json(response))
        },
        Err(e) => Ok(handle_diesel_error(e)),
    }
}

// 获取特定用户
async fn get_user(
    path: web::Path<(i32,)>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    let user_id = path.into_inner().0;

    let mut conn = db_pool.get()
        .expect("无法获取数据库连接");

    match users::find_by_id(user_id, &mut conn) {
        Ok(user) => {
            let response = ApiResponse {
                status: "success".to_string(),
                data: Some(user),
                message: None,
            };
            Ok(HttpResponse::Ok().json(response))
        },
        Err(e) => Ok(handle_diesel_error(e)),
    }
}

// 创建新用户
async fn create_user(
    user: web::Json<NewUser>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    let mut conn = db_pool.get()
        .expect("无法获取数据库连接");

    match users::create(user.into_inner(), &mut conn) {
        Ok(new_user) => {
            info!("创建新用户: id={}", new_user.id);
            let response = ApiResponse {
                status: "success".to_string(),
                data: Some(new_user),
                message: None,
            };
            Ok(HttpResponse::Created().json(response))
        },
        Err(e) => Ok(handle_diesel_error(e)),
    }
}

// 更新用户
async fn update_user(
    path: web::Path<(i32,)>,
    user: web::Json<UpdateUser>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    let user_id = path.into_inner().0;

    let mut conn = db_pool.get()
        .expect("无法获取数据库连接");

    match users::update(user_id, user.into_inner(), &mut conn) {
        Ok(updated_user) => {
            let response = ApiResponse {
                status: "success".to_string(),
                data: Some(updated_user),
                message: None,
            };
            Ok(HttpResponse::Ok().json(response))
        },
        Err(e) => Ok(handle_diesel_error(e)),
    }
}

// 删除用户
async fn delete_user(
    path: web::Path<(i32,)>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    let user_id = path.into_inner().0;

    let mut conn = db_pool.get()
        .expect("无法获取数据库连接");

    match users::delete(user_id, &mut conn) {
        Ok(count) if count > 0 => {
            Ok(HttpResponse::NoContent().finish())
        },
        Ok(_) => {
            let response = ApiResponse::<()> {
                status: "error".to_string(),
                data: None,
                message: Some("用户不存在".to_string()),
            };
            Ok(HttpResponse::NotFound().json(response))
        },
        Err(e) => Ok(handle_diesel_error(e)),
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // 初始化日志
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));

    // 创建数据库连接池
    let pool = db::establish_connection_pool();

    info!("启动 API 服务器在 http://127.0.0.1:8080");

    // 启动 HTTP 服务器
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .service(
                web::scope("/api")
                    .route("/users", web::get().to(get_users))
                    .route("/users/{id}", web::get().to(get_user))
                    .route("/users", web::post().to(create_user))
                    .route("/users/{id}", web::put().to(update_user))
                    .route("/users/{id}", web::delete().to(delete_user))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

现在，我们已经创建了一个与数据库交互的完整 RESTful API。你可以运行并测试它：

```bash
RUST_LOG=info cargo run
```

## 16.4 中间件与认证

在实际应用中，API 通常需要实现认证和授权功能，以确保只有授权用户才能访问特定资源。Actix-web 通过中间件系统支持这些功能。

### 中间件基础

中间件允许你拦截请求和响应，在处理程序执行前后执行代码。这对于实现日志记录、认证、压缩等功能非常有用。

让我们创建一个简单的日志中间件 `src/middlewares.rs`：

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures::future::LocalBoxFuture;
use log::info;
use std::{
    future::{ready, Ready},
    time::Instant,
};

// 性能日志中间件
pub struct Logger;

impl<S, B> Transform<S, ServiceRequest> for Logger
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = LoggerMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(LoggerMiddleware { service }))
    }
}

pub struct LoggerMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for LoggerMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let method = req.method().clone();
        let path = req.path().to_owned();
        let start_time = Instant::now();

        let fut = self.service.call(req);

        Box::pin(async move {
            let res = fut.await?;
            let elapsed = start_time.elapsed();

            info!(
                "{} {} - 状态: {} - 耗时: {:.2}ms",
                method, path, res.status().as_u16(),
                elapsed.as_secs_f64() * 1000.0
            );

            Ok(res)
        })
    }
}
```

### JWT 认证

一种常见的 API 认证方式是使用 JSON Web Tokens (JWT)。让我们为我们的 API 添加 JWT 认证。

首先，添加相关依赖到 `Cargo.toml`：

```toml
[dependencies]
# ... 现有依赖
jsonwebtoken = "8.3.0"
actix-web-httpauth = "0.8.0"
derive_more = "0.99"
```

创建认证模块 `src/auth.rs`：

```rust
use actix_web::{error::ErrorUnauthorized, Error};
use actix_web_httpauth::extractors::bearer::BearerAuth;
use chrono::{Duration, Utc};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::env;

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,      // 用户 ID
    pub role: String,     // 用户角色
    pub exp: i64,         // 过期时间
    pub iat: i64,         // 颁发时间
}

impl Claims {
    pub fn new(user_id: i32, role: &str) -> Self {
        let now = Utc::now();
        Claims {
            sub: user_id.to_string(),
            role: role.to_string(),
            exp: (now + Duration::hours(24)).timestamp(), // 24小时后过期
            iat: now.timestamp(),
        }
    }
}

// 生成 JWT
pub fn generate_token(user_id: i32, role: &str) -> Result<String, Error> {
    let secret = env::var("JWT_SECRET").unwrap_or_else(|_| "default_secret".to_string());
    let claims = Claims::new(user_id, role);

    encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(secret.as_bytes()),
    )
    .map_err(|e| {
        ErrorUnauthorized(format!("无法创建令牌: {}", e))
    })
}

// 验证 JWT
pub fn verify_token(token: &str) -> Result<Claims, Error> {
    let secret = env::var("JWT_SECRET").unwrap_or_else(|_| "default_secret".to_string());

    decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &Validation::default(),
    )
    .map(|data| data.claims)
    .map_err(|e| {
        ErrorUnauthorized(format!("无效的令牌: {}", e))
    })
}

// 从请求中提取并验证 JWT
pub async fn validate_token(auth: BearerAuth) -> Result<Claims, Error> {
    verify_token(auth.token())
}
```

创建认证相关的处理程序 `src/auth_handlers.rs`：

```rust
use actix_web::{web, HttpResponse, Responder, Result};
use diesel::prelude::*;
use log::info;
use serde::{Deserialize, Serialize};

use crate::db::DbConnection;
use crate::auth;
use crate::schema::users;

#[derive(Deserialize)]
pub struct LoginRequest {
    email: String,
    password: String,
}

#[derive(Serialize)]
pub struct LoginResponse {
    token: String,
    user_id: i32,
    name: String,
}

// 简化起见，我们没有实现密码哈希
pub async fn login(
    login_data: web::Json<LoginRequest>,
    db_pool: web::Data<crate::db::Pool>,
) -> Result<impl Responder> {
    let mut conn = db_pool.get()
        .expect("无法获取数据库连接");

    // 查找用户（实际应用中应验证密码哈希）
    let user_result = users::table
        .filter(users::email.eq(&login_data.email))
        .first::<crate::models::User>(&mut conn);

    match user_result {
        Ok(user) => {
            // 实际应用应验证密码
            // 这里简化处理，仅作为演示
            let token = auth::generate_token(user.id, "user")?;

            info!("用户登录成功: id={}", user.id);

            Ok(HttpResponse::Ok().json(LoginResponse {
                token,
                user_id: user.id,
                name: user.name,
            }))
        },
        Err(_) => {
            Ok(HttpResponse::Unauthorized().json(crate::ApiResponse::<()> {
                status: "error".to_string(),
                data: None,
                message: Some("无效的邮箱或密码".to_string()),
            }))
        }
    }
}
```

最后，更新 `main.rs` 以应用中间件和认证：

```rust
mod auth;
mod auth_handlers;
mod middlewares;
// ... 其他现有模块

use actix_web::{web, App, HttpResponse, HttpServer, Responder, Result as ActixResult};
use actix_web_httpauth::middleware::HttpAuthentication;
use actix_web::dev::HttpServiceFactory;

// ... 现有代码

// 受保护的路由组
fn protected_routes() -> impl HttpServiceFactory {
    web::scope("/protected")
        .wrap(HttpAuthentication::bearer(auth::validate_token))
        .route("/profile", web::get().to(get_profile))
}

// 获取当前用户个人资料
async fn get_profile(
    claims: web::ReqData<auth::Claims>,
    db_pool: web::Data<db::Pool>,
) -> ActixResult<HttpResponse> {
    let user_id = claims.sub.parse::<i32>().unwrap();

    let mut conn = db_pool.get()
        .expect("无法获取数据库连接");

    match users::find_by_id(user_id, &mut conn) {
        Ok(user) => {
            let response = ApiResponse {
                status: "success".to_string(),
                data: Some(user),
                message: None,
            };
            Ok(HttpResponse::Ok().json(response))
        },
        Err(e) => Ok(handle_diesel_error(e)),
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // ... 现有初始化代码

    // 启动 HTTP 服务器
    HttpServer::new(move || {
        App::new()
            .wrap(middlewares::Logger)  // 应用日志中间件
            .app_data(web::Data::new(pool.clone()))
            .service(
                web::scope("/api")
                    .route("/login", web::post().to(auth_handlers::login))
                    .route("/users", web::get().to(get_users))
                    .route("/users/{id}", web::get().to(get_user))
                    .route("/users", web::post().to(create_user))
                    .route("/users/{id}", web::put().to(update_user))
                    .route("/users/{id}", web::delete().to(delete_user))
                    .service(protected_routes())  // 添加受保护路由
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

现在，API 服务器包含基本的认证功能。用户可以登录并接收 JWT 令牌，然后使用该令牌访问受保护的路由。

## 16.5 API 测试与文档生成

高质量的 API 不仅需要良好的实现，还需要全面的测试和清晰的文档。本节将介绍如何为我们的 API 编写测试和生成文档。

### 集成测试

Actix-web 提供了测试工具，允许你在不启动实际服务器的情况下测试 API 端点。

创建测试目录和文件 `tests/api_tests.rs`：

```rust
use actix_web::{test, web, App};
use diesel::prelude::*;
use dotenv::dotenv;
use rust_api_server::{
    auth, auth_handlers, db, middlewares, models, schema, users,
    ApiResponse, handle_diesel_error, get_users, get_user, create_user, update_user, delete_user
};
use serde_json::{json, Value};
use std::{env, sync::Once};

// 确保 .env 文件只被加载一次
static INIT: Once = Once::new();

fn setup() {
    INIT.call_once(|| {
        dotenv().ok();
        env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));
    });
}

// 测试数据库工具函数
fn get_test_db_pool() -> db::Pool {
    let database_url = env::var("TEST_DATABASE_URL")
        .unwrap_or_else(|_| env::var("DATABASE_URL").expect("DATABASE_URL must be set"));

    let manager = diesel::r2d2::ConnectionManager::<diesel::PgConnection>::new(database_url);
    diesel::r2d2::Pool::builder()
        .build(manager)
        .expect("Failed to create test pool")
}

// 清理测试数据
fn clean_test_data(conn: &mut db::DbConnection) {
    use schema::users::dsl::*;
    diesel::delete(users)
        .execute(conn)
        .expect("Failed to clean test data");
}

// 创建测试用户
fn create_test_user(conn: &mut db::DbConnection) -> models::User {
    let new_user = models::NewUser {
        name: "测试用户".to_string(),
        email: "test@example.com".to_string(),
    };

    diesel::insert_into(schema::users::table)
        .values(&new_user)
        .get_result(conn)
        .expect("Failed to create test user")
}

#[actix_web::test]
async fn test_get_users() {
    setup();
    let pool = get_test_db_pool();
    let mut conn = pool.get().expect("Failed to get db connection");

    // 清理并准备测试数据
    clean_test_data(&mut conn);
    create_test_user(&mut conn);

    // 创建测试应用
    let app = test::init_service(
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .service(web::resource("/api/users").to(get_users))
    ).await;

    // 发送测试请求
    let req = test::TestRequest::get().uri("/api/users").to_request();
    let resp: Value = test::call_and_read_body_json(&app, req).await;

    // 验证响应
    assert_eq!(resp["status"], "success");
    assert!(resp["data"].is_array());
    assert!(resp["data"].as_array().unwrap().len() > 0);
}

#[actix_web::test]
async fn test_create_user() {
    setup();
    let pool = get_test_db_pool();
    let mut conn = pool.get().expect("Failed to get db connection");

    // 清理测试数据
    clean_test_data(&mut conn);

    // 创建测试应用
    let app = test::init_service(
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .service(web::resource("/api/users").to(create_user))
    ).await;

    // 准备测试数据
    let new_user = json!({
        "name": "新用户",
        "email": "new@example.com"
    });

    // 发送测试请求
    let req = test::TestRequest::post()
        .uri("/api/users")
        .set_json(&new_user)
        .to_request();

    let resp: Value = test::call_and_read_body_json(&app, req).await;

    // 验证响应
    assert_eq!(resp["status"], "success");
    assert_eq!(resp["data"]["name"], "新用户");
    assert_eq!(resp["data"]["email"], "new@example.com");
}

// 可以添加更多测试...
```

为了使测试可以访问我们的模块，我们需要调整 `src/lib.rs`，导出必要的项：

```rust
// src/lib.rs
pub mod schema;
pub mod models;
pub mod db;
pub mod users;
pub mod auth;
pub mod auth_handlers;
pub mod middlewares;

// 重新导出常用的项
pub use crate::models::{User, NewUser, UpdateUser};
pub use crate::users::{find_all, find_by_id, create, update, delete};

use actix_web::{web, HttpResponse, Responder, Result as ActixResult};
use diesel::result::Error as DieselError;
use log::{info, error};
use serde::Serialize;

// 响应模型
#[derive(Serialize)]
pub struct ApiResponse<T> {
    pub status: String,
    pub data: Option<T>,
    pub message: Option<String>,
}

// 处理 Diesel 错误
pub fn handle_diesel_error(err: DieselError) -> HttpResponse {
    // 实现同前
}

// 导出处理函数
pub async fn get_users(db_pool: web::Data<db::Pool>) -> ActixResult<HttpResponse> {
    // 实现同前
}

pub async fn get_user(
    path: web::Path<(i32,)>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}

pub async fn create_user(
    user: web::Json<NewUser>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}

pub async fn update_user(
    path: web::Path<(i32,)>,
    user: web::Json<UpdateUser>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}

pub async fn delete_user(
    path: web::Path<(i32,)>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}
```

然后，我们需要更新 `src/main.rs` 以从库中导入这些项：

```rust
// src/main.rs
use rust_api_server::{
    auth, auth_handlers, db, middlewares,
    ApiResponse, handle_diesel_error, get_users, get_user, create_user, update_user, delete_user
};
use actix_web::{web, App, HttpResponse, HttpServer, Responder, Result as ActixResult};
use actix_web_httpauth::middleware::HttpAuthentication;
use log::info;

// ... 其余代码保持不变
```

运行测试：

```bash
cargo test
```

### API 文档生成

使用 Swagger/OpenAPI 规范可以为 API 生成交互式文档。在 Rust 中，我们可以使用 `utoipa` 库来实现。

首先，添加依赖到 `Cargo.toml`：

```toml
[dependencies]
# ... 现有依赖
utoipa = { version = "3.3.0", features = ["actix_extras"] }
utoipa-swagger-ui = { version = "3.1.3", features = ["actix-web"] }
```

接下来，我们为模型和 API 端点添加 OpenAPI 注解。更新 `src/models.rs`：

```rust
use crate::schema::users;
use chrono::NaiveDateTime;
use diesel::prelude::*;
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

#[derive(Queryable, Serialize, Debug, ToSchema)]
pub struct User {
    pub id: i32,
    pub name: String,
    pub email: String,
    #[schema(format = "date-time")]
    pub created_at: NaiveDateTime,
    #[schema(format = "date-time")]
    pub updated_at: NaiveDateTime,
}

#[derive(Insertable, Deserialize, ToSchema)]
#[diesel(table_name = users)]
pub struct NewUser {
    pub name: String,
    pub email: String,
}

#[derive(AsChangeset, Deserialize, ToSchema)]
#[diesel(table_name = users)]
pub struct UpdateUser {
    pub name: Option<String>,
    pub email: Option<String>,
}
```

更新 `src/lib.rs` 以添加 API 文档：

```rust
// ... 现有代码

use utoipa::{OpenApi, Modify};
use utoipa::openapi::security::{SecurityScheme, HttpBuilder, HttpAuthScheme};

#[derive(OpenApi)]
#[openapi(
    paths(
        get_users,
        get_user,
        create_user,
        update_user,
        delete_user,
        auth_handlers::login
    ),
    components(
        schemas(User, NewUser, UpdateUser, ApiResponse<User>, auth_handlers::LoginRequest, auth_handlers::LoginResponse)
    ),
    modifiers(&SecurityAddon),
    tags(
        (name = "users", description = "用户管理 API"),
        (name = "auth", description = "认证 API")
    )
)]
pub struct ApiDoc;

struct SecurityAddon;

impl Modify for SecurityAddon {
    fn modify(&self, openapi: &mut utoipa::openapi::OpenApi) {
        let components = openapi.components.as_mut().unwrap();
        components.add_security_scheme(
            "bearer_auth",
            SecurityScheme::Http(
                HttpBuilder::new()
                    .scheme(HttpAuthScheme::Bearer)
                    .bearer_format("JWT")
                    .description(Some("使用 JWT 令牌进行认证"))
                    .build()
            )
        );
    }
}

/// 获取所有用户
///
/// 返回系统中的所有用户列表。
#[utoipa::path(
    get,
    path = "/api/users",
    responses(
        (status = 200, description = "成功获取用户列表", body = inline(ApiResponse<Vec<User>>)),
        (status = 500, description = "服务器错误", body = inline(ApiResponse<()>))
    ),
    tag = "users"
)]
pub async fn get_users(db_pool: web::Data<db::Pool>) -> ActixResult<HttpResponse> {
    // 实现同前
}

/// 获取特定用户
///
/// 根据 ID 获取特定用户信息。
#[utoipa::path(
    get,
    path = "/api/users/{id}",
    params(
        ("id" = i32, Path, description = "用户 ID")
    ),
    responses(
        (status = 200, description = "成功获取用户", body = inline(ApiResponse<User>)),
        (status = 404, description = "用户不存在", body = inline(ApiResponse<()>)),
        (status = 500, description = "服务器错误", body = inline(ApiResponse<()>))
    ),
    tag = "users"
)]
pub async fn get_user(
    path: web::Path<(i32,)>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}

/// 创建新用户
///
/// 创建一个新用户并返回创建的用户信息。
#[utoipa::path(
    post,
    path = "/api/users",
    request_body = NewUser,
    responses(
        (status = 201, description = "用户创建成功", body = inline(ApiResponse<User>)),
        (status = 500, description = "服务器错误", body = inline(ApiResponse<()>))
    ),
    tag = "users"
)]
pub async fn create_user(
    user: web::Json<NewUser>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}

/// 更新用户
///
/// 更新指定 ID 的用户信息。
#[utoipa::path(
    put,
    path = "/api/users/{id}",
    params(
        ("id" = i32, Path, description = "用户 ID")
    ),
    request_body = UpdateUser,
    responses(
        (status = 200, description = "用户更新成功", body = inline(ApiResponse<User>)),
        (status = 404, description = "用户不存在", body = inline(ApiResponse<()>)),
        (status = 500, description = "服务器错误", body = inline(ApiResponse<()>))
    ),
    tag = "users"
)]
pub async fn update_user(
    path: web::Path<(i32,)>,
    user: web::Json<UpdateUser>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}

/// 删除用户
///
/// 删除指定 ID 的用户。
#[utoipa::path(
    delete,
    path = "/api/users/{id}",
    params(
        ("id" = i32, Path, description = "用户 ID")
    ),
    responses(
        (status = 204, description = "用户删除成功"),
        (status = 404, description = "用户不存在", body = inline(ApiResponse<()>)),
        (status = 500, description = "服务器错误", body = inline(ApiResponse<()>))
    ),
    tag = "users"
)]
pub async fn delete_user(
    path: web::Path<(i32,)>,
    db_pool: web::Data<db::Pool>
) -> ActixResult<HttpResponse> {
    // 实现同前
}
```

更新 `src/auth_handlers.rs` 以添加文档：

```rust
use actix_web::{web, HttpResponse, Responder, Result};
use diesel::prelude::*;
use log::info;
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

use crate::db::DbConnection;
use crate::auth;
use crate::schema::users;

#[derive(Deserialize, ToSchema)]
pub struct LoginRequest {
    pub email: String,
    pub password: String,
}

#[derive(Serialize, ToSchema)]
pub struct LoginResponse {
    pub token: String,
    pub user_id: i32,
    pub name: String,
}

/// 用户登录
///
/// 验证用户凭据并返回 JWT 令牌。
#[utoipa::path(
    post,
    path = "/api/login",
    request_body = LoginRequest,
    responses(
        (status = 200, description = "登录成功", body = LoginResponse),
        (status = 401, description = "登录失败", body = inline(crate::ApiResponse<()>))
    ),
    tag = "auth"
)]
pub async fn login(
    login_data: web::Json<LoginRequest>,
    db_pool: web::Data<crate::db::Pool>,
) -> Result<impl Responder> {
    // 实现同前
}
```

最后，在 `src/main.rs` 中集成 Swagger UI：

```rust
use rust_api_server::{
    auth, auth_handlers, db, middlewares, ApiDoc,
    ApiResponse, handle_diesel_error, get_users, get_user, create_user, update_user, delete_user
};
use actix_web::{web, App, HttpServer};
use actix_web_httpauth::middleware::HttpAuthentication;
use log::info;
use utoipa_swagger_ui::SwaggerUi;

// ... 其他代码

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // ... 现有初始化代码

    // 启动 HTTP 服务器
    HttpServer::new(move || {
        App::new()
            .wrap(middlewares::Logger)
            .app_data(web::Data::new(pool.clone()))
            // 添加 Swagger UI
            .service(
                SwaggerUi::new("/swagger-ui/{_:.*}")
                    .url("/api-docs/openapi.json", ApiDoc::openapi())
            )
            .service(
                web::scope("/api")
                    .route("/login", web::post().to(auth_handlers::login))
                    .route("/users", web::get().to(get_users))
                    .route("/users/{id}", web::get().to(get_user))
                    .route("/users", web::post().to(create_user))
                    .route("/users/{id}", web::put().to(update_user))
                    .route("/users/{id}", web::delete().to(delete_user))
                    .service(protected_routes())
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

现在，你可以启动服务器，并访问 `http://localhost:8080/swagger-ui/` 查看交互式 API 文档。

### 性能测试

最后，我们可以使用工具如 `wrk` 或 `Apache Bench` 对 API 进行性能测试：

```bash
# 使用 wrk 工具进行基准测试
wrk -t12 -c400 -d30s http://localhost:8080/api/users

# 使用 Apache Bench
ab -n 10000 -c 100 http://localhost:8080/api/users
```

这些测试可以帮助你了解 API 的性能特征，并进行优化。

## 练习与实践

完成以下练习，巩固本章所学知识：

### 练习 16.1: 扩展用户 API

**目标：** 为用户 API 添加更多功能。

**要求：**

1. 添加按名称搜索用户的功能
2. 实现分页功能，允许客户端指定页码和每页数量
3. 添加用户排序功能（按名称、创建时间等）
4. 为新功能编写测试和文档

### 练习 16.2: 实现产品管理 API

**目标：** 创建一个产品管理 API，与用户 API 集成。

**要求：**

1. 设计并实现产品数据模型
2. 创建产品 CRUD 操作的 API 端点
3. 实现用户购物车功能
4. 添加产品与用户之间的关联（如收藏、评论）
5. 使用中间件确保只有授权用户可以修改产品信息

### 练习 16.3: 完善认证系统

**目标：** 完善 API 的认证和授权系统。

**要求：**

1. 实现用户注册功能
2. 增加密码哈希和验证
3. 添加用户角色和权限系统
4. 实现刷新令牌机制
5. 为认证相关功能编写测试

## 小结

本章介绍了使用 Rust 和 Actix-web 框架构建 RESTful API 的完整过程。通过学习这些内容，你现在应该能够：

- 遵循 RESTful 设计原则设计 API
- 使用 Actix-web 框架构建高性能 Web 服务
- 集成数据库并实现数据持久化
- 实现认证和中间件功能
- 编写 API 测试并生成 API 文档

这些知识为你使用 Rust 构建企业级 Web 应用奠定了基础。Rust 的性能、安全性和可靠性使其成为后端服务开发的理想选择，而丰富的生态系统提供了构建完整解决方案所需的工具。

在下一章中，我们将探索如何使用 Rust 构建命令行工具，这是 Rust 的另一个强项领域。

## 扩展阅读

想要深入了解本章内容，推荐以下资源：

1. [Actix-web 官方文档](https://actix.rs/docs/) - 详细的框架文档和示例
2. [Diesel 指南](https://diesel.rs/guides/) - Rust ORM 的深入指南
3. [RESTful API 设计最佳实践](https://restfulapi.net/) - 全面的 RESTful API 设计指南
4. [JWT 认证指南](https://jwt.io/introduction/) - JWT 认证的完整介绍
5. [OpenAPI 规范](https://swagger.io/specification/) - API 文档标准
