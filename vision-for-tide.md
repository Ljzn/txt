# 水涨船高：开放地构建一个模块化web框架

> Sep 11,2018 Aaron Turon

网络服务工作组今年地目标是在多个方面提升web开发的体验：提升类似async/await的基础设施，优化web相关的crates的生态，把这些组合在一起变成一个框架和一本书，名为“潮水(Tide)”。取“水涨船高”的意思，意在增进分享，兼容，并且提高所有rust的web开发和框架。

## web框架的角色

简而言之，web框架的目的是让你以一种语言“原生”的感觉来开发web应用。关于其确切的意思有很多争论，大多数语言都有不同路线的流行的web框架。

总之，主要的问题是解决以下的不匹配：

- 一个web应用最终需要通过HTTP，但是程序员通常更愿意用原生语言来思考核心的“业务逻辑”。最简单的例子，你想让核心逻辑直接使用整数和结构体来工作，而不是用生的字符串数据。这种割裂也体现在例如身份验证，解析URL，甚至是如何表现你的应用中的服务等方面。

- 以上功能中一个尤为重要的方面是处理序列化以及模板：在生的请求和回复的内容中，是什么类型的数据，它们如何与rust的类型连接起来？

- 许多web应用也要和数据库交互，“原生”和数据库的视角也有不匹配。类似于Diesel的对象关系映射是一种解决方式。

除了解决这些不匹配，web框架有时还有其它目的，例如：

- 默认提供良好的安全性。

- 一致的应用结构，使得从其它使用了这个框架的代码仓库迁移过来很简单。

- 标准化的解决方案，例如对于线程和连接池等等。

- 工作流用于快速生成代码。

这些很难！这需要非常多底层的库来做重复工作，以及仔细并符合人体工程学的设计，以达到不同需求的目标。

## Tide的愿景

Tide的目标有两点：

- 首先，增强那些提供了核心的web功能的“重复工作”库。例如http和url都是这方面很成熟的库。我们想要更多这种成熟的库！很大的一部分努力是在辨识并找到方式来提升，标准化，文档化这些库。

- 其次，要在这些库之上构建一个严肃的框架，最好是非常简洁的一层。无论我们是否想要使用现有的库，我们都想创造一个小的，独立的库，而不是一个巨大的整体的框架。

所有这些会集结成一本书，并提供底层库和框架的文档。

我们想以开放协作的方式完成。这片博客是一个开始，探索不同的设计问题并寻求反馈以及别的想法。如果你想参与到其中，加入WG-net-web的Discord频道吧！

## 开篇

文章的剩余部分是讨论一些核心的话题，来调查现有的生态。我们计划接下来稳定地发表一些文章，来提出问题，表达设想，并逐渐将框架开发作为一个社区项目。

### 基础服务API

大多数语言生态中都有一个提供关键web服务接口的库：Ruby有Rack，Python有WSGI，Java有Servlets，等等。

这些接口定义了什么是一个web服务器，通常看起来像是一种输入请求然后产生返回值的函数，尤其是HTTP。这些接口意味着你可以把web框架（帮助你生成这类函数）和底层的web服务器解耦（实际实现HTTP协议）。这也使得提供一个通用的底层中间件，可选任意的服务器和框架，成为可能。

现在大多数，但不是全部，的rust web框架都直接使用hyper来提供基础的HTTP功能，这阻止了解耦以及中间件应用。然而，tower-service准备提供一个更通用的接口，通过它的Service trait，而且在Tower项目中已经有了大量的中间件，尽管它还没有被发布到 crates.io 上。

简单来说，`Service`trait构建了一个异步的请求到回复的函数，请求和回复的类型是它们自己通用的。对于HTTP，它们可能就使用`http`包里的类型。这个trait也可以处理每个连接的状态（通过`NewService`工厂trait）以及背压（通过`ready`方法）。

这里有一些重要的提问：

- `Service`trait是在提议“使用async/await来借用”被提出之前设计的，所以可能会需要一些修改。特别地，在某些情况下你可能已经借用了数据，你本想不产生额外的复制就把它写入到一个回复中。尚不清楚现在是否有可能这样做。这个问题目前正在讨论。

