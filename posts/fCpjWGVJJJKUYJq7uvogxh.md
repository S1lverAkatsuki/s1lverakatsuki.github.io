# ScholarAI-backend 开发笔记

也即我参与 「2026 年软件工程实践」课程的记录。
没事找事与 *身为开发者的* 自傲作祟的产物，同时抱着写一个正经后端的想法开始了编写。
会随项目进度的进展而更新。

🦀 技术栈：``Axum` 作为后端大框架 + `sqlx` 库处理数据库交互

## 后端的分层架构

简单地写当然可以一个函数进去 `Request` 出来 `Response`，但是如果要往大了搞就得尽可能多拆一些功能出来。
不然混一起有你好受的。

我在这里使用的是三层架构，进来的数据流向是这样的：

```text
前端---(`Request`)---> 中间件 ---(处理后的 `Request`)---> `Handle` 层 ---(解析结构化的类型)---> `Service` 层 ---(数据合法的结构化类型)---> `Repository` 层 ---(拼接成的 SQL 语句)---> 数据库
```

出来就反过来就好咯。

---

拿一个用户注册的例子为准，后端有一个表：

```sql
CREATE TABLE "User" (
    email character varying COLLATE pg_catalog."default",
    password_hash character varying COLLATE pg_catalog."default",
)
```

不用考虑竞争和主键，这只会有下面一个用户。

前端发一个长这样的 JSON `POST` 请求去注册：

```json
{
  "email": "test@example.com",
  "password": "password123"
}
```

需要接收只有邮箱字段的 JSON 作为注册成功的返回值。

一个邮箱一个密码，这够简单了吧。

样例省略了 *thiserror 库提供的* 统一错误处理与 `AppState` 这个全局状态。
*我相信你一定看得懂的！看不懂去翻我仓库*

### `Handler` 层

接受前端发来的请求，吐出相应响应。 *谐音梗嘿嘿*
路由挂的就是这个函数。
会直接调用**全局状态**中的**服务对象**的对应方法。
如果有**与具体业务无关的**校验，可以放在这里进行，比如说密码长度。

---

```rust
pub async fn register(State(state): State<AppState>, Json(payload): Json<RegisterRequest>) -> Result<RegisterReturn, AppError> {
  payload.validate()?;  // 库 validator 可以给结构体字段标记上简单限定，然后用这个方法进行校验

  let response = state.user_service.register(payload).await?;

  Ok(Json(response))
}
```

### `Service` 层

接受 `Handle` 层发来的「可能不符合业务逻辑」的结构体，按业务逻辑校验后把合法的结构体送给 `Repository` 层。
这一层也是连接所有其他服务的地方，诸如 JWT 校验 / 生成的安全服务，哈希服务等等。甚至可以说下一层 `Repository` 是一个读写服务。
这层肯定是最麻烦的一个。

---

```rust
pub async fn register(&self, request: RegisterRequest) -> AppResult<RegisterSucceed> {
    let existing_user = UserRepository::find_user_by_email(&self.pool, &&request.email).await?;

    if existing_user.is_some() {
        return Err(AppError::EmailAlreadyExists);
    }

    let password_hash = self.security_service.hash_password(&request.password);
    let now = Local::now().date_naive();

    let user = User {
        email: request.email,
        password_hash,
    };

    let user = UserRepository::register(&self.pool, user).await?;

    Ok(RegisterSucceed { email: user.email })
}
```

注意到 `&self.pool` 没，这一层的字段是这样：

```rust
pub struct UserService {
    pool: Arc<PgPool>,
    security_service: Arc<SecurityService>,
}
```

含有连接池对象和用到的服务。

有一种设计哲学是说把连接池对象改成下一层 `Repository` 的实例，这样服务层就不要管具体的数据库实现了。
但我仍然选择保留连接池，在下一层用纯静态方法解决数据库访问。
除了实例套实例像是在写 Jvav，这样很难跨服务开事件的！*比如说注册送九块九，同时需要改用户与优惠券两个数据库*

### `Repository` 层

*它将直接与主对决*
负责把合法的字段填到 SQL 语句内，然后找个数据库对象，或是连接池什么的跑一遍吐出结果，再把结果 *这里稍后说* 返回。

---

```rust
pub async fn register(pool: &PgPool, user: User) -> AppResult<User> {
    let rec = sqlx::query_as::<_, User>(
        r#"
        INSERT INTO "User" (email, password_hash)
        VALUES ($1, $2)
        RETURNING email,
        "#,
    )
    .bind(&user.email)
    .bind(&user.password_hash)
    .fetch_one(pool)
    .await
    .map_err(|e| AppError::DatabaseError(e.to_string()))?;

    Ok(rec)
}
```

### 用到的数据结构

目前总共有三种，进来的请求，数据库实体的 Rust 映射，返回的响应。

```rust
// 省去各种 derive 派生 Trait 写的方便
pub struct RegisterRequest {
    pub email: String,
    pub password: String
}

