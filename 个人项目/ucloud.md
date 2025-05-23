## 项目一、Ucloud：类Openstack云操作系统项目-网络子系统组件开发

### Situation（背景）

电信云平台分为Iaas和Paas，Iaas产品叫做FusionSphere，是一个增强的商业版openstack。

openstack是一个开源的云操作系统，主要负责创建管理虚拟机、网络、存储资源。

HW因为美国的制裁，面临业务连续性风险，说白了就是怕不让用了，于是启动Ucloud项目。

### Target（目标）

完成三大模块自研：计算nova、网络neutron、存储cinder，无损替换电信云Iaas的openstack。

### Action（行动）

本人负责网络子系统，即neutron组件的自研。

neutron组件主要分为neutron-server、plugin、agent，我们的网络子系统分为三个模块：

- quark：负责提供北向API接口，处理认证、鉴权、网络资源的生命周期管理（网络、子网、端口、路由的增删查改等）
- controller：负责将命令式资源转换成声明式资源（<u>为什么要声明式</u>）
- agent：list&watch到资源变化后，在计算或网络节点上调用ovs、linux-bridge等去执行具体的网络操作

本人是作为quark的owner，负责相关特性的设计和开发

#### quark的主要流程

使用initialStartUp方法（使用@PostConstruct注解，项目启动的时候会执行该方法）去启动akka框架的初始化和各项配置

akka主要创建多个actor，每个actor只监听一个事件（项目中，一次请求过来实际用到两个actor，一个preprocessActor，一个处理业务的actor），重点主要在业务actor，业务actor监听requestParam类型的消息，当收到这个消息时，就会通过策略模式（策略配置在xml文件中，在inistialstartUp方法中会被预先加载到内存中）对输入做前置校验，校验通过后才会走到真正的业务代码，然后做后续处理。

#### 难点1

开发oslo-policy权限控制组件

作用

- 权限控制：根据不同角色的用户，判断是否允许执行API请求
- 策略解耦：使用配置文件（policy.json）去定义权限规则，从代码中分离，并且支持热更新

达到的业务效果

- 资源隔离：用户只能操作自己项目的网络资源
- 动态策略管理：无需重启服务就能调整权限规则

难点

- 策略规则的解析：涉及复合逻辑表达式，如`"create_network:admin or project_id:%(project_id)s"`， 以及需要运行时动态注入上下文参数
- 如何集成到现有框架：复用已有的actor+前置处理，或者使用AOP，不考虑spring拦截器（因为只作用于@Controller，而处理业务逻辑的是actor，由@Component修饰），最终选择复用已有方案，因为复杂的业务场景，将前置处理和业务逻辑都聚合在actor中，而AOP考虑到依赖动态代理，性能消耗大，并且不好访问actor的消息上下文

解决办法

- 梳理原生的oslo policy的代码流程
- 服务启动时，预先将每个操作对应的规则都转换成AST语法树，使用ConcurrentHashMap将规则缓存起来

```java
rule:admin_or_owner or (is_admin:True and not expired:True)
// 对应的AST如下
LogicalRule(OR,
  ReferenceRule("admin_or_owner"),
  LogicalRule(AND,
    AtomicRule("is_admin", "True"),
    LogicalRule(NOT, AtomicRule("expired", "True"))
  )
)
```

#### 难点2

性能优化

遇到的问题：

- Quark内存占用大
- 创建port单一接口性能不达标，处理时间很慢

解决办法：

- Quark内存占用大：quark整体内存占用大是因为启动时没有配置最大最小堆，导致内存一直飙高（初始堆通常是物理内存的1/64，最大堆是1/4），因此设置堆内存大小，最大最小堆设置成一样（避免堆扩展导致性能波动）

- 创建port单一接口性能不达标，处理时间很慢：查看GC日志，发现GC频繁（[GC分析](#GC分析))

  使用MAT分析内存情况，发现有port对象内存占用特别高，再通过druid分析数据库操作时长，发现在查询port资源时，没有使用条件查询，而是把库里所有port资源都查询出来了，导致查询时长慢，并且查出来的列表内存很高，最终通过加上条件查询解决

### Result（结果）

实现网络子系统可控

和原生openstack的neutron相比

api并发上限：200->3000，提升15倍

内存占用：内存占用60G->10G，减少80%+

[test](Java/JVM.md/#GC)
