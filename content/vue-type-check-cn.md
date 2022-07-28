+++
title = "vue-type-check: Vue 模板中的 Typescript 类型检查"
date = 2019-10-02
category = "Vue"

[taxonomies]
tags = ["Vue", "Typescript"]

+++

越来越多人开始尝试使用 Typescript 编写他们的 Vue 项目，Vue 本身也在不断加强对 Typescript 的支持（官方提供 vue-class-component 库、使用 Typescript 编写 Vue 3.0 等），但是对于组件中模板部分的类型检查仍然有很大的局限性。

<!-- more -->

为此我们开源了一个易于使用的 Vue 类型检查器： [vue-type-check](https://github.com/Yuyz0112/vue-type-check)，可以对 Typescript 编写的 Vue 组件中模板和脚本的部分均进行类型检查。

vue-type-check 同时提供了 CLI 和 API 两种使用方式，并且输出清晰的错误提示以便和现有的工作流无缝衔接。

## 示例

我们对以下 Vue 单文件组件进行类型检查，其中包含了两个类型错误：

- 在模板中使用的变量 `msg` 并没有在组件中定义。
- `printMessage` 方法中错误地对字符串类型的值使用 `toFixed` 方法。

```vue
<template>
  <div id="app">
    <p>{{ msg }}</p>
  </div>
</template>

<script lang="ts">
import Vue from "vue";

export default Vue.extend({
  name: "app",
  data() {
    return {
      message: "Hello World!"
    };
  },
  methods: {
    printMessage() {
      console.log(this.message.toFixed(1));
    }
  }
});
</script>
```

![example.gif](https://raw.githubusercontent.com/Yuyz0112/vue-type-check/master/assets/vtc.gif)

更多使用方式可参考[文档](https://github.com/Yuyz0112/vue-type-check#usage)。

## 工作原理

目前 vue-type-check 完全基于 [vetur](https://github.com/vuejs/vetur) 的 [interpolation feature](https://vuejs.github.io/vetur/interpolation.html) 实现，interpolation 的内部实现可以参考[这篇文章](https://blog.matsu.io/generic-vue-template-interpolation-language-features)。

之所以要在 vetur 的基础上开发 vue-type-check 是为了避免其局限性：

- vetur 以 vscode 编辑器插件的形式存在，不能很好地和持续集成等工作流进行集成，使用场景有限。
- vetur interpolation 目前还是实验性功能，并且由于实现方式较为 hack，因此使用时稳定性还有所欠缺，实际使用时会遇到 vue language server crash 需要重启的问题。相比之下 vue-type-check 只对已经完成编辑的文件进行类型检查，稳定性更佳。
- vetur interpolation 还未做太多的性能优化，在我们一个有大量自动生成的 Typescript 代码的项目中会导致 vscode 非常卡顿。

实际上在此之前我们还尝试过其它实现方式，但最终我们还是转向了基于 vetur 进行开发，因为我们希望避免重复的开发，并且持续地将 vetur 中最新的功能和优化应用在 vue-type-check 中。

另一方面 vetur 也有[提供 CLI 使用方式的规划](https://github.com/vuejs/vetur/issues/1149)，我们也会尝试将 vue-type-check 中完成的工作反馈给社区。

## 其它尝试

对于 Vue 组件模板代码的类型检查社区中也陆续进行过其它尝试，我们从 [katashin](https://github.com/ktsn) 的[这篇文章](https://katashin.info/2019/04/28/261)了解到了几种思路的利弊。

### 方式一：将 Vue 模板编译后进行检查

因为 Vue 实际上也是将模板编译成了 JS 代码，因此可以实现一个模板 -> TS 的编译器，对编译后的结果进行类型检查。这一方式的问题是 vue-template-compiler 不能提供 source map 的支持，因此在转化之后无法标记出发生错误的位置。

### 方式二：实现一个类型检查器

参考 Angular 的方式重新实现一个类型检查器，这种方式可以使用 Typescript API 的部分能力，但与 Typescript 本身还有较大差距，想要实现一些复杂的类型检查（例如函数重载、泛型）是非常困难的。

### 方式三：将模板转化为 Typescript AST

这种方式可以完全利用 Typescript 编译器的能力，并且也可以正确标记出错误发生的位置。这也是最终 katashin 给 vetur 的 patch 中所使用的方式，目前是通过 vue-eslint-parser 完成了这一转化过程。