pub struct User {
    pub email: String,
    pub password_hash: String
}

pub struct RegisterSucceed {
    pub email: String
}
```

### `Repository` 层的返回值

在样例的 **Create 操作**中，直接返回了创建完成表映射的对象 `User`。
也就导致在 `Service` 层中有两个一模一样的 `user` 对象，尽管说由于变量遮蔽可以正常跑，但不觉得奇怪吗？
而且 **RUD** 这剩下的三个对象还要返回什么呢。

我更期望的是后端中心而不是数据库中心的架构，也就是后端发给数据库的都是确定的值，连时间戳和 ID 都是后端提供的 *可能以前没有功能的 JSON 结构化文本存储写多了吧*

和大模型讨论一下决定如此返回（`Result::Ok()` 的值）：

- C：本来应该返回完整的数据库映射实体，综合我那种后端中心架构就返回 `()`
- R：返回 `Option` 或是 `Vec` 包裹的实体
- U：简单的返回 `()`，复杂的返回更新后的实体
- D：返回 `()` 或是受影响的行数（如果语法一切正常但没得东西删，这时候 `sqlx` 是不会报错的，必须由影响行数来判断删没删）

`()` 只是为了看看错误，没错误就一切正常。

### `Service` 层与更底层

实际上我认为，`Service` 层就是最底层了，剩下的 `Repository` 层就是一个和数据库操作的 `utils` 类似物，如果以后有别的服务，诸如文件解析或是 Bot 相关，那也是和 `Repository` 层同一地位的东西。

## Rust 函数传参类型与所有权转移

用泛型 `T` 来做样例：

- `T`：最简单的形参类型，代表函数持有实参的所有权
- `mut T`：和 `T` 没区别，在需要在函数内部对实参进行修改时可以少进行一步赋值到可变变量
- `&T`：按引用传递，不可变
- `&mut T`：可变引用，拿来改外部的值时使用，这是**唯一一个带有副作用**的参数
- `mut &T`：不可变地去借用外部变量，但是这个形参可以绑到别的不可变引用，没法改引用内对应的值
- `mut &mut T`：可变绑定到可变引用，什么都可以变

---

类比到 C++ 就是：

| 🦀 | C++ |
| -- | --- |
| `T`, `mut T` | `T` |
| `&T` | `const T&` |
| `&mut T` | `T&` |
| `mut &T` | `const T*` |
| `mut &mut T` | `T*` |

## Tracing 日志

> 参考：[Crate tracing](https://docs.rs/tracing/latest/tracing/)

一个日志，仅此而已。它会用很「日志」的风格打印出信息或是错误。
比如一个简单的提示当前服务已开启的 `INFO`：

```rust
use axum::{routing::get, Router};
use tracing_subscriber::FmtSubscriber;

