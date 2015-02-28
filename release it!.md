#《Release It!:Design and Deploy Production-Ready Software 》

------

《release it!》是我在Tim Yang 的博文[应用层的容错与分层设计](http://timyang.net/service/application-failure-managment/)，扫了一眼看到了评论里的介绍，todo note:学习Netflix的Monkey系列，以及他的“价值较高的那部分Stability Patterns”。

------
[TOC]

##[作者访谈](http://www.infoq.com/cn/articles/nygard-release-it)
###production-ready software and feature-complete software
feature-complete:特定版本的所有功能都通过了功能测试测试
production-ready:承受住real world的系统压力
###转变
为系统设计完备的异常处理机制，架构图上的每个箭头、框图都有靠不住的时候。
###一个有趣的例子
某电商系统在上线前按照评估的系统容量做了三个月的完备测试，最终它还是在首次启动后十五分钟就宕了机。怎么会发生这种事情呢？没错，我们是做了负载测试，问题是测试在某种程度上还是太过“温和”了。例如，在测试中所有的VU都默认使用cookie。后来我们发现开发工程师对会话(session)的设计存在很大问题，禁用cookie的浏览器会制造出大量的冗余会话。在测试中，VU是不会在一个会话中对同一个URL做多次访问的，因此负载大都被合理地分配到多个应用程序服务器上了。事实上，我们没有对刷屏者和代理购买程序采取任何防范措施。

如果你是一个.com系统的开发工程师，有一天测试人员过来告诉你说“我登记了一个程序错误，因为在短时间内用一个不使用cookie的浏览器去重复访问某个页面的时候，系统就挂掉了”。你真的应该感到幸运，因为有人这么早就给你指出了这个问题。

即使测试人员及时报告了此类bug，它们也可能马上被标为“关闭”，**理由是需求计划书上标明了只支持使用cookie的用户**。你就理所当然地只想着使用cookie的用户了。但是要知道，这并不意味着如果有人使用不带cookie的脚本访问网站的时候，它就该宕机啊！还就是有这么一批人，他们虽然不知道怎样使用cookies，但是他们能用"wget" 、 "curl" 或者VB编写脚本来访问你的网站。结果就是，不管你是愿意还是不愿意，他们都会来“光顾”的。
##keep system alive---maintaining system uptime

##system capacity

##general design issues should be considered

##examine the system's ongoing life



> Written with [StackEdit](https://stackedit.io/).