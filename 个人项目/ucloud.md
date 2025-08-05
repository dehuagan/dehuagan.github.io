## Ucloud：类Openstack云操作系统项目-网络子系统组件开发

### Situation（背景）

19年后hw被美国制裁，启动了一系列bmc项目，即业务连续性项目，也就是对关键系统或组件的自研，防止在未来不让使用

我们部门负责的业务是电信云平台，分为IaaS和PaaS，PaaS的底层是k8s，IaaS的底层是Openstack，

openstack是一个开源的云计算平台，负责创建管理虚拟机、网络、存储资源。

为了防止Openstack未来不让用，于是启动Ucloud项目。

### Target（目标）

完成三大模块自研：计算nova、网络neutron、存储cinder，无损替换电信云Iaas的openstack。

### Action（行动）

本人负责网络子系统，即neutron组件的自研。

neutron组件主要分为neutron-server、plugin、agent，我们的网络子系统分为三个模块：

- quark：负责提供北向API接口，处理认证、鉴权、网络资源的生命周期管理（网络、子网、端口、路由的增删查改等）
- controller：负责将命令式资源转换成声明式资源（<u>为什么要声明式</u>）
- agent：list&watch到资源变化后，在计算或网络节点上调用ovs、linux-bridge等去执行具体的网络操作，比如南向配置流表、网桥、ip link等相关业务配置，注意，具体是哪个节点的agent去处理资源，是由nova决定的，nova创建虚机时，会调neutron的创建port接口，其中入参会带host-id指定节点，所以neutron不需要做调度处理，由nova处理，因此创建的CR也会带节点信息，从而让节点的agent知道自己是否要处理这个资源

本人是作为quark的owner，负责相关大颗粒特性的设计开发、问题攻关以及可靠性看护

#### quark的主要流程

使用initialStartUp方法（使用@PostConstruct注解，项目启动的时候会执行该方法）去启动akka框架的初始化和各项配置

akka主要创建多个actor，每个actor只监听一个事件（项目中，一次请求过来实际用到两个actor，一个preprocessActor，一个处理业务的actor），重点主要在业务actor，业务actor监听requestParam类型的消息，当收到这个消息时，就会通过策略模式（策略配置在xml文件中，在inistialstartUp方法中会被预先加载到内存中）对输入做前置校验，校验通过后才会走到真正的业务代码，然后做后续处理。每个类型的actor默认会预先创建16个，其中写和读的比例是1:2，即5个actor负责处理增删改，11个actor负责处理查询操作

为什么使用akka-actor，因为actor框架支持高并发，每个actor都是消息驱动，只处理属于他的消息，同时框架使用dispatcher处理actor的消息（底层使用了forkjoinpool），当一个请求过来时，akka-http先把请求发送给对应的actor，请求会进入该actor的消息队列，然后由dispatcher从actor的消息队列中获取消息，然后提交到forkjoinpool中，由forkjoinpool中的线程执行该actor的指令逻辑(调用receive方法)

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

查看接口时延日志，其中某个dhcp相关的接口时延很高（4~5s），需要优化

解决办法：

- 首先想到是数据库，某个sql查询耗时长，开启druid的sql时长日志打印，发现果然有一处根据dhcp信息查询port的sql耗时很久（2~3s）
- 另外想到一个因素，GC频率高，加长了耗时，于是查看GC日志，发现full gc频繁（几秒左右一次，1次1s左右），再调接口的时候，使用MAT去分析内存情况，发现port对象内存占用特别高（几乎30M+）
- 结果很明显，就是因为没有加上一些过滤条件，查询了过多的不相关的port（每次上万），导致sql执行缓慢，内存飙高，gc频繁
- 着手优化，代码中增加过滤条件（带上dhcp某些属性），使得查询出来的port对象大量减少，从而降低内存占用，<u>jmeter压测的时延具体忘了，但是记得看接口时延日志是不到1s；同时，原本设置的堆大小是2G，通过减少port对象查询，观察服务稳态下，即full gc后的老年代占用，发现维持在800M左右，于是重新设置堆大小为1.5G（新生代400M+老年代800M+300M的buffer），实现堆内存优化

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
- 考虑到k8s的cr本身具备状态存储与事件通知能力，相比引入kafka更轻量

解决办法：

quark在处理业务前，调k8s接口创建action资源，actyionType为write-ahead，在处理完业务后，创建一个actionType为done的action资源，controller负责list&watch这个action资源，同时启动一个协程维护一个最小堆，堆顶元素就是那个最先过期的action，观察整体的超时情况，当controller监听到write_ahead的action时，将action放入一个最小堆，监听到对应的done的action时，就从最小堆中删去这个action，如果在超时时间内没有监听到done，就直接去根据write_ahead的action中携带的资源信息查quark服务，然后创建自定义资源，比如port，这样就避免了每组action都要创建一个定时器监控超时时间，相当于要创建很多协程，造成性能损耗

### Result（结果）

实现网络子系统可控

和原生openstack的neutron相比

api并发上限：200->3000，提升15倍

内存占用：内存占用60G->10G，减少80%+

还有哪些可优化：

GC调优，换垃圾回收器
限流、容灾、降级

## 有可能问到的问题

### 为什么要拆成三部分（quark、controller、agent）？这样做的好处和代价分别是什么

1、参考了 OpenStack Neutron 的架构：Neutron Server 提供 API 和调度，Plugin 层负责资源与底层驱动的解耦，Agent 在各节点上做实际网络配置

### 为什么使用akka-actor+SpringBoot的架构

1. akka-actor基于消息传递，每个actor拥有自己的状态和消息队列，内部串行处理消息，天然支持并发
2. akka-actor和SpringBoot兼容性好，我们将每个actor作为一个bean来管理，方便控制其生命周期