#[tokio::main]
async fn main() {
    let subscriber = FmtSubscriber::builder()
        .with_max_level(Level::INFO)
        .finish();
    tracing::subscriber::set_global_default(subscriber).expect("无法设置 tracing 订阅器");

    let app = Router::new()
        .route("/", get(|| async { "Hello, World!" }));

    tracing::info!("服务已启动，监听地址 http://127.0.0.1:3000");

    axum::Server::bind(&"127.0.0.1:3000".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

库总共有五种级别的日志：`TRACE`，`DEBUG`，`INFO`，`WARN`，`ERROR`，最左边那个最详细。
它和 Axum 没有任何关系，但是还是配合的很不错，对吧。

## Rust 与 Postgre 交互

实践里用的库 `sqlx`，又有连接池又有迁移功能，比其他还得好几个 crate 拼一块才能实现的好多了，而且带了静态检查。

### 初始化与连接

```rust
use sqlx::postgres::PgPoolOptions;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let database_url = "...";   // 样例当然随便搞，实际建议拿 dotenv 读环境变量

    let pool = PgPoolOptions::new()
        .max_connections(5)     // 最大连接个数
        .connect(&database_url)
        .await?;

    // 接下来 pool 就是个哪里都能用的连接池了

    Ok(())
}
```

### CRUD

这里用一个用户表做样例：

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    active BOOLEAN NOT NULL DEFAULT TRUE
);
```

*直接丢 CREATE 语句比任何描述都准确，是吧*

---

sqlx 提供了两种宏：`query!`和 `query_as!` 构造一个 SQL 对象，当然也有用函数的写法。
这两个的区别在返回类型上。
返回一个或是数个得看后续的函数如何调用这个对象，`fetch_one` 与 `fetch_all`，如果可能找不到还得用 `fetch_optional`。
关于宏与函数的区别：

- 宏可以在编译期就能连接到数据库进行检查。它也支持绑定变量，但要求 SQL 模板必须是字符串字面量 `'&static str` 这种。
- 函数就刚好相反，拿运行时性能换来了灵活性。比如你想根据条件动态拼接一段 WHERE 子句。

---

- `query!` 与 `sqlx::query()`

  ```rust
  // 返回的是匿名结构体，可以直接用 SQL 语句中的键名访问
  // 写 SQL 的时候用 AS 取别名会很有用

  let row = sqlx::query!(
      "SELECT id, username, email FROM users WHERE id = $1",
      1
  )
  .fetch_one(&pool) // 只要一个就行了
  .await?;

  println!("User: {} <{}>", row.username, row.email);


  let target = 42;

  let rows = sqlx::query("SELECT username FROM users WHERE id = $1")
      .bind(target) // 需要运行时变化的只能用函数了
      .fetch_all(&pool) // 返回的是数组
      .await?;

  ```

- `query_as!` 与 `sqlx::query_as()`

  ```rust
  // 需要映射到自定义的结构体
  // 如果想要很「规范」的设计而且喜欢让各种层的结构体占满代码文件就用这种

  #[derive(sqlx::FromRow)] // 如果用函数，建议派生这个 Trait，不然访问的时候会非常麻烦
  struct User {
      id: i32,
      username: String,
      email: String,
      active: bool,
  }

  // 出的类型是 Result，找不到返回 sqlx 自己的错误
  let user: Result<User, sqlx::Error> = sqlx::query_as!(
      User,
      "SELECT id, username, email, active FROM users WHERE username = $1",
      "alice"
    )
    .fetch_optional(&pool)  // 可能找不到
    .await?;

  let target = "marisa";

  // 多行字符串很好用
  let user = sqlx::query_as::<_, User>(r#"
      SELECT id, username, email, active
      FROM users
      WHERE username = $1
      "#
    )
    .bind(target)
    .fetch_optional(&pool)
    .await?;

  
  // 如果没有给结构体派生 sqlx::FromRow，访问的时候会特别麻烦
  // println!("User: {} <{}>", user.try_get("username")?, user.try_get("email")?);
  // 但是如果用函数，它的限定已经强制结构体必须实现 sqlx::FromRow 了
  ```

### 更灵活的动态查询

```rust
let mut builder = sqlx::QueryBuilder::new("SELECT * FROM users WHERE 1=1");

if let Some(name) = name_filter {
    builder.push(" AND name = ").push_bind(name);
}

if let Some(age) = age_filter {
    builder.push(" AND age > ").push_bind(age);
}

let query = builder.build_query_as::<User>();
let users = query.fetch_all(&pool).await?;
```

如果对应的 `filter` 不是 `None` 就能自动添加对应的限定语句。

## Postgre 的特殊类型

### 枚举

> 参考：[枚举类型](https://postgres.ac.cn/docs/current/datatype-enum.html)

原本在 MySQL 中只能约定几个字符串往 `TEXT` 里存，但是现在可以用更有类型化的方式限定，以免后端写一半拼错的情况发生。
和其他语言的枚举一样：

```sql
CREATE TYPE sex AS ENUM ('male', 'female', 'heli');
```

之后就可以正常用了：

```sql
CREATE TABLE person (
    name text,
    sex sex
);
```

枚举很省空间，因为最后会被转成数字存储，只占 4B。
命名都是用小写下划线字母，无论是枚举类型还是枚举值。
枚举值是区分大小写的，所以 `'happy'` 和 `'HAPPY'` 是不同的。标签中的空格也很重要。

### JSON

> 本征：「有了这个 PG 就能超越任何 nosql 了。」

<br/>

> 参考：[JSON类型](https://postgres.ac.cn/docs/current/datatype-json.html)

有两种 JSON 类型，原版文本文件的 `json` 和编码后的二进制 `jsonb`。前者写的快查的慢，后者相反。前者会保留空格和字段顺序这种后者认为无关紧要的信息。

```sql
CREATE TABLE orders (
    id serial PRIMARY KEY,
    info jsonb
);

-- 插入只是和正常字符串一样，这里是 jsonb 类型也是会自动编码的
-- 打了几个换行符和空格更可读一些
INSERT INTO orders (info) VALUES 
('{
    "customer": "张三",
    "items": [
        {
            "name": "手机", 
            "price": 5000
        }
    ],
    "tags": ["电子", "数码"]
}');

-- 对 JSON 字段做 GIN 索引，这是做到 nosql 的核心
CREATE INDEX idx_info_gin ON orders USING GIN (info);
```

JSON 类型有特殊的读取符号：

- `->` 返回 JSON 对象（带引号的那种）
- `->>` 返回 文本/字符串（不带引号）
- `#>` 按路径获取 JSON 对象

```sql
-- 获取客户姓名
SELECT info->>'customer' FROM orders;

-- 获取第一个商品的名称
-- 第一层 items 是对象，所以可以继续提取
SELECT info->'items'->0->>'name' FROM orders;

-- 确保 JSON 里的 price 必须大于 0
ALTER TABLE orders ADD CONSTRAINT check_price 
CHECK ((info->'items'->0->>'price')::numeric > 0);
```

同样有自己的条件判定，`@>` 可查找左边的对象是否包括右边的内容。

```sql
-- 查询包含「电子」标签的所有订单
SELECT * FROM orders WHERE info->'tags' @> '["电子"]';
```

---

我在 ScholarAI 中用了一个枚举标记后面 JSON 的类型，然后按不同的方式读取，做到一种伪多态的效果。
*写的很爽呢，避免了 SQL 笑传之拆拆表*

## 靠 `mod.rs` 组织项目

> 参考：[【rust】rsut基础：模块的使用一、mod关键字、mod.rs文件的含义等](https://www.cnblogs.com/night-ride-depart/p/17111306.html)
> *怎么这都有错字的？*

当前项目目录如下：

```text
src
├── garden
│   ├── mod.rs
│   └── vegetable.rs
├── main.rs
└── utils.rs
```

目标是把 `utils.rs` 与 `garden/vegetable.rs` 导入到 `main.rs` 中使用。

```rust
// garden/vegetable.rs
pub fn vegetable_func() {
    println!("Hello from vegetable");
}


// utils.rs
pub fn utils_func() {
    println!("Hello from utils");
}
```

在 `garden/mod.rs` 下，其中声明的内容会被作为父文件夹 `garden` 模块的导出：

```rust
pub mod vegetable;

pub use vegetable::vegetable_func;
```

这样在 `main.rs` 内，就可以直接导入这俩使用了。

```rust
mod garden;
mod utils;

fn main() {
    vegetable_func();
    utils_func();
}
```

## Rust 测试的输出细节

这里有一个非常简单的测试：

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn test_add() {
        let ans = add(2, 3);
        assert_eq!(ans, 5);
        println!("{}", ans);
    }
}
```

用 `cargo test` 去运行会发现其并没有任何输出。
这是因为 Rust 会捕捉 stdout 里的内容，只有 stderr 里的才会有输出。
加上 `--no-caputre` 参数就好了：

```bash
cargo test -- --no-capture
```

## 集成测试与单元测试

用 `#[cfg(test)]` 在主要代码下组织的是单元测试，会随着代码本体的 crate 编译。
一般情况下会单独分离出来，这样方便改而且避免测试在本体里拉屎。

```rust
// src/utils.rs
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// src/main.rs
use utils::add;

fn main() {
    println!("{}", add(2, 3));
}
```

我们现在对它进行改造：

```text
my_crate
├── 其他配置项...
├── src
│   ├── utils.rs
│   ├── main.rs
│   └── lib.rs  // <- 多了
└── tests   // 专门的测试文件夹
    └── test_add.rs  // 测试代码
```

这种就是所谓集成测试，它们不会以内联模块的方式编译到 `main.rs` 这个二进制 crate 里，而是作为「外部 crate 使用者」来编译。
它只能测试库 crate，所以需要一个 `lib.rs` 导出。

```rust
// src/lib.rs
pub mod utils;
```

这样在测试里就能用了：

```rust
// tests/test_add.rs
use my_crate::utils::add; // 项目名叫 my_crate

#[test]
fn test_add_integration() {
    assert_eq!(add(2, 3), 5);
}
```

**Tip**：在 `lib.rs` 中保留内部独立可以减少导入树的长度：

```rust
// src/lib.rs
mod utils;
pub use utils::add;

// tests/test_add.rs
use my_crate::add;  // 少了中间层
```
