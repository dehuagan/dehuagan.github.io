## GCode
#### 介绍
是一个仿leetcode写的网站，主要用到Springboot，mysql，activemq。
#### 系统架构&主要流程
主要说一下后端，后端分为应用服务和判题服务，应用服务就是处理用户的request，判题服务专门处理判题。

用户在网页上编写代码，然后点击提交，后端收到提交请求后，会把提交信息包括：

```
- submission_id
- code
- date
- user_id
- language
- 使用内存（初始为0）
- 运行使用时间（初始为0）
- 通过率
```
持久化到mysql，存储到mysql成功的话，就会返回ok，让客户端跳转到提交列表页面，然后调用接口同步获取提交运行的结果（利用sseemitter作消息推送）。

同时将submission_id作为消息发送到mq（通过事务来保证消息发送不丢失）。

判题机服务作为消费者，收到消息后，首先通过redis判断是否消费过这个submission_id（防止重复消费），接着根据submission_id从数据库取到submission对象，将其中的code存到一个脚本文件（比如java的话就存储为java文件），然后通过java的runtime执行相对应脚本文件，调用从网上找的一个脚本获取运行时间和内存。

判题过程就是一个for循环，遍历testcase，通过对testcase的答案和脚本运行结果比较，得到通过率，然后更新到mysql。

最后得到结果会通过sseemitter返回给客户端，完成一次提交操作。