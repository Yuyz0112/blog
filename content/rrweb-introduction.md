+++
title = "Record and Replay the Web: A rrweb Tale"
date = 2018-12-23
category = "rrweb"

[taxonomies]
tags = ["rrweb"]
+++

Recently we have released [rrweb](https://www.rrweb.io), which is an open source web session replay library that provides easy-to-use APIs to record user's interactions and replay them remotely.

In this post, I'd like to dive into more details about the project.
<!-- more -->

## Why rrweb?
Today we already have some commercial session replay products like [Logrocket](https://logrocket.com/), [Fullstory](https://www.fullstory.com/), etc.

If you are just looking for a ready-to-use tool and would like to pay for its service, I would recommend you to use the above products, because they have well-tested backend services that can store the data for you and perform some higher order features.

But if your web app has some sensitive data you do not want to store in a third-party database, or your web app was deployed in a private network environment which cannot access these public services, or you are just interested in how session replay works, then you are encouraged to continue the reading and find the answer.

## Hello, World!
Always a classic way, one of the best ways to learn a new tool is to show some hello-world level code example. But for rrweb, you will find that the usage is as simple as "Hello, World!" And it just works.

After installing rrweb, you can init the recorder with the following codes:

```js
rrweb.record({
  emit(event) {
    // store the event in any way you like
  },
});
```

During recording, the recorder will emit when there is some event incurred, all you need to do is to store the emitted events in any way you like.

A more real-world usage may looks like this:

```js
let events = [];

rrweb.record({
  emit(event) {
    // push event into the events array
    events.push(event);
  },
});

// this function will send events to the backend and reset the events array
function save() {
  const body = JSON.stringify({ events });
  events = [];
  fetch('http://YOUR_BACKEND_API', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body,
  });
}

// save events every 10 seconds
setInterval(save, 10 * 1000);
```

When you intend to replay the recorded events, which we call a session, you can init the replayer with the following codes:

```js
const events = YOUR_EVENTS;

const replayer = new rrweb.Replayer(events);
replayer.play();
```

And that's it! The replayer will replay the recorded interactions in a pixel-perfect way.
## How rrweb works?
The view of a webpage can be described as a DOM tree, so when we are talking about recording a web, we are actually recording the state of its DOM tree.

It is easy to access the DOM tree via the browser's document object, but it is a little harder to transform the state into a serializable data structure.

Our recorder implement this by traversing the DOM tree and handling some corner cases. The recorder will return a serialized DOM tree state which we call full snapshot.

After we got the full snapshot of the initial DOM tree state, the recorder will start several observers to observe the mutations that could mutate the view of the webpage. Every mutation will also be serialized to the corresponding data structure which we call incremental snapshot.

![snapshot chain](/images/snapshot-chain.png)

So, at last, we have a snapshot chain which consists of one full snapshot and many incremental snapshots. The replayer will rebuild every snapshot at the corresponding timestamp, which replays all the recorded interactions.

## What's next for rrweb?
Before the end,  I would like to take a moment to describe the future vision for rrweb.

From day one, rrweb has a design principle of doing less in the recorder side because the web should not be aware of that it was being recorded. But there are still some important features we want to add into rrweb, such as:

- record in the web worker
- implement transmission data compression
- hijack the console API and record corresponding events
- hijack Ajax/fetch API and record request events
- use TraceKit to log exception events

To keep rrweb's recorder be lightweight, these features may be provided as extensions so that users can decide which extension will be bundled into their web app.
