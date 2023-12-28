---
title: "Spring ProblemDetail 探究"
date: 2023-08-13T00:00:00+08:00
lastmod: 2023-12-28T00:00:00+08:00
draft: false
description: "Spring 6 新功能"
featuredImage: "featured-image.webp"
tags: ["Spring"]

lightgallery: true
---

时隔许久，我终于写了一篇新的博客（哭

这次主要是来自与我对 `Spring 6` 新出的 `ProblemDetails` 的想法与探讨。

`ProblemDetails` 是 `Spring 6` 新出的一个功能。目的是实现 `RFC 7807` 的规则， `RFC 7807`
具体是定义了一个通用的错误返回体。引用一下 `RFC 7807` 的文档说明：
> This document defines a "problem detail" as a way to carry machine readable details of errors in a HTTP response to
> avoid the need to define new error response formats for HTTP APIs.

## 传统做法

### 统一返回值

我个人认为 `Spring` 的这次跟进是一个很好的推动统一返回的行为。在以往如果需要开发一个前后段分离式的项目时，通常都会定义一个统一的返回格式用来“区分”这次请求是否成功和成功结果。例如：

```java

@Data
class ResultResponse<T extends Serializable> implements Serializable {
    private String code;
    private String message;
    private LocalDateTime timestamp;
    private T data;
}
```

这样我们就可以在控制器中编写响应的代码，让我们可以使用这个包含定义格式的封装返回值了。

```java

@RestController
@RequestMapping("/")
public class AccountController {

    @GetMapping("get")
    public ResultResponse get() {
        return new ResultResponse("200", "SUCCESS", LocalDateTime.now(), "xxx data");
    }
}
```

当请求被正确处理的情况下，我们通常会返回这样的格式：

  ```json
  {
  "code": "200",
  "message": "SUCCESS",
  "timestamp": "2023-08-13 12:00:00",
  "data": "xxx data"
}
  ```

如果不幸出现了某种异常错误，我们通常会返回：

  ```json
  {
  "code": "404",
  "message": "FAILED",
  "timestamp": "2023-08-13 12:00:00",
  "data": null
}
  ```

这样客户端就可以根据返回的 `code` 来判断这次请求是否成功，或者失败原因是什么，这通常意味着后端和前端会有一个
“密码本”，也就是一个包含所有错误代码的枚举类。这时候我们就会想，如果有一个可以帮助我们一劳永逸的处理所有返回值，不用手动进行封装就好了。

### 统一异常处理

很高兴的是 `Spring` 已经为我们想到了这个问题，`Spring`
提供了一个可以在函数处理完成后进行拦截的方法，我们可以在这里对返回值进行处理，这和我们不想每个控制器中都手动写封装不谋而合，而且这样也不够优雅：

```Java
package io.github.cgglyle.boson.system.result;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.github.cgglyle.boson.system.exception.ResultException;
import jakarta.validation.constraints.NotNull;
import lombok.RequiredArgsConstructor;
import org.springframework.core.MethodParameter;
import org.springframework.core.io.Resource;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.lang.Nullable;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

import java.io.Serializable;
import java.util.Optional;

/**
 * @author Lyle Liu
 */
@RestControllerAdvice(basePackages = {"io.github.cgglyle"})
@RequiredArgsConstructor
public class ResultResponseAdvice implements ResponseBodyAdvice<Object> {
    private final ObjectMapper objectMapper;

    // 这部分适用于判断返回值是否需要进行处理
    @Override
    public boolean supports(MethodParameter returnType,
                            @Nullable Class<? extends HttpMessageConverter<?>> converterType) {
        final boolean isSupports;
        if (returnType.getParameterType().isAssignableFrom(ResultResponse.class)) {
            isSupports = false;
        } else if (returnType.getParameterType().isAssignableFrom(Resource.class)) {
            isSupports = false;
        } else if (returnType.getParameterType().isAssignableFrom(Serializable.class)) {
            isSupports = false;
        } else {
            isSupports = true;
        }
        return isSupports;
    }

    // 这部分则是封装的主要部分
    @Override
    public Object beforeBodyWrite(Object body,
                                  @NotNull MethodParameter returnType,
                                  @Nullable MediaType selectedContentType,
                                  @Nullable Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  @Nullable ServerHttpRequest request,
                                  @Nullable ServerHttpResponse response) {
        final ResultResponse<?> resultResponse;
        Optional.ofNullable(response).ifPresent(
                r -> r.getHeaders().set("Content-Type", "application/json;charset=utf-8")
        );
        if (body == null) {
            resultResponse = ResultResponse.success();
        } else if (body instanceof Serializable data) {
            if (returnType.getParameterType().isAssignableFrom(String.class)) {
                try {
                    return objectMapper.writeValueAsString(ResultResponse.success(new Data<>(data)));
                } catch (JsonProcessingException e) {
                    throw new ResultException(e.getMessage());
                }
            } else {
                resultResponse = ResultResponse.success(new Data<>(data));
            }
        } else {
            throw new ResultException("Result object does not implement the serializable interface!");
        }
        return resultResponse;
    }
}

```

