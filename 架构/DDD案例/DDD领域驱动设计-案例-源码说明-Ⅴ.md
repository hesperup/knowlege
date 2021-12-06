[案例源码地址：基于DDD设计-售后补偿系统](https://github.com/wuya11/ddd_demo)



## 1.案例说明

1. 该源码为实际项目的脱敏版本，改造过程中，部分功能无法重现。由于售后涉及到订单服务，用户服务等这种跨系统的交互，在案例中基于防腐层做模拟实现。
2. 案例的主要目是展示DDD应用传统项目的流程，具体实现功能的代码不是重点关注的对象，读者可主要了解业务流程，业务规则在分层目录中的实现。切不可对流程中的细节功能做过多分析。
3. 构建实体与需求文档相对应，构建实体的合理性不在本例中过多讨论，本例前提是已经确定实体，聚合根后，具体编码落地的细节展示。
4. 源码中所有实体（包括聚合根）均继承BaseEntity，实体的唯一主键均继承自BaseID。BaseEntity，BaseID是一个只包括uuid的基础值对像。
5. 源码中所有的仓库层均继承BaseRepository接口，便于统一定义仓库的通用操作，如实体的保存，查询，移除（设置基础对象，基础接口，有助于统一管理实体和扩展实体的功能）。参考代码如下：

![img](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)View Code



## 2.案例启动

1. 本例采用SpringBoot构建一个微服务，依赖请参考pom.xml文件。
2. 本例使用了数据库（mysql），相关的数据脚本，请在资源文件中查看，路径为：resources/static/demosql。
3. 先在本地或服务器上面构建好数据库，导入数据表脚本。
4. 修改application.properties，数据库的连接地址请修改为本地可连接的配置地址。
5. 启动或调试DDDApplication.java，项目启动成功后，才可以测试整个流程。启动成功如图：

![img](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/DDD%E6%A1%88%E4%BE%8B/DDD%E9%A2%86%E5%9F%9F%E9%A9%B1%E5%8A%A8%E8%AE%BE%E8%AE%A1-%E6%A1%88%E4%BE%8B-%E6%BA%90%E7%A0%81%E8%AF%B4%E6%98%8E-%E2%85%A4.assets/626790-20211101104424965-1436099082.png)



## 3.案例测试

1. 本例所用的订单信息，是虚拟设置的一个(在源码中可修改订单信息)。且只能基于这个订单号做售后补偿业务测试。
2. 为了便于理解，请查看com.wangling.base.tool.ddd.compensate.CompensateControllerTest.java文件，作者编写了测试用例，读者可基于测试用例一步一步的测试或查看代码。

![img](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/DDD%E6%A1%88%E4%BE%8B/DDD%E9%A2%86%E5%9F%9F%E9%A9%B1%E5%8A%A8%E8%AE%BE%E8%AE%A1-%E6%A1%88%E4%BE%8B-%E6%BA%90%E7%A0%81%E8%AF%B4%E6%98%8E-%E2%85%A4.assets/626790-20211101104501000-956666851.png)

1. 创建补偿单过程源码参考：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
@Override
public long save(CompensateApplyCommand compensateApplyCommand) {
// 应用层体现 业务流程：
// 获取订单信息-基于防腐层获取
OrderV orderV = compensateSelectFacade.getOrderResponse(compensateApplyCommand.getCompensateBillCommand().getSubOid());
// 获取人员信息-基于防腐层获取
UserResponse userResponse = compensateSelectFacade.getUserResponse(compensateApplyCommand.getCompensateBillCommand().getActuid());
// 基于工厂创建实体
CompensateBillA compensateBillA = compensateBillFactory.createCompensateBillA(compensateApplyCommand, orderV, userResponse);
// 调用领域层处理保存的业务逻辑
CompensateBillA compensateBillAdd = compensateBillA.process(compensateBillDomainService);
// 调用仓库保存数据
compensateBillRepository.save(compensateBillAdd);
// 保存完成后，消息通知其他系统
sendCreateMessage(compensateBillAdd);
// 保存完成后，主动发起审核
long coid = compensateBillAdd.getCompensateBillId().getCoid();
check(CheckTypeEnum.AUTO_CHECK, coid, compensateBillAdd.getActuid());
return coid;
}
```

