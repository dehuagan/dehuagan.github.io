## Ucloud：类Openstack云操作系统项目-网络子系统组件开发

### Situation（背景）

电信云平台分为Iaas和Paas，Iaas产品叫做FusionSphere，是一个增强的商业版openstack。

openstack是一个开源的云计算平台，负责创建管理虚拟机、网络、存储资源。

HW因为美国的制裁，面临业务连续性风险，说白了就是怕不让用了，于是启动Ucloud项目。

### Target（目标）

完成三大模块自研：计算nova、网络neutron、存储cinder，无损替换电信云Iaas的openstack。

### Action（行动）

本人负责网络子系统，即neutron组件的自研。

neutron组件主要分为neutron-server、plugin、agent，我们的网络子系统分为三个模块：

- quark：负责提供北向API接口，处理认证、鉴权、网络资源的生命周期管理（网络、子网、端口、路由的增删查改等）
- controller：负责将命令式资源转换成声明式资源（<u>为什么要声明式</u>）
- agent：list&watch到资源变化后，在计算或网络节点上调用ovs、linux-bridge等去执行具体的网络操作，注意，具体是哪个节点的agent去处理资源，是由nova决定的，nova创建虚机时，会调neutron的创建port接口，其中入参会带host-id指定节点，所以neutron不需要做调度处理，由nova处理，因此创建的CR也会带节点信息，从而让节点的agent知道自己是否要处理这个资源

本人是作为quark的owner，负责相关特性的设计和开发

#### quark的主要流程

使用initialStartUp方法（使用@PostConstruct注解，项目启动的时候会执行该方法）去启动akka框架的初始化和各项配置

akka主要创建多个actor，每个actor只监听一个事件（项目中，一次请求过来实际用到两个actor，一个preprocessActor，一个处理业务的actor），重点主要在业务actor，业务actor监听requestParam类型的消息，当收到这个消息时，就会通过策略模式（策略配置在xml文件中，在inistialstartUp方法中会被预先加载到内存中）对输入做前置校验，校验通过后才会走到真正的业务代码，然后做后续处理。当同时有多个相同请求时，会根据配置的策略扩容同一个类型的actor？

为什么使用akka-actor，因为actor框架支持高并发，每个actor都是消息驱动，只处理属于他的消息，同时框架使用dispatcher处理actor的消息（底层使用了forkjoinpool），当一个请求过来时，akka-http先把请求发送给对应的actor，请求会进入该actor的消息队列，然后由dispatcher从actor的消息队列中获取消息，然后提交到forkjoinpool中，由forkjoinpool中的线程执行该actor的指令逻辑

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
- ~~如何集成到现有框架：复用已有的actor+前置处理，或者使用AOP，不考虑spring拦截器（因为只作用于@Controller，而处理业务逻辑的是actor，由@Component修饰），最终选择复用已有方案，因为复杂的业务场景，将前置处理和业务逻辑都聚合在actor中，而AOP考虑到依赖动态代理，性能消耗大，并且不好访问actor的消息上下文~~

解决办法

- 梳理原生的oslo policy的代码流程
- 服务启动时，预先将每个操作对应的规则都转换成AST语法树，使用ConcurrentHashMap将规则缓存起来

```java
// rule:admin_or_owner or (is_admin:True and not expired:True)
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

同事使用jmeter之类的工具测试主要接口的指标，其中某个dhcp相关的接口时延很高，需要优化

解决办法：

- 首先想到是数据库，某个sql查询耗时长，开启druid的sql时长日志打印，发现果然有一处根据dhcp信息查询port的sql耗时很久（多少s？）

- 另外想到一个因素，GC频率高，加长了耗时，于是查看GC日志，发现full gc频繁（怎样算频繁？），再使用MAT去分析内存情况，发现port对象内存占用特别高

- 结果很明显，就是因为没有加上一些过滤条件，查询了过多的不相关的port（上万），导致sql执行缓慢，内存飙高，gc频繁

- 着手优化，代码中增加过滤条件（带上dhcp某些属性），使得查询出来的port对象大量减少，从而降低内存占用；同时，原本设置的堆大小是4g，整体服务的内存占用大概6g，通过减少port对象查询，观察服务稳态下full gc后的老年代占用，发现维持在1.多G左右，于是重新设置堆大小为2G，把整体服务内存占用降低到大概4G

-

-

- ~~创建port单一接口性能不达标，处理时间很慢：查看GC日志，发现GC频繁（[GC分析](#GC分析))~~

  ~~使用MAT分析内存情况，发现有port对象内存占用特别高，再通过druid分析数据库操作时长，发现在查询port资源时，没有使用条件查询，而是把库里所有port资源都查询出来了，导致查询时长慢，并且查出来的列表内存很高，最终通过加上条件查询解决~~

- ~~Quark内存占用大：quark整体内存占用大是因为启动时没有配置最大最小堆，导致内存一直飙高（初始堆通常是物理内存的1/64，最大堆是1/4），因此设置堆内存大小，最大最小堆设置成一样（避免堆扩展导致性能波动），设置的值是根据峰值时，full gc后，老年代大小*x~~

  > ~~叙事流程：1. 单一接口性能不达标，时延大； 2. 服务内存占用高峰值6G（现象）--》查看GC日志，发现gc频繁（full 还是 minor？），druid分析数据库查询时长久，MAT分析port对象内存占用特别高（排查）--》确定是在接口逻辑中，存在一处查询，把所有port都查出来，导致内存飙高（原因）--》加上条件查询，显著减少查询的port数量（解决单一接口问题）--》单一接口性能达标，又因为没有设置堆大小，导致内存占用高 --》 根据优化逻辑后，服务稳态下full gc后的老年代占用，设置最大最小堆~~

可优化点：

1. 分析sql慢查询，优化sql，使用索引
2.

#### 难点3

解决一致性问题

原有方案（主要说第一个）：

1. quark处理业务逻辑和资源入库（commit）后，发送消息给laccon-controller，通知让它查数据库，根据新增资源信息创建k8s的CR，这个方案的问题是：quark写入DB后还没发消息给controller就崩溃了，controller永远不知道要处理此资源，导致系统状态不一致
2. quark处理业务逻辑和资源入库（commit）后，直接调k8s的接口创建CR，这个方案的问题是：需要保证数据库操作和CR创建的原子性，否则，数据库操作成功，但是CR创建失败，导致系统状态不一致；同时，会造成controller和quark组件间的功能耦合

代替方案：

quark在处理业务前，使用kafka向controller发送write-ahead消息，再处理业务，在处理完业务后，向controller发送一个done消息，controller在收到write-ahead消息后，启动一个定时器，在超时时间内收到done消息的话，删除定时器，无论是否在超时时间内收到done，都会调quark接口查询最新资源，并根据资源信息把CR创建出来，把数据同步到etcd

遇到的问题：

- 使用kafka有连续性风险，需要替换

解决办法：

quark在处理业务前，调k8s接口创建action资源，actyionType为write-ahead，在处理完业务后，创建一个actionType为done的action资源，controller负责list&watch这个action资源

### Result（结果）

实现网络子系统可控

和原生openstack的neutron相比

api并发上限：200->3000，提升15倍

内存占用：内存占用60G->10G，减少80%+

还有哪些可优化：

GC调优，换垃圾回收器



