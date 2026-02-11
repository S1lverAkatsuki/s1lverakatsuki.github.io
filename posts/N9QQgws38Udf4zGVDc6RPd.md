# 非常简单地上手 Axum 后端

这个是对 Simply Writer 一大部分的总结单独提了出来，这也是我第一次写 Rust 后端呢，之前只是写 Tauri 这种套皮浏览器应用的「后端」。
由于涉及到前端怎么去调用后端接口，所以我也打了前端的 Tag。

自备依赖：

```toml
[dependencies]
axum = "0.8.8"
tokio = { version = "1.49.0", features = ["macros", "rt-multi-thread"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tower-http = { version = "0.6", features = ["cors", "fs"] }
```

## 最简单的样例

> 来自[官网](https://docs.rs/axum/latest/axum/)上的样例

```rust
use axum::{
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(|| async { "Hello, World!" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

无论是主函数还是处理函数，全是**异步**的，很显然你不想要在读写的时候卡死整个后端吧。

`Router::new()` 创建一个路由对象，后面的 `route` 方法就是把每一层的路由标注出来，对同一个路由只能用一行：

```rust
let app = Router::new()
    .route("/", get(|| async { "First" }))
    .route("/", get(|| async { "Second" }));
```

这里同时给 `/` 用 `route` 挂了两个处理函数，会直接 panic。
但是可以链式调用在一个 `route` 内挂两种不同的处理函数，见下文。

`route` 第一个参数为**路由**，第二个参数为请求类型。
请求类型分为 `GET` 与 `POST` 两种，一个从后端拿东西一个给后端送东西。
我们这里先不讨论 `POST`，后面展开说。
注意这里的闭包都是异步的，一般来说都是写成异步函数：

```rust
let app = Router::new()
    .route("/", get(get_handler));
```

```rust
async fn get_handler() -> &"static str {
    "Hello World!"
}
```

`/` 代表根路由，一般在上面挂 `index.html` 这种主页面，然后不同的子域名在挂不同的页面或是请求：

```rust
let app = Router::new()
    // 根路由显示主页
    .route("/", get(show_index))
    // 静态页面路由
    .route("/about", get(show_about))
    // API 路由负责提供服务，比如获取或是更新数据
    .route("/api/jobs", get(list_something).post(upsert_something))
    // 带有路径参数的 API 路由，这里 id 是参数
    .route("/api/user/:id", get(get_user_by_id));
```

`/api/jobs` 路由挂了俩请求，一个 `GET` 一个 `POST`，这是没问题的。

---

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
```

在 `0.0.0.0` 这个 IP 的 `3000` 端口申请监听服务，让走这个地址的所有 TCP 包都归后端处理。

**Tip**：监听 `0.0.0.0` 地址代表监听**所有** IP 的端口，注意与 `localhost` 区分开来 *计网课白上了*

如果这个**端口被占**了，那么 `unwrap` 就会 panic。

---

```rust
axum::serve(listener, app).await.unwrap();
```

正式启动服务，程序在这里无限循环，阻塞下面的部分。

## POST 请求处理

现在来写一个最简单的 `POST`，前端给后端发个字符串。

后端主函数，注意我改了服务的地址到了 `127.0.0.1:3000`。

```rust
#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", post(handler));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

// 先省略 handler 的内容
// 很显然你不能直接拿这个过编译
```

前端部分这样写：

```ts
const send = async () => {
  try {
    // fetch 地址是 API 接口
    // 后端服务直接挂在根路由上，所以也不用多一些 /api/... 之类的域名
    const response = await fetch("http://127.0.0.1:3000/", {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      // 哪怕是字符串，前后端交换的数据仍然是基于 JSON 的
      body: JSON.stringify("Hello Backend")
    });

    // 2xx 就是 ok
    if (response.ok) {
      const data = await response.json();
      console.log("收到后端返回：", data);
    }
  } catch (error) {
    // 注意这里不会处理 404，只会处理后端炸了或是 CORS 拦截错误
    console.error("请求失败：", error);
  }
};
```

错误处理写的很详尽。*我不是写教程一个 `try-catch` 就解决了，懒狼一条*

前端会给后端发一个 JSON *长得不太像，但仍然是合法的*：

```json
"Hello Backend"
```

---

后端需要把这个 JSON 接住解包（参数里的 `Json(text)`），然后状态码丢回前端。

```rust
async fn handler(Json(text): Json<String>) -> StatusCode {
  println!("{}", text);
  StatusCode::OK
}
```

这就是一个最简单的数据传递

## POST 传递 JSON 对象

考虑到大部分时候前后端传递的都是一个对象而不仅仅是一个字符串：

```ts
const response = await fetch("http://127.0.0.1:3000/", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  // 套了层括号就变成对象了
  // 记住这里的字段名 message
  body: JSON.stringify({ message: "Hello Backend" })
});
```

