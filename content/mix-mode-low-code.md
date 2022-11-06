+++
title = "More productive low-code: create UI in code, perfect it in the GUI editor"
date = 2022-11-06
category = "Sunmao"

+++

## Recap

In our previous [blog](/painless-web-ui-in-go), we've shown a concept of building web UI in non-JS languages on top of Sunmao, a low-level low-code framework.

After tracking the process of building several Apps with this approach, we primarily received positive feedback from backend developers who wish to create a full-stack Web App.

And based on the feedback, we found a bonus in this approach. Developers can now use the best practices they are already familiar with in the new low-code area, namely:

- do abstraction
- do code review
- use version control systems

## Be more productive

Although the experiment works quite well, we found there are still some situations a GUI-based low-code editor will win.

For example, developers may tweak visual properties and write CSS stylesheets to customize the UI. Most GUI-based editors let you see the changes in real-time without leaving your browser. And as you know, writing CSS lets many backend developers feel a headache, a visualize stylesheet builder can be a life saver.

The solution we are going to demo is mix mode low-code. In this mode, developers can build the App either in programming languages or in the GUI editor.

## A demo

In the following demo video, we have:

1. remove the style code from our go example
2. edit style in the GUI editor
3. edit properties in the GUI editor

<video src="/videos/mix-mode.mp4" width="100%" controls></video>

And there are additional powerful features you can get from Sunmao's GUI editor, like:

- auto-complete when writing a JS expression
- real-time validation

## How were the two worlds combined?

Thanks to the flexibility of Sunmao, the problem of combining code and GUI editor has been transformed into combining two JSON-based schemas.

We call this approach layered schemas. The upper layer schema is modified in the GUI editor, while the base layer schema is written in the code.

Instead of storing the complete top layer structure in a file, we merely store the delta after diffing the two layers. This lessens the likelihood of conflicts while updating the base layer and makes it easier to assess what has been modified in the GUI editor.

Since we'd like to support multiple languages, implementing the diff strategy in the JS world will apply to all bindings.

Currently, we build the implementation on top of [jsondiffpatch](https://github.com/benjamine/jsondiffpatch), which minimalize the delta and provide a beautiful diff view of the changes.

<div style="display: flex;">
  <img src="/images/patch-diff-1.png" height="350px" width="auto" />
  <img src="/images/patch-diff-2.png" height="350px" width="auto" />
</div>

## Credits

Although we've long considered the mix mode concept, [Theatre.js](https://www.theatrejs.com/) is the first practical implementation we've seen, and it clarifies our vision.

## Show me the code

The code in the demo video can be found [here](https://github.com/Yuyz0112/sunmao-ui-go-binding), and it's just a POC code and every API is not stabilized.

We've opened a [discussion](https://github.com/smartxworks/sunmao-ui/discussions/600) in Sunmao's Github repo, you can comment on it or join our slack channel to give your feedback about this idea.
