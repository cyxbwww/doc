# Egg.js 学习笔记

---



## 1. Egg.js 是什么

---

**Egg.js 为企业级框架和应用而生**，帮助开发团队和开发人员降低开发和维护成本

### 1.1 设计原则

Egg 没有选择社区常见框架的大集市模式（集成如数据库、模板引擎、前端框架等功能），专注于提供 Web 开发的核心功能和一套灵活可扩展的插件机制，通过 Egg，团队的架构师和技术负责人可以非常容易地基于自身的技术架构在 Egg 基础上扩展出适合自身业务场景的框架

Egg 的插件机制有很高的可扩展性，**一个插件只做一件事**（比如 [Nunjucks](https://mozilla.github.io/nunjucks) 模板封装成了 [egg-view-nunjucks](https://github.com/eggjs/egg-view-nunjucks)、MySQL 数据库封装成了 [egg-mysql](https://github.com/eggjs/egg-mysql)）。Egg 通过框架聚合这些插件，并根据自己的业务场景定制配置，这样应用的开发成本就变得很低



## 2. 基础功能

---

### 2.1 目录结构

```` 
egg-project
├── package.json
├── app.js (可选)
├── agent.js (可选)
├── app
|   ├── router.js
│   ├── controller
│   |   └── home.js
│   ├── service (可选)
│   |   └── user.js
│   ├── middleware (可选)
│   |   └── response_time.js
│   ├── schedule (可选)
│   |   └── my_task.js
│   ├── public (可选)
│   |   └── reset.css
│   ├── view (可选)
│   |   └── home.tpl
│   └── extend (可选)
│       ├── helper.js (可选)
│       ├── request.js (可选)
│       ├── response.js (可选)
│       ├── context.js (可选)
│       ├── application.js (可选)
│       └── agent.js (可选)
├── config
|   ├── plugin.js
|   ├── config.default.js
│   ├── config.prod.js
|   ├── config.test.js (可选)
|   ├── config.local.js (可选)
|   └── config.unittest.js (可选)
└── test
    ├── middleware
    |   └── response_time.test.js
    └── controller
        └── home.test.js
````

由框架约定的目录：

- `app/router.js` 用于配置 URL 路由规则，具体参见 [Router](https://www.eggjs.org/zh-CN/basics/router)
- `app/controller/**` 用于解析用户的输入，处理后返回相应的结果，具体参见 [Controller](https://www.eggjs.org/zh-CN/basics/controller)
- `app/service/**` 用于编写业务逻辑层，可选，建议使用，具体参见 [Service](https://www.eggjs.org/zh-CN/basics/service)
- `app/middleware/**` 用于编写中间件，可选，具体参见 [Middleware](https://www.eggjs.org/zh-CN/basics/middleware)
- `app/public/**` 用于放置静态资源，可选，具体参见内置插件 [egg-static](https://github.com/eggjs/egg-static)
- `app/extend/**` 用于框架的扩展，可选，具体参见[框架扩展](https://www.eggjs.org/zh-CN/basics/extend)
- `config/config.{env}.js` 用于编写配置文件，具体参见[配置](https://www.eggjs.org/zh-CN/basics/config)
- `config/plugin.js` 用于配置需要加载的插件，具体参见[插件](https://www.eggjs.org/zh-CN/basics/plugin)
- `test/**` 用于单元测试，具体参见[单元测试](https://www.eggjs.org/zh-CN/core/unittest)
- `app.js` 和 `agent.js` 用于自定义启动时的初始化工作，可选，具体参见[启动自定义](https://www.eggjs.org/zh-CN/basics/app-start)。关于`agent.js`的作用参见[Agent 机制](https://www.eggjs.org/zh-CN/core/cluster-and-ipc#agent-机制)

由内置插件约定的目录：

- `app/public/**` 用于放置静态资源，可选，具体参见内置插件 [egg-static](https://github.com/eggjs/egg-static)
- `app/schedule/**` 用于定时任务，可选，具体参见[定时任务](https://www.eggjs.org/zh-CN/basics/schedule)

**若需自定义自己的目录规范，参见 [Loader API](https://eggjs.org/zh-cn/advanced/loader.html)**

- `app/view/**` 用于放置模板文件，可选，由模板插件约定，具体参见[模板渲染](https://www.eggjs.org/zh-CN/core/view)
- `app/model/**` 用于放置领域模型，可选，由领域类相关插件约定，如 [egg-sequelize](https://github.com/eggjs/egg-sequelize)



### 2.2 框架内置基础对象

#### 2.2.1 Application

Application 是全局应用对象，在一个应用中，只会实例化一个

##### 获取方式

几乎所有被框架 [Loader](https://www.eggjs.org/zh-CN/advanced/loader) 加载的文件（Controller，Service，Schedule 等），都可以 export 一个函数，这个函数会被 Loader 调用，并使用 app 作为参数

```` javascript
// app.js
module.exports = (app) => {
  app.cache = new Cache()
};

// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    this.ctx.body = this.app.cache.get(this.ctx.query.id)
  }
}
````

在 Context 对象上，可以通过 `ctx.app` 访问到 Application 对象

```` javascript
// app/controller/user.js
class UserController extends Controller {
  async login() {
    this.ctx.body = this.ctx.app.cache.get(this.ctx.query.id)
  }
}
````

在继承于 Controller, Service 基类的实例中，可以通过 `this.app` 访问到 Application 对象

```` javascript
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    this.ctx.body = this.app.cache.get(this.ctx.query.id)
  }
}
````



#### 2.2.2 Context

Context 是一个**请求级别的对象**，在每一次收到用户请求时，框架会实例化一个 Context 对象，这个对象封装了这次用户请求的信息，框架会将所有的 [Service](https://www.eggjs.org/zh-CN/basics/service) 挂载到 Context 实例上

##### 获取方式

最常见的 Context 实例获取方式是在 [Middleware](https://www.eggjs.org/zh-CN/basics/middleware), [Controller](https://www.eggjs.org/zh-CN/basics/controller) 以及 [Service](https://www.eggjs.org/zh-CN/basics/service) 中，在 Service 中获取和 Controller 中获取的方式一样，在 [Middleware](https://www.eggjs.org/zh-CN/basics/middleware) 中获取 Context 实例的方式

```` javascript
// Koa v1
function* middleware(next) {
  // this is instance of Context
  console.log(this.query)
  yield next
}

// Koa v2
async function middleware(ctx, next) {
  // ctx is instance of Context
  console.log(ctx.query)
}
````



#### 2.2.3 Request & Response

Request 是一个**请求级别的对象**。封装了 Node.js 原生的 HTTP Request 对象，提供了一系列辅助方法获取 HTTP 请求常用参数

Response 是一个**请求级别的对象**。封装了 Node.js 原生的 HTTP Response 对象，提供了一系列辅助方法设置 HTTP 响应

##### 获取方式

可以在 Context 的实例上获取到当前请求的 Request(`ctx.request`) 和 Response(`ctx.response`) 实例

```` javascript
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    const { app, ctx } = this;
    const id = ctx.request.query.id
    ctx.response.body = app.cache.get(id)
  }
}
````

- [Koa](http://koajs.com/) 会在 Context 上代理一部分 Request 和 Response 上的方法和属性
- 如上面例子中的 `ctx.request.query.id` 和 `ctx.query.id` 是等价的，`ctx.response.body=` 和 `ctx.body=` 是等价的
- 需要注意的是，获取 POST 的 body 应该使用 `ctx.request.body`，而不是 `ctx.body`



#### 2.2.4 Controller

框架提供了一个 Controller 基类，并推荐所有的 [Controller](https://www.eggjs.org/zh-CN/basics/controller) 都继承于该基类实现。这个 Controller 基类有下列属性

- `ctx` - 当前请求的 [Context](https://www.eggjs.org/zh-CN/basics/objects#context) 实例
- `app` - 应用的 [Application](https://www.eggjs.org/zh-CN/basics/objects#application) 实例
- `config` - 应用的[配置](https://www.eggjs.org/zh-CN/basics/config)
- `service` - 应用所有的 [service](https://www.eggjs.org/zh-CN/basics/service)
- `logger` - 为当前 controller 封装的 logger 对象

在 Controller 文件中，可以通过两种方式来引用 Controller 基类

```` javascript
// app/controller/user.js

// 从 egg 上获取（推荐）
const Controller = require('egg').Controller
class UserController extends Controller {
  // implement
}
module.exports = UserController

// 从 app 实例上获取
module.exports = (app) => {
  return class UserController extends app.Controller {
    // implement
  }
}
````



#### 2.2.5 Service

框架提供了一个 Service 基类，并推荐所有的 [Service](https://www.eggjs.org/zh-CN/basics/service) 都继承于该基类实现，Service 基类的属性和 [Controller](https://www.eggjs.org/zh-CN/basics/objects#controller) 基类属性一致，访问方式也类似

```` javascript
// app/service/user.js

// 从 egg 上获取（推荐）
const Service = require('egg').Service
class UserService extends Service {
  // implement
}
module.exports = UserService

// 从 app 实例上获取
module.exports = (app) => {
  return class UserService extends app.Service {
    // implement
  }
}
````



#### 2.2.6 Helper

Helper 用来提供一些实用的 utility 函数。t它自身是一个类，有和 [Controller](https://www.eggjs.org/zh-CN/basics/objects#controller) 基类一样的属性，它也会在每次请求时进行实例化，因此 Helper 上的所有函数也能获取到当前请求相关的上下文信息

##### 获取方式

可以在 Context 的实例上获取到当前请求的 Helper(`ctx.helper`) 实例

```` javascript
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    const { app, ctx } = this
    const id = ctx.query.id
    const user = app.cache.get(id)
    ctx.body = ctx.helper.formatUser(user)
  }
}
````

##### 自定义 helper 方法

应用开发中，我们可能经常要自定义一些 helper 方法，例如上面例子中的 `formatUser`，我们可以通过[框架扩展](https://www.eggjs.org/zh-CN/basics/extend#helper)的形式来自定义 helper 方法

```` javascript
// app/extend/helper.js
module.exports = {
  formatUser(user) {
    return only(user, ['name', 'phone'])
  }
}
````



### 2.3 路由（Router）

Router 主要用来描述请求 URL 和具体执行动作的 Controller 对应关系

#### 2.3.1 定义 Router

`app/router.js` 里面定义 URL 路由规则

```` javascript
// app/router.js
module.exports = (app) => {
  const { router, controller } = app
  router.get('/user/:id', controller.user.info)
};
````

`app/controller` 目录下面实现 Controller

```` javascript
// app/controller/user.js
class UserController extends Controller {
  async info() {
    const { ctx } = this;
    ctx.body = {
      name: `hello ${ctx.params.id}`
    };
  }
}
````

当用户执行 `GET /user/123`，`user.js` 里面的 info 方法就会执行



#### 2.3.2 参数获取

##### Query String 方式

```` javascript
// app/router.js
module.exports = (app) => {
  app.router.get('/search', app.controller.search.index)
};

// app/controller/search.js
exports.index = async (ctx) => {
  ctx.body = `search: ${ctx.query.name}`
};

// curl http://127.0.0.1:7001/search?name=egg
````



##### 参数命名方式

```` javascript
// app/router.js
module.exports = (app) => {
  app.router.get('/user/:id/:name', app.controller.user.info)
};

// app/controller/user.js
exports.info = async (ctx) => {
  ctx.body = `user: ${ctx.params.id}, ${ctx.params.name}`
};

// curl http://127.0.0.1:7001/user/123/xiaoming
````



### 2.4 控制器（Controller）

Controller 负责**解析用户的输入，处理后返回相应的结果**，对用户的请求参数进行处理（校验、转换），然后调用对应的 [service](https://www.eggjs.org/zh-CN/basics/service) 方法处理业务，得到结果封装并返回

1. 获取用户通过 HTTP 传递过来的请求参数
2. 校验、组装参数
3. 调用 Service 进行业务处理，必要时处理转换 Service 的返回结果，让它适应用户的需求
4. 通过 HTTP 将结果响应给用户

定义的 Controller 类，会在每一个请求访问到 server 时实例化一个全新的对象，而项目中的 Controller 类继承于 `egg.Controller`，会有下面几个属性挂在 `this` 上

- `this.ctx`: 当前请求的上下文 [Context](https://www.eggjs.org/zh-CN/basics/extend#context) 对象的实例，通过它我们可以拿到框架封装好的处理当前请求的各种便捷属性和方法
- `this.app`: 当前应用 [Application](https://www.eggjs.org/zh-CN/basics/extend#application) 对象的实例，通过它我们可以拿到框架提供的全局对象和方法
- `this.service`：应用定义的 [Service](https://www.eggjs.org/zh-CN/basics/service)，通过它我们可以访问到抽象出的业务层，等价于 `this.ctx.service` 
- `this.config`：应用运行时的[配置项](https://www.eggjs.org/zh-CN/basics/config)



##### 2.4.1 获取 HTTP 请求参数

##### query

在 URL 中 `?` 后面的部分是一个 Query String，这一部分经常用于 GET 类型的请求中传递参数，我们可以通过 `ctx.query` 拿到解析过后的这个参数体

```` javascript
class PostController extends Controller {
  async listPosts() {
    const query = this.ctx.query
  }
}
````



##### queries

有时候我们的系统会设计成让用户传递相同的 key，例如 `GET /posts?category=egg&id=1&id=2&id=3`。针对此类情况，框架提供了 `ctx.queries` 对象，这个对象也解析了 Query String，但是它不会丢弃任何一个重复的数据，而是将他们都放到一个数组中

```` javascript
// GET /posts?category=egg&id=1&id=2&id=3
class PostController extends Controller {
  async listPosts() {
    console.log(this.ctx.queries)
    // {
    //   category: [ 'egg' ],
    //   id: [ '1', '2', '3' ],
    // }
  }
}
````

`ctx.queries` 上所有的 key 如果有值，也一定会是数组类型



##### body

虽然我们可以通过 URL 传递参数，但是还是有诸多限制

- [浏览器中会对 URL 的长度有所限制](http://stackoverflow.com/questions/417142/what-is-the-maximum-length-of-a-url-in-different-browsers)，如果需要传递的参数过多就会无法传递
- 服务端经常会将访问的完整 URL 记录到日志文件中，有一些敏感数据通过 URL 传递会不安全

一般请求中有 body 的时候，客户端（浏览器）会同时发送 `Content-Type` 告诉服务端这次请求的 body 是什么格式的。Web 开发中数据传递最常用的两类格式分别是 JSON 和 Form

框架内置了 [bodyParser](https://github.com/koajs/bodyparser) 中间件来对这两类格式的请求 body 解析成 object 挂载到 `ctx.request.body` 上。HTTP 协议中并不建议在通过 GET、HEAD 方法访问时传递 body，所以我们无法在 GET、HEAD 方法中按照此方法获取到内容

框架对 bodyParser 设置了一些默认参数，配置好之后拥有以下特性：

- 当请求的 Content-Type 为 `application/json`，`application/json-patch+json`，`application/vnd.api+json` 和 `application/csp-report` 时，会按照 json 格式对请求 body 进行解析，并限制 body 最大长度为 `100kb`。
- 当请求的 Content-Type 为 `application/x-www-form-urlencoded` 时，会按照 form 格式对请求 body 进行解析，并限制 body 最大长度为 `100kb`。
- 如果解析成功，body 一定会是一个 Object（可能是一个数组）。

一般来说我们最经常调整的配置项就是变更解析时允许的最大长度，可以在 `config/config.default.js` 中覆盖框架的默认值

```` javascript
module.exports = {
  bodyParser: {
    jsonLimit: '1mb',
    formLimit: '1mb'
  }
}
````



### 2.5 服务（Service）

Service 就是在复杂业务场景下用于做业务逻辑封装的一个抽象层，提供这个抽象有以下几个好处

- 保持 Controller 中的逻辑更加简洁
- 保持业务逻辑的独立性，抽象出来的 Service 可以被多个 Controller 重复调用
- 将逻辑和展现分离，更容易编写测试用例



#### 使用场景

- 复杂数据的处理，比如要展现的信息需要从数据库获取，还要经过一定的规则计算，才能返回用户显示。或者计算完成后，更新到数据库
- 第三方服务的调用，比如 GitHub 信息获取等



#### 属性

每一次用户请求，框架都会实例化对应的 Service 实例，由于它继承于 `egg.Service`，故拥有下列属性方便我们进行开发

- `this.ctx`: 当前请求的上下文 [Context](https://www.eggjs.org/zh-CN/basics/extend#context) 对象的实例，通过它我们可以拿到框架封装好的处理当前请求的各种便捷属性和方法
- `this.app`: 当前应用 [Application](https://www.eggjs.org/zh-CN/basics/extend#application) 对象的实例，通过它我们可以拿到框架提供的全局对象和方法
- `this.service`：应用定义的 [Service](https://www.eggjs.org/zh-CN/basics/service)，通过它我们可以访问到其他业务层，等价于 `this.ctx.service` 
- `this.config`：应用运行时的[配置项](https://www.eggjs.org/zh-CN/basics/config)



#### 注意事项

- Service 文件必须放在 `app/service` 目录，可以支持多级目录，访问的时候可以通过目录名级联访问
- 一个 Service 文件只能包含一个类， 这个类需要通过 `module.exports` 的方式返回
- Service 需要通过 Class 的方式定义，父类必须是 `egg.Service`
- Service 不是单例，是 **请求级别** 的对象，框架在每次请求中首次访问 `ctx.service.xx` 时延迟实例化，所以 Service 中可以通过 this.ctx 获取到当前请求的上下文



### 2.6 中间件（Middleware）

一般来说中间件也会有自己的配置。在框架中，一个完整的中间件是包含了配置处理的。我们约定一个中间件是一个放置在 `app/middleware` 目录下的单独文件，它需要 exports 一个普通的 function，接受两个参数

- options: 中间件的配置项，框架会将 `app.config[${middlewareName}]` 传递进来
- app: 当前应用 Application 的实例



#### 使用中间件

中间件编写完成后，我们还需要手动挂载，支持以下方式



##### 在应用中使用中间件

在应用中，我们可以完全通过配置来加载自定义的中间件，并决定它们的顺序。如果我们需要加载 gzip 中间件，在 `config.default.js` 中加入下面的配置就完成了中间件的开启和配置，该配置最终将在启动时合并到 `app.config.appMiddleware`

```` javascript
module.exports = {
  // 配置需要的中间件，数组顺序即为中间件的加载顺序
  middleware: ['gzip'],

  // 配置 gzip 中间件的配置
  gzip: {
    threshold: 1024, // 小于 1k 的响应体不压缩
  },
};
````



##### 在框架和插件中使用中间件

框架和插件不支持在 `config.default.js` 中匹配 `middleware`，需要通过以下方式

````javascript
// app.js
module.exports = (app) => {
  // 在中间件最前面统计请求时间
  app.config.coreMiddleware.unshift('report')
}

// app/middleware/report.js
module.exports = () => {
  return async function (ctx, next) {
    const startTime = Date.now()
    await next()
    // 上报请求时间
    reportTime(Date.now() - startTime)
  }
}
````

应用层定义的中间件（`app.config.appMiddleware`）和框架默认中间件（`app.config.coreMiddleware`）都会被加载器加载，并挂载到 `app.middleware` 上



##### router 中使用中间件

以上两种方式配置的中间件是全局的，会处理每一次请求。 如果你只想针对单个路由生效，可以直接在 `app/router.js` 中实例化和挂载，如下

```` javascript
module.exports = (app) => {
  const gzip = app.middleware.gzip({ threshold: 1024 })
  app.router.get('/needgzip', gzip, app.controller.handler)
}
````



##### 通用配置

无论是应用层加载的中间件还是框架自带中间件，都支持几个通用的配置项

- enable：控制中间件是否开启
- match：设置只有符合某些规则的请求才会经过这个中间件
- ignore：设置符合某些规则的请求不经过这个中间件



## Sequelize

---



