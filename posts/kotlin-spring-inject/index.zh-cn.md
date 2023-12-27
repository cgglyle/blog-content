---
title: "如何用 Kotlin 实现一个 Spring - 依赖注入"
date: 2023-12-14T00:00:00+08:00
lastmod: 2023-12-26T00:00:00+08:00
draft: false
description: "用自己的方式实现一个只属于你的 Spring。"
featuredImage: "featured-image.webp"

tags: ["Spring", "Kotlin", "技术"]
categories: ["Spring"]
series: ["用自己的方式实现一个只属于你的 Spring"]
series_weight: 2
lightgallery: true
---


上节完成了基本的功能，这节我们将探索依赖注入和如何解决循环依赖。

<!--more-->

## 依赖注入

目前我们实现了基本的功能，但是你发现了么？我们测试的类是没有成员变量的，完全没有。  我们只能实现通过无参构造函数进行实例化。  

所以，我们现在来实现依赖注入。这样 `Spring` 就会自动在容器中找出符合的 `Bean` 注入进去。

我们先来创建一个有变量的类：

```kotlin
class TestClassB {
    // lateinit 意思延迟赋值，否则现在就要赋值，这是 kotlin 的特性
    lateinit var testClassA: TestClassA
}
```

这样我们在生成 `TestClassB` 的时候就需要注入 `TestClassA` 否则用户使用的时候就会出问题。

如何做到呢？还是`反射` ，使用反射创建比较方便注入，我们目前就使用这种方法。在哪里注入？当然是在 `doCreateBean` 位置注入。

但是我们现在需要考虑， `Spring` 怎么知道成员需要注入什么东西？实际上是 `Spring` 在加载配置文件和扫描的时候构建了 `BeanDefinition` 时将参数放入了定义，这样在实例化类的时候就可以根据定义进行注入。

我们先创建一个用于存放参数的类：

```kotlin
class PropertyValue(
    val name: String,
    val value: Any,
)
```

我们修改一下 `BeanDefinition`：

```kotlin {linenos=table,hl_lines=[4,"6-9"]}
class BeanDefinition(
    val type: Class<*>,
    // 提供默认值
    val scope: BeanScope = BeanScope.SINGLETON,
){
    val propertyValues: MutableList<PropertyValue> = mutableListOf()
    fun setProperty(name: String, value: Any) {
        propertyValues.add(PropertyValue(name, value))
    }
}
```

添加了一个 `propertyValues` 作为存储 `依赖`。

我们在 `BeanContainer` 类中修改以下方法：

```kotlin {linenos=table,hl_lines=[6,7,"12-24"]}
private fun doCreateBean(beanName: String, beanDefinition: BeanDefinition): Any {
    val beanClass = beanDefinition.type
    // 反射创建
    val instance = beanClass.getDeclaredConstructor().newInstance()
    // 如果 bean 的定义是单例模式，就将创建出的 bean 加入容器
    if (beanDefinition.scope == BeanScope.SINGLETON) saveBean(beanName, instance)
    // 注入依赖
    doInjectProperty(beanDefinition, instance, beanClass)
    return instance
}

private fun doInjectProperty(beanDefinition: BeanDefinition, bean: Any, beanClass: Class<*>) {
    // 循环依赖
    beanDefinition.propertyValues.forEach {
        // 使用反射找到与依赖名称相同的方法
        val declaredField = beanClass.getDeclaredField(it.name)
        // 如果方法不能访问，就设置访问
        if (!declaredField.canAccess(bean)) {
        declaredField.trySetAccessible()
        }
        // 注入依赖
        declaredField.set(bean, it.value)
    }
}
```

我们修改一下测试类：

```kotlin
class TestClassA {
    // lateinit 意思是延迟赋值。
    // 因为 kotlin 特性，所有变量都需要有初始值
    lateinit var name: String
}
```

编写一个单元测试：

```kotlin
@Test
fun getBeanClassANameMustNotBeNull() {
    //given
    val beanContainer = BeanContainer()
    val beanDefinitionA = BeanDefinition(TestClassA::class.java)
    beanDefinitionA.setProperty("name", "testClassA")
    beanContainer.saveBeanDefinition("testClassA", beanDefinitionA)

    //when
    val beanA = beanContainer.getBean("testClassA")

    //then
    //as 关键字为强制转换的意思，等同于 (TestClasB) beanB
    Assertions.assertEquals("testClassA", (beanA as TestClassA).name)
}
```

完美通过测试，测试类正常的注入了名称。

## 注入 Bean

这里我们考虑一种情况，假如我们需要注入的是其他的 `Bean` 我们该怎么做？

