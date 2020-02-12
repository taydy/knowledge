#### 什么是 IoC ？

简单来说，IoC 是反转控制，类似于好莱坞原则，主要有依赖查找和依赖注入实现。

#### 依赖查找和依赖注入的区别？

依赖查找是主动或手动的依赖查找方式，通常需要依赖容器或标准 API 实现。而依赖注入则是手动或自动依赖绑定的方式，无需依赖特定的容器和 API。

#### Spring 作为 IoC 容器有什么优势？

- 典型的 IoC 管理，依赖查找和依赖注入
- AOP 抽象
- 事务抽象
- 事务机制
- SPI 扩展
- 强大的第三方整合
- 易测试性
- 更好的面向对象

#### 轻量级容器有哪些特征？

- 能够管理应用代码；
- 能够快速启动；
- 容器不需要特殊的配置；
- 容器能够达到比较轻量级的内存占用以及最小化的 API 依赖；

#### Spring IoC 依赖来源

- 自定义 bean
- 容器内建 Bean 对象，如 Environment
- 容器内建依赖，如 BeanFactory

#### Spring IoC 配置元信息

- Bean 定义配置
  - 基于 XML 文件
  - 基于 Properties 文件
  - 基于 Java 注解
  - 基于 Java API
- IoC 容器配置
  - 基于 XML 文件
  - 基于 Java 注解
  - 基于 Java API
- 外部化属性配置
  - 基于 Java 注解

#### BeanFactory 与 FactoryBean 的区别？

BeanFactory 是 IoC 底层容器；

FactoryBean 是创建 Bean 的一种实现方式，帮助实现复杂的初始化逻辑。



#### BeanFactory 和 ApplicationContext 谁才是 Spring IOC 容器？

BeanFactory 是底层的 IoC 容器，ApplicationContext 具有应用特性的 BeanFactory 超集。额外提供了以下功能：

- 面向切面 AOP；
- 配置元信息 Configration Metadata;
- 资源管理 Resources;
- 事件 Events;
- 国际化 i18n;
- 注解 Annotations;
- Environment 抽象。

BeanFactory 和 ApplicationContext 是同一类事物，只不过在底层实现上 ApplicationContext 组合了一个 BeanFactory 实现。

#### Spring IoC 容器启动时做了哪些准备？

IoC 配置元信息读取和解析、IoC 容器生命周期、Spring 事件发布、国际化等。

#### Spring IoC 启动过程

1. 创建 BeanFactory，并进行初步的初始化，加入一些内建的 Bean 对象或者 Bean 依赖，以及加上一些内建的非 Bean 的依赖。
2. 。。。



### Spring Bean

#### 什么是 BeanDefinition？

BeanDefinition 是 Spring Framework 中定义 Bean 的配置元信息接口，包含：

- Bean 的类名；
- Bean 行为配置元信息，如作用域、自动绑定的模式、生命周期回调等；
- 其他 Bean 引用，又可称作合作者(Collaborators)或者依赖(Dependencies)；
- 配置设置，比如 Bean 属性(Properties)。