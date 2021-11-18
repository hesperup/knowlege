 基于DDD领域驱动设计的思想，在开发具体系统时，需要先建立不同的层级包。主要是梳理不同层面（应用层，领域层，基础设施层，展示层）包括的功能目录，每一个层面应该包括哪些模块。本例所讲述的分层是DDD落地方案中常用的一种（参考），且本例适当做了调整和细化。详细分层目录参考下图：

![img](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/DDD%E6%A1%88%E4%BE%8B/DDD%E9%A2%86%E5%9F%9F%E9%A9%B1%E5%8A%A8%E8%AE%BE%E8%AE%A1-%E9%A1%B9%E7%9B%AE%E5%8C%85%E7%BB%93%E6%9E%84%E8%AF%B4%E6%98%8E-%E2%85%A3.assets/626790-20211029174848954-1059485780.png)



### 1. 展示层

展现层(用户接口层)（Presentation Layer）：负责以Restful的格式接受Web请求，然后将请求路由给Application层执行，并返回视图模型（View Model），其载体通常是DTO（Data Transfer Object）。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
* userinterfaces: 展示层，又称用户接口层
* 用户接口层为外部用户访问底层系统提供交互界面和数据表示。
* 用户接口层在底层系统之上封装了一层可访问外壳，为特定类型的外部用户（人或计算机程序）访问底层系统提供访问入口，并将底层系统的状态数据以该类型客户需要的形式呈现给它们。
*
* 用户接口层有两个任务：
* 从用户处接收命令操作，改变底层系统状态；
* 从用户处接收查询操作，将底层系统状态以合适的形式呈现给用户。
*
* 本例用户接口层包括如下子模块：
* 命令目录(command): 对象命名为XxxCommand，指调用方明确想让系统操作的指令，其预期是对一个系统有影响，也就是写操作。通常来讲指令需要有一个明确的返回值（如同步的操作结果，或异步的指令已经被接受）。
* 查询目录(query): 对象命名为XxxQuery，指调用方明确想查询的东西，包括查询参数、过滤、分页等条件，其预期是对一个系统的数据完全不影响的，也就是只读操作。
* 事件目录(event): 对象命名为XxxEvent，指一件已经发生过的既有事实，需要系统根据这个事实作出改变或者响应的，通常事件处理都会有一定的写操作。事件处理器不会有返回值。这里需要注意一下的是，* Application层的Event概念和Domain层的DomainEvent是类似的概念，但不一定是同一回事，这里的Event更多是外部一种通知机制而已。
* 返回数据对象目录(dto): 对象命名为XxxDto，作为ApplicationService的出参
* 控制层（Controller）: 提供restful接口，供外部系统调用
*
* 规范：
* ApplicationService的接口入参只能是一个Command、Query或Event对象，CQE对象需要能代表当前方法的语意。唯一可以的例外是根据单一ID查询的情况，可以省略掉一个Query对象的创建
* CQE,Dto,都是Value Object，但是从语义上来看有比较大的差异，主要是从命名上区别出来。
* CQE：CQE对象是ApplicationService的输入，是有明确的“意图”的，所以这个对象必须保证其"正确性"。为验证部分字段的格式，必填性，可基于Spring Validation等模式做基础数据验证。
* DTO：Dto对象只是数据容器，只是为了和外部交互，所以本身不包含任何逻辑，只是贫血对象。
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)



### 2. 应用层

应用层（Application Layer）：主要负责获取输入，组装上下文，做输入校验，调用领域层做业务处理，如果需要的话，发送消息通知。当然，层次是开放的，若有需要，应用层也可以直接访问基础实施层。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
* application 应用层
* 相对于领域层,应用层是很薄的一层,应用层定义了软件要完成的任务,要尽量简单.
* 它不包含任务业务规则或知识, 为下一层的领域对象协助任务、委托工作。这一层也很适合写一些任务处理,消息处理
* 它没有反映业务情况的状态,但它可以具有反映用户或程序的某个任务的进展状态。
*
* 对外:为展现层提供各种应用功能(service)。
* 对内:调用领域层（领域对象或领域服务）完成各种业务逻辑任务
*
* 事件(event): 对象命名为XxxEvent，跨聚合根，或部分业务处理完成后，需要通知其他模块的,本例采用 Spring event模式。本例是将领域事件放在应用层的。
* 应用服务(service): 对象命名为XxxService，应用层的服务
*
* 一个应用层通常包括以下三种服务：
*
* 业务处理类：XxxCommandService
* 业务查询类：XxxQueryService
* 业务事件类：XxxEventService
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

