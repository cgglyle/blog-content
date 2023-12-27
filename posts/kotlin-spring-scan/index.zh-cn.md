---
title: "如何用 Kotlin 实现一个 Spring - 扫描"
date: 2023-12-16T00:00:00+08:00
lastmod: 2023-12-26T00:00:00+08:00
draft: false
description: "用自己的方式实现一个只属于你的 Spring。"
featuredImage: "featured-image.webp"

tags: ["Spring", "Kotlin", "技术"]
categories: ["Spring"]
series: ["用自己的方式实现一个只属于你的 Spring"]
series_weight: 3
lightgallery: true
---

在上两篇文章中我们实现了基础的 `IoC` 容器和依赖注入功能。我们知道早期 `Spring` 是使用 `XML` 来作为配置文件的， `Spring` 通过读取 `XML` 文件来配置系统。但是考虑到 `XML` 配置的繁琐和易错，现在 `Spring` 使用注解的形式来处理配置。接下来我们就实现这个想法。

<!--more-->

## 路径扫描

我们首先想一下，我们如何才能根据一个路径扫描出路径下的所有类。

{{< admonition question "假设我们想扫描 io.github.cgglyle 下的所有类，我们该如何做呢？" >}}
你：直接使用 `Class.forName` 直接找出来。  
{{< /admonition >}}

想法很好，但是不可行。`Class.forName` 一定要完全限定名！否则是加载不出来的。

现在需要考虑的是如何获取全限定名，怎么样做呢？

好在 `Java` 为我们提供了一个方便的功能： `ClassLoader` 类加载器，使用类加载器我们可以通过相对路径获取类的绝对路径。

类似 `io/github/cgglyle` -> `/home/spring/io/github/cgglyle`

其中 `/home/spring` 就是我们启动类时的路径，这部分将会根据启动时传入的参数发生变化。而后方的相对路径则是不会变动的。通过这种方式我们可以获取的真实的类路径。

```kotlin
this.javaClass.classLoader.getResource( path )
```

正如上方的代码，我们可以得到对应的 `path` 的绝对路径，这个方法的返回值为 `URL`，通过这个 `URL` 我们可以直接使用 `File` 将文件加载进内存。

当拿到这个文件后，我们就可以判断是否为目录，如果不是目录就直接获取类的 `path` 如果不是我们就继续递归加载。直到拿到所有的类。

这里我们需要了解从 `File` 中拿到的类是绝对路径，且带有 `.class` 后缀，因为我们实际加载的是 `class` 文件，而不是源码。

我们将绝对路径进行转换，变成完全限定名，这样我们就拿到了完全限定名

例如：`/home/spring/io/github/cgglyle/TestClassA.class` -> `io.gihub.cgglyle.TestClassA`

需要注意的是，我们如何获得 `/home/spring/` 这个地址，因为我们没法确定传入的类路径到底是什么，我们这里也不能写死。我们需要编写一个动态的方式获取这个地址。好在我们可以邪道获取这个地址。

我们可以通过获取 `“”` 这个空地址来得到类路径，也就是：

```kotlin
this.javaClass.classLoader.getResource("")
// get /home/spring/
```

然后我们直接截取就好，接下来我们看一下，如何实现这个功能。

### 实现

```kotlin
import java.io.File

class ComponentScanProcessor {
    fun scan(path: String): List<Class<*>> {
        // io.github.cgglyle -> io/github/cgglyle
        val filePath = path.replace(".", "/")
        // 从 ClassLoader 中取出资源，因为在系统启动时会配置 ClassPath，这样我们就可以取出绝对路径
        val resource = this.javaClass.classLoader.getResource(filePath)
        resource ?: throw ScanException("扫描 '$path' 不存在！")

        val file = File(resource.file)
        val scanFilePath = scanFilePath(file, mutableListOf())
        val absoluteFilePath =
            this.javaClass.classLoader.getResource("") ?: throw ScanException("扫描 ClassPath 绝对路径失败")
        val classList = scanFilePath.map {
            val classPath = it.substring(absoluteFilePath.path.length)
                .replace("/", ".")
                .substringBeforeLast(".class")
            Class.forName(classPath)
        }
        return classList
    }

    private fun scanFilePath(file: File, pathList: MutableList<String>): List<String> {
        // 如果是目录就继续处理，如果是文件就直接加入列表
        if (file.isDirectory) {
            val listFiles = file.listFiles()
            // 语法糖，如果 listFiles 不为 null 执行 let 后面的语句，否则跳过
            listFiles?.let {
                for (f in listFiles) {
                    if (f.isFile) {
                        pathList.add(f.path)
                    } else if (f.isDirectory) {
                        // 如果是目录就递归扫描
                        scanFilePath(f, pathList)
                    }
                }
            }
        } else if (file.isFile) {
            pathList.add(file.path)
        }
        return pathList.toList()
    }
}
```