> 有人说：那我们直接用名字加 `Bean` 的名字就好了啊。  

你说的对，但是目前我们的方法只会将 `Bean` 的名字注入，而名字是 `String` 类型， `Bean` 是其他类型，不能强制转换肯定会抛出异常。

这里我们可以使用类引用来处理，也就是 `BeanReference` 来处理，当发现 `PropertyValue` 的 `value` 是 `BeanReference` 类型，我们就去 `getBean` 这样我们就得到了 `Bean` 然后注入进去就好了。

我们创建一个 `BeanReference`：

```kotlin
class BeanReference(
    val beanName: String,
)
```

我们更改一下 `doInjectProperty` 方法：

```kotlin {linenos=table,hl_lines=["11-17"]}
private fun doInjectProperty(beanDefinition: BeanDefinition, bean: Any, beanClass: Class<*>) {
    // 循环依赖
    beanDefinition.propertyValues.forEach {
        // 使用反射找到与依赖名称相同的方法
        val declaredField = beanClass.getDeclaredField(it.name)
        // 如果方法不能访问，就设置访问
        if (!declaredField.canAccess(bean)) {
            declaredField.trySetAccessible()
        }
        // is 关键字等同于 instanceof, 而且 is 关键字会自动将在上下文中将 value 转换为 BeanReference，不用显式转换。
        if (it.value is BeanReference) {
            // 通过 getBean 获得 Bean
            declaredField.set(bean, getBean(it.value.beanName))
        } else {
            // 注入依赖
            declaredField.set(bean, it.value)
        }
    }
}
```

我们编写一个单元测试：

```kotlin
class TestClassB {
    // lateinit 意思延迟赋值，否则现在就要赋值，这是 kotlin 的特性
    lateinit var testClassA: TestClassA
}

@Test
fun getBeanClassBPropertyClassAMustNotBeNull() {
    //given
    val beanContainer = BeanContainer()
    val beanDefinitionA = BeanDefinition(TestClassA::class.java)
    beanDefinitionA.setProperty("name", "testClassA")
    beanContainer.saveBeanDefinition("testClassA", beanDefinitionA)

    val beanDefinitionB = BeanDefinition(TestClassB::class.java)
    beanDefinitionB.setProperty("testClassA", BeanReference("testClassA"))
    beanContainer.saveBeanDefinition("testClassB", beanDefinitionB)

    //when
    val beanA = beanContainer.getBean("testClassA")
    val beanB = beanContainer.getBean("testClassB")

    //then
    //as 关键字为强制转换的意思，等同于 (TestClasB) beanB
    Assertions.assertEquals(beanA, (beanB as TestClassB).testClassA)
}
```

完美通过！

## 循环依赖

很高兴你能看到这里，你也许已经对解决 `Spring` 循环依赖问题已经非常熟悉，但是也许你对为什么要这么样做有些疑问，请继续向下看。

让我们先写一个单元测试：

```kotlin
class TestClassC {
    lateinit var testClassD: TestClassD
}
class TestClassD {
    lateinit var testClassC: TestClassC
}
@Test
fun circularDependencyTest() {
    //given
    val beanContainer = BeanContainer()
    
    val beanDefinitionC = BeanDefinition(TestClassC::class.java)
    beanDefinitionC.setProperty("testClassD", BeanReference("testClassD"))
    beanContainer.saveBeanDefinition("testClassC", beanDefinitionC)
    
    val beanDefinitionD = BeanDefinition(TestClassD::class.java)
    beanDefinitionD.setProperty("testClassC", BeanReference("testClassC"))
    beanContainer.saveBeanDefinition("testClassD", beanDefinitionD)
    
    
    //when
    val beanC = beanContainer.getBean("testClassC") as TestClassC
    val beanD = beanContainer.getBean("testClassD") as TestClassD
    
    //then
    Assertions.assertNotNull(beanC)
    Assertions.assertNotNull(beanD)
    
    Assertions.assertNotNull(beanC.testClassD)
    Assertions.assertNotNull(beanD.testClassC)
}
```

这里我们构建了一个 `C` 依赖 `D`，而 `D` 由依赖 `C` 的类，这样写并没有错。正常也可以通过其他方式构建出来这种类。例如：

```kotlin
fun test(){
    val c = TestClassC()
    val d = TestClassD()
    c.testClassD = d
    d.testClassC = c
}
```

这样我们就构建出了一个循环依赖的类，实际上这就是我们想要的结果。如何打破依赖，只需要临时存储就好了。

