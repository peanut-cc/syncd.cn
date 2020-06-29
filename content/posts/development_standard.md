---
title: "开发中的一些规范"
date: 2020-04-29T14:10:24+08:00
draft: false
author: "fan"
subtitle: "开发中的一些规范"
tags: ["Go"]
categories: ["Go"]
toc:
  enable: true
  auto: true
---

## 关于Go开发编码规范

### 基本要求

* 项目代码必须通过lint工具的风格检查
* 必须go fmt
* 建议使用Go Modules
* 必须有单元测试
* 必要的CI

### 编码风格

[Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)

## Git 规范

### 基本要求

* commit message 可以使用英文，也可以使用中文，但应尽可能描述清楚 commit 所做的事情。如：**<type>: <subject>**
* commit 中type为：feat(实现功能)，fix(bug修复)， docs(完善文档)，style(格式化代码)， refactor(仅重构不改动功能)， test(增加，重构，修复测试，不改动功能代码) ，chore(其他小的修复)。subject 需要简短的描述做了件什么事情。
* 可以使用 [Commitizen](http://commitizen.github.io/cz-cli/) 等工具进一步规范 commit message 格式
* MR 由两部分组成：MR 说明，以及一系列的 commit。
* 通常 MR 的标题部分应当包含对应的 Issue ID（例如 XXXXX-1234），以及简短的描述做了件什么事情。如：**<issue>: <subject>**
* 一个commit 只做一件事，一个大的变更，拆成多个MR(方便Code Review)

### Code Review

实际开发中，我们通常都是在各自开发分支进行开发，那么功能开发完成之后，或修复bug之后，就需要除了自己之外的其他人进行code review。从而提高代码的质量以及避免合并到主线上之后对主线代码造成影响。所以code review 最直接的目的就是：**代码长期的可用性与可维护性**

Code Review 流程：

1. 开发者从master分支切换开发分支进行功能开发
2. 开发者将代码提交到新的分支，并提交MR，开发者需要自己解决冲突rebase master
3. 开发者选择至少不小于一个人的相关人员进行review
4. review 代码的人若认为有问题，可以以评论的方式进行记录
5. review 代码的人认为没有问题你，Approve MR
6. 合并到master


## 延伸阅读

[Effective Go](https://golang.org/doc/effective_go.html)

[Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments#go-code-review-comments)

[How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)