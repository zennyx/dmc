# 备忘录

## Development

### 语言

1. Go

   优先度高。生产力语言，性能高。适用于微服务开发及DevOps（为了看懂`k8s`和`Istio`的源代码也必须看）。

1. Kotlin

   优先度中。生产力语言，实现同样逻辑，代码量约为`Java`的2/3。可与`Java`互操作，适用于所有`Java`应用的场合，包括Android开发。和`Java`语法并不非一一对应，且有些设定反人类，早期对`Eclipse`的支持也不够好。

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

   3.0。新特性很多，`Tree-shaking`有望进一步减小前端包的体积，配合开发的`Vite`也要了解，也许能大幅减少开发时等待转译的时间。

1. Kubernetes/Istio

   看完`go`看源代码。

1. Spark/Flink

   流处理。

## DevOps

### 系统

1. Linux

   快被DevOps里关于Linux的内容搞死了。

### OpenJDK

1. dragonwell

   8用的是`hotspot`，确认下11用的是什么`JVM`。另外配套的诊断工具`Arthas`也得看。

1. Java-link

   9及之后的`JDK`具备模块化能力，有可能减小`JRE`体积，要确认。

## GitHub

### springfield

1. 对`key-value`模块再进行代码复审，虽然重新设计了KeyValueManager，依然不 · 满 · 意。

1. 完成`spring-data`和`mybatis`的桥接功能模块。简化MBG的使用（用下`JSR-269`？）。

1. 检查项目中`javax`包的使用情况，看看是否需要替换成`Jakarta`命名空间。

### vspx

废弃，由`vspx2`配合`vue3`继续。

### vspx2

1. 需要实现一个类似于`spring-boot`的模块负责将`vue`主节点`mount`到`Html`中，为开发者挂载`vue`节点、配置`vue`（路由、状态机等）、在`vue`生效前后进行处理（启动画面、广告？等）及其他需求提供一个统一的“门面”，避免在index.js里写一大堆。

1. 路由、状态机、i18n等能否独立出来，让`vue-router`等作为其实现使用？

1. 状态机相关的编程依然反人类，需要简化。

### dmc

1. 阶段一

   翻译、整理、学习各功能点。

1. 阶段二

   验证、确认。

1. 阶段三

   整理成册，编写可用于生产的`shell`或者`playbook`。