要是觉得一直写 fetch 麻烦，也可以用库[axios](https://www.axios-http.cn/)，能很容易写成 Tauri 的 `invoke` 那种很简单的样子：

```ts
const send = async () => {
  try {
    const res = await axios.post("http://127.0.0.1:3000/", {
        message: "Hello Backend"
    });
  } catch (error) {
    //  服务器响应了，但状态码超出了 2xx 范围 (比如说 404, 500, 403)
    if (error.response) {
      console.error("后端报错了：", error.response.status);
      console.error("后端返回的错误信息：", error.response.data);
      
      if (error.response.status === 403) {
        alert("你没有权限执行此操作");
      }
    } else if (error.request) {
      // 请求发出了，但没收到回应（CORS 拦截 / 后端炸了）
      console.error("服务器无响应");
    } else {
      // 在设置请求时就触发了错误 (例如写错 URL)
      console.error("请求设置错误：", error.message);
    }
  }
};
```

这时候前端给后端发的就是：

```json
{ message: "Hello Backend" }
```

---

```rust
#[derive(Deserialize, Serialize)]
struct Data {
    message: String,
    // 这里的字段名和前面传的一样
}

async fn handler(Json(payload): Json<Data>) -> StatusCode {
    println!("{}", payload.message);
    StatusCode::OK
}
```

这里必须定义一个和前端一模一样字段名的结构体 `Data`，它就是那个对象的映射。
结构体要实现序列化与反序列化。
之后就是简单替换参数就好了。

## 请求处理函数的返回值

你有没有注意到，我在上面的样例中写的返回值是不一样的，`GET` 处理函数返回的是字符串 `&'static str`，`POST` 处理返回的是状态码 `StatusCode`。
按理说任何一个后端给前端的东西都应该有状态码啊。

其实是因为 Axum 默认给一些基本类型（`&'static str`，`String`，`Json<...>`，`Html`，元组，`Result`）实现了 `IntoResponse` 这个 Trait，把内容转换成请求对象：
返回 `String` ，Axum 会自动调用其 `into_response` 方法，将字符串作为响应体，并将 `Content-Type` 标头设置为 `text/plain`
返回 `Json<...>` 时，Axum 会给你序列化，设置标头为 `application/json`。
返回元组，一般来说是第一个状态码，第二个内容，比如 `(StatusCode, Json<...>)`，
返回 `Result`，其实应该是 `Result<impl IntoResponse, impl IntoResponse>`，结果和错误都可以组成请求发给前端。

状态码默认都是 `200 OK`。

---

一般的思路：

1. 不需要返回数据 -> `StatusCode`
2. 需要返回数据 -> `Json<...>` （没有错误）/ `Result<Json<...>, StatusCode>`（有错误）
3. 需要自定义非 `200 OK` 的成功码（比如 `201 Created`）-> `(StatusCode, Json<T>)`

这里都传的 JSON 对象，除了写样例和返回 HTML 文件，最好不要裸传数据，哪怕就是传一个字符串。

## CORS

跨域资源调用。*很高深的名词对吧*
很简单的例子：在 `localhost:5173` 的网页去访问 `localhost:3000` 的后端就会出来 CORS 错误，因为系统可能认为你访问了不该访问的东西。
用 Axum 的中间件 [CorsLayer](https://docs.rs/tower-http/latest/tower_http/cors/index.html) 就可以解决：

```rust
let cors = CorsLayer::new()
    .allow_origin(Any)   // 允许任何域名
    .allow_methods(Any)  // 允许任何 HTTP 方法
    .allow_headers(Any); // 允许任何标头

let app = Router::new()
    .route(...)
    .layer(cors);
```

如果想要自定义域名：

```rust
let origins = [
    "http://localhost:3000".parse().unwrap(),
];

let cors = CorsLayer::new()
    .allow_origin(AllowOrigin::list(origins))
```

## Status

和 Tauri 的 `Manager` 几乎一模一样。

```rust
#[derive(Clone)]
struct AppState {
    count: Arc<Mutex<i32>>,
}

async fn get_handler(State(state): State<AppState>) -> Json<i32> {
    let count = state.count.lock().unwrap();
    Json(*count)
}

async fn set_handler(State(state): State<AppState>) -> Json<i32> {
    let mut count = state.count.lock().unwrap();
    *count += 1;
    Json(*count)
}
```

哦，还不是一模一样。
给 `AppState` 里面的字段包锁，而不是和 Tauri 那样给整个 `AppState` 包锁。因为这里是互联网后端，肯定有远超过一个的用户。难道我只读一个字段要锁住其他字段的读写吗？
由于 Axum 会把状态在不同对象间复制，所以需要实现 `Clone` Trait。
处理函数用和对 JSON 对象处理一样的解包拿到值。
主函数里面记得实例化和挂载：

```rust
let state = AppState {
    count: Arc::new(Mutex::new(42)),
};

let app = Router::new()
    .route(...)
    .with_state(state);
```