但是当你使用单元测试进行测试的时候，你发现竟然通过了！怎么可能？我就等着循环依赖出现呢，我等着堆栈溢出的异常呢！

你开始 `Debug` 找出没有引发异常的异常最终你发现了问题的所在。

```kotlin {linenos=table,hl_lines=["5-10"]}
private fun doCreateBean(beanName: String, beanDefinition: BeanDefinition): Any {
    val beanClass = beanDefinition.type
    // 反射创建
    val instance = beanClass.getDeclaredConstructor().newInstance()
    // 就在这里 ===================================================================
    // 如果 bean 的定义是单例模式，就将创建出的 bean 加入容器
    if (beanDefinition.scope == BeanScope.SINGLETON) saveBean(beanName, instance)
    // 注入依赖
    doInjectProperty(beanDefinition, instance, beanClass)
    // 就在这里 ===================================================================
    return instance
}
```

我们知道当 `doInjectProperty` 方法进行依赖注入的时候，如果发现需要注入的是一个 `Bean` 就去 `getBean`，在容器中寻找。

而我们这里为什么没有出现异常呢？是因为我们在注入之前就将还没有完全实例化走完完整声明周期的 `Bean` 放入了容器中。当 `getBean` 去寻找的时候就会发现相应的 `Bean` ，这就是问题的所在。

> 你说：挺好的啊？这不是解决了么？也不用处理循环依赖了。  

乍一看确实挺好，以后再也没有循环依赖的问题了。但是！你没有考虑到并发问题，如果在你实例化 `Bean` 的时候有另一个线程去请求了这个 `Bean` 呢？这样的话，这个线程就会拿到一个没有完整实例化的 `Bean`，并发问题就出现了，而且难以排查。你根本没有办法知道当时的 `Bean` 是什么样子的，实例化到什么程度了。

> 你说：害，不就是并发么，直接上锁，我 TM 锁锁锁！  

锁挺好，但是不要“贪杯”哦！这样确实能解决问题，但是对性能的影响也是巨大的！

> 你说：Spring 不都是在启动时初始化么，启动的时候慢一点也没啥。之后全部生成完不就快了。  

也对，但是你没有考虑到，还有很多的 `Bean` 都是懒加载，当使用的时候才会进行实例化。这时候就会出问题了。

现在我们需要让容器中的 `Bean` 必须都是完整实例化的 `Bean`，只有这样我们才能保证你能 `getBean` 出来的都是完整的。`get` 不出来的要么是不存在，要么是出问题了。

我们将 `doInjectProperty` 移动到存入容器方法的上方。再次启动单元测试，喜闻乐见的 `StackOverflowError` 出现了。

```kotlin {linenos=table,hl_lines=[6,8]}
private fun doCreateBean(beanName: String, beanDefinition: BeanDefinition): Any {
    val beanClass = beanDefinition.type
    // 反射创建
    val instance = beanClass.getDeclaredConstructor().newInstance()
    // 注入依赖
    doInjectProperty(beanDefinition, instance, beanClass)
    // 如果 bean 的定义是单例模式，就将创建出的 bean 加入容器
    if (beanDefinition.scope == BeanScope.SINGLETON) saveBean(beanName, instance)
    return instance
}
```

### 解决

那我们该如何解决这个问题呢？

实际上解决的重点就是刚刚的方法，将没有实例化完成的方法临时放到一个容器中，这样就可以打破循环。我们新创建一个容器用于存放没有完成实例化的实例。

```kotlin {linenos=table,hl_lines=[7]}
class BeanContainer {
    private val container: MutableMap<String, Any> = mutableMapOf()

    // 用于存储 bean 的定义
    private val beanDefinitions: MutableMap<String, BeanDefinition> = mutableMapOf()
    // 早期未实例化的 Bean 容器
    private val earlyBeanContainer: MutableMap<String, Any> = mutableMapOf()

    // ...
｝
```

先让我们暂停一下，我们需要考虑一个新的问题。我们怎么知道这个类会不会循环依赖？我们不可能等真的堆栈溢出的时候才处理这个问题，那时候就晚了。（好像也行，但是太不优雅了，而且一直抛出异常也会影响性能）。

我们考虑这种情况：

```text
    实例化 TCA(TestClassA) -> 发现需要注入 TCB(TestClassB) -> 实例化 TCB -> 发现需要注入 TCA
    ==> 发生循环依赖
```

假设我们在外部存放一个列表 `List<beanName>` 每当正在创建一个类的时候就将这个类的名字存入，等创建完成后就删除这个名字，这样就可以解决发现循环依赖的问题。

