+++
title = "vue-type-check: type checking in the template part"
date = 2019-10-02
category = "Vue"

[taxonomies]
tags = ["Vue", "Typescript"]

+++

Nowadays more people start trying to build Vue project with Typescript. Vue itself also provides better support to Typescript such as the vue-class-component lib and rewriting version 3.0's codebase in Typescript.

But the limitation of type checking in the template is still a big problem preventing Vue component from being type-safe.

<!-- more -->

We have just open-sourced an easy-to-use Vue type checker, [vue-type-check](https://github.com/Yuyz0112/vue-type-check), to help solve this problem. This type checker can do type checking on the template and script code of a Vue single-file-component.

And it also provides CLI and programmatical API usages with clear error messages which is helpful to integrate the existing workflows.

## Example

We are going to check a simple Vue component with two type errors:

- The variable `msg` we use in the template is not defined in the component.
- We use the `toFixed` method on a string which is not allowed.

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

More details can be found in the [doc](https://github.com/Yuyz0112/vue-type-check#usage).

## How it works?

Currently, vue-type-check is built on the top of [vetur](https://github.com/vuejs/vetur)'s [interpolation feature](https://vuejs.github.io/vetur/interpolation.html). You may find some internal designs of the interpolation in [this post](https://blog.matsu.io/generic-vue-template-interpolation-language-features).

We decided to make vue-type-check because of some vetur's limitations:

- vetur is a vscode editor plugin, which means it could not be integrated into CI or other workflows easily.
- vetur interpolation is still an experimental feature and there are some hacks in the implementation. This makes it a little unstable and sometimes needs a restart when the Vue language service crashed.
- vetur interpolation does not have many performance optimizations right now. We are experiencing critical performance issues when using it in a large codebase with many auto-gen typescript codes.

We have tried other approaches before this, but finally, we choose to stick with vetur because we do not like over-wheeling, and want to keep bringing vetur's latest feature and optimization into vue-type-check.

Also, we found vetur has [a plan for offering a CLI usage](https://github.com/vuejs/vetur/issues/1149), so we will try to contribute to the upstream later.

## Other attempts

The community also has other attempts on checking the types in the template. We have learned the trade-off of them in [this post](https://katashin.info/2019/04/28/261) from [katashin](https://github.com/ktsn).

### Approach 1: check the compiled template

Since Vue compiled the template into JS code, we can also implement a template to TS compiler and check the compiled code.

The limitation of this approach is that the vue-template-compiler does not have source map support so we could not get the position of the error in the source file.

### Approach 2: implement a ad-hoc type checker

Like what Angular did, we could implement an ad-hoc type checker that can use part of Typescript's APIs.

But it would be extremely hard to implement some complex checks like generics and function overloading.

### Approach 3: transform the template into Typescript AST

We can fully adapt the Typescript compiler's type checks in this way with source map support.

This approach is also choosed in Katashin's patch for vetur and it use vue-eslint-parser under the hood.