我们编写一个单元测试去测试。

因为要扫描路径，所以我先编写了一个测试路径：

```shell
.
└── io
    └── github
        └── cgglyle
            ├── bean
            │   ├── BeanContainerTest.kt
            │   ├── TestClassA.kt
            │   ├── TestClassB.kt
            │   ├── TestClassC.kt
            │   ├── TestClassD.kt
            │   └── TestClassE.kt
            └── scan
                ├── ComponentScanProcessorTest.kt
                └── test
                    ├── dir
                    │   ├── TestC.kt
                    │   └── TestD.kt
                    ├── TestA.kt
                    └── TestB.kt

8 directories, 11 files
```

请忽略 `bean` 路径，那是上两篇的单元测试

```kotlin
import io.github.cgglyle.ComponentScanProcessor
import io.github.cgglyle.scan.test.TestA
import io.github.cgglyle.scan.test.TestB
import io.github.cgglyle.scan.test.dir.TestC
import io.github.cgglyle.scan.test.dir.TestD
import org.junit.jupiter.api.Assertions
import org.junit.jupiter.api.Test

class ComponentScanProcessorTest {

    @Test
    fun testClassPathScan() {
        //given
        val componentScanProcessor = ComponentScanProcessor()

        //when
        val scan = componentScanProcessor.scan("io.github.cgglyle.scan.test")

        //then
        Assertions.assertEquals(scan.size, 4)
        Assertions.assertTrue(scan.any { it == TestA::class.java })
        Assertions.assertTrue(scan.any { it == TestB::class.java })
        Assertions.assertTrue(scan.any { it == TestC::class.java })
        Assertions.assertTrue(scan.any { it == TestD::class.java })
    }
}
```

最终是测试通过的，而且我们也可以正确的处理多级嵌套扫描。

## 判断 Bean

我们虽然现在扫描出了所有类，但是我们不能全部都设为 `Bean`。我们需要根据某种方式判断这个类是否为 `Bean`

我们照抄 `Spring` 的方式，使用注解 `@Component` 来作为条件。只有标注的 `@Component` 注解的类才是需要管理的 `Bean`。

这个功能只需要判断获取到的 `Class` 上面有没有相应的注解就好。

### 实现

```kotlin
@Retention(AnnotationRetention.RUNTIME)
// 这里和 java 不同，使用这种方式限制只能标记在类上
@Target(AnnotationTarget.CLASS)
annotation class Component()

class ComponentScanProcessor {
    fun scan(path: String): List<Class<*>> {
    // ...
        val classList = scanFilePath.map {
            val classPath = it.substring(absoluteFilePath.path.length)
                .replace("/", ".")
                .substringBeforeLast(".class")
            Class.forName(classPath)
        }.filter { isBean(it) } // 此处添加一个过滤
        return classList
    }

    private fun isBean(clazz: Class<*>): Boolean {
        val componentAnnotation = clazz.getDeclaredAnnotation(Component::class.java)
        return componentAnnotation != null
    }
}
```

我们更改以下测试类：

我们在 `TestA` 和 `TestC` 上标记 `@Component` 其他两个不标记

```kotlin
@Test
fun testClassPathScan() {
    //given
    val componentScanProcessor = ComponentScanProcessor()

    //when
    val scan = componentScanProcessor.scan("io.github.cgglyle.scan.test")

    //then
    Assertions.assertEquals(scan.size, 2)
    Assertions.assertTrue(scan.any { it == TestA::class.java })
    Assertions.assertTrue(scan.any { it == TestC::class.java })
}
```

成功通过。

## 解析 BeanDefinition

还记得我们之前编写的 `BeanDefinition` 吗？它才是我们创建 `Bean` 的定义，我们需要在扫描到需要管理的 `Bean` 的时候构建出它。

当我们成功获取到需要管理的 `Bean` 后，我们解析 `Class` 中的数据，并且结合 `@Component` 中的配置进行定义。

### 实现

```kotlin
class ComponentScanProcessor {
    fun scan(path: String): List<BeanDefinition> {
        val classList = scanClassPath(path)
        return classList.map { doBeanDefinitionParse(it) }
    }

    private fun scanClassPath(path: String): List<Class<*>> {
        // io.github.cgglyle -> io/github/cgglyle
        val filePath = path.replace(".", "/")
        // 从 ClassLoader 中取出资源，因为在系统启动时会配置 ClassPath，这样我们就可以取出绝对路径
        val resource = this.javaClass.classLoader.getResource(filePath)
        resource ?: throw ScanException("扫描 '$path' 不存在！")

        val file = File(resource.file)
        val scanFilePath = scanFilePath(file, mutableListOf())
        // 通过获取 “” 的资源可以获得类路径
        val absoluteFilePath =
            this.javaClass.classLoader.getResource("") ?: throw ScanException("扫描 ClassPath 绝对路径失败")
        val classList = scanFilePath.map {
            val classPath = it.substring(absoluteFilePath.path.length)
                .replace("/", ".")
                .substringBeforeLast(".class")
            Class.forName(classPath)
        }.filter { isBean(it) }
        return classList
    }
    
    // ...

    private fun doBeanDefinitionParse(clazz: Class<*>): BeanDefinition {
        return BeanDefinition(clazz)
    }
}
```