特别说明：一个实体处理完成后，若存在副作用，可基于事件模式处理。关于事件模块到底是放置在领域层还是应用层，网上存在不同的模式（参考网址：[DDD-领域事件](https://zhuanlan.zhihu.com/p/129345423)）。本例是基于Spring Event模式，放在应用层的。



### 3. 领域层

领域层（Domain Layer）：主要是封装了核心业务逻辑，并通过领域服务（Domain Service）和领域对象（Entities）的函数对外部提供业务逻辑的计算和处理。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
* domain 领域层
* 领域层主要负责表达业务概念,业务状态信息和业务规则。是一个纯内存化的操作。
* Domain层是整个系统的核心层,几乎全部的业务逻辑会在该层实现。领域层不关注数据是如何落地存储的，领域层也不直接调用底层仓库接口保存数据。
* 领域模型层主要包含以下的内容:
*
* 实体(Entities): 对象命名为XxxE，具有唯一标识的对象,所有实体统一用E作为后缀，如PersonE
* 工厂(factory): 接口命名规则为XxxFactory，创建复杂的实体，聚合根，只做创建处理
* 值对象(vo): 对象命名为XxxV，无需唯一标识的对象,所有值对像统一同V作为后缀 ,如PersonV,实体的主键编码以Id结尾
* 领域服务(Domain Services): 接口命名规则为XxxDomainService，一些行为无法归类到实体对象或值对象上,本质是一些操作,而非事物(与本例中domain/service包下的含义不同)
* 仓储(Repository): 接口命名规则为XxxRepository，创建复杂对象,隐藏创建细节,提供查找和持久化对象的方法。本层仅编写仓库的接口。具体实现再基础层
* 聚合/聚合根(Aggregates,Aggregate Roots): 对象命名为XxxA，聚合是指一组具有内聚关系的相关对象的集合,每个聚合都有一个root和boundary,所有聚合统一用A作为后缀，如PersonA
*
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)



### 4. 基础实施层

基础实施层（Infrastructure Layer）主要包含Tunnel（数据通道）、防腐层，Config和Common。这里我们使用Tunnel这个概念来对所有的数据来源进行抽象，这些数据来源可以是数据库（MySQL，NoSql）、搜索引擎、文件系统、也可以是SOA服务等；Config负责应用的配置；Common是通用的工具类。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
* infrastructure 基础实施层
* 向其他层提供 通用的 技术能力(比如工具类,第三方库类支持,常用基本配置,数据访问底层实现)
* 基础实施层主要包含以下的内容:
* 为应用层 传递消息(比如通知)
* 为应用层 提供持久化机制(最底层的实现)
*
* 防腐层(acl): 实体对接外部系统，实体与外部系统之间，不同领域之间，不同的参数转换，语义转换等
* 转换层(assembler): 数据转换工具类，如Dto转换为实体，实体转换为数据表pojo对象，基于org.mapstruct.Mapper实现
* 仓库层(repository): 仓库实现层，实体与DB之间存储的功能层
* 异常管理(exception): 封装具体业务的异常处理信息
* 配置模块(config): 封装配置信息，包括一些基础静态字段，基于阿波罗等获取的配置信息
* 枚举模块(enum): 封装该模块的枚举信息
* 数据库映射的基础数据对象(database): 命名规则为：XxxDo,数据表翻译为java基础的pojo对象
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)



### 5.项目分层

一个微服务里，通常包括多个不同的领域业务。一个领域业务基于DDD的模式，通常都包括上述的分层模块，为了区分不同的领域业务，有两种方案建立包的层级。

1. DDD层级分类：基于分层包，在每一个包下面，新建具体业务的名称，如在application.service包下面建立退款，售后补偿业务。建立两个分别为application.service.refund（退款包）,application.service.compensate（售后包）。
2. 业务分类：基于业务分层，在四层包的前提下，每一个业务均包括DDD的分层包。如下图所示：

![img](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/DDD%E6%A1%88%E4%BE%8B/DDD%E9%A2%86%E5%9F%9F%E9%A9%B1%E5%8A%A8%E8%AE%BE%E8%AE%A1-%E9%A1%B9%E7%9B%AE%E5%8C%85%E7%BB%93%E6%9E%84%E8%AF%B4%E6%98%8E-%E2%85%A3.assets/626790-20211029174904808-289519078.png)

本例选用的方案2，便于归类业务在各个层的模块代码。最后层级结构确定下来后，每个层只负责该层应该负责的功能，不可混用，滥用。好的设计需要开发人员遵守一定的规则，关于层级划分，没有绝对的标准，可参考开源COLA 4.0或其他架构学习。