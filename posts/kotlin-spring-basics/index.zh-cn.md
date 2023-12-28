---
title: "如何用 Kotlin 实现一个 Spring - 基础"
date: 2023-12-12T00:00:00+08:00
lastmod: 2023-12-26T00:00:00+08:00
draft: false
description: "用自己的方式实现一个只属于你的 Spring。"
featuredImage: "cover.webp"

tags: ["Spring", "Kotlin", "技术"]
categories: ["Spring"]
series: ["用自己的方式实现一个只属于你的 Spring"]
series_weight: 1
lightgallery: true
---

最近正在研究 `Spring` 的原理（源码）（程序员苦逼的生活）

<!--more-->

开始之前先问自己几个问题：

- 你是否也想实现一个自己的 `Spring IoC` 容器？
- 你是否对 `Spring` 的运行原理感兴趣？
- 你是否想更深一步的理解 `Spring` 的工作状态？

如果你对上面任意个想法感兴趣，就请继续看吧。本篇会教你如何实现一个最基础的 `Spring IoC` 容器。

至于为什么使用 `Kotlin` 来实现。是因为 `Kotlin` 的语法糖和开发速度要比 `Java` 多很多。
当然，如果你不懂 `Kotlin` 也没关系，涉及 `Kotlin` 特殊语法的地方我会标注出来。

不要害怕不要哭，这个非常简单。

{{< admonition tip "为什么不按照 Spring 源码的结构编写？" >}}
需要注意的是，这系列教程并不是带你手撕源码，带你照抄 Spring 源码。Spring 是一个发展超过 20 年的大型框架，里面有厚重的历史包袱。  

同时它有大量的“历史遗留问题”各种各样的兼容问题，这是不可避免的，Spring 源码中的大量弯弯绕绕，一层又一层的嵌套也有这层原因。  

所以我们只需要理解 Spring 这么做的理念，为什么要这么做，这么做有什么好处，学习精益求精的设计模式和工业级的书写技巧。  

理解了这些，相信你可以在本系列教程中收获颇丰。
{{< /admonition >}}

## 为什么要实现 IoC 容器
对啊！为什么要实现一个 [`IoC`]^(Inversion of Control/控制反转) 容器啊？还有！`IoC` 到底是什么？很好的问题，如果你不理解什么是 `IoC` 就会无法理解 `Spring` 到底做了什么。

要理解什么是控制反转，只需要一个简单的例子：

```kotlin
// 这么写，Kotlin 会默认提供一个有参构造函数
class A(
    // val 意味着一个不可变的成员，类似 java 的 final
    val b: B
) {}

class B(
    val c: C
) {}

class C {}
```

在这个例子中类 `A` 使用类 `B` 作为成员变量，类 `B` 使用类 `C` 作为成员变量，考虑一个情形，我如何才能使用类 `A`？

> 某些人：太简单了，直接 `new` 一个就好了。就像这样
> ```kotlin
> // Kotlin 中 fun 表示为函数或者说是方法
> fun main() {
>     // Kotlin 没有 new 关键字，直接忽略就好，也不需要在句末标记 ';' 
>     val a = A(B(C()))
> }
> ```

很棒，你说的非常对，你成功的创建出了类 `A` 但是如果类 `A` 中有 100 个成员变量呢？或者说如果需要嵌套 100 个类呢？

> 你说：不可能！绝对不可能！什么人写类需要嵌套 100 个类啊！

你说的很对，但是就是有可能，而且你需要了解每一个类的构造方式，你也要构造每一个类，要处理众多的可能性，想想都头皮发麻。

> 这时有个聪明人说：不如这样！我们设计一个系统，让所有类都让它来构建，我们使用的时候就直接去找它要就好了！
> 这样我们就不用自己处理这些复杂的构建了！

这简直太聪明了！这就是 `IoC` 的雏形，我们提供配置信息，由 `IoC` 容器去构建存储所有类，我们只需要调用就好。

借用一下 `Spring` 官方教程中的图片
{{< image src="container-magic.png" caption="容器的逻辑" width="2450" height="1562" >}}

当然，考虑到大家肯定已经是熟练的 `Spring` 板砖工了，我就不细说 `IoC` 的意思了。

## 一个基础的 IoC 容器
现在大家想一下如何实现一个容器？容器，容器，当然需要一个容器去装已经构建好的实例了。
我们目前不用考虑如何去构建等其他情况，我们就考虑如何去存储一个构建好的实例。

