---
layout: post
title: "关于 Code Review"
description: ""
category: 
comments: true
tags: [Thought]
---

最近把 Martian Folower 的《Refactoring》读了一遍，虽然原作创作于上世纪九十年代，但是老马对很多重构方法论的介绍，尤其是针对每一种重构方法的步骤分解，让人茅塞顿开；有些平时写代码不知不觉已经使用的方法，或者使用不到位的，经作者概括总结出来，也让人有蓦然回首的感觉。但是今天没有对这本书作过多评述，而是对最近工作中遇到的代码质量问题发发牢骚。

现在打工的这家公司的节奏很快，工作流采用最简单粗暴的方式（其实也只有我们这个部门是这样，每个人的代码不用进行 code review，可以直接 push 到 master 分支，然后进行上线。大部分的代码质量问题只能通过自发的 review，或者线上事故后的修复来把控。

由于没有 reivew，大家各自为政，代码中到处充斥着重复功能定义。一些新人同学动辄写出几百行的函数，过长的函数签名，没有任何意义的 commit comment，或者是爆炸的 if/else 逻辑，而没有使用 early return。看着这些代码想要保持身心愉悦真是一件不容易的事情，而这些问题，在一个有良好 code review 习惯的开发团队中都很容易规避。

采用这种粗暴的工作流原因一是和 leader 自己的开发习惯有关，二是公司的开发节奏确实太快，即使采用这种节奏也才能堪堪赶上 deadline，加入其它开发流程势必会拖慢开发进度。

此外公司两个月就能换三次大的产品方向，每次几乎是重新开发。在这种情况下，花太多精力进行代码质量的维护表面上确实不划算，先把功能用代码堆出来再说，因为指不定过段时间代码就丢掉不用了。

但是即使有这些也不应该放弃 reivew 环节，code review 不只是为了保证代码质量，也是同事之间相互学习的重要手段，尤其能帮助新人成长。从个人经验来看，新人水平的提高，最有效的途径就是通过 code review。

即使代码出自老司机之手，review 环节也不可忽略。因为每个人都会有思维定势，让别人帮你看代码并不完全是因为别人比你牛，而是因为别人能从不同的角度思考你遇到的问题。

有机会呆在一个关注成员成长，并且有科学可行的方法的团队，是一大幸事，且行其珍惜。如果处在一个急功近利，开发节奏异常快的团队，个人成长这些也只能往后排了。但是关注团队成员的成长，从长线来说对 leader，对公司也是有收益的。试问一个自己本身就满腹牢骚的员工，他怎么可能真心推荐他的朋友或者学弟学妹来这样一个团队呢。





