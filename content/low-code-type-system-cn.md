+++
title = "穿越类型边界：在 TS、JSON schema 与 JS 运行时之间构建统一类型系统"
date = 2022-07-28
category = "Sunmao"

+++

近期我们开源了用于开发低代码工具的框架 [Sunmao（榫卯）](https://github.com/smartxworks/sunmao-ui)。在 Sunmao 中，我们为了提升多个场景下的开发、使用体验，设计了一套贯穿 TS（Typescript）、[JSON schema](https://json-schema.org/) 和 JS（Javascript）运行时的类型系统。

<!-- more -->

## 为什么 Sunmao 需要类型系统

首先要介绍一下 Sunmao 中的两项核心设计：

- 角色划分
- 可扩展性

**角色划分**是指 Sunmao 将使用者划分为了组件开发者与应用构建者两个角色。

组件开发者更加关注代码质量、性能以及用户体验等部分，并以此为标准创造出可复用的组件。当组件开发者以他们趁手的方式开发了一个新的组件，他们就可以将该组件封装为一个 Sunmao 组件并注册到组件库中。

应用构建者则选用已有的组件，并实现应用相关的业务逻辑。结合组件与 Sunmao 的平台特性，应用构建者可以更高效地完成这一工作。

之所以将角色进行划分，**是因为每时每刻都有应用被开发，但组件迭代的频率则低得多**。所以在 Sunmao 的帮助下，用户可以将组件开发的任务交给少量高级前端工程师或者基于开源项目完成，而将应用搭建的工作交由初级前端工程师、后端工程师、甚至是无代码开发经验的人完成。

**可扩展性**则是指大部分 Sunmao 组件代码并不维护在 Sunmao 内部，而是动态注册的。这也就要求 Sunmao 的 GUI 编辑器能够感知到各个组件可配置的内容，并呈现出合理的编辑器 UI。

### 应用元数据

不难看出，为了满足角色划分与可扩展性的需求，我们需要在不同角色之间维护一份元数据供双方协同，并最终将元数据渲染为应用。

<img src="/images/sunmao-metadata.png" width="500px" />

组件开发者在实现组件代码后，将可配置的部分定义为元数据的格式，应用构建者根据场景配置具体的元数据。

### 响应式状态管理

另一方面，Sunmao 中为了降低应用构建者的开发难度，设计了一套[高效的响应式状态管理机制](https://github.com/smartxworks/sunmao-ui/issues/30)，我们会在一篇单独的文章中分享它的设计细节。

眼下，可以简单的理解为 Sunmao 允许每个组件将自己的状态对外暴露，其余任意组件可以访问该状态并建立依赖关系，当状态发生变更时自动重新渲染。

例如一个 id 为 demo_input 的输入框对外暴露了`当前输入内容`这一状态，另一个 id 为 demo_text 的文本组件响应式的展示了当前输入内容的长度。

<img src="/images/sunmao-reactive.png" width="500px" />
<img src="/images/sunmao-reactive.gif" width="500px" />

### 类型与开发体验

因此，我们在 Sunmao 中为了提升不同角色的开发体验，就产生了以下类型需求：

- 应用元数据是有类型的。
  - GUI 编辑器可以根据元数据类型呈现出合理的编辑器 UI。
  - 基于元数据类型，可以对应用构建者配置的具体值进行校验。
- 组件对外暴露的状态是有类型的。
  - 应用构建者在使用这些状态时，可以获得编辑器补全等特性。
- Sunmao 组件 SDK 是有类型的。
  - 组件开发者定义元数据类型之后，实际使用 SDK 开发组件时应该获得类型保护，降低将组件接入 Sunmao 的成本。

考虑到可移植性、序列化能力以及生态，我们最终选择使用 JSON schema 描述元数据类型。

一个简化过后的输入框组件元数据定义如下：

```json
{
  "version": "demo/v1",
  "metadata": {
    "name": "input"
  },
  "spec": {
    "properties": {
      "type": "object",
      "properties": {
        "defaultValue": {
          "type": "string"
        },
        "size": {
          "type": "string",
          "enum": ["sm", "md", "lg"]
        }
      }
    },
    "state": {
      "type": "object",
      "properties": {
        "value": {
          "type": "string"
        }
      }
    }
  }
}
```

这段元数据描述了：

- 输入框接受 `defaultValue` 和 `size` 两种配置，用于指定输入框的初始值与大小。
- 输入框会将 `value` 状态对外暴露，用于访问输入框当前输入的内容。

基于 JSON schema 的元数据定义已经足够让 GUI 编辑器根据它呈现 UI。接下来的目标就是将 JSON schema 的类型定义复用到 Sunmao 组件 SDK 与编辑器的运行时中。

<img src="/images/three-types.png" width="500px" />

## 连通 TS 与 JSON schema

为了提供最好的类型体验，Sunmao 的组件 SDK 基于 TS 开发，目前使用 React 作为 UI 框架。

元数据中 `spec.properties` 即应用构建者配置的部分，会以 React 组件 props 参数的形式传入，用于实现组件的逻辑。

通常，我们会使用 TS 定义 props 的类型，同样以输入框组件为例，props 的类型定义如下。

```ts
type InputProps = {
  defaultValue: string;
  size: "sm" | "md" | "lg";
};

function Input(props: InputProps) {
  // implement the component
}
```

这时问题出现了，作为组件开发者，即需要定义 JSON schema 类型，也需要定义 TS 类型，不论是初次定义还是后续维护都是额外的负担。

所以我们要寻求一种方式使组件开发者仅定义一次，就能同时生成 JSON schema 与 TS 类型。

首先我们用 TS 实现一个简单的 JSON schema builder，仅支持构建 number 类型的 schema：

```ts
class TypeBuilder {
  public Number() {
    return this.Create({ type: "number" });
  }

  protected Create<T>(schema: T): T {
    return schema;
  }
}

const builder = new TypeBuilder();
const numberSchema = builder.Number(); // -> { "type": "number" }
```

JSON schema 顾名思义，是以 JSON 形式存在的，属于运行时的一部分，而 TS 的类型只存在于编译阶段。

在这个简易的 TypeBuilder 中我们构建了运行时对象 `numberSchema`，值为 `{ type: "number" }`。下一个目标就是如何将 `numberSchema` 这个运行时对象与 TS 中的 `number` 类型建立关联。

沿着建立关联这个思路，我们定义一个 TS 类型 `TNumber`：

```ts
type TNumber = {
  static: number;
  type: "number";
};
```

在 `TNumber` 中，包含了一个 number 类型的 JSON schema 结构，同时有一个 `static` 字段指向了 TS 中的 number 类型。

在此基础上，优化一下我们的 TypeBuilder：

```ts
class TypeBuilder {
  public Number(): TNumber {
    return this.Create({ type: "number" });
  }

  protected Create<T>(schema: Omit<T, "static">): T {
    return schema as any;
  }
}

const builder = new TypeBuilder();
const numberSchema = builder.Number(); // typeof numberSchema -> TNumber
```

这里的关键技巧是 `return schema as any` 的处理。在 `this.Create` 的调用中，并没有真正传入 `static` 字段。但当调用 `Number` 时，期望 `this.Create` 的泛型返回 `TNumber` 类型，包含 `static` 字段。

所以正常情况下，`this.Create` 无法通过类型校验，而 `as any` 的断言可以欺骗编译器，使它认为我们返回了包含 `static` 的 `TNumber` 类型，但在运行时并没有真正的引入额外的 `static` 字段。

此时，运行时对象的 `numberSchema` 的 TS 类型已经指向了 TNumber，而 `TNumber['static']` 指向的就是最终期望的 `number` 类型。

**至此，我们就连通了 TS 与 JSON schema。**

为了简化代码中的使用，我们还可以实现一个泛型 `Static` 用于获取 TypeBuilder 构建出来的运行时对象的类型：

```ts
type Static<T extends { static: unknown }> = T["static"];

type MySchema = Static<typeof numberSchema>; // -> number
```

将这一技巧拓展至 string 类型的 JSON schema 同样适用：

```diff
type TNumber = {
  static: number;
  type: "number";
};

+type TString = {
+  static: string;
+  type: "string";
+};

class TypeBuilder {
  public Number(): TNumber {
    return this.Create({ type: "number" });
  }

+ public String(): TString {
+   return this.Create({ type: "string" });
+ }

  protected Create<T>(schema: Omit<T, "static">): T {
    return schema as T;
  }
}
```

当然在实际使用的过程中还有许多的细节，例如 JSON schema 除基础类型信息之外，还支持配置许多其他附加信息；以及更复杂的 JSON schema 类型 AnyOf、OneOf 等如何与 TS 类型结合。

在 Sunmao 中，我们最终使用的是更为完善的开源项目 [typebox](https://github.com/sinclairzx81/typebox) 作为 TypeBuilder。

## 在 JS 运行时中推断类型

实现了 JSON schema 与 TS 的结合之后，我们进一步思考如何在编辑器的 JS 运行时中推断类型，为应用构建者提供自动补全等特性。

在 Sunmao 编辑器中，通过名为`表达式`的特性支持编写 JS 代码，并可以访问应用中所有组件的响应式状态。

<img src="/images/sunmao-expression.png" width="500px" />

表达式的灵活之处在于支持任意合法的 JS 语法，所以也可以编写更为复杂一些的多行表达式：

<!-- prettier-ignore -->
```js
{{(() => {
  function response(value) {
    if (value === 'hello') {
      return 'world'
    }
    return value
  }
  const res = response(demo_input.value);
  return String(res);
})()}}
```

在分析 JS 运行时类型推断方法之前，先展示一下类型推断在表达式中的应用：

<img src="/images/expression.gif" width="500px" />

从演示中可以清晰的看到，对函数 `response` 返回的变量 `res` 我们准确推断了其类型，从而进一步补全了对应类型变量的方法。

值得注意的是，当 `response` 传入 string 类型的变量使，`res` 的类型也被推断为了 string，而当传入值变为 number，返回值的推断结果也变为了 number。这与 `response` 函数的内部实现逻辑是相符的。

并且表达式中包含的只是常规的 JS 语法，而不是拥有类型的 TS 代码，那么 Sunmao 是如何从中推断类型的呢？实际上我们使用了 JS 代码分析引擎 [tern](https://ternjs.net/) 来实现这一点。

### Tern 的由来

Tern 的作者 Marijn Haverbeke 也是前端领域使用广泛的开源项目 [CodeMirror](https://codemirror.net/)、[Acorn](https://github.com/acornjs/acorn) 等项目的作者。

Marijn 在开发基于运行在 Web 中的代码编辑器 CodeMirror 的过程中产生了对于“代码补全”这一功能的需求，因此开发了 Tern，用于分析 JS 代码，并推断代码中的类型，最终实现代码补全。

又在开发 Tern 的过程中发现编辑器场景下，代码通常处于不完整且语法不合法的状态，因此又开发了能够解析“不合法 JS”的 JS parser：Acorn。

值得一提的是，Tern 中所实现的类型推断算法主要参考了论文[《Fast and Precise Hybrid Type Inference for JavaScript》](https://rfrn.org/~shu/papers/pldi12.pdf)，该篇论文的作者是当时在 Mozilla 负责开发火狐浏览器 JS 引擎 SpiderMonkey 的工程师 Brian Hackett 和 Shu-yu Guo，论文中描述了 SpiderMonkey 所使用的类型推断算法。

不过 Marijn 也在自己的[博客](https://marijnhaverbeke.nl/blog/tern.html)中介绍，Tern 的场景与 SpiderMonkey 并不相同。Tern 从编辑器补全的场景出发，可以实现的更为激进，使用更多近似、牺牲一定精度以提供更好的推断结果。

### Tern 的类型推断算法。

Tern 通过代码静态分析，构建代码对应的类型图结构（type graph）。Graph 的每个 node 为程序中的变量或表达式，以及当前推断出得类型；每条 edge 则为变量之间的**传播**关系。

首先从一段简单的代码理解 type graph 与传播。

```js
const x = Math.E;
const y = x;
```

对于这段代码，tern 会构建出如下图所示的 type graph：

<img src="/images/type-graph-1.png" width="300px" />

`Math.E` 作为 JS 标准变量在 tern 中已经被预先定义为 number 类型，而对变量 `x` 和 `y` 的赋值则生成了 type graph 中的 edge，`Math.E` 的类型也顺着 edge 传播，将 `x` 和 `y` 的类型传播为 number，完成了类型推断。

如果将代码略作修改，tern 的推断结果可能会出乎你的意料：

```js
const x = Math.E;
const y = x;
x = "hello";
```

<img src="/images/type-graph-2.png" width="300px" />

当再次对 `x` 赋值为 string 类型时，其实变量 `y` 的结果并不会随之改变（number 在 JS 中是基础类型，没有引用关系）。但是在 tern 的 type graph 中，向 `x` 赋值的动作会为其增加 string 类型，并顺着 edge 传播给 `y`，在 tern 的类型推断下，`x` 和 `y` **都同时具备 string 和 number 两个类型**。

这与实际的代码结果（`x` 为 string，`y` 为 `number`）显然是不符的，但这就是 tern 为了降低 type graph 构建成本与算法逻辑所做的近似处理：忽略控制流，假设程序中的所有操作均在同一个时间点发生。并且通常这样的近似推理方式对于代码补全场景并没有太大的不利影响。

在代码中还存在更为复杂的类型传播场景，比较典型的是函数的调用。以另一段代码为例：

```js
function foo(x, y) {
  return x + y;
}
function bar(a, b) {
  return foo(b, a);
}
const quux = bar("goodbye", "hello");
```

<img src="/images/type-graph-3.png" width="300px" />

可以看出根据 tern 构建的 type graph，在多次函数调用后仍然可以推断出 `quxx` 的类型为 string。

对于更复杂的场景，例如逆向推断、继承、泛型函数等的 type graph 构建技巧，可以参考上文中的博客链接。

### 在 Sunmao 中使用 tern

基于 tern 提供的类型推断能力，已经可以解决 Sunmao 表达式中常规 JS 的代码补全需求。但上文提到 Sunmao 的表达式可以访问所有组件的响应式状态。这些状态被自动注入到 JS scope 中而不存在于表达式的代码内，所以 tern 无法感知它们的存在以及类型。

不过 tern 提供了一套 definition 机制，可以向它声明环境中已经存在的变量及类型。而在 Sunmao 中组件通过 JSON schema 定义了对外暴露的状态类型，所以我们可以通过一个 JSON schema 与 tern definition 的转换函数自动为 tern 提供这部分类型声明：

```ts
function generateTypeDefFromJSONSchema(schema: JSONSchema7) {
  switch (schema.type) {
    case "array": {
      const arrayType = `[${Types.ARRAY}]`;
      return arrayType;
    }
    case "object": {
      const objType: Record<string, string | Record<string, unknown>> = {};
      const properties = schema.properties || {};
      Object.keys(properties).forEach((k) => {
        if (k in properties) {
          const nestSchema = properties[k];
          if (typeof nestSchema !== "boolean") {
            objType[k] = generateTypeDefFromJSONSchema(nestSchema);
          }
        }
      });
      return objType;
    }
    case "string":
      return "string";
    case "number":
    case "integer":
      return "number";
    case "boolean":
      return "bool";
    default:
      return "?";
  }
}
```

在一些场景中组件的状态 JSON schema 较为宽松，因此我们还会对上述方法稍加改造，从状态的运行时实际值中读取类型，动态生成 tern definition，提供更多类型声明信息。

## 小结

通过文中的方法，我们实现了只维护一份类型定义，自动在 TS、JSON schema 与 JS 运行时三者之间构建统一的类型系统，提升 Sunmao 中不同角色的开发体验。

后续我们还会介绍与之相关的 Sunmao 功能设计，包括

- 响应式状态如何实现按需渲染
- 完成类型推断后如何开发支持混合高亮与代码补全的编辑器
- ...

如果你感兴趣，可以在开源社区中关注、参与 [Sunmao 项目](https://github.com/smartxworks/sunmao-ui)，也欢迎向我们投递简历。