经过统一返回的处理后我们就可以得到一个可以无视封装的方式，也就是统一返回值的封装已经 `透明化`。经过这样处理的控制器可以直接写成：

```java

@RestController
@RequestMapping("/")
public class AccountController {

    @GetMapping("get")
    public String get() {
        return "xxx data";
    }
}
```

我们通过这样的方式进一步屏蔽了统一返回的复杂性，这时候有一些聪明的老哥会提出疑问：出现异常了怎么办？如果出现了异常肯定不会走这套机制的。
我们需要使用别的方式来进行统一的异常处理。没错，`spring` 也提供了 `异常控制` 机制。

```java

@Slf4j
@RequiredArgsConstructor
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * Undefined exception
     */
    @ExceptionHandler(value = Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResultResponse<String> exceptionHandler(Exception e) {
        log.error("Undefined exception: {}", e.getMessage(), e);
        return new ResultResponse("500", "System error", LocalDateTime.now(), "");
    }
}
```

这样就完成了一个 `异常控制器` 这个控制器在 `Spring` 层面直接拦截所有错误，并根据错类型进行匹配来执行响应的代码并返回错误值。
至此我们屏蔽了统一返回和统一异常处理，在控制器中或其他位置发生的异常可以直接被 `异常控制器` 拦截处理，这样我们就可以实现业务逻辑和异常分离。
更好的将关注点集中在业务逻辑中。

## 思考

很好，貌似我们解决了一切的问题。但是这和我们今天要聊得 `Problem Details` 有什么关系呢？

为了引出我们的主角，我们需要思考一下这样实现有什么问题？

有些人可能会说，阿里或者其他什么人都是这么实现的，有什么问题？
其实这会引出一个问题，也就是 `RESTful API` 是否要将所有返回状态都设置为 `200`。

如果不需要全部设成 `200`我们就不需要留下 `统一返回值` 这个 “无用” 功能了，因为我们完全可以在客户端判断返回值是否是 `200`
状态来实现判断是否为正确返回。 其余的状态值就可以根据状态值定义和具体的返回信息来判断错误类型和错误信息。

这样我们就只需要处理异常。这就有请 `Problem Details` 上场。
不过在此之前，我们先了解一下 `RFC 7807` 是如何定义 `Problem Details` 的。

{{< admonition note "RFC 7807" >}}

- type (string)
    - 一个 `URI` 提供关于这个异常的人类可读的说明文档。如果不存在可以使用 `about:blank` 替代。
- title (string)
    - 一个简短的说明，同种异常不应该因为异常的具体信息发生变化，除非是因为需要国际化。
- status (number)
    - `HTTP` 状态码
- detail (string)
    - 问题详细信息
- instance (string)
    - `URI` 引用
- 扩展信息
  {{< /admonition >}}

一个完整的 `Problem Details` 由上面的 6 部分组成。

## 我们来了解一下 `Spring` 提供的实现。

### `ErrorResponse`

