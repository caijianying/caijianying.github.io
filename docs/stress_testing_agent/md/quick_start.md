## 介绍
为避免一头雾水先说明一下，开始本项目需具备的前置知识
* [什么是全链路压测？](/md/total_chain_testing)
* JavaAgent
* 字节码增强框架 ByteBuddy

演示项目
* [演示Agent](https://github.com/caijianying/stress-testing-app-agent)
* [测试应用](https://github.com/caijianying/stress-testing-app)


本站将在highLight(测试应用)接入highLight(演示Agent)，逐步完善Agent的主要功能。为方便理解，请按顺序查看。
1. 识别压测标
2. 压测标透传
3. 数据隔离
   * 3.1 -影子表方式
   * 3.2 -影子库方式
4. 链路监控
5. 熔断通知

创作不易，觉得对你有帮助的可以顺便支持(highLight(其实也就是star))一下我的开源项目，非常感谢！

highLight(演示项目)用于演示视频的讲解，偏向于highLight(基本功能的实现)。highLight(开源项目)则是在此基础上做的增强，更偏向于highLight(架构)。

[支持STA](/)




