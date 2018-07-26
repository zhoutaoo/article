spring-cloud打造微服务平台

# 简述

和Spring Boot 是什么关系
Spring Boot 是 Spring 的一套快速配置脚手架，可以基于Spring Boot 快速开发单个微服务，Spring Cloud是一个基于Spring Boot实现的云应用开发工具；Spring Boot专注于快速、方便集成的单个个体，Spring Cloud是关注全局的服务治理框架；Spring Boot使用了默认大于配置的理念，很多集成方案已经帮你选择好了，能不配置就不配置，Spring Cloud很大的一部分是基于Spring Boot来实现。

spring -> spring booot > Spring Cloud 这样的关系。

服务注册、发现: eureka

配置管理:spring config , spring security

集群容错: hystrix（待实现）

API网关: zuul（待实现）

服务负载:feign+ribbon

api文档输出:swagger2

代码简化:lombok

消息队列:rabbitmq

分布式锁: redis （待实现）

链路跟踪:spring cloud sletuh ->zipkin

安全认证:oauth2/JWT(待实现)

服务监控:spring-boot-admin

# 微服务开发

## 注册中心


## 配置中心


## 消息中心


## 网关服务


## 授权与认证



# 微服务发布

## 服务打包

## 服务部署



# 微服务运维

## 监控中心

## 调度中心

## 追踪中心