这是实现了 `RFC 7807` 的一个接口，接口中提供了 7807 所有定义的功能。通过实现这个接口我们可以实现 7807 定义的功能。
接口已经提供了一个默认实现 `ErrorResponseExcpetion`，通过继承此实现我们可以快速的完成定义。

### `ProblemDetails`

这是实现了 `RFC 7807` 具体信息的封装。实际上就是将此封装中的信息映射成 `Json` 发送给客户端。
特别的是，如果你使用默认的 `Jackson` 库时， `ProblemDetails` 中的 `properties` 会被映射为顶级元素。
（说实话，我真的很不喜欢 `Fastjson` 也不推荐使用此序列化库）

{{< admonition tip "有关阿里系 “开源” " false>}}
尽量减少使用阿里系的开源框架，他们大多数的开源框架都是 **KPI** 任务。

有极大可能烂尾或出现漏洞，即便有人提交了 PR，合并都很慢。特别是 `Fastjson` 漏洞常年不修。

> 更新(2023/12/28)：最近阿里 “开猿节流，降本增笑“ 的场景让人忍俊不禁。
> {{< /admonition >}}

### `ErrorResponseException`

这是 `ErrorResponse` 的默认实现，通过此异常，可以快速实现 `RFC 7807` 中定义的返回信息。实际上所有的自定义异常都是继承这个类。

### `ResponseEntityExceptionHandler`

这是可以拦截上述 `ErrorResponseException` 的控制器。你可以将这个作为一个全局异常处理的基类。通常来说你不需要重写类中的方法。

## 实践

现在我们了解了 `Spring` 提供的类，我们现在可以基于此来完成我们想像中的方法。

你首先需要开启这个功能，你需要在配置文件中设置：

```
spring.mvc.problem-details=true
```

我们需要自定义一些异常

```java
package io.github.cgglyle.system.exception;

import jakarta.annotation.Nullable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.ErrorResponseException;
import org.springframework.web.util.DefaultUriBuilderFactory;

import java.net.URI;

/**
 * @author Lyle Liu
 */
public class SystemException extends ErrorResponseException {
    private static final String ERROR_CODE_URI = "http://localhost/system/error/";
    private static final String TITLE = "E.S.SystemException";

    public SystemException(final String errorDetail, final String errorCode, @Nullable final Throwable cause) {
        super(HttpStatus.INTERNAL_SERVER_ERROR, ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR), cause,
                null, new Object[]{errorCode});
        setTitle(TITLE);
        setDetail(errorDetail);
        setType(getTypeFromErrorCode(errorCode));
    }

    private URI getTypeFromErrorCode(final String errorCode) {
        return new DefaultUriBuilderFactory()
                .uriString(ERROR_CODE_URI)
                .pathSegment(errorCode)
                .build();
    }
}
```

我们继承 `ErrorResponseException`，并实现构造函数，父类提供了 4 个构造函数，你可以选择其中一种进行实现。
（ps：这里 Spring 实现的很奇怪，如果你需要带参数的国际化就只能实现第四个有四个参数的构造函数，
却没有给一个简便的方法。难道这个不是一个常见的功能么？）参数的详细功能会在下面国际化部分一起说明。

实际上到这里我们就已经完成了全部的异常处理问题，我们在需要的地方直接抛出这个异常就可以了。
`ResponseEntityExceptionHandler` 会捕获这个异常并封装成 `RFC 7807` 格式的消息并返回给客户端。

我们可以实验一下：

```java

@GetMapping
public void get() {
    throw new SystemException("Test error", "C001", null);
}
```

调用后的结果是

```http request
connection: close
content-type: application/problem+json
date: Sun,13 Aug 2023 12:43:29 GMT
transfer-encoding: chunked

{
    "type": "http://localhost/system/error/C001",
    "title": "Oops! System error!",
    "status": 500,
    "detail": "System error, please contact administrator: C001",
    "instance": "/"
}
```

可以看到返回体中含有 `type` `title` `status` `detail` `instance`，
你甚至什么都还没开始做呢，简直完美。不过你肯定疑惑我什么都没做我懂，但是你的 `title` `detail` 为啥直接就有详细信息。
明明代码中没有这几句话啊。实际上这是国际化的功劳。

