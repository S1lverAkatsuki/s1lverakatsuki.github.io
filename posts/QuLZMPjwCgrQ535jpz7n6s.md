# 苏大科协网站后端重写开发笔记

[仓库地址](https://github.com/SudaKX/kxpage)，其中 `backend-rs` 文件夹下为重写内容。

## 编译期初始化 & `lazy_static!` 宏

`const` 关键字是拿来声明常量的，除了在代码中硬编码的值还可以用 `const fn` 表达式来进行一些简单的计算：

```rust
const FOO: i32 = 1 + 1;
```

但是 `const fn` 没法完成很复杂的操作，比如进行哈希计算：

```rust
const FOO: i32 = 42;
const BAR: String = {
    use sha2::{Digest, Sha512};
    let mut hasher = Sha512::new();
    hasher.update(FOO.to_be_bytes());
    format!("{:x}", hasher.finalize())
};
// ⨉ 不能编译，无法访问非常量函数
```

当然，可以用 `static` 来实现类似的功能。

```rust
const FOO: i32 = 42;

lazy_static! {
    static ref BAR: String = {
        use sha2::{Digest, Sha512};
        let mut hasher = Sha512::new();
        hasher.update(FOO.to_be_bytes());
        format!("{:x}", hasher.finalize())
    };
}
```

尽管说并不是真正的编译期计算，但仍然比一个动态的变量访问开销小。
当然，更推荐使用自带的 `std::sync::OnceLock` 或是 `std::sync::LazyLock`，更符合直觉：

```rust
use std::sync::LazyLock;

const FOO: i32 = 42;
static BAR: LazyLock<String> = LazyLock::new(|| {
    use sha2::{Digest, Sha512};
    let mut hasher = Sha512::new();
    hasher.update(FOO.to_be_bytes());
    format!("{:x}", hasher.finalize())
});
```

## `pub(crate)` 组织代码

在优化代码结构的时候会需要把一个文件里的代码拆分到不同的文件当中。

- **before**:

  ```rust
  // main.rs
  fn add(a: i32, b: i32) -> i32 {
      a + b
  }

  fn main() {
      println!("{}", add(1, 1));
  }
  ```

- **after**:

  ```rust
  // add.rs
  pub(crate) fn add(a: i32, b: i32) -> i32 {
      a + b
  }

  // main.rs
  mod add;

  use crate::add::add;

  fn main() {
      println!("{}", add(1, 1));
  }
  ```

`pub(crate)` 的目的为向整个 `crate` （这里就是项目内）内部公开，而不必干扰外部的接口。

## Rust 与 SQLite 交互

SQLite 是个很棒的数据库！

这里有几个模板可以用 `rusqlite` 库与 SQLite 数据库进行交互。

> 参考: [Crate rusqlite](https://docs.rs/rusqlite/latest/rusqlite/index.html)

### `Connection::execute`

```rust
use rusqlite::Connection;

fn create() -> Result<(), Box<dyn Error>> {
    let conn = Connection::open("a.db")?;
    // open 语句会根据提供的目录去找数据库文件，如果找不到会创建一个

    conn.execute("CREATE TABLE IF NOT EXISTS users (
                id VARCHAR(36) PRIMARY KEY,
                name VARCHAR(10),
                age INTEGER
                )", ())?;  // 现场编译现场跑

    Ok()
}
```

`execute` 语句用在执行一次且不返回结果的语句更简洁，比如这里的 `CREATE`，以及 `INSERT`。
每次调用都会重新编译 SQL 语句。

### `Connection::prepare` + `Statement::query`

使用上面创建的 `users` 表作为样例。

```rust
use rusqlite::Connection;

struct User {
    id: String,
    name: String,
    age: u8
}

fn select_as_age(age: u8) -> Result<(), Box<dyn Error>> {
    let conn = Connection::open("a.db")?;

    let stmt = conn.prepare("SELECT id, name, age
                            FROM users
                            WHERE age >= ?1")?;
    let filtrated_users = stmt.query_map([&age], |row| {
      Ok(User {
        id: row.get(0)?,
        name: row.get(1)?,
        age: row.get(2)?
      })
    })?;

    Ok(())
}

fn insert_users(users: Vec<User>) -> Result<(), Box<dyn Error>> {
    let conn = Connection::open("a.db")?;

    let stmt = conn.prepare("INSERT INTO users (id, name, age) VALUES (:id, :name, :age)")?;
    // Connection::prepare 支持具名参数

    for user in users {
      stmt.execute([user.id, user.name, user.age])?;  // 预编译好的语句
    }

    Ok(())
}

fn insert_user(user: User) -> Result<(), Box<dyn Error>> {
    let conn = Connection::open("a.db")?;

    conn.execute("INSERT INTO users (id, name, age) VALUES (?1, ?2, ?3)", [user.id, user.name, user.age])?;
    // Connection::execute 仅支持位置参数

    Ok(())
}
```

`Connection::prepare` 是预编译好的语句，在调用的时候不必再次编译。
在需要多次调用的时候（比如 `insert_users`）使用 1 + 1，在只要跑一次的时候（比如 `insert_user`）使用 `Connection::execute`

---

注意这里每个 `handler` 都打开了一遍数据库，这只是样例写法。
实际最好把数据库连接放进 Axum 的 `state` 里面，不然卡飞你。

**Tip**: 如果直接放会因为连接没有实现 `Send` Trait 导致必须要套锁才能访问，推荐使用 `r2d2_sqlite` 的连接池解决：

```rust
#[tokio::main]
async fn main() {
    let manager = r2d2_sqlite::SqliteConnectionManager::file("a.db");
    let pool = r2d2::Pool::new(manager).unwrap();

    let app = Router::new()
        // ...
        .with_state(pool); // 将 pool 注入 state

    // ...
}
```

## 后端项目组织

不得不说，Darksky 的项目组织能力非常强大。
他使用了一个 `StateResponse` 包装返回回去的所有状态文本。

对此进行扩展可得到一个后端处理函数的返回值 `Result<impl IntoResponse, (StatusCode, Vec<u8>)`。
值就不用管了，自己实现转请求去。
错误类型把状态码包进去，方便前端用 `response.ok` 之类的判断。
`Vec<u8>` 是因为这里走了 Protobuf，出去的是二进制信息。

在项目内可以用 `map_err` 把错误都丢给前端显示：

```rust
fn foo(...) -> Result<impl IntoResponse, (StatusCode, Vec<u8>) {
    // ...

    do_something().map_err(|e| {
        eprintln!("{}", e);
        let resp = StateResponse {
            message: "failure".to_string(),
        };
        (StatusCode::INTERNAL_SERVER_ERROR, resp.encode_to_vec())
    })?;

    // ...

    let resp = StateResponse {
        message: "success".to_string(),
    };

    // 成功的返回值，可以自定义返回的 http 请求头
    // 请求体同样是编码成 Protobuf 的二进制
    Ok((
        HeaderMap::from_iter([(
            header::CONTENT_TYPE,
            HeaderValue::from_static("application/octet-stream"),
        )]),
        resp.encode_to_vec(),
    ))
}
```

---

如果觉得每个业务函数都写这一坨麻烦，那可以用 `IntoResponse` Trait 优化一下写法：

```rust
// 定义自己的错误类型
enum AppError {
    Database(rusqlite::Error),
    Internal(anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AppError::Database(_) => (StatusCode::INTERNAL_SERVER_ERROR, "DB Error"),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "Server Error"),
        };
        (status, msg).into_response()
    }
}

// 这样业务函数就可以直接用 ? 了
async fn foo() -> Result<impl IntoResponse, AppError> {
    let _ = do_db_work()?;
    Ok(StatusCode::OK)
}
```

## `anyhow` 库与万用 `Result<T, E>`

`Result<T, E>` 的两个参数分别是成功时的结果和失败后的错误对吧。
但是这种情况呢：

```rust
fn main() -> Result<(), _> {
    some_func()?;  // 我不知道会出来什么样的错误
} // ⨉ 无法使用未指定的错误类型
```

有一个很无赖的解法：

```rust
fn main() -> Result<(), Box<dyn Error>> {
    some_func()?;
} // ✓ 可以过编
```

`dyn Error` 是标准库中 `std::error::Error` Trait 的错误 Trait 对象，`Box<dyn Error>` 可以容纳任何实现了 Error Trait 的错误类型。

---

当然如果出现需要多次使用 `Box<dyn Error>` 的情况，建议使用 `anyhow` 库，这就非常简单了：

```rust
use anyhow::Result;  // 注意这会顶掉标准库的 Result

fn main() -> Result<()> {
    some_func()?;
}
```

## Rust 使用 Protobuf

如果不知道什么是 Protobuf 的话，可以把它理解为一种二进制数据封装格式，比 JSON 更小更快（JSON 还要打包键名呢）

> 参考：[Protocol Buffers 文档](https://protobuf.com.cn/overview/)

### 1. 依赖

```toml
[dependencies]
prost = "0.14.3"

[build-dependencies]
prost-build = "0.14.1"
```

构建依赖是必须的，小心没法编译 `proto` 文件。

### 2. 写 Proto 定义文件

这就直接是官网的样例了：

```text
// a.proto

edition = "2023";   // 代表使用 Proto3 语法

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;
}
```

### 3. 在 `build.rs` 中编译

`a.proto` 只是个声明文件，想要真正使用得需要转成 `rs` 文件才行。

```rust
fn main() {
    std::fs::create_dir_all("src/pb").expect("Failed to create output directory");

    prost_build::Config::new()
        .out_dir("src/pb") // 设置proto输出目录
        .compile_protos(&["a.proto"], &["."]) // 要处理的proto文件
        .expect("Failed to compile proto files");
}
```

试着编译一次后会多一个文件 `a.rs`，这就是 `proto` 对应过来的数据结构。

```text
src
├── pb
│   └── a.rs
├── main.rs
└── ...
```

### 4. 在代码中使用

很简单，`include!` 就行了 *我在写 C++ 吗*

```rust
include!("pb/a.rs");

fn decode(data: Bytes) -> Result<(), Box<dyn Error
>> {
    // 数据都是对上的，里面的字段也能正常访问
    let person = Person::decode(data.as_ref())?;
    println!("{:?}", person);
    Ok(())
}

fn encode() -> Vec<u8> {
    let person = Person { name: "Foo".to_string(), id: 42, email: "bar@baz.com".to_string() };
    person.encode_to_vec()
}
```

---

如果觉得到处 `include!` 丑陋，这里有一个办法：

```rust
// src/pb.rs
pub mod a {
    include!("pb/a.rs");
}

// src/main.rs
mod pb;

use crate::pb::a::Person;  // 这样按需引入
```

## HTTP 响应状态码语义与场景

> 参考：[HTTP 响应状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Status)

就用这几种就行了：

- 什么问题都没有：`200 OK`
- `POST` 成功创建资源：`201 Created`
- 后端出问题：`500 Internal Server Error`
- 参数出问题：`400 Bad Request`
- 未登录：`401 Unauthorized`
- 权限不足：`403 Forbidden`
- 请求的不存在：`404 Not Found` *最熟悉的一集*

## 环境变量读取与使用 `.env` 文件

环境变量可以方便把一些需要经常更改的内容从源码中提出来。

```rust
env::var("USER_PWD").unwrap_or_else(|_| "1234567890".to_string());
```

这可以读取一个名为 `USER_PWD` 的环境变量，如果找不到就会用默认值返回。

---

当然，如果每个应用都要去 PATH 里瞎改一通未免也拉太多屎了，用 `.env` 文件才是更优雅的做法。

用库 `dotenv` 就行了，它能读取 `.env` 里的环境变量并临时注入到运行时的应用当中 *放心，这不是永久的*

- 项目根目录下的 `.env` 文件

  ```text
  USER_PWD=114514
  ```

- `src/main.rs`

  ```rust
  use dotenv::dotenv;

  fn main() {
      dotenv().ok();
      env::var("USER_PWD").unwrap();  // 114514
  }
  ```

`.env` 仅在开发环境使用，生产环境建议直接在系统里改了。

---

用 `envy` 库还可以写的更规则一些，带有类型定义的环境变量解析：

```rust
#[derive(Deserialize)]
struct Config {
    user_pwd: String
}

let config = envy::from_env::<Config>().unwrap();
let user_pwd = config.user_pwd;
```

## Axum 默认包大小与更改方法

在做测试的时候，发现一个诡异的现象，有一张 JPG 老传不上去，经过控制变量测试后我发现原来是图片太大了，2K 的图片呢。
然后发现 Axum 居然默认最大只能传 **2 MB** 的请求体，图片好几 MB 肯定传不上。
用下面的语句改的更大一些：

```rust
let app = Router::new()
    // ...
    .layer(axum::extract::DefaultBodyLimit::max(10 * 1024 * 1024)) // 10MB
    // ...
```

## Axum 的 `Query` 与 `Path` 提取器解析路径参数的区别

### `Query`

用于解析形如 `/api?id=114514` 这样的参数，通常是可选的，不需要在路由内声明：

```rust
#[derive(Deserialize)]
struct Params {
    id: Option<String>
}

async fn handler(Query(params): Query<Params>) -> StatusCode {
    let id = params.id.unwrap_or("42");
    println!("{}", id);
    StatusCode::Ok
}

#[tokio::main]
async fn main() {
    // ...
    let app = Router::new()
        // ...
        .route("/api", get(handler))
        // ...

    // ...
}
```

### `Path`

用于解析形如 `/api/114514` 这样的参数，用以声明唯一的值，需要在路由内声明：

```rust
async fn handler(Path(id): Path<String>) -> StatusCode {
    println!("{}", id);
    StatusCode::Ok
}

#[tokio::main]
async fn main() {
    // ...

    let app = Router::new()
        // ...
        .route("/api/:id", get(handler))
        // ...

    // ...
}
```

## 验证前端接口的方法

开了后端后如何验证它在工作？
最简单的就是写一个 JS 文件发请求：

```js
fetch("...")
  .then(data => console.log(data))
  .catch(error => console.error('Error:', error));
```

然后直接拿 node 跑。

这样太蠢了，还是用 `crul` 命令吧，Windows 和 Linux 都有。
