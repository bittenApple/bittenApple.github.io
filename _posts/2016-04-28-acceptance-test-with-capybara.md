---
layout: post
title: "使用 Capybara 进行验收测试"
description: "使用 Capybara 进行验收测试"
category: "Programming" 
comments: true
tags: [Capybara, Acceptance Test]
---
## 背景  
公司里没有专门负责测试的同学，日常的测试工作都是开发的同学兼顾。随着项目规模的扩大，迭代开发后的回归测试已经很难覆盖到所有功能点，尤其是公共组件的升级更容易导致线上事故。这时候我们希望有一个针对 WEB 的[验收测试](https://en.wikipedia.org/wiki/Acceptance_testing)方案，保证每次开发后项目的核心功能不会出现问题。

## 选型 
这里先要提测试界两个鼎鼎大名的项目。  
一个是浏览器自动化工具 [Selenium]，可以驱动浏览器完成定制的操作。   
另一个是 [Phantomjs]，基于 WebKit 的服务器端 JavaScript API。除了没有可视化界面的浏览器，可以完成浏览器的几乎所有操作，访问网页并支持执行 Javascript。因为不需要启动浏览器，所以经常被用于轻量级测试方案。  
在选择哪一个测试框架的时候，我纠结了一番。计划的测试项目上线后，所有的开发同学都应该参与维护。产品功能如果有更新，负责开发的同学也要更新相应的测试用例。从这个角度来讲，前端同学维护测试项目的机会最多，所以测试项目本来希望是通过 Node 来写，比如 Jasmine，Mocha，以减少前端同学的上手难度。
不过经过调研，最终还是选择了 Ruby 社区的 Capybara。先摘录 Capybara 官方描述的优势。  
1. **No setup** necessary for Rails and Rack application. Works out of the box.  
2. **Intuitive API** which mimics the language an actual user would use.  
3. **Switch the backend** your tests run against from fast headless mode to an actual browser with no changes to your tests.  
4. **Powerful synchronization** features mean you never have to manually wait for asynchronous processes to complete.  

其中让我评价最高的是第三条。开发同学在本地环境跑测试的时候，肯定是通过 [Seleinum] 启动浏览器观察最直观。此外我希望将自动化测试融入上线流程，在远端服务器上运行测试，这就需要支持 [Phantomjs] 的测试方案。Capybara 只用一个配置项，就可以完美切换具体的测试驱动。  
第一条和第二条不再赘述，解释下第四条。在 Web 页面中，难免有 Ajax 请求，我们的测试预期很可能是基于这些异步请求结果的。如果正常写代码，我们不得不手动设置等待时间，异步请求的返回结果，Capybara 把这个过程帮你完成了。但是在实际开发过程中，发现 Capybara 的这个特性并不好用，最终还是手动 sleep 了:(  

## 总结
如果前端交互比较复杂，开发 UI 测试用例比较费时，同时也有产品变化后继续维护的成本。如果只是测试产品页面是否能正常渲染，没有 500 和 Javascript 报错，开发量并不大。在快速迭代的情况下就已经能规避很多问题了，收益率很高。  
Capybara 支持 Cucumber，RSpec 等 [BDD](https://en.wikipedia.org/wiki/Behavior-driven_development) 测试框架。个人认为在测试项目不是很重的情况下，使用 Test::Unit 足以。  
最终一期的测试项目覆盖了核心功能的交互，以及所有产品类型页面的检查。同时上线包含测试流程，在测试机运行测试项目，测试不通过会停止上线。投入运行后第一个月内规避了三次线上事故。  


[Selenium]: http://www.seleniumhq.org/
[Phantomjs]: http://phantomjs.org/