最容易想到的是，我们使用一个 `Map<beanName, beanObject>` 来存储，`Map` 最合适不过了，
有高性能的索引，只需要知道名字就可以找到存储的实例。我们可以这样写：
```kotlin
class BeanContainer {
    // mutable 表示可变 map，如果只有 Map 的话是不可变的，不能添加删除成员，只能取出。
    private val container: MutableMap<String, Any> = mutableMapOf()
}
```

这样我们就完成了一个容器，简单吧？

> 你说：就这？这我还用你教？

实际上就是这么简单，之后会变得越来越复杂。

我们可以提供一个方法，帮助用户取出 `Bean` （这里直接使用 Spring 的叫法了）。

```kotlin
class BeanContainer {
    private val container: MutableMap<String, Any> = mutableMapOf()

    fun getBean(beanName: String): Any {
        // 如果 beanName 找出的实例为 null 就直接返回异常
        // container[beanName] 是语法糖，类似 container.get(beanName)
        // ?: 也是语法糖，a ?: b 意思就是：如果 a 为 null 就执行 : 后面的语句
        return container[beanName] ?: throw BeanException("$beanName 不存在!")
    }
}
```

现在我们就有了一个可以使用的容器，当然，目前我们无法向容器中存入任何实例，
也无法让容器自动构建任何类。为了测试，我们提供一个方法，让我们可以手动的存入
一个实例。

```kotlin
    fun saveBean(beanName: String, bean: Any) {
        // 这里也是语法糖，等同 container.put(beanName, bean)
        container[beanName] = bean
    }
```

我们创建一个单元测试来看看，我们实现的容器到底是否可以使用。

```kotlin
import org.junit.jupiter.api.Assertions
import org.junit.jupiter.api.Test

class BeanContainerTest {
    @Test
    fun getBeanTest() {
        //given
        val beanContainer = BeanContainer()
        val testClassA = TestClassA()
        beanContainer.saveBean("testClassA", testClassA)

        //when
        val bean = beanContainer.getBean("testClassA")

        //then
        Assertions.assertEquals(testClassA, bean)
    }

    @Test
    fun getBeanButBeanNameDoseNotExistShouldThrowException() {
        //given
        val beanContainer = BeanContainer()
        val testClassA = TestClassA()
        beanContainer.saveBean("testClassA", testClassA)

        //when
        //这里断言应该抛出一个异常。这个写法是一个语法糖，当函数的最后一个变量是 lambda 语句
        //可以将 lambda 语句单独用 ｛｝花括号提出，更加清晰
        Assertions.assertThrows(BeanException::class.java) {
            val bean = beanContainer.getBean("testClassB")
        }
    }
}
```

经过测试，容器显然是可以使用的。

## 如何自动构建一个 Bean
你已经实现了一个 `Bean`，但是发觉还是要自己构建一个 `Bean` 在放入容器中。这和自己手动构建一个 
`Bean` 没有什么两样，接下来我们就来思考如何构建一个 `Bean`。

你可能会想到使用**反射**来构建一个类，这是一个很好的想法，我们现在就来实现一个。

不过使用反射的话，你至少需要提供 `Class` 信息或完全限定名，当然是提供 `Class` 信息比较方便，
完全限定名太长了。

```kotlin
class BeanContainer {
    private val container: MutableMap<String, Any> = mutableMapOf()
    // 用于存储 bean 的定义，也就是 Class 信息
    private val beanDefinitions: MutableMap<String, Class<*>> = mutableMapOf()

    fun getBean(beanName: String): Any {
        // 如果 beanName 找出的实例为 null 就创建一个
        // container[beanName] 是语法糖，类似 container.get(beanName)
        // ?: 也是语法糖，a ?: b 意思就是：如果 a 为 null 就执行 : 后面的语句
        return container[beanName] ?: doCreateBean(beanName)
    }

    fun saveBean(beanName: String, bean: Any) {
        // 这里也是语法糖，等同 container.put(beanName, bean)
        container[beanName] = bean
    }

    fun saveBeanDefinition(beanName: String, beanClass: Class<*>) {
        beanDefinitions[beanName] = beanClass
    }
    
    private fun doCreateBean(beanName: String): Any {
        val beanClass = beanDefinitions[beanName] ?: throw BeanException("$beanName 的定义不存在！")
        // 反射创建
        return beanClass.getDeclaredConstructor().newInstance()
    }
}
```

这里我们创建了一个 `beanDefinitions` 容器，用于存储 `bean` 的定义，提供了 `doCreateBean` 方法，并在 `getBean` 中使用，如果在容器中没有发现 `Bean` 我们就现场创建一个，如果没有定义我们就抛出异常。