- 尽管`http`包提供了基本的请求和回复类型，主体数据还是被通用地处理。所以，为了发布一个对于HTTP的`Service`的标准，我们还需要将这些类型（或者它们的限制条件）标准化，这意味着要检查数据流的主体。最近有一篇关于tower-web的文章提出一种基于新的`BufStream`trait的方式。

总之，rust的服务抽象的基础已经非常稳固，希望工作组可以帮助它们走向标准化，并且做出必要的提升。这个抽象也是Tide项目的起点。

### 路由策略

有了上面这些抽象，你可以把web框架看作是一种帮助你编写一个请求到回复的函数的方式。如本文开始所说的，一个框架所做的很大一部分工作是让你可以专注于用自然的方式编写核心业务逻辑。

大多数框架是从一个路由系统开始的，它将URL和HTTP请求映射到特定的包含业务逻辑的函数中 -- 通常称为 endpoints。路由通常也是其它HTTP相关业务发生的地方，例如合法性检查。因此，它可以定义一个框架是如何拼装起来的。

路由的方法趋向于这三种：

#### endpoint为中心

这是Rocket和Tower Web使用的方式。你“装饰”了一个标准的rust函数，装饰品中包括了将要映射到这个函数的URL的信息，以及如何解构各个参数，还有（对于Rocket）额外的检查。以下是Tower Web中的片段：

```rust
#[get("/")]
#[content_type("json")]
fn hello_world(&self) -> Result<HelloResponse, ()> { .. }
```

目的很明显：这种配置上手很简单，并且把所有重点都放在了用于实现endpoints的“原生rust”函数上。另一方面，这通常需要一个基于宏的实现，可能会比其它方式更难以扩展或自定义。

#### 内容表路由

这种方式被Gotham,Rouille和Actix Web所使用。你通过使用构造者风格或者基于宏的API，来构建独立于endpoints的路由逻辑。例如，在Gotham中，有一个确定的`Router`类型：

```rust
fn router() -> Router {
  build_simple_router(|route| {
    route.request(vec![Get, Head], "/").to(index);
    route.get_or_head("/products").to(products::index);
    route.get("/bag").to(bag::index);

    route.scope("/checkout", |route| {
      route.get("/start").to(checkout::start);

      route
        .post("/payment_details")
        .to(checkout::payment_details::create);

      route
        .put("/payment_details")
        .to(checkout::payment_details::update);

      route.post("/complete").to(checkout::complete)
    });
  })
}
```

这种“内容表”的形式使得应用的高层解构能够和endpoint的逻辑分离开。这也有助于处理多endpoint的情况，例如一系列的路由共享一个通用的合法性检查标准。另一方面，这比endpoint为中心的方式要更笨重，处理解构（从请求中拉出信息）也会更困难。

#### 自由结构

Warp采用了这种方式。这也被称为"无路由"方式。endpoint和应用的其它部分没有区别。相反，所有东西都是一个“筛子”，实际上是一个感知HTTP的函数，你通过组合筛子来构建你的应用。因此路由是用特定的筛子来处理的：

```rust
// Just the path segment "todos"...
let todos = wrap::path("todos");

// Combined with `index`, this means nothing comes after "todos".
// So, for example: `GET /todos`, but not `GET /todos/32`.
let todos_index = todos.and(wrap::path::index());

// `GET /todos`
let list = wrap::get2()
  .and(todos_index)
  .and(db.clone())
  .map(list_todos);

// `POST /todos`
let create = wrap::post2()
  .and(todos_index)
  .and(wrap::body::json())
  .and(db.clone())
  .and_then(create_todo);

// Combine out endpoints, since we want requests to match any of them:
let api = list
  .or(create);

// View access logs by setting `RUST_LOG=todos`
let routes = api.with(warp::log("todos"));
```

现在分析这种方式的优缺点还太早。

### Tide里的路由

那么哪种方式最适合Tide呢--能够平衡对于生态系统的益处，同时适合于模块化（这样就不仅限于Tide）。

目前，“内容表”的路由方式似乎是最合适的。这在其它语言里已经很常见，在rust里是首创--但可以很好地使用一些标准和API审查。和endpoint中心的路由相比，这更容易模块化和扩展。和自由结构相比，更容易计算开销。

为了深入讨论这个话题，在这个系列的下一篇文章中，我们将会呈现一个雏形，讨论在内容表路由中的API设计问题，并探索模块化和分享的方式。