我们将扫描类路径的代码单独提取出来 `scanClassPath`，我们在使用 `doBeanDefinitionParse` 来处理解析

这样我们就完成了。我们来试着将扫描器和容器组合起来实验一下效果。编写一个单元测试。

诶诶诶，先等等，你是不是忘了点什么？你好像忘记 `BeanName` 怎么处理了吧？我们解析的时候并没有生成相应的名字。回忆一下，我们向容器中存入 `BeanDefinition` 的时候需要名字。（我才不会说我写单元测试的时候发现没名字的呢！这也许就是没有实施 `TDD` 的问题吧，如果先编写单元测试在写代码就会发现这个问题。）

## Bean 名字生成

我们采用和 `Spring` 相同的做法，有下面几种情况：

`TestA` -> `testA`

`URLTestA` -> `URLTestA`

目前我们就考虑这两种，不考虑什么代理类内部类之类的。我们发现 `Spring` 在处理像 `URLTestA` 的时候会保持原样。这是因为开头前两个字母都是大写的情况说明这个单词很可能是一个专有名词，例如： `URL` `URI` `DNS` `HTTP`，专有名词还是大写为妙，方便辨识。

### 实现

我们使用和 `Spring` 一样的方法处理：

```kotlin
class AnnotationBeanNameGenerator {
    fun generatorBeanName(beanDefinition: BeanDefinition): String {
        val type = beanDefinition.type
        val simpleName = type.simpleName
        return uncapitalizeAsProperty(simpleName)
    }

    fun uncapitalizeAsProperty(string: String): String {
        // 如果不为空，且第一个首字母和第二个字母为大写就直接返回。例如：DNS
        if (string.isNotBlank() && string.first().isUpperCase() && string.toCharArray()[1].isUpperCase()) {
            return string
        }
        return changeFirstCharacterCase(string, false)
    }

    /**
    * 通过控制 [capitalize] 来决定 [string] 的首字母大写或者小写
    */
    fun changeFirstCharacterCase(string: String, capitalize: Boolean): String{
        if (string.isBlank()) {
            return string
        }
        val baseChar = string.toCharArray()[0]
        val updateChar = if (capitalize) {
            baseChar.uppercaseChar()
        } else {
            baseChar.lowercaseChar()
        }
        if (baseChar == updateChar) {
            return string
        }
        val chars = string.toCharArray()
        chars[0] = updateChar
        return String(chars)
    }
}
```

单元测试：

```kotlin
class AnnotationBeanNameGeneratorTest {
    private lateinit var beanNameGenerator: AnnotationBeanNameGenerator

    @BeforeEach
    fun setUp() {
        beanNameGenerator = AnnotationBeanNameGenerator()
    }

    @Test
    fun generatorBeanName() {
        // given
        val beanDefinition = BeanDefinition(TestA::class.java)
        val DNSBeanDefinition = BeanDefinition(DNSTestA::class.java)

        // when
        val beanName = beanNameGenerator.generatorBeanName(beanDefinition)
        val dnsBeanName = beanNameGenerator.generatorBeanName(DNSBeanDefinition)

        // then
        assertEquals("testA", beanName)
        assertEquals("DNSTestA", dnsBeanName)
    }
}
```

完美通过！

## 依赖注入

在上面我们已经可以通过注解成功注册 `Bean`，但是我们还没有实现依赖注入。在 `Spring` 中最常用的是 `@Autowired` 注入和 `构造函数` 这两种方式，虽然现在 `Spring` 已经不推荐 `@Autowired` 注入方式，但是它比较好实现，所以我们现在优先实现 `@Autowired` 注入方式。

当然最重要的就是编写 `@Autowired` 注解。

```kotlin
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.FIELD)
annotation class Autowired()
```

我们该如何注入呢？考虑到我们已经实现了 `BeanDefinition` 和 `PropertyValue` 以及 `BeanReference` 这三个类。我们只需要在扫描的过程中判断当前类中的成员是否有 `@Autowired` 注解，并提取它的名字即可。（因为 `BeanReference` 只需要 `Bean` 的名字来寻找 `Bean`）

