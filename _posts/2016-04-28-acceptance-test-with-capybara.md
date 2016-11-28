---
layout: post
title: "使用 Capybara 进行验收测试"
description: "使用 Capybara 进行验收测试"
category: "Programming" 
comments: true
tags: [Capybara, Acceptance Test]
---
## 背景  
公司里没有专门负责测试的同学，日常的测试工作都是开发的同学兼顾。随着项目规模的扩大，迭代开发后的回归测试已经很难覆盖到所有功能点，尤其是公共组件的升级更容易导致线上事故。这时候我们希望有一个支持 Web 项目的[验收测试](https://en.wikipedia.org/wiki/Acceptance_testing)方案，保证每次开发后项目的核心功能不会出现问题。

## 选型 
这里先要提测试界两个鼎鼎大名的项目。  
1. [Selenium]， 浏览器自动化工具，可以驱动浏览器完成定制的操作。   
2. [Phantomjs]，一个基于 WebKit 的 JavaScript API。除了没有可视化界面，可以完成浏览器的几乎所有操作，访问网页并支持运行 Javascript。因为不需要启动浏览器，所以经常被用于轻量级测试方案。  
在选择哪一个测试框架的时候，我确实纠结了一番。计划中的测试项目上线后，所有的开发同学都应该参与维护。产品功能如果有变更，负责开发的同学也要更新相应的测试用例。从这个角度来讲，负责交互实现的前端同学维护测试项目的机会最多，所以本来希望使用基于 Node 的解决方案，比如 Jasmine，Mocha，以减少前端同学的上手难度。  

不过经过调研，最终还是选择了 Ruby 社区的 Capybara。  
先摘录 Capybara 官方描述的优势。  
1. **No setup** necessary for Rails and Rack application. Works out of the box.  
2. **Intuitive API** which mimics the language an actual user would use.  
3. **Switch the backend** your tests run against from fast headless mode to an actual browser with no changes to your tests.  
4. **Powerful synchronization** features mean you never have to manually wait for asynchronous processes to complete.  

其中让我评价最高的是第三条，可以很方便地更改测试实现。开发同学在本地环境跑测试的时候，最直观的方式莫过于通过 [Selenium] 启动浏览器来观察；此外我希望将自动化测试融入到上线流程中，在没有 UI 的服务器上运行测试，这就需要支持 [Phantomjs] 的测试方案。Capybara 正好符号要求，只用一个配置项，就可以切换具体的测试驱动。  

第一条和第二条不再赘述，解释下第四条。在 Web 页面中难免有 Ajax 请求，我们的测试预期很可能是基于这些异步请求结果的。如果正常写代码，我们不得不手动设置等待时间，来获取异步请求的返回结果。Capybara 帮我们完成了这个过程，只需要按顺序写你期望的操作就可以了。但是在实际开发过程中，发现 Capybara 的这个特性有时并不好用，有时还是得手动 sleep:(  

## 总结  
- 如果前端交互比较复杂，开发 UI 测试用例会比较费时，同时产品变化也会带来测试用例后续的维护成本。建议一开始只需要检查产品页面是否能正常渲染，有没有 5xx 和 Javascript 报错，这样开发量不大，并且就已经能规避很多问题了，是一件投资回报率很高的事情。  
- Capybara 支持 Cucumber，RSpec 等 [BDD](https://en.wikipedia.org/wiki/Behavior-driven_development) 测试框架。个人认为在测试项目不是很重的情况下，使用 Test::Unit 足以。  

最终一期的测试工程覆盖了线上项目核心功能的交互，以及所有课程类型页面的检查。自动化测试流程也整合到上线操作中，测试不通过会终止上线操作。投入运行后的第一个月内规避了三次线上事故。  


[Selenium]: http://www.seleniumhq.org/
[Phantomjs]: http://phantomjs.org/
