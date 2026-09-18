---
title: 谈谈Spring的启动过程
date: 2026-00-00
categories:
  - 八股
tags:
  - Java
  - Spring
---

## 谈谈Spring的启动过程

> **创建容器 → 注册 BeanDefinition → BeanFactory 后置处理器 → Bean 后置处理器 → 实例化 → 依赖注入 → 初始化 → AOP 代理 → 容器启动完成**

1. **创建并刷新 ApplicationContext**

   * 创建 Spring IoC 容器
   * 加载配置和环境信息

2. **解析并注册 BeanDefinition**

   * 通过组件扫描、配置类等方式发现 Bean
   * 将 Bean 的定义信息封装成 `BeanDefinition`
   * 注册到 `BeanFactory`

3. **执行 BeanFactory 后置处理器**

   * 在 Bean 实例化之前执行
   * 对 `BeanDefinition` 进行修改和处理

4. **注册 Bean 后置处理器**

   * 在 Bean 创建和初始化过程中执行
   * 提供各种扩展能力

5. **实例化 Bean**

   * 创建非懒加载的单例 Bean
   * 通过构造方法等方式创建对象

6. **依赖注入**

   * 解析 Bean 之间的依赖关系
   * 完成 `@Autowired` 等依赖注入

7. **初始化 Bean**

   * 执行 `Aware` 回调
   * 执行 `@PostConstruct`
   * 执行 `InitializingBean`
   * 执行 Bean 后置处理器相关逻辑

8. **创建 AOP 代理**

   * 对需要代理的 Bean 创建代理对象
   * 例如 `@Transactional`、切面等

9. **容器启动完成**

   * 所有需要初始化的 Bean 创建完成
   * 发布容器刷新完成事件
   * `ApplicationContext` 启动完成


### BeanFactory 和ApplicationContext有什么区别?

BeanFactory 是 Spring IoC 容器的基础接口，主要负责 Bean 的创建、获取、依赖注入和生命周期管理。ApplicationContext 在 BeanFactory 的基础上进行了扩展，增加了事件发布、国际化、资源加载、环境配置等功能，是实际项目中更常用的 IoC 容器