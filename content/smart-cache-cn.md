+++
title = "GraphQL 客户端缓存的正确打开方式"
date = 2019-08-19
category = "GraphQL"

+++

一个设计精巧的缓存不仅可以让客户端在运行时变得更加高效，还可以极大地提升开发效率，并且减少各类数据不一致问题引发的 bug。而与其它 API 规范相比，GraphQL 和客户端缓存的结合可以让这些优势再次被放大。

<!-- more -->

在过去的半年中我们重新设计并实现了项目中的数据平面，最终形态是一个完全运行在浏览器中的 GraphQL 网关，并且随时可以迁移为独立运行的 NodeJS 服务。之后我还会写一些文章（或者公开我们的设计文档）来介绍这个过程中我们遇到的挑战与解决方案，而在本文中我会先介绍其中最有趣且最强大的部分：缓存。

在这个过程中分别会涉及客户端缓存如何解决 UI 工程中的问题、GraphQL 社区中的缓存方案以及**我们如何增强社区方案**。

所有的内容**不需要** GraphQL 相关使用经验，不过我们还是先花 2 分钟时间讲解 GraphQL 最基本的语法和类型系统，便于读者理解。

## 背景知识

### GraphQL

首先声明，一套**严格遵守最佳实践**的 RESTful API 也可以使用这套缓存方案大部分的功能，但是**任何**一套 GraphQL API（包含按照 Quick Start 攻略 5 分钟写出的）都可以发挥缓存的最大威力，后者在现实工作中的优势不言而喻。

言归正传，我们用一个最简单的 CMS 系统（只有作者和文章两类数据）熟悉 GraphQL。

```graphql
# 这是一段 GraphQL 的 schema 定义，可以理解为 API 文档，只不过在 GraphQL 中 schema 是运行时的一部分，因此也可以理解为“必须编写并且持续维护的 API 文档”。

# type 是用于声明类型的关键字，Query 则是一种特殊的类型，可以称之为 ROOT type，也可以理解为 API endpoint。GraphQL 中一般将查询类 API 都定义在 Query 下。
# 在这个示例中 Query 包含了 getUser 和 getPosts 两个 API，分别接受一个参数，以及有对应的返回值。如果用 RESTful 的方式表示，大概相当于两个这样的 endpoints：
# GET /users/:id
# GET /posts?page=:page
type Query {
  # getUser 是这个查询的名称，id: String! 则表示接受一个名为 id 的参数，类型是 String，! 代表该值不能为 null。User 的部分则是返回值的类型，之后我们会看到它的类型声明。
  getUser(id: String!): User!
  # [Post!]! 这样的类型声明代表值整体不能为 null，并且数组中的每一项也不能为 null。
  getPosts(page: Int!): [Post!]!
}

# Mutation 是另一个特殊的 ROOT type，一般将对数据有改动的 API 都定义在 Query 下。
# updatePost 和 deletePost 这两个 API 用 RESTful 的方式表示大概如下：
# PATCH /posts/:id
# DELETE /posts/:id
type Mutation {
  updatePost(id: String!, title: String!): Post!
  deletePost(id: String!): Post!
}

# User 和下方的 Post 都是普通的类型声明，GraphQL 的类型系统中只可以使用基本类型或声明的类型。
# User 类型对应的一段 JSON 数据如下：
# {
#   "id": "1",
#   "name": "Adam",
#   "posts": [
#     { "id": "1", "title": "Adam's post" }
#   ]
# }
type User {
  id: String!
  name: String!
  posts: [Post!]!
}

type Post {
  id: String!
  title: String!
}
```

在示例中可以看出 GraphQL 是一个强类型的 API 规范，相比 RESTful 不仅有明确的 endpoint，而且参数、返回值也都有明确地类型。RESTful API 想要提供同样的信息往往需要借助工具自动生成并且有一定的维护成本。

### Normalization

Normalization 一般指将各种形式的数据转化为统一标准格式的过程，这个过程对于保持数据一致性有很大帮助。以我们的 CMS 系统为例：

```json
// get user
{
  "id": "1",
  "name": "Adam",
  "posts": [{ "id": "1", "title": "Adam's post" }]
}[
  // get posts
  { "id": "1", "title": "Adam's post" }
]
```

