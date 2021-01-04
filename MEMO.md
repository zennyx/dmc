# 备忘录

## Development

### 语言

1. Go

   优先度高。生产力语言，性能高。适用于微服务开发及DevOps（为了看懂`k8s`和`Istio`的源代码也必须看）。

1. Kotlin

   优先度高。生产力语言，实现同样逻辑，代码量约为`Java`的2/3。可与`Java`互操作，适用于所有`Java`应用的场合，包括Android和IOS（详见KMM，目前alpha阶段）的开发。和`Java`语法并不非一一对应，且有些设定反人类，早期对`Eclipse`的支持也不够好。

1. Rust

   优先度低。同时具备`Go`和`Java`的优点，但最近Mozilla出现大规模裁员，谨慎观望。

### 框架/库

1. spring-webflux

   异步非阻塞，希望能对标`go`+`iris`。

1. spring-cloud

   3.0。

1. iris

   配合`go`食用的`Web`应用框架。

1. React

   17.0。子节点能和父节点版本不同，要看下原理。

1. Vue

   3.0。新特性很多，`Tree-shaking`有望进一步减小前端包的体积，配合开发的`Vite`也要了解，也许能大幅减少开发时等待转译的时间。目前发布延迟了，依然处于beta阶段。

1. Kubernetes（k3s）/Istio（OSM）

   看完`go`看源代码。k8s没有捐赠给CNCF，需要当心。查一下rancher的k3s，istio也是一样。

1. Spark/Flink

   流处理。优先度低，分布式/云都没搞定，流处理免谈。

### 项目（Java + Maven）结构

1. 公共库（参考springfield）

1. 业务项目

   ``` text
   project-parent
          ├ project-commons
          ├ project-module[n]
          |        ├ project-module[n]-dal
          |        |        └ src
          |        |           └ com.[cn].[pn].[mn].dal
          |        |                    ├ config
          |        |                    └ repository
          |        ├ project-module[n]-buz
          |        |        └ src
          |        |           └ com.[cn].[pn].[mn].buz
          |        |                    └ service
          |        |                         └ impl
          |        ├ project-module[n]-web
          |        |        └ src
          |        |           └ com.[cn].[pn].[mn].web
          |        |                    ├ config
          |        |                    └ controller
          |        ├ project-module[n]-pojo
          |        |        └ src
          |        |           └ com.[cn].[pn].[mn].pojo
          |        |                    ├ config
          |        |                    ├ entity
          |        |                    ├ domain
          |        |                    └ model
          |        ├ project-module[n]-boot
          |        |        └ src
          |        |           └ com.[cn].[pn].[mn]
          |        |                    └ Application.java
          |        ├ project-module[n]-job
          |        |        └ src
          |        |           └ com.[cn].[pn].[mn].job
          |        |                    ├ config
          |        |                    └ task
          |        ├ project-module[n]-api
          |        |        └ src
          |        |           └ com.[cn].[pn].[mn].api
          |        |                    └ client
          |        └ project-module[n]-bus
          |                 └ src
          |                    └ com.[cn].[pn].[mn].bus
          |                             ├ config
          |                             ├ listener
          |                             ├ producer
          |                             └ consumer
          ├ ...
          ├ project-standalone
          |        └ src
          |           └ com.[cn].[pn]
          |                    └ Application.java
          ├ project-gateway
          |        └ src
          |           └ com.[cn].[pn].gateway
          └ project-samples
   ```

   >简要说明：
   >
   >\[cn\]：公司名
   >
   >\[pn\]：项目/产品/业务线名，\[pn\]和\[mn\]直接可能还有一个次业务线名
   >
   >\[mn\]：模块/业务名
   >
   >project-parent：Maven主项目，用于：
   >
   >   1. 项目/各子模块的打包、本地/远程仓库的发布管理
   >   1. 各子模块的依赖、插件统一管理
   >   1. 版本统一管理（可考虑使用`${revision}${changelist}`）
   >
   >project-commons：项目公共库子模块
   >
   >project-module\[n\]：各功能子模块，每个模块代表一组业务功能
   >
   >project-module\[n\]-pojo：子模块数据/领域模型层，用于：
   >
   >   1. 为其他子模块提供业务用数据接口
   >   1. 数据校验（JSR303，可使用Hibernate Validation）
   >   1. 数据防锈（可选，解耦用，可使用MapStruct）
   >
   >project-module\[n\]-dal：子模块数据访问层，用于：
   >
   >   1. 访问数据库
   >   1. Shading（非代理模式）
   >
   >> ※ 依赖数据/领域模型层
   >
   >project-module\[n\]-buz：子模块业务层（服务），用于：
   >
   >   1. 业务的实装
   >   1. 事务管理
   >
   >>   ※ 可调用同模块下其他服务，可调用数据访问层
   >>
   >>   ※ 依赖数据访问层、数据/领域模型层
   >
   >project-module[n]-web：子模块网络层，用于：
   >
   >   1. 对外暴露基于RESTFul的业务（服务）API调用
   >   1. 数据校验（利用数据/领域模型层提供的功能）
   >   1. 异常处理
   >
   >>   ※ 可调用业务层
   >>
   >>   ※ **不可**调用同模块下其他网络api
   >>
   >>   ※ 依赖业务层、数据/领域模型层
   >>
   >>   ※ 404异常无法由`@ErrorController`捕获（看过源代码就懂了），须额外配置（如果需要的捕获的话，作为内部服务时出现404的可能性不高）
   >
   >project-module\[n\]-boot：子模块引导层，通常仅包含注解有`@SpringApplication`的`Application.java`类，用于：
   >
   >   1. 子模块作为分布式项目模块打包发布时使用
   >
   >>   ※ 依赖网络层、数据/领域模型层
   >
   >project-module\[n\]-job：子模块任务层（可选），用于：
   >
   >   1. 实装业务的批/流处理
   >   ※ 依赖数据访问层、数据/领域模型层
   >
   >project-module\[n\]-bus：子模块总线层（可选），用于：
   >
   >   1. 实装业务消息队列相关处理
   >
   >>   ※ 依赖业务层、数据/领域模型层
   >
   >project-module\[n\]-api：子模块接口层，用于：
   >
   >   1. 实装基于OpenFeign/dubbo的客户端，供其他模块通过RESTFul/RPC调用
   >
   >>   ※ 依赖网络层、数据/领域模型层
   >
   >project-standalone：通常仅包含注解有`@SpringApplication`的`Application.java`类，用于：
   >
   >   1. 项目作为standalone打包发布时使用
   >
   >>   ※ 依赖所有功能子模块的网络层、数据/领域模型层
   >
   >project-gateway：网关子项目，用于：
   >
   >   1. 路由
   >   1. 鉴权
   >   1. 限流

