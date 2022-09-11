+++
title = "Painless Web UI development in Go, and can also be other languages"
date = 2022-09-11
category = "Sunmao"

+++

In our experience, many back-end programs need to build a user interface for end users. Web-based UI is a great choice, but the learning curve of HTML/CSS/JS and other modern front-end toolings stopped a lot of back-end programmers. And when it starts to introduce front-end programmers to collaborate on this, human resource cost comes.

In this post, we will introduce a new way to solve this problem.

## A demo

First, you can check the demo video including:

- Building Web-based UI in Go, with less code.
- Use custom UI libraries.
- Use back-end data in UI, no matter static, dynamic, or real-time.
- Handle front-end interactions in the back-end.
- Call UI's method from the back-end.

<video src="/videos/go-binding.mp4" width="100%" controls></video>

## What's happening here

The demo is built on top of [Sunmao](https://github.com/smartxworks/sunmao-ui/), which is an open-source framework for developing low-code tools.

Compared to most of the drag-and-drop-based GUI low-code editors, Sunmao is much more low-level and flexible.

With Sunmao's fully serializable and structured spec, we can describe UI in JSON, and implement a JSON spec builder in any programming language. We call this 'headless Sunmao'.

And since Sunmao has built-in reactive state management in its runtime, users can write far less code to accomplish common business logic.

## Implement a JSON spec builder

```json
{
  "id": "text",
  "type": "core/v1/text",
  "properties": {
    "value": { "raw": "demo text", "format": "plain" }
  }
}
```

In the demo video, we choose to implement a fluent-style API in Go, so the Go binding looks like this:

```go
b.NewComponent().Type("core/v1/text").Id("text").
  Properties(map[string]interface{}{
    "value": map[string]interface{}{
      "raw": "demo text",
      "format": "plain"
    }
  })
```

And it's quite straightforward to build some short-cut methods to simplify the code:

```go
b.NewText().Id("text").Content("demo text")
```

Of course, you can choose other API styles to implement your own JSON spec builder in any language or choose a different abstraction aspect to speed up development.

## Not only the layout but also the logic

Most Web apps are dynamic. Imagine you want to click a button and send a notification, then you may need some Javascript code like this:

```js
const btn = document.createElement("button");
btn.innerHTML = "click to show notification";
document.body.appendChild(btn);

btn.addEventListener("click", () => $notify("foo"));
```

To avoid learning these kinds of browser APIs, Sunmao let users describe logic in a more structured way.

```json
[
  {
    "id": "button",
    "type": "some_lib/v1/button",
    "properties": {
      "text": "click to show notification"
    },
    "traits": [
      {
        "type": "core/v1/event",
        "properties": {
          "handlers": [
            {
              "type": "onClick",
              "componentId": "test_notification",
              "method": {
                "name": "open",
                "parameters": {
                  "title": "foo"
                }
              }
            }
          ]
        }
      }
    ]
  },
  {
    "id": "test_notification",
    "type": "some_lib/v1/notification",
    "properties": {}
  }
]
```

And it becomes easy to porting to other programming languages:

```go
b.NewNotification().Id("test_notification")
b.NewButton().Id("button").Content("demo text").OnClick(&sunmao.EventHandler{
  ComponentId: "test_notification",
  Method: "open",
  Parameters: map[string]interface{}{
    "title": "foo"
  }
})
```

## Calculate on the client side wisely

Doing all the calculations on the server side is not always the best choice. For example, if you want to render an input element's content's length as text, Javascript code can do it like this:

```js
const text = document.createElement("div");

input.addEventListener("change", (evt) => {
  text.innerHTML = `length is ${evt.currentTarget.value.length}`;
});
```

When you try to implement this on the server side, it may cause you to keep sending the latest input value to the back-end and return the result of the calculation to the front-end.

This kind of code is hard to maintain and not performant. So the idea is just to let the client side do this but in a more battery-include way.

```json
b.NewInput().Id("test_input")
b.NewText().Id("text").Content("length is {{ test_input.value.length }}")
```

The key point is you can use the syntax `{{}}` to evaluate Javascript expressions on the client side.

And as all the modern front-end frameworks do, Sunmao implements reactive state management at its core. Reactive here means you can access the input element's value like `{{ test_input.value }}`, and the expression will re-evaluate whenever the input's value changed.

## Custom components are first-class citizens

Although the Sunmao team maintains some component libraries, there are no 'built-in components in the framework. Every component is a custom component, which means front-end developers can wrap any existing code into a Sunamo component.

Instead of building simple toy apps, this lets users build complex Web apps that can fit your company's design system, implement complex business logic, and reuse current front-end code.

## Comparing to other projects

Several open-source projects are doing the similar things, writing Web UI in back-end runtime, namely:

- [PyWebIO](https://www.pyweb.io/)

- [Streamlit](https://streamlit.io/)

- [Octant](https://octant.dev/)

Although these projects inspired us, we think the headless Sunmao proposal has more potential in the area because:

  1. Custom components greatly leverage the power of front-end code, not limit. And front-end developers can implement custom components without writing any Go/Python/... binding code.
   2. The full-featured front-end framework helps users write less code and get better performance.

## Show me the code

The code in the demo video can be found [here](https://github.com/Yuyz0112/sunmao-ui-go-binding), and it's just a POC code and every API is not stabilized.

We've opened a [discussion](https://github.com/smartxworks/sunmao-ui/discussions/600) in Sunmao's Github repo, you can comment on it or join our slack channel to give your feedback about this idea.
