## 1. 背景

>  QMSSO早期流程很鸡肋，用户登录成功后系统顶级域名下写Cookie，用户再次访问其他应用通过Cookie到SSO获取会话数据。

**老方案存在的问题**

* 子系统之间处理数据同步的场景比较麻烦。
* 统一退出比较慢，需要通过HTTP协议逐一通知子系统。
* 登录页面样式不统一，且没有使用到组件化的方式构建页面。
* 项目中夹杂许多难以维护的代码。

## 2. 系统改造

### 目标

* 抽象会话数据存储方案。
* 使用消息通知机制。
* 多应用之间使用Spring Session统一会话数据。
* 解决单应用多负载数据同步问题。
* 重构前端页面。
* 简化部署，方便测试。

### 场景

* 业务场景思维导图

![应用场景](http://p2y607sfp.bkt.clouddn.com/WechatIMG120.jpeg)

**系统结构划分**

| 模块名            | 模块介绍      | 使用端口 | 必须https | 请求路径           | 启动顺序 |
| -------------- | --------- | ---- | ------- | -------------- | ---- |
| sso-server     | cas服务     | 8443 | √       | cas            | 2    |
| sso-config     | 配置中心      | 8889 | ×       | config         | 1    |
| sso-management | service管理 | 8081 | √       | cas-management | 3    |
| sso-monitor    | 监控服务      | 8444 | ×       | /              | 5    |
| sso-frontend   | 前端项目      | 3000 | ×       | /              | 单独启动 |
| sso-support    | 基础模块      | -    | -       | -              | -    |
| sso-rest-api   | 接口服务      | 8881 | ×       | api            | 4    |

**1. CAS简介**

CAS系统由两部分组件构成，分别是CAS Clients和CAS Server。Clients和Server之间可以通过多种协议通信。例如：CAS协议、SAML协议、OAuth协议等。

**1.1 CAS应用场景**

* 分布式多系统用户集中管理
* 用户权限集中管理
* 多因素认证

**1.2 CAS特性**

* Service Management
  * Authorized services: 授权哪些Service可以加入cas单点会话中
  * 强制认证模式
  * 自定义属性: Attribute release
  * Proxy control: 授权策略
  * 主题控制: 自定义客户端登录页

**1.3 讨论问题**

* SSO存在的意义—用户只需要登录一次就可以访问相互信任的应用系统
* Ticket票据统一存储问题，各节点访问的一致性。—Tomcat Session集群同步。
* 移动端REST API认证。

## 3. 技术架构

CAS系统实现鉴权功能，鉴权成功后将用户信息写入到REDIS中，通过Spring Session+REDIS实现分布式共享会话机制。



**3.1 认证主题页渲染**



**3.2 客户端接入规则**



**3.3 REST认证模式**

QMCAS对于采用RESTful接口认证模式的客户端是透明的，客户端将用户名及密码提交给SSO-REST-API而不需要直接和SSO交互，API首先通过SSO鉴权后获取用户信息创建token返回给客户端。客户端通过token请求API相关数据接口。

![客户端认证模式](http://p2y607sfp.bkt.clouddn.com/1516956533852.jpg)