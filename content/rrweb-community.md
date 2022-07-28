+++

title = "The new adventure of the rrweb community"
date = 2020-10-04
category = "rrweb"

+++

[中文版](/rrweb-community-cn)

It has been nearly two years since I released [rrweb](https://github.com/rrweb-io/rrweb) as an open-source project.

As the creator and one of the maintainers of this project, I learned a lot during this period.

<!-- more -->

## 2019

Back in 2019, the changes in the repo mainly focus on the correctness of the observers. And outside code, I received many custom-develop requests, most of them are user behavior analytics Apps.

Since I'm working as an R&D manager in my daily work, I do not have time to do any kind of customization requests and only can maintain rrweb at weekends.

Thanks to our kind community, there is no one complaining about the speed of developing new features.

Even more, they always create issues/PRs with some warm words that makes me more energetic.

## 2020

Time goes to 2020, rrweb meets more opportunities this year.

We saw some quite exciting commercial Apps which were built on top of rrweb. And the most interesting part is people started to use rrweb beyond the scope of user behavior analysis. Some of the Apps are using rrweb as a general web serialization library.

And because of the COVID-19 pandemic, some online-sale scenarios need such a record and replay tool to do audit things. For example, many insurance companies in China are requested to record the sales process and store them for a relatively long term.

Personally, I participated in two hackathons and built two proof-of-concept Apps with rrweb. Both of them won the first prize in the hackathon and show some bright future of web serialization tech.

All of these use cases help rrweb grow into a more feature-rich, good-performance, and solid library.

And the most important thing is more people are contributing to the rrweb community. There are some creative and active contributors to help the project move faster. Also, some generous companies are sponsoring the project's development.

## Next

In the coming days, we will try to expand the rrweb community to provide better services to our users. And here are three new plans to achieve this.

### rrweb partners

Firstly, we will have **rrweb partners**. This is the plan for our enterprise users, who may have some long-term goal to accomplish with rrweb. Companies in this plan will:

1. Have the free consulting service from the core team.
2. Discuss the roadmap and priority of rrweb's new features with the core team.
3. Get periodic training about rrweb and its ecosystem.
4. All the rrweb pro features.(rrweb pro is the third plan I will explain later)

The rrweb partners need to pay an annual fee, which will be used to encourage the core team and active community contributors.

### rrweb joint lab

Secondly, we are going to set up a **rrweb joint lab**. The joint lab will incubate some Apps built on top of rrweb to discover the best use cases of rrweb.

There is an example if you are not sure about the difference between the joint lab and the 'partners' plan.

> When we build a co-browsing tool with rrweb, we may need a solid network transport layer. If some IaaS vendor can provide such a network layer and make some integration with our experiment tool, then they will be a member of the joint lab.

There is no annual fee for the joint lab members, but they are asked to provide some tech supports for the experimental projects.

### rrweb pro

Finally, we will have the **rrweb pro** plan.

Different from many commercial projects, rrweb will always be open-source and I'm even not going to encourage users to choose the pro plan.

I've written down some very detailed design docs for rrweb because I'd love users to know how it works. But during the maintaining, I find there are two kinds of users we need to help:

- Developers use rrweb in their daily work. Some of them are junior engineers and just want to accomplish the task ASAP. It was a little hard for them to understand the architecture of rrweb. Instead, some consulting may be quite useful.
- Companies use rrweb in some complex scenarios. For example, we've received many requests for recording iframe, optimize storage size, etc. There is no silver bullet for these use cases, and some tailor-made solutions may work much better than a general one.

The way we plan to help them is to do some consulting works and provide some ad-hoc solutions.

Currently, features in the rrweb pro plan include:

- same-origin iframe recording
- cross-origin iframe recording
- events storage size optimization
- high-performance PDFjs recording

We are going to charge an hourly fee for these works and may open-source some of the pro plan solutions when they are stable and general to use. So please stick with our open-source version if you are not in a hurry, or become our partners to participate in the process deeply.

All of these plans are in the early stage, and we may have some more detailed explanation on it. But you can start to mail me(aryu0112@gmail.com) if you have some interest in it.

## Conclusion

Some of the situations have changed in the past two years, but the goal of rrweb not. I'd love more people to know the power of modern browser's APIs and build creative Apps with rrweb. So none of the plans will hurt the current open-source community of rrweb. Instead, the plans may provide more choices to make the community better.
