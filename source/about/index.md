---
title: 关于我
date: 2019-01-16 11:15:44
type: "about"
layout: "about"
---
# 联系方式
- Email：hilin2333@gmail.com
- 微信号：idiot_wakaka

# 个人信息
 - 男/1992.08.25
 - 本科/西华师范大学/信息与计算科学
 - 工作年限：4年
 - Github：http://github.com/silencecorner
 - 公司后台主程、研发组长，拥有较强的技术能力，喜欢钻研技术

# 工作期望
 - 期望职位：高级Java开发工程师
 - 期望薪资：税前月薪20k~30k
 - 期望城市：北京

# 技能清单
- 熟练java、java8编程，代码习惯良好。
- 能够熟练使用常见的设计模式
- 能够熟练使用持续集成工具Jenkins、docker、k8s
- 熟练使用mysql、redis等关系型、非关系型数据库
- 熟练使用spring boot、spring cloud技术栈下的相关技术
- 熟悉grpc、protobuf、graphql-java
- 熟悉go、kotlin
- 有一定的shell脚本能力
- 对service mesh架构(istio、sofamesh)有一定的认识

## 北斗国信智能科技有限公司（2018年3月 ~ 至今）
### 北斗位置服务平台 
#### 工作描述
- 系统设计，做项目技术报告，编写设计文档
- 编写认证、授权服务、项目基础代码
- 编写对外的open api文档
- 功能拆解，分派任务
- 对其他开发同学提供技术支持
- 设计代码规范
- code review

#### 项目介绍
北斗位置服务平台是一个saas平台，主要为公司内部和第三方提供位置数据托管、基于位置数据的功能抽象，提供统一的车、人、物位置信息服务，减少重复开发成本，解决位置数据并发度高、数据处理复杂。项目设计之初就对可靠性、稳定性、可维护性、减少技术债务做一定的考量，最终采用了传统的分布式->微服务的方式。
#### 项目技术
- mycat
- spring boot/spring security
- mybatis
- redis
- mysql

### 北斗关爱平台
#### 工作描述
- 服务设计，定义grpc、graphql
- 编写模块设计文档
- 功能拆解，分派任务
- 对其他开发同学提供技术支持
- 为服务添加istio支持以及istio运维实例
- code review

#### 项目介绍
该项目采用了service mesh架构体系，istio工具集将断路器、注册发现、动态路由等微服务中常见功能都已实现，开发者无从感知。网关采用的是graphql，提供灵活的接口，减少重复代码；微服务之间采用的protobuf接口定义，使用grpc作为传输载体；所有的服务在测试环境都已经容器化，更方便运维人员做持续集成。该平台主要服务对象为老人以及儿童，通过一些iot设备与平台组织建立联系，老人可通过来电订购服务或商品，子女通过关爱端app为老人下单结账，还可以建立亲情圈加强亲人之间的联系。项目主要分为几大模块：人员组织管理、电商系统、iot应用以及通用数据管理，这其中有每个模块中又分为几个单独的应用、应用之间通过grpc交互，例如：电商系统中分为产品微服务、店铺管理、订单系统。
#### 项目技术
- docker/k8s/istio
- atlassian tools (研发工具，持续集成)
- graphql-java
- emqtt (iot broker)
- kafka
- redis/mysql

## 四川企之道软件有限公司（2016年6月 ~ 2018年3月）
### 圈知道APP 
#### 工作描述
- 参与产品功能设计，平衡实现和功能两级分化
- 基础框架搭建编写项目基础代码
- 功能拆解，分派任务，提供技术支持
- 编写代码规范、code review

#### 项目介绍
该项目采用了微服务架构，包括了红包系统、朋友圈、人脉系统、任务系统、用户系统等；在该系统中红包系统作为类秒杀系统，采用预先分包策略、数据库添加乐观锁，使用redis缓存提高系统的并发度和可靠性，在红包系统中与常见商品秒杀系统不太一样的是商品秒杀可以通过边缘计算减少应用服务器的命中次数，红包因为是顺序的，所以所有流量都在应用服务器上，其中对服务器代码进行了大量优化，经过大量压测之后红包系统的上线之后稳定运行;使用nginx+keepalived实现高可用，使用的编程式tcc事务控制（比较麻烦），使用该技术方案是为了技术复杂度，相应的代码量是正常6倍左右；在设计之初项目中就不允许使用连接查询，对复杂的sql进行拆分以提高性能，为后期数据库拆分留下了条件。
#### 项目技术
- nginx + keepalived
- spring cloud
- mybatis
- redis
- mysql

### 宏华技术服务管理信息系统 
#### 工作描述
- 负责项目实施工作，将用户意见反馈给leader
- 对新增模块进行代码编写，对部分模块进行重写

#### 项目介绍
该项目是对已有老系统进行升级改造，新增培训模块，对现有的业务流程进行改造, 对人员调度功能进行优化, 对合同规定的其他模块进行优化;其中主要是变更系统中的售后服务流程实现逻辑，从需求接收、需求任务核实到任务收款实现任务提交给个人体现人员调动的一个流程控制、任务明细、报表统计等，使系统符合客户公司实际的售后服务流程。

#### 项目技术
- SpringMvc
- Hibernate
- Oracle
- Extjs
  
  