```http request
{
    "type": "http://localhost/system/error/C001",
    "title": "啊偶，系统出错了！",
    "status": 500,
    "detail": "系统错误，请联系管理员：C001",
    "instance": "/"
}
```

你看，这是中文版本。
这归功于完备的国际化（实际上感觉实现的很奇怪）

## 国际化

```properties
problemDetail.io.github.cgglyle.system.exception.SystemException=System error, please contact administrator: {0}
problemDetail.title.io.github.cgglyle.system.exception.SystemException=Oops! System error!
```

它会在 `Spring` 配置文件中读取以 `problemDetail.` + 异常类完全限定名来读取国际化消息。
特别的是 `title` 也是可以国际化的， `problemDetail.title` + 异常类完全限定名。（实际上我就是因为这段才想写这个博客的）

我在研究国际化功能的时候发现了问题，我先入为主的以为在 `ProblemDetails` 中 `title` 填入一个唯一的表识符来识别国际化功能，
但是发现这是无效的。正如上方 `private static final String TITLE = "E.S.SystemException";` 这段代码，
我尝试在此处使用 `E.S.SystemException` 来标示系统异常国际化信息。
`Spring` 官方这部分写的也异常的模糊，反正我时没看懂。Debug 后发现是异常全限定名，我很疑惑为什么要这样设计。

这时我想到，如果我设置了国际化的 `title` 那么我在异常中写的 `private static final String TITLE = "E.S.SystemException";`
还有效么？实际上，这是无效的，可以想到，国际化的优先级是高于 `setTitle(TITLE)`，这是因为 `ProblemDetails` 是最终产物，
而你现在的行为属于直接设置最终产物的值，而国际化部分在你设置启用后会替换掉你手动设置的值。
也就是说， `ResponseEntityExceptionHandler` 并不会处理 `ProblemDetails` 中的值，它只是最终产物。

{{< mermaid >}}flowchart TD;
A[是否配置了国际化] -->|有| B(MessageDetailCode 是否为空)
A --> |没有| G
B --> |空| D(调用 Spring 的 MessageSource 处理默认的国际化配置)
B --> |不为空| E(调用 Spring 的 MessageSource 处 MessageDetailCode 的配置)
E --> F
D --> F(调用结果是否为 NULL)
F --> |为空| G(最终 Detail 是 defaultDetail)
F --> |不为空| H(最终 Detail 是国际化结果)
{{< /mermaid >}}


实际上，如果你设置了国际化，你手动设置的 `title` 和 `detail` 都是无效的。你完全可以删除这部分。

```java
public SystemException(final String errorDetail, final String errorCode, @Nullable final Throwable cause) {
    super(HttpStatus.INTERNAL_SERVER_ERROR, ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR), cause,
            null, new Object[]{errorCode});
    setTitle(TITLE); // 这里
    setDetail(errorDetail); // 这里
    setType(getTypeFromErrorCode(errorCode));
}
```

这是你可能想到，那我应该如何替换我的 `title` 呢？很高兴的告诉你，你无法替换，如果你设置了国际化。同时，你也不应该替换。
在 `RFC 7807` 中 `title` 部分定义了：

> A short, human-readable summary of the problem type. It SHOULD NOT change from occurrence to occurrence of the
> problem, except for purposes of localization

定义中很明确的表示这是一个简短的人类可读的问题描述，同时他不应该在同种异常中发生该变，除非是国际化。
这点很好理解，在一个异常中，我们有 `detail` 来描述具体的错误原因。
甚至有扩展信息可以添加无限长的错误描述。这个标题只应该用来形容这个异常，仅此而已。

但是高兴的是我们可以覆盖 `detail`，看到

```java
super(HttpStatus.INTERNAL_SERVER_ERROR,ProblemDetail.

forStatus(HttpStatus.INTERNAL_SERVER_ERROR),cause, null,new Object[]{errorCode});
```

