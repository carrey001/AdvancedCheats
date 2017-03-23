## Dagger2
---
Dagger2 是一种依赖注入（Dependency Injection，简称DI） 是用于减少计算机程序之间耦合的一个方法。
Dagger2 使用代码字段生成创建依赖关系所需要的代码，减少公式化的代码。降低耦合。

**Dagger2的引用 **
- 在整个项目的 build.gradle中加入：

```
dependencies {
   // other classpath definitions here
   classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
}
```
- 在app/build.gradle中分别加入：

```
// add after applying plugin: 'com.android.application'  
apply plugin: 'com.neenbedankt.android-apt'

```

```
dependencies {
  // apt command comes from the android-apt plugin
  apt 'com.google.dagger:dagger-compiler:2.2'
  compile 'com.google.dagger:dagger:2.2'
  provided 'javax.annotation:jsr250-api:1.0'
}

```
> 需要注意的是provided代表编译时需要的依赖，Dagger的编译器生成依赖关系的代码，并在编译时添加到IDE 的class path中，只参与编译，并不会打包到最终的apk中。apt是由android-apt插件提供，它并不会添加这些类到class path中，这些类只用于注解解析，应当避免使用这些类。

** Dagger2中的注解**
---

注解|用法
-------|--------
@module|Modules类里面的方法专门提供依赖，所以我们定义一个类，用@Module注解，这样Dagger在构造类的实例的时候，就知道从哪里去找到需要的 依赖。modules的一个重要特征是它们设计为分区并组合在一起（比如说，在我们的app中可以有多个组成在一起的modules)
@Provide|在modules中，我们定义的方法是用这个注解，以此来告诉Dagger我们想要构造对象并提供这些依赖。
@Singleton|	当前提供的对象将是单例模式 ,一般配合@Provides一起出现
@Component|	用于接口，这个接口被Dagger2用于生成用于模块注入的代码
@Inject|	在需要依赖的地方使用这个注解。（你用它告诉Dagger这个 构造方法，成员变量或者函数方法需要依赖注入。这样，Dagger就会构造一个这个类的实例并满足他们的依赖。）
@Scope|	Scopes可是非常的有用，Dagger2可以通过自定义注解限定注解作用域。

----