```kotlin
class ComponentScanProcessor {
    // ...

    private fun doBeanDefinitionParse(clazz: Class<*>): BeanDefinition {
        val beanDefinition = BeanDefinition(clazz)
        // 获取所有成员
        clazz.declaredFields.forEach {
        // 判断成员是否有 @Autowired 注解
            if (it.getDeclaredAnnotation(Autowired::class.java) != null) {
                beanDefinition.setProperty(it.name, BeanReference(it.name))
            }
        }
        return beanDefinition
    }
}
```

当然这里我们没有考虑 `Set 注入` 等方案。我们先编写个单元测试：

```kotlin
@Component
class TestC {
    @Autowired
    lateinit var testA: TestA
    lateinit var testC: TestC

    fun testCIsInitialized(): Boolean {
        // 判断 testC 是否已经初始化
        // 因为没有自动注入，我们希望它是没有初始化的
        return this::testC.isInitialized
    }
}	

@Component
class TestA {
    // 我们还希望可以自动循环注入自己
    @Autowired
    lateinit var testA: TestA
}
```

我们在 `TestC` 类中注入 `TestA` 变量

```kotlin
@Test
fun testAutowired() {
    //given
    val componentScanProcessor = ComponentScanProcessor()
    val beanDefinitions = componentScanProcessor.scan("io.github.cgglyle.scan.test")
    val beanContainer = BeanContainer()
    val annotationBeanNameGenerator = AnnotationBeanNameGenerator()
    val pairs = beanDefinitions.map { Pair(annotationBeanNameGenerator.generatorBeanName(it), it) }
    pairs.forEach { (beanName, beanDefinition) ->
        beanContainer.saveBeanDefinition(beanName, beanDefinition)
    }

    // when
    val beanA = beanContainer.getBean("testA") as TestA
    val beanC = beanContainer.getBean("testC") as TestC

    // then
    Assertions.assertNotNull(beanA)
    Assertions.assertNotNull(beanC)
    Assertions.assertEquals(beanA, beanC.testA)
    Assertions.assertEquals(false, beanC.testCIsInitialized())
    Assertions.assertEquals(beanA, beanA.testA)
}
```

完美通过！

## 作用域配置

我们还没有实现作用域，我们需要现在 `@Component` 中添加一个作用域配置成员。

```kotlin
@Retention(AnnotationRetention.RUNTIME)
// 这里和 java 不同，使用这种方式限制只能标记在类上
@Target(AnnotationTarget.CLASS)
annotation class Component(
    // 添加作用域配置
    val scope: BeanScope = BeanScope.SINGLETON
)
```

在 `doBeanDefinitionParse` 中解析：

```kotlin {linenos=table,hl_lines=["2-8"]}
private fun doBeanDefinitionParse(clazz: Class<*>): BeanDefinition {
    val componentAnnotation = clazz.getDeclaredAnnotation(Component::class.java)
    // if 可以将块中的值返回，实际上 if 本身也是一个表达式，本身和 java 中的三元表达式相同。
    val beanDefinition = if (componentAnnotation != null) {
        BeanDefinition(clazz, componentAnnotation.scope)
    } else {
        BeanDefinition(clazz)
    }

    clazz.declaredFields.forEach {
        if (it.getDeclaredAnnotation(Autowired::class.java) != null) {
            beanDefinition.setProperty(it.name, BeanReference(it.name))
        }
    }
    return beanDefinition
}
```

编写一个单元测试：

```kotlin
private lateinit var beanContainer: BeanContainer
@BeforeEach
fun setUp() {
    //given
    val componentScanProcessor = ComponentScanProcessor()
    val beanDefinitions = componentScanProcessor.scan("io.github.cgglyle.scan.test")
    val beanContainer = BeanContainer()
    val annotationBeanNameGenerator = AnnotationBeanNameGenerator()
    val pairs = beanDefinitions.map { Pair(annotationBeanNameGenerator.generatorBeanName(it), it) }
    pairs.forEach { (beanName, beanDefinition) ->
        beanContainer.saveBeanDefinition(beanName, beanDefinition)
    }
    this.beanContainer = beanContainer
}
    
@Test
fun testScope() {
    // when
    val prototypeTest = beanContainer.getBean("prototypeTest")
    val prototypeTestB = beanContainer.getBean("prototypeTest")
    
    // then
    Assertions.assertNotEquals(prototypeTest, prototypeTestB)
}
    
@Component(BeanScope.PROTOTYPE)
class PrototypeTest {
}
```

我将构建扫描和注册提取出来，避免重复编写。当然了，测试也是完美通过。

## 小结

我们目前为止已经完成了扫描类，并且可以使用扫描的方式完成 `Bean` 的注册。

{{< admonition tip "源码" >}}
项目所有源码可以在 [Cgglyle's Github Kotlin-Spring](https://github.com/cgglyle/kotlin-spring)
中找到
{{< /admonition >}}