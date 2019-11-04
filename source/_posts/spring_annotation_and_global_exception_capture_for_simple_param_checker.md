---
title: 使用Spring注解和全局异常捕获对简单参数校验的代码优化
date: 2017-08-06
---
## 前言

在对前端传来的参数进行校验是后端程序开发中不可或缺的步骤，但是大量参数校验的代码混杂在业务逻辑代码中实在是令人无奈何，无形之中使得简单的代码逻辑趋于复制。本文就以int类型为例，使用Spring注解和全局异常捕获对简单参数校验的代码简化技巧进行介绍。
<!--more-->
### 示例场景
这里定义了一个最基本的Controller，代码如下：
``` java
@RestController
public class MainController {
	@RequestMapping("/test")
	public String test(int id) {
		return String.format("id: %s", id);
	}
}
```
通常情况下Spring可以自动对参数进行注入，并转换位相应的类型。这里的id参数会被自动转换为int类型。
```
url| http://localhost:8080/test?id=1
res| id: 1
```

但是如果前端以下面这种URL进行请求，代码就会抛出异常
```
url| http://localhost:8080/test?id=
res| Failed to convert value of type 'java.lang.String' to required type 'int'; nested exception is java.lang.NumberFormatException: For input string: ""
```
原因是空字符串并不能转换为有效的数字。同理不带id参数的URL也会抛出异常，毕竟null是无法转换成有效的数字的。

为了解决这个问题，我们可以用Spring的注解来解决，代码如下：
``` java
@RestController
public class MainController {
	@RequestMapping("/test")
	public String test(@RequestParam(defaultValue = "1000", name = "id") int id) {
		return String.format("id: %s", id);
	}
}
```

我们在id参数前面加了@RequestParam(defaultValue = "1000", name = "id")这段注解，它的主要作用是在id参数为空字符或null的时候使用自定义的默认值（defaultValue）。效果如下：
```
url| http://localhost:8080/test?id=
res| id: 1000
```
看起来已经比原来好很多的，但是还是有不足之处，如果是这样的请求：
```
url| http://localhost:8080/test?id=abc
res| Failed to convert value of type 'java.lang.String' to required type 'int'; nested exception is java.lang.NumberFormatException: For input string: "abc"
```

显然字符串"abc"无法被转换成数字，而且在spring的文档中也没找到可以用参数注解来解决这个问题的。不过，网上有一种通过配置全局捕获异常，从而可以自定义返回信息，实现如下：
``` java
@ControllerAdvice
@ResponseBody
public class ControllerExceptionHandler {
	// 添加全局异常处理流程，根据需要设置需要处理的异常，本文以MethodArgumentTypeMismatchException为例
	@ExceptionHandler({ MethodArgumentTypeMismatchException.class })
	public String MethodArgumentNotValidHandler(Exception ex, HttpServletRequest req) {
		String errorMsg = ex.getMessage();
		String[] strArray = errorMsg.split("\""); // 获取""中的字符
		if (strArray.length > 1) {
		    // 获取到出错的参数值
			String errorValue = strArray[1]; 
			// 获取到出错的参数名称
			Optional<String> keyOp = req.getParameterMap()
							.entrySet().stream()
							.filter(e -> e.getValue()[0].equals(errorValue))
							.map(e -> e.getKey())
							.findAny();
			if (keyOp.isPresent()) {
				return String.format("参数错误，请检查以下参数：%s=%s", keyOp.get(), errorValue);
			}
		}
		return errorMsg;
	}
}
```

测试一下
```
url| http://localhost:8080/test?id=abc
res| 参数错误，请检查以下参数：id=abc
```

如果对于一些必传参数，注解设置默认值的做法还是略显多余，毕竟这样还是要手动对参数进行校验，不妨去掉@RequestParam注解，在全局异常处理类中添加所需的异常类型，然后问题就解决了。