```text
    实例化 TCA -> 存入 List -> 发现需要注入 TCB -> 检查 List 是否有 TCB 名字 ->
    没有 -> 实例化 TCB -> 存入 List -> 发现需要注入 TCA -> 检查 List 是否有 TCA 名字 -> 
    发现名字 -> 存在循环依赖
```

这样我们就可以解决这个问题了。

```kotlin {linenos=table,hl_lines=[11,24,27,"44-53"]}
import kotlin.text.StringBuilder

class BeanContainer {
    private val container: MutableMap<String, Any> = mutableMapOf()

    // 用于存储 bean 的定义
    private val beanDefinitions: MutableMap<String, BeanDefinition> = mutableMapOf()
    // 早期未实例化的 Bean 容器
    private val earlyBeanContainer: MutableMap<String, Any> = mutableMapOf()
    // 循环依赖标记
    private val earlyMarks: MutableList<String> = mutableListOf()
    
    // ...
    
    // ...


    private fun doCreateBean(beanName: String, beanDefinition: BeanDefinition): Any {
        val beanClass = beanDefinition.type
        // 反射创建
        val instance = beanClass.getDeclaredConstructor().newInstance()
        // 注入依赖
        // 注入前添加标记
        earlyMarks.add(beanName)
        doInjectProperty(beanDefinition, instance, beanClass)
        // 注入后删除标记
        earlyMarks.remove(beanName)
        // 如果 bean 的定义是单例模式，就将创建出的 bean 加入容器
        if (beanDefinition.scope == BeanScope.SINGLETON) saveBean(beanName, instance)
        return instance
    }

    private fun doInjectProperty(beanDefinition: BeanDefinition, bean: Any, beanClass: Class<*>) {
        // 循环依赖
        beanDefinition.propertyValues.forEach {
            // 使用反射找到与依赖名称相同的方法
            val declaredField = beanClass.getDeclaredField(it.name)
            // 如果方法不能访问，就设置访问
            if (!declaredField.canAccess(bean)) {
                declaredField.trySetAccessible()
            }
            // is 关键字等同于 instanceof, 而且 is 关键字会自动将在上下文中将 value 转换为 BeanReference，不用显式转换。
            if (it.value is BeanReference) {
                // 当标记中已经存在正要注入的名字，就是循环依赖
                if (earlyMarks.contains(it.name)) {
                    val stringBuilder = StringBuilder()
                    val firstBeanName = earlyMarks[0]
                    earlyMarks.forEach{
                        stringBuilder.append(it).append(" -> ")
                    }
                    stringBuilder.append(firstBeanName)
                    throw BeanException("发生循环依赖！$stringBuilder")
                }
                // 通过 getBean 获得 Bean
                declaredField.set(bean, getBean(it.value.beanName))
            } else {
                // 注入依赖
                declaredField.set(bean, it.value)
            }
        }
    }
}
```

这样，我们在注入前放入标记，注入后删除标记，当注入过程中发现有标记，就是存在循环依赖。

出现循环依赖后的异常。

```log
    io.github.cgglyle.BeanException: 发生循环依赖！testClassC -> testClassD -> testClassC
```

还挺好看的，哈哈哈哈。但是我们都知道，发生循环依赖了应该将其处理掉，而不是抛出异常，除非是构造函数的循环依赖，那个没法处理只能抛出。

让我们将临时容器利用起来：

```kotlin {linenos=table,hl_lines=["3-7","22-25","38-49"]}
fun saveBean(beanName: String, bean: Any) {
    // 这里也是语法糖，等同 container.put(beanName, bean)
    synchronized(container) {
        this.container[beanName] = bean
        this.earlyBeanContainer.remove(beanName)
        this.earlyMarks.remove(beanName)
    }
}
    
private fun doCreateBean(beanName: String, beanDefinition: BeanDefinition): Any {
    val beanClass = beanDefinition.type
    // 反射创建
    val instance = beanClass.getDeclaredConstructor().newInstance()
    // 注入依赖
    doInjectProperty(beanName, beanDefinition, instance, beanClass)
    // 如果 bean 的定义是单例模式，就将创建出的 bean 加入容器
    if (beanDefinition.scope == BeanScope.SINGLETON) saveBean(beanName, instance)
    return instance
}
    
private fun doInjectProperty(beanName: String, beanDefinition: BeanDefinition, bean: Any, beanClass: Class<*>) {
    // 注入前添加标记
    earlyMarks.add(beanName)
    // 将 Bean 临时放入早期容器
    earlyBeanContainer[beanName] = bean
    // 循环依赖
    beanDefinition.propertyValues.forEach {
        // 使用反射找到与依赖名称相同的方法
        val declaredField = beanClass.getDeclaredField(it.name)
        // 如果方法不能访问，就设置访问
        if (!declaredField.canAccess(bean)) {
            declaredField.trySetAccessible()
        }
        // is 关键字等同于 instanceof, 而且 is 关键字会自动将在上下文中将 value 转换为 BeanReference，不用显式转换。
        if (it.value is BeanReference) {
        // 当标记中已经存在正要注入的名字，就是循环依赖
            if (earlyMarks.contains(it.name)) {
                // 从早期容器中找出早期 Bean
                val earlyBean = earlyBeanContainer[it.name]
                if (earlyBean == null) {
                    val stringBuilder = StringBuilder()
                    val firstBeanName = earlyMarks[0]
                    earlyMarks.forEach {
                        stringBuilder.append(it).append(" -> ")
                    }
                    stringBuilder.append(firstBeanName)
                    throw BeanException("发生循环依赖！$stringBuilder")
                }
                declaredField.set(bean, earlyBean)
            } else {
                // 通过 getBean 获得 Bean
                declaredField.set(bean, getBean(it.value.beanName))
            }
        } else {
        // 注入依赖
            declaredField.set(bean, it.value)
        }
    }
}
```