Adam 的第一篇文章和我们从文章列表中获得的第一篇文章实际上是同一份数据，但是当客户端从不同的 API 以及数据层级中读取这篇文章时，它很难识别这一点（因为我们肯定不会对所有数据进行递归地深比较）。

Normalization 则是为同一份数据提供唯一的标识，方便我们更高效地进行识别。在以上示例中 `Post:1` 就是一个很好的唯一标识，通过数据类型结合数据 id 形成了不会冲突且容易识别的标识。

不过如何知道一段数据代表 `Post` 对于 RESTful API 来说不是一个容易的回答的问题，往往需要维护的 schema 信息来辅助判断。但是对于 GraphQL 来说一段数据的 typename 是规范的一部分。

## UI 中的数据同步问题

继续以我们的 CMS 系统为例，我们可能有以下几部分 UI：

- 所有文章列表
- 用户的文章列表
- 文章详情页面
- 用于编辑文章的编辑器

当用户在文章编辑器内修改了一篇文章的标题，我们期望以上所有 UI 中的对应文章标题都能及时、正确地被更新为新标题。

为了达到这一目标，我们一般会采用以下几种方式。

### 方式一：不缓存任何数据，在 UI 被创建时重新获取数据

绝大部分现代化的 UI 框架（例如 React，Vue 等）都会为 UI 组件提供“生命周期”，这样我们可以在组件被创建时获取它对应的数据。在上述场景中，编辑文章标题之后只要再次打开文章列表，UI 就会重新获取最新的文章列表，当然也包含了修改后的文章标题。

但是这种方式也可能导致问题，因为我们无法保证数据更新后所有依赖它的 UI 都会重新被创建，一些在修改前已经完成初始化的 UI 还会展示错误的旧数据。

### 方式二：修改数据后主动重新获取数据

在上述例子中，这一方式的逻辑如下：

```
编辑文章标题，之后：
	重新获取所有文章列表数据
	重新获取文章作者的文章列表数据
	重新获取文章详情页面数据
```

这一方式在逻辑上更为正确，但是在真实的应用中更容易导致维护性方面的问题。例如在后续的功能迭代中我们又增加了一个“最近被修改的文章列表”，那么我们就需要将它也增加到这个重新获取数据的列表中，这在有一定规模的项目中是极其容易出错的。

### 方式三：维护统一的数据存储

现代化的 UI 框架也都提供了响应式的数据存储（例如 redux，vuex）以解决这类问题。我们可以将所有文章数据放在统一的数据存储中，当文章标题被修改时我们只需要更新数据存储中的对应文章数据，对应的 UI 就能响应式地进行更新。

这一方式在逻辑上更为正确，但是想要在实践中完全做对并不容易。因为开发者需要在数据存储中正确地将他们的数据进行 normalization 才能保证更新数据的过程正确且高效。

这一过程通常意味着更多的模板代码以及抽象封装，因此在很多项目中我们最后会发现大家会采用以上三种方式的混合体来避免各式各样的数据同步 bug。

## apollo-client in-memory-cache

apollo-client 是 GraphQL 社区中使用最广泛的客户端之一，因为它不仅提供了基本的数据请求功能，还包含了一套完善的数据缓存机制让开发者可以轻松解决上述的数据同步问题。

如果我们通过 apollo-client 发出以下请求并获得对应数据结果

```graphql
# 查询第一页文章
query {
  getPosts(page: 1) {
    id
    title
  }
}
```

```json
// 查询结果
{
  "getPosts": [{ "id": "1", "title": "Adam's post" }]
}
```

**这次请求 apollo-client 也会在它的缓存中保存为以下结构**：

```js
{
  ROOT_QUERY: {
    getPosts(page: 1): [{id: "Post:1", typename: "Post"}]
  }
  Post:1: {id: "1", __typename: "Post", title: "Adam's post"}
}
```

当 UI 再一次发出同样的请求时，apollo-client 会优先通过以下方式查询是否命中缓存：

