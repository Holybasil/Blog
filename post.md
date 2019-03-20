## You should know about GraphQL

> GraphQL 一种用于 API 的查询语言

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;正如官网上所列的，GraphQL 能够使客户端刚刚好获取到需要的数据，一次请求可以获取全部的数据，具备类型系统保证数据可靠性，提供强大的 API 查询工具，API 渐进改动更加友好，可以凌驾在现有的代码之上。

### 概览

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;首先我们得了解，GraphQL 是一种用于描述如何请求数据的语法，它主要由客户端的查询语言和服务端的 schema 以及解析器组成。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;它允许客户端自己选择所需要的数据，比如说我们需要做一个产品的 PC 端和手机端，但是在 list 展示时 PC 端需要的信息（字段）肯定是比手机端要多的，并且出于美观的要求，手机端需要展示 list 中 item 的概览信息，这样，两者需要的字段既有重复又有不同，服务端就需要出两套 API，这就比较难受了。但是如果用 GraphQL，客户端可以自己选择需要的字段，服务端会根据客户端的需求返回特定的结果。这也预示着，如果产品后期迭代的时候，API 版本不需要跟着迭代，只需要添加相应的 schema 和解析器就行了。这就是 GraphQL 的好处之一，服务器会声明好提供什么资源，客户端按需取用。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;接着，我们试想一个场景，我们需要获取当前用户的信息以及当前用户的朋友的信息。在 REST 中，我们会需要一次用户查询和 n 次用户的朋友查询，这就是比较经典的 n+1 问题。这样的查询很不符合前端优化的理念，并且拿到数据后还需客户端聚合数据进行展示。然而在 GraphQL 中，可以通过一次查询拿到所有的信息。因为当一次 GraphQL 查询请求发出之后，服务端接受这个请求，并遍历这个请求的所有字段，为每一个字段调用适当的解析器。所以在 GraphQL 查询中用户的 friends 字段虽然存着 id 数组，但是解析器会根据 id 拿到相应的用户信息，返回一个聚集用户信息的数组给客户端。这也意味着，每个字段都是可以有参数的。看到了吗，我们讲 n+1 变成了 1，并且客户端不需要知道 friends 的信息要怎么获取，也不需要担心数据获取、聚合数据和业务的复杂关系。这样，大部分的数据都在同一数据中心的服务器或者更直接，在同一服务器之间流动。显然，这些服务器之间的数据获取具有更低的延迟和更高的带宽，大大提高了客户端性能。同时说明了，GraphQL 更适合于基于图的数据结构的数据获取，这一技术出自于研究社交网络的 Facebook 也就不足为奇了。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但是在上面的场景中其实存在着一个数据库重复操作的隐患。试想一下，拿到当前用户所有朋友的信息，而朋友的信息中也包括爱好的信息，也就是对于每一个 User 对象的结构大概这样`{id:"", friends:[], likes: []}`，那么在每次 getFriendById 的时候，都会调用几次 getLikeById。这样显然会对数据库造成 1+n 的问题。而 Facebook 提供了 Dataloader 用来解决这种问题，简而言之就是，在每次 getFriendById 的时候，通过 load 函数（或者 loadMany）将 likes 的 id 缓存起来，也就是重复的 id 不会进入缓存。最后将这个 id 数组交给 Dataloader 的批处理函数，返回顺序一致且长度相等的结果数组。经过这样的缓存机制，对每个朋友的爱好查询就不是每个朋友都要进行一次查询，而是一次这些爱好 id 的并集查询。

### 核心思想

#### type 和 schema

一个 GraphQL 的查询就是在选择一个对象上的字段，所以我们需要事先在服务端定义所有客户端能够获取到的数据的 schema, 一个基本的 schema（数据类型）类似下面这样

```js
type User {
  id: ID!
  name: String!
  gender: Int
  todos(type: Type): [Todo]
  email: String!
  partners: [User]
}
```

这表示了，我们可以从服务器获取 User 对象，并且这个 User 对象有以上 6 个字段，在查询时，我们可以选择 User 下的任意字段返回，每个字段都被赋予了 type，这样对于查询结果的描述就更加具体。（并且 GraphQL 是严格执行类型系统的）

#### query 和 mutation

这两个是 graphQL 最常用的操作类型，query 可以类比 get，mutation 可以类比 post。（还有一个 subscription，是配合 websocket 向客户端推送消息的，暂不涉及）
一个 query 类似下面这样

```js
type Query {
  user(id: ID!): User
  users(gender: Int): [User]
}
```

这表示我们可以通过 id 查询 User，其中参数 id 为必传项。也可以查询 users，得到一个由 User 组成的数组，gender 是可选参数。并且，当我们查询时，如果其返回值不是标量或者枚举型（特殊的标量 enum 限制可选值集合），那我们就需要指明想要从这个字段中获取的数据。
例如

```
query {
    user(id:"0"){
        name
        friend{
            name
        }
    }
}
```

由于 user 的类型为 User，不为标量，所以需指明其中需要的字段，如果当前字段仍不为标量，仍需指明其下的标量。也就是说，GraphQL 查询始终以标量值结束。

一个 mutation 类似下面这样

```js
mutation($todo: PostTodo) {
    addTodo(todo: $todo) {
        type
        content
    }
}
```

其中 PostTodo 是一个输入对象类型

```js
input PostTodo{
  owner: ID!
  content: String!
  type: Type
}
```

当传递的参数不为 graphQL 中规定的标量（Int，Float，String，Boolean，ID）而是一个复杂对象时，需要将这个数据类型的关键字从 type 改为 input。

需要注意的是，query 是并行执行的，而 mutation 是串行执行的。

#### Apollo

GraphQL 只是一套标准，而现在已经有很多语言对其支持。在服务端方面，Java，Javascript，Go 等等耳熟能详的后端语言都有了相应的服务端库。而在客户端方面，Relay 是 Facebook 出品，所以很不幸的仅支持 React。Apollo 是在 GraphQL 社区里最活跃的一个组织。客户端 Apollo Client 全面支持了 React，Vue，Angular 以及原生 Android 和原生 IOS，服务端 Apollo Server 可以作为一个独立的服务器，一个现有 REST 之上的中间层，或者直接处于 serverless 中。
![apollo.png](https://i.loli.net/2019/03/20/5c92011442f1d.png)

#### 改变