我们在注入参数的第一步就是添加标记，表示当前类正在构建，接着将还未完全实例化的 `Bean` 放入早期容器中，之后执行注入。

在发现需要注入的是一个 `Bean` 时，我们就需要判断需要注入的 `Bean` 是否正在构建，如果正在构建中，我们就知道现在出现了循环依赖，我们就去早期容器中拿出之前已经存入的还没有完全实例化的 `Bean`，将其注入。

当我们彻底完成一个 `Bean` 的实例化，我们就将这个 `Bean` 放入容器中，并移除构建中标记和早期容器中的实例，这样可以避免后续错误的使用早期实例。

我们创建一个复杂的循环依赖单元测试：

```kotlin
class TestClassC {
    lateinit var testClassD: TestClassD
    lateinit var testClassE: TestClassE
}

class TestClassD {
    lateinit var testClassC: TestClassC
    lateinit var testClassE: TestClassE
}

class TestClassE {
    lateinit var testClassC: TestClassC
    lateinit var testClassD: TestClassD
}

@Test
fun circularDependencyTest() {
    //given
    val beanContainer = BeanContainer()
    
    val beanDefinitionC = BeanDefinition(TestClassC::class.java)
    beanDefinitionC.setProperty("testClassD", BeanReference("testClassD"))
    beanDefinitionC.setProperty("testClassE", BeanReference("testClassE"))
    beanContainer.saveBeanDefinition("testClassC", beanDefinitionC)
    
    val beanDefinitionD = BeanDefinition(TestClassD::class.java)
    beanDefinitionD.setProperty("testClassC", BeanReference("testClassC"))
    beanDefinitionD.setProperty("testClassE", BeanReference("testClassE"))
    beanContainer.saveBeanDefinition("testClassD", beanDefinitionD)
    
    val beanDefinitionE = BeanDefinition(TestClassE::class.java)
    beanDefinitionE.setProperty("testClassC", BeanReference("testClassC"))
    beanDefinitionE.setProperty("testClassD", BeanReference("testClassD"))
    beanContainer.saveBeanDefinition("testClassE", beanDefinitionE)
    
    //when
    val beanC = beanContainer.getBean("testClassC") as TestClassC
    val beanD = beanContainer.getBean("testClassD") as TestClassD
    val beanE = beanContainer.getBean("testClassE") as TestClassE
    
    //then
    Assertions.assertNotNull(beanC)
    Assertions.assertNotNull(beanD)
    Assertions.assertNotNull(beanE)
    
    Assertions.assertEquals(beanC.testClassD, beanD)
    Assertions.assertEquals(beanC.testClassE, beanE)
    Assertions.assertEquals(beanD.testClassC, beanC)
    Assertions.assertEquals(beanD.testClassE, beanE)
    Assertions.assertEquals(beanE.testClassC, beanC)
    Assertions.assertEquals(beanE.testClassD, beanD)
}
```

完美通过！

## 小结
目前为止，我们实现了依赖注入功能。

当然，我们不是完全按照 `Spring` 的方式编写。我们也没有实现三级缓存，因为目前还用不到。我们还没有实现后置处理器等关键部分。

{{< admonition tip "源码" >}}
项目所有源码可以在 [Cgglyle's Github Kotlin-Spring](https://github.com/cgglyle/kotlin-spring)
中找到
{{< /admonition >}}