1. 进入 ROOT_QUERY
2. 查询是否有 `getPosts(page: 1)` 对应的结果，得到 `[{id: "Post:1", typename: "Post"}]`
3. 查询是否有 `Post:1` 对应的数据
4. 所有查询均命中时则按对应结构将缓存中的数据拼装为正确的结构返回给 UI

当然 apollo-client 还提供了多种网络策略让 UI 可以决定请求是否要从缓存中查询，不过这并不是本文要讨论的重点。

### apollo-client 缓存如何解决数据同步问题

和 redux，vuex 等方式类似，apollo-client 的数据缓存也是响应式的。发起数据请求的 UI 会**订阅**它所依赖的数据，当缓存中的数据更新时，依赖对应数据的 UI 会正确地更新到最新状态。

与以往的数据管理方案不同之处在于：

1. apollo-client 将这部分数据存储和网络请求紧密结合在了一起，并且完全内置。这意味着开发者不需要再在网络请求和数据存储之间编写额外的模板代码以及抽象封装。
2. 当发送数据更新时（例如编辑文章标题），apollo-client 也能够感知这一变化（因为我们通过它发出请求），并自动更新数据缓存而不需要编写额外代码。
3. apollo-client 通过 data object id 作为唯一标识进行 normalization。就像我们在 normalization 部分所说，GraphQL 要求数据是强类型的，因此用 typename 结合一个唯一 id 组成的 data object id （例如 Post:1）可以保证全局唯一，并且能让 apollo-client 从不同的 query 和 mutation 中识别相同的数据。

对应我们在上文中的例子，编辑文章标题之后所有包含这篇文章数据的 UI 都会更新至最新状态，这一过程既不需要编写额外的业务逻辑代码，也不需要发出网络请求。

### apollo-client 缓存的重大缺陷

许多开发者在初步尝试 apollo-client 之后都被它的数据缓存所提供的便利性深深吸引，但很快也会发现它的不足之处，并在发现社区迟迟没有解决方案后不得不选择放弃使用数据缓存。

在所有**更新**操作中 apollo-client 缓存都表现良好，但是对于**创建**和**删除**类的操作数据缓存就不那么美好了。

apollo-client 提供了两种方式用于解决创建和删除数据后的缓存更新：

- 直接读写缓存数据，将其调整到正确状态。
- 触发数据重新获取。

#### 直接读写缓存数据

在官方文档中用一个 Todo App 演示了第一种方式的操作步骤：

1. 从缓存数据中读取所有 Todo
2. 将新增的 Todo 加入到数组中
3. 将包含新 Todo 的数组写入缓存数据中

这一方式的局限性在于真实项目中的数据往往不会这么简单。例如我们可能有一个后端分页、排序、过滤后返回的 Todo 列表，当我们新增一项 Todo 时意味着**我们也要使用和后端一致的分页、排序、过滤逻辑判断是否应该将这项 Todo 插入缓存的列表中**。

这显然是不可行的，不仅因为我们不想重复维护一份和后端相同的业务逻辑，更因为一些复杂的逻辑判断我们无法在前端实现。

#### 触发数据重新获取

重新获取数据更像是一种倒退，因为我们需要维护某次创建或删除后要重新获取的数据列表。

并且重新获取数据还可能导致大量不必要的网络请求，想象一下这样的操作过程：

1. 用户查看文章列表，从第一页一直查看到第十页，缓存中会包含 10 条对应的查询记录。
2. 用户编写了一篇新文章，这时所有的缓存数据都失效了（文章都需要向后挪到一项）。
3. 触发数据重新获取，十页数据**立刻**重新请求，但此时我们实际需要马上更新的其实只有 UI 正在展示的第十页数据。

综合来看，我们需要的是一种更有效的**缓存失效机制**，能够从缓存中删除过期数据，并且只重新获取 UI 正在展示的数据。

## smart-cache-invalidation

