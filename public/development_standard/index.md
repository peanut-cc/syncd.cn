# 开发中的一些规范


## 基本要求(Go)

* 项目代码必须通过lint工具的风格检查
* 必须go fmt
* 建议使用Go Modules
* 必须有单元测试
* 必要的CI

### 编码风格

[Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)


## code commit


### 原则

代码的commit遵循的原则：
1. 一个commit 只做一件事
2. 一个MR不能引入过多的commits,导致很难甚至无法完成review


最小化原则体现：
1. 尽量不同时修改多个文件
2. 尽量不同时修改多个逻辑上游划分的模块
3. 不破坏已有逻辑
4. 不修改无关的部分
5. 一个大的变更尽量拆解成多个更独立的MR

### 要求

* commit message 可以使用英文，也可以使用中文，但应尽可能描述清楚 commit 所做的事情。如：type: subject
* commit 中type为：feat(实现功能)，fix(bug修复)， docs(完善文档)，style(格式化代码)， refactor(仅重构不改动功能)， test(增加，重构，修复测试，不改动功能代码) ，chore(其他小的修复)。subject 需要简短的描述做了件什么事情。
* 可以使用 [Commitizen](http://commitizen.github.io/cz-cli/) 等工具进一步规范 commit message 格式
* MR 由两部分组成：MR 说明，以及一系列的 commit。
* 通常 MR 的标题部分应当包含对应的 Issue ID（例如 XXXXX-1234），以及简短的描述做了件什么事情。如：**<issue>: <subject>**

## Code Review

### 什么是Code Review

代码评审(Code Review)是指在功能开发过程中，邀请原作者之外的开发者(审阅人)来对功能代码 进行评审的步骤。其目的是为了在代码合并入主线之前确保其质量，避免对主线代码的质量造成负面的影响。
    

实际开发中，我们通常都是在各自开发分支进行开发，那么功能开发完成之后，或修复bug之后，就需要除了自己之外的其他人进行code review。从而提高代码的质量以及避免合并到主线上之后对主线代码造成影响。所以code review 最直接的目的就是：**代码长期的可用性与可维护性**


### Code Review 流程

流程：
1. 开发者从master分支切换开发分支进行功能开发
2. 开发者将代码提交到新的分支，并提交MR，开发者需要自己解决冲突rebase master
3. 开发者选择不少于一个(推荐两个)审阅人，请求其帮忙 review 代码;通常直接在 MR 下 at 审阅人请求 review 或将 MR assign 给相应的审阅人即可
4. 审阅人认为存在问题，告知开发者，由开发者进一步完善;推荐以评论的方式进行记录，即使当面沟通也需要以评论的方式记录一下讨论结果;另外，MR 下的问题讨论应当由审阅人来 resolve，在审阅人明确表示问题得到解决之前，开发者应当避免随意关闭审阅人提出的问题
5. 审阅人认为没有问题，Approve MR;或直接在 MR 下评论 “reviewed by @someone”、“LGTM” 等
6. 在至少有一位审阅人完成评审的情况下，由模块负责人将 MR 合入主线，如存在冲突则需要开发者先将分支重新 rebase 主线

Git 团队协作中常用术语:
WIP   Work in progress, do not merge yet. // 开发中
LGTM Looks good to me. // Riview 完别人的 PR ，没有问题
PTAL Please take a look. // 帮我看下，一般都是请别人 review 自己的 PR
CC Carbon copy // 一般代表抄送别人的意思
RFC  —  request for comments. // 我觉得这个想法很好, 我们来一起讨论下
IIRC  —  if I recall correctly. // 如果我没记错
ACK  —  acknowledgement. // 我确认了或者我接受了,我承认了
NACK/NAK — negative acknowledgement. // 我不同意


### 如何提交一个MR

定方案-->写代码-->自测-->提 MR

MR所包含的内容：
1. 代码的改动： 实现功能/修复缺陷/补充测试....
2. MR自身的描述信息：帮助审阅人理解上述改动
3. commit历史：commit历史应该是被整理之后的

关于commit历史：
通常我们在开发过程中的commit历史是会比较糟糕的，可能也commit message 也会不规范，所以我们在提交mr之前就需要对我们的commit历史进行整理，如：
1. 合并一些无用的commit历史
2. 更改不规范的commit message
3. .....



**注：对于没有任何描述的MR审阅人可以直接拒绝审阅**

**注：永远做自己MR的第一个审阅人**

### 对于MR代码质量要求

1. 如果认为一个 MR 在被接受那一刻的得分为 90~100 分的话，提交 MR 时分数不应低于 80 分
2. 将 MR 完成到 80 分是开发者的责任，在达成之前，审阅者没有义务帮助开发者审阅并完善 MR
3. 低于 80 分的 MR 认为完成度过低，应当被打上 WIP 标记
4. 当审阅人认为完成度过低时，可以直接将 MR 标记为 WIP 并拒绝审阅(甚至可以不解释)

当代码质量出现(不限于)以下情况时，可以认为完成度过低：
1. 代码风格/格式不符合编码规范
2. 缺少必要的(单元)测试代码
3. 破坏兼容性且未标注或说明(包含改变了特定接口的行为但未更新注释)
4. 明显未经过自测便提交
5. 存在较多(3 个及以上)低级逻辑错误(偏主观，无明确标准)
6. .....




## 延伸阅读

[Effective Go](https://golang.org/doc/effective_go.html)

[Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments#go-code-review-comments)

[How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)