## DevOps

### 系统

1. Linux

   快被DevOps里关于Linux的内容搞死了。

1. 鸿蒙2

   [官网](https://www.harmonyos.com/cn/home/)

   [源代码](https://openharmony.gitee.com/openharmony)

   [IDE](https://developer.harmonyos.com/cn/develop/deveco-studio)（和安卓类似，是IDEA的魔改版）

### OpenJDK

1. dragonwell

   8用的是`hotspot`，确认下11用的是什么`JVM`。另外配套的诊断工具`Arthas`也得看。目前已提供Windows兼容版（体验版）。**如果不好用，开发者的开发环境需考虑替换成Ubuntu Desktop或者Deepin。**

1. Java-link

   9及之后的`JDK`具备模块化能力，有可能减小`JRE`体积，要确认。

## GitHub

### springfield

1. ~~对`key-value`模块再进行代码复审，虽然重新设计了KeyValueManager，依然不 · 满 · 意。~~ 已重构，已解决了`KeyValueHolder`被注册为bean后可能被滥用的风险。有时间考虑以下场景：用户切换`Locale`。设计Loader时有考虑过这个问题，但切换`Locale`后，holder内的`key-value`s需要up-to-date，可能要为`KeyValues`追加refreshAll方法？

1. 完成`spring-data`和`mybatis`的桥接功能模块。简化MBG的使用（用下`JSR-269`？）。为什么要写个桥接功能？`mybatis-spring`不是用吗？因为`mybatis-spring`和`spring-data`作几乎同样的事但`mybatis`享受不到`spring-data`带来的其他功能，比如分页。只要分页也用不着写这么个玩意儿吧？是的，所以包里还提供了只使用`mybatis`拦截器实现的简易版。

1. 添加一组功能，通过`ThreadLocal`，直接传递分页数据至数据访问端，尽量让开发者无感（配合`spring-data-commmon`的既有功能）

1. 检查项目中`javax`包的使用情况，看看是否需要替换成`Jakarta`命名空间。

### vspx

借用 aspx `CODE BEHIND` 的思想，将ui和api分离，使美工和前端开发者的工作内容分离；使用`JSX/TSX`+`class`语法，希望在必要的时候，可以随时从`vue`、`react`、`java`开发者中抽调人员开发。
**废弃，由`vspx2`配合`vue3`继续。**

### vspx2

1. 需要实现一个类似于`spring-boot`的模块负责将`vue`主节点`mount`到`Html`中，为开发者挂载`vue`节点、配置`vue`（路由、状态机等）、在`vue`生效前后进行处理（启动画面、广告？等）及其他需求提供一个统一的“门面”，避免在index.js里写一大堆。

1. 路由、状态机、i18n等能否独立出来，让`vue-router`等作为其实现使用？

1. 状态机相关的编程依然反人类。目前使用js 装饰器方式实现，需要细化。

### dmc

1. 阶段一

   翻译、整理、学习各功能点。

1. 阶段二

   验证、确认。

1. 阶段三

   整理成册，编写可用于生产的`shell`或者`playbook`。