缓存失效是 apollo-client 在社区中最长久也最受关注的问题之一，一些已有方案在易用性和通用性方面都无法满足我们的需求，因此我们重新实现了 [smart-cache-invalidation](https://github.com/Yuyz0112/smart-cache)，提供了简单易用的缓存失效方案，其效果可以参考[测试用例](https://github.com/Yuyz0112/smart-cache/blob/master/test/index.spec.ts)。

> 需要注意我们的实现使用了一些 apollo-client 的内部 API，仅在 2.6 版本进行过测试。之后我们会尝试将这部分工作贡献给社区，推进 apollo-client 缓存失效工作的进展。

smart-cache-invalidation 主要包含三个方面的实现：

1. 清理缓存中某一类型的（GraphQL type）数据或者某一特定数据。
2. 基于 GraphQL schema 生成静态的依赖关系，让清理结果更为正确。
3. 惰性重新获取被 UI 依赖所依赖的活跃数据。

### 缓存清理

smart-cache-invalidation 提供了三种清理 API：

```js
// 清理某一类型的数据，例如所有 Post 数据
client.deleteCache({ typeName: "Post" });

// 清理某一特定的数据，例如 id 为 1 的 Post
client.deleteCache({
  typeName: "Post",
  value: {
    __typename: "Post",
    id: "1",
  },
});

// 清理某一特定的 query
client.deleteCache({ query: "getPosts" });
```

这三种使用方式分别对应不同的使用场景：

1. 清理某一类型的数据是一种较为激进的方式，在创建或删除数据后都可以使用，它能够彻底的清理缓存中和这一类型相关的所有数据。大部分时候应该选择这一方式，因为它会提供最为正确的结果。
2. 清理某一特定的数据是一种较为保守的方式，在删除数据后可以使用，但是在[复杂场景](https://github.com/Yuyz0112/smart-cache/blob/master/test/index.spec.ts#L214)下它可能无法完全正确地工作。
3. 清理某一特定的 query，这种特殊的方式一般用于清理特殊类型的返回值，例如查询只返回字符串而不是类型数据。

smart-cache-invalidation 在清理的过程中会遍历缓存，以保证所有关联数据都被正确清理。并且由于我们支持从 schema 中识别类型之间的依赖关系，所以潜在的关联数据也能够被正确清理，下文中我们会进一步讲解其实现。

### 类型依赖关系

在我们的 CMS 示例中用户数据的 schema 如下：

```graphql
type User {
  id: String!
  name: String!
  posts: [Post!]!
}
```

假设我们有一个 UI 展示了用户名称及其文章数量，其中文章数量是通过获取用户数据中 posts 数组的长度得到。

当用户新发布一篇文章后，我们会清理所有 Post 相关的缓存，此时我们需要一种更高效、直接的方式知道 User 类型和 Post 类型存在依赖关系，当 Post 相关数据缓存失效时 User 数据也应该失效以便更新。

幸运的是我们从静态的 GraphQL schema 中就可以得到这一依赖关系，smart-cache-invalidation 提供了一个从 schema 中生成依赖关系的 CLI 工具，当然使用者也可以手动编写依赖关系并传入 smart-cache-invalidation 中。

### 惰性重新获取数据

在分析 apollo-client 重新获取数据的方式时我们已经提到：我们仅需要立刻重新获取被 UI 订阅的活跃数据，其余部分在缓存失效后无需立刻触发请求，可以等到 UI 重新创建时自行获取。

对于数据我们做了如下划分：

1. 总数据。“总数据”包含缓存中的数据和无缓存（例如 fetch-policy 为 no-cache）的数据，其中无缓存的数据我们在缓存失效后无需处理。
2. 缓存中的数据。“缓存中的数据”可以按其对应的 Query 状态分为活跃和非活跃两类，活跃 Query 即对应的 UI 仍然在订阅缓存中的数据，非活跃的 Query 即 UI 曾经订阅过缓存中的数据，但现在已经取消订阅。非活跃的 Query 我们也在缓存失效后也无需处理，因为他们会在下一次重新开始订阅时发现缓存未命中，而通过网络获取最新数据。
3. 缓存中的活跃 Query 数据。“缓存中的活跃 Query 数据”根据我们我们清除的缓存可以分为已失效和未失效两类，未失效即 Query 对应的数据在缓存中仍然完整，我们在缓存失效后也无需处理未失效的活跃 Query。
4. 缓存已失效的活跃 Query 数据。这也就是在清理缓存数据后我们会触发重新获取数据的部分。