这部分了么？这部分第一个参数是 `HTTP` 状态值，这里我们希望系统错误时会返回 500 状态。
同时第二个参数是封装的 `ProblemDetails`。我相信你肯定会疑惑为什么要设置两遍：`HttpStatus.INTERNAL_SERVER_ERROR`，
这是因为第一个参数是用于控制返回的 `HTTP` 状态值，也就是 `RESTful` 的状态值，
第二个参数的状态值显示的是返回体中 `status` 的值。

我不清楚为什么要这么设计。但是如果这两个值不同的话，`Spring` 的日志会提示你：

  ```log
  WARN 77173 --- [nio-8080-exec-1] o.s.w.s.m.m.a.HttpEntityMethodProcessor  : public final org.springframework.http.ResponseEntity<java.lang.Object>
  	org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler.handleException(java.lang.Exception,org.springframework.web.context.request.WebRequest)
      throws java.lang.Exception returned ResponseEntity: <404 NOT_FOUND Not Found,ProblemDetail[type='http://localhost/system/error/C001', title='Oops! System error!', status=500, detail='This is TEST: C001', instance='/', properties='null'],[]>,
      but its status doesn't match the ProblemDetail status: 500
  ```

第三个参数是具体的错误原因，这部分并不会返回给客户端，只是用来打印错误日志。第四个参数就是错误描述，这个参数可以覆盖掉国际化部分。第五个参数是可替换的错误标示。（我不知道这里应该怎么说，你大概理解意思就好（笑））

当然，如果你选择覆盖 `detail` 的话，你也可以使用国际化。也就是说，你覆盖的那部分也可以使用国际化标示。（所以这部分就很奇怪）

经过上方的一顿输出，你可能看出来：“那你这个异常类写的不怎么样啊！” 很显然是这个样子的。这只是用来测试的产物。实际上，（天啊，我怎么这么愿意说实际上）最终的异常应该是这个样子的：

```java
public class SystemException extends ErrorResponseException {
    private static final String ERROR_CODE_URI = "http://localhost/system/error/";

    public SystemException(final String errorDetail, final String errorCode, @Nullable final Throwable cause) {
        super(HttpStatus.INTERNAL_SERVER_ERROR, ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR), cause,
                errorDetail, new Object[]{errorCode});
        setType(getTypeFromErrorCode(errorCode));
    }

    public SystemException() {
        this(null, null, null);
    }

    private URI getTypeFromErrorCode(final String errorCode) {
        return new DefaultUriBuilderFactory()
                .uriString(ERROR_CODE_URI)
                .pathSegment(errorCode)
                .build();
    }
}

```

看起来好了一些是吧，实际上也没有好多少。你可能注意到了 `ERROR_CODE_URI`这是一个用户友好的信息页面，通常来说你可以在这里展示一些问题的原因，如何处理，怎么报告错误之类的。这里肯定是没有了。

-----

## 后话

现在你已经完全了解了 `ProblemDetails`的工作原理（并不），你可能会想到，那 `RuntimeException`可以抛出这样的格式么？
可以，但是不完全可以。如果你完全不做处理的话，抛出的还是老样子，包含错误堆栈的错误信息。
不过你可以使用继承 `ResponseEntityExceptionHandler`来实现重写 `RuntimeException`。

```java
@Slf4j
@RestControllerAdvice

public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    @ResponseStatus(HttpStatus.INSUFFICIENT_STORAGE)
    @ExceptionHandler(Exception.class)
    ProblemDetail exceptionHandler(Exception e, WebRequest request) {
        log.error("Exception", e);
        return createProblemDetail(e,
                HttpStatus.INTERNAL_SERVER_ERROR,
                "System error!",
                null,
                null,
                request);
    }
}
```

实际上这和我们最开始使用的方式没什么区别。至此，我们已经学会使用 `Spring 6 ` 提供的 `ProblemDetail` 方法来实现 `RFC 7807`。

-----

## References

1. https://datatracker.ietf.org/doc/html/rfc7807
2. https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html
3. https://www.kapresoft.com/java/2023/05/15/spring-error-handling-best-practices.html