单元测试：
```kotlin
@Test
fun getBeanNewInstance() {
    //given
    val beanContainer = BeanContainer()
    beanContainer.saveBeanDefinition("testClassA", TestClassA::class.java)

    //when
    val bean = beanContainer.getBean("testClassA")

    //then
    Assertions.assertNotNull(bean)
}

@Test
fun getBeanButDefinitionDoesNotExistShouldThrowException() {
    //given
    val beanContainer = BeanContainer()

    //when
    Assertions.assertThrows(BeanException::class.java) {
        beanContainer.getBean("testClassB")
    }
}
```

测试通过！

到这里我们已经正常的实现了一个简单的 `IoC` 容器。

## 作用域
经常用 `Spring` 的人知道，默认情况下 `Spring` 的 `bean` 是单例模式的，也就是说，从容器中取出的
`bean` 默认都是相同的。

不知道你发现了么？我们上面编写的代码并不能实现这个功能，我们只能每次取出一个不同的 `bean`，
也就是说，默认都是多例模式，我们下载可以来实现一下单例模式。

当然，不能设置所有的类都是多例或者单例，我们需要可以设置。`beanDefinitions` 当然就是最好的选择。
目前我们的定义只能设置类型，不能设置作用域。我们可以将这个定义单独提取出来作为一个类存放。

```kotlin
class BeanDefinition(
    val type: Class<*>,
    // 提供默认值
    val scope: BeanScope = BeanScope.SINGLETON,
)

enum class BeanScope{
    SINGLETON, PROTOTYPE
}
```
这样我们就实现了一个初步的 `beanDefinition` 其中包含，bean 的类型，作用域。作用域中有单例模式
和多例模式的枚举。

```kotlin
class BeanContainer {
    private val container: MutableMap<String, Any> = mutableMapOf()
    // 用于存储 bean 的定义
    private val beanDefinitions: MutableMap<String, BeanDefinition> = mutableMapOf()

    fun getBean(beanName: String): Any {
        // 如果 beanName 找出的实例为 null 就创建一个
        // container[beanName] 是语法糖，类似 container.get(beanName)
        // ?: 也是语法糖，a ?: b 意思就是：如果 a 为 null 就执行 : 后面的语句
        val beanDefinition = beanDefinitions[beanName] ?: throw BeanException("$beanName 的定义不存在！")
        return container[beanName] ?: doCreateBean(beanName, beanDefinition)
    }

    fun saveBean(beanName: String, bean: Any) {
        // 这里也是语法糖，等同 container.put(beanName, bean)
        container[beanName] = bean
    }

    fun saveBeanDefinition(beanName: String, beanDefinition: BeanDefinition) {
        beanDefinitions[beanName] = beanDefinition
    }

    private fun doCreateBean(beanName: String, beanDefinition: BeanDefinition): Any {
        val beanClass = beanDefinition.type
        // 反射创建
        val instance = beanClass.getDeclaredConstructor().newInstance()
        // 如果 bean 的定义是单例模式，就将创建出的 bean 加入容器
        if (beanDefinition.scope == BeanScope.SINGLETON) saveBean(beanName, instance)
        return instance
    }
}
```

修改以下 `doCreateBean` 方法，让他直接传入定义，然后用定义创建实例，创建后判断是否为单例模式，
如果是就存入容器，如果不是就直接返回。下次请求 `bean` 的时候就会重新创建。

单元测试：
```kotlin
@Test
fun getBeanMustSame() {
    //given
    val beanContainer = BeanContainer()
    val beanDefinition = BeanDefinition(TestClassA::class.java)
    beanContainer.saveBeanDefinition("testClassA", beanDefinition)

    //when
    val beanA = beanContainer.getBean("testClassA")
    val beanB = beanContainer.getBean("testClassA")

    //then
    Assertions.assertEquals(beanA, beanB)
}

@Test
fun getBeanPrototypeMustNotSame() {
    //given
    val beanContainer = BeanContainer()
    val beanDefinition = BeanDefinition(TestClassA::class.java, BeanScope.PROTOTYPE)
    beanContainer.saveBeanDefinition("testClassA", beanDefinition)

    //when
    val beanA = beanContainer.getBean("testClassA")
    val beanB = beanContainer.getBean("testClassA")

    //then
    Assertions.assertNotEquals(beanA, beanB)
}
```

这样我们就完成了作用域功能。

## 小结
到目前为止，我们完成了容器的基本功能，提供了作用域和自动创建功能。

下节我们将探索如何进行依赖注入，尽情期待。

{{< admonition tip "源码" >}}
项目所有源码可以在 [Cgglyle's Github Kotlin-Spring](https://github.com/cgglyle/kotlin-spring)
中找到
{{< /admonition >}}