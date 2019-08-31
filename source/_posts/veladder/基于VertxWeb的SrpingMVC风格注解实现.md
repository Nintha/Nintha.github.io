---
title: 基于VertxWeb的SrpingMVC风格注解实现
date: 2019-08-29
---

## 前言

最近了解到了vertx这个异步框架，但平时用的比较多的还是spring，出于好奇，尝试基于vertx web去实现spring mvc风格注解。

最终效果如下所示

```java
@Slf4j
@RestController
public class HelloController {

    @RequestMapping("hello/world")
    public String helloWorld() {
        return "Hello world and nintha veladder";
    }

    @RequestMapping("echo")
    public Map<String, Object> echo(String message, Long token, int code, RoutingContext ctx) {
        log.info("uri={}", ctx.request().absoluteURI());

        log.info("message={}, token={}, code={}", message, token, code);
        HashMap<String, Object> map = new HashMap<>();
        map.put("message", message);
        map.put("token", token);
        map.put("code", code);
        return map;
    }

    @RequestMapping(value = "hello/array")
    public List<Map.Entry<String, String>> helloArray(long[] ids, String[] names, RoutingContext ctx) {
        log.info("ids={}", Arrays.toString(ids));
        log.info("names={}", Arrays.toString(names));
        return ctx.request().params().entries();
    }

    @RequestMapping("query/list")
    public List<Map.Entry<String, String>> queryArray(List<Long> ids, TreeSet<String> names, LinkedList rawList, RoutingContext ctx) {
        log.info("ids={}", ids);
        log.info("names={}", names);
        log.info("rawList={}", rawList);
        return ctx.request().params().entries();
    }

    @RequestMapping("query/bean")
    public BeanReq queryBean(BeanReq req) {
        log.info("req={}", req);
        return req;
    }

    @RequestMapping(value = "post/body", method = HttpMethod.POST)
    public BeanReq postRequestBody(@RequestBody BeanReq req) {
        log.info("req={}", req);
        return req;
    }
}
```

完整实现见[github repo: veladder](https://github.com/nintha/veladder)

<!--more-->

## @RestContoller

现在主流spring mvc的控制器一般是使用`@RestController`注解，它的主要作用是告诉框架这个类是请求控制器，让框架主动扫描并加载这个类。

这个注解本质是个标记注解，所以目前不需要其他字段。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface RestController {
}
```

由于目前没有实现任何DI（依赖注入），为了方便我们直接在代码里面手动创建实例并加载。

```java
@Override
public void start() throws Exception {
    HttpServer server = vertx.createHttpServer();

    Router router = Router.router(vertx);
    routerMapping(new HelloController(), router);

    server.requestHandler(router).listen(port, ar -> {
        if (ar.succeeded()) {
            log.info("HTTP Server is listening on {}", port);
        } else {
            log.error("Failed to run HTTP Server", ar.cause());
        }
    });
}
```

上面这段代码是在Verticle里面启动一个HTTP服务器，并进行路由构建。里面的`routerMapping`方法我们下面实现一下

```java
private <ControllerType> void routerMapping(
    ControllerType annotatedBean, Router router) throws NotFoundException {
    
    Class<ControllerType> clazz = (Class<ControllerType>) annotatedBean.getClass();
    if (!clazz.isAnnotationPresent(RestController.class)) {
        return;
    }
    
    // other code
}
```

先简单判断下这个加载的类是否为带`@RestController`注解。好了，`@RestContoller`注解的任务已经完成了。



## @RequestMapping

`@RequestMapping`注解的作用是把请求路径和我们的处理逻辑关联在一起。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface RequestMapping {
    String value() default "";

    HttpMethod[] method() default {};
}

```

value对应的path，而method对应http的请求方法，如GET、POST等。使用效果如下所示：

```java
@RequestMapping(value = "post/body", method = HttpMethod.POST)
```

我们继续上一节的路由解析。

首先我们要获取到控制器类中被该注解标记的方法

```java
Method[] methods = clazz.getDeclaredMethods();
for (Method method : methods) {
    if (!method.isAnnotationPresent(RequestMapping.class)) continue;
    
    // ... handle method code
}
```

这里使用了反射，并且只处理了带`@RequestMapping`注解的方法，然后从注解中获取请求路径和请求方法，具体代码如下所示

```java
// ... handle method code
RequestMapping methodAnno = method.getAnnotation(RequestMapping.class);
String requestPath = methodAnno.value();

Handler<RoutingContext> requestHandler = ctx -> {
    // ... call annotated method and inject parameters
};

// bind handler to router
HttpMethod[] httpMethods = methodAnno.method();
if (httpMethods.length == 0) {
    // 默认绑定全部HttpMethod
    router.route(formatPath).handler(BodyHandler.create()).handler(requestHandler);
} else {
    for (HttpMethod httpMethod : httpMethods) {
        router
            .route(httpMethod, formatPath)
            .handler(BodyHandler.create())
            .handler(requestHandler);
    }
}
```

httpMethods这个参数我们采用了数组类型，这样一个处理函数可以绑定到多个HttpMethod中。这里还做了一个特殊处理，当用户省略这个参数时，将绑定全部HttpMethod，毕竟不绑定HttpMethod的处理函数没有意义。

上面代码中的`requestHandler`的功能是调用被注解的处理函数并注入所需要的参数，并将函数返回值转换为合适的格式再返回给请求者。个人感觉强大的参数注入功能可以极大的提升开发者的编码体验。接下来我们实现下`requestHandler`。

### 返回值处理

返回值一般有多种情况：

1. void类型，这种是无返回值，一般是特殊情况会使用。比如下载文件，无法以JSON形式返回，需要调用RoutingContext进行处理。
2. 基础类型和字符串，直接转换为JSON中的数字和字符串就好。
3. 其他类型，当POJO处理，直接序列化成JSON

处理返回值代码如下所示：

```java
// Write to the response and end it
Consumer<Object> responseEnd = x -> {
    if (method.getReturnType() == void.class) return;

    HttpServerResponse response = ctx.response();
    response.putHeader(HttpHeaders.CONTENT_TYPE, "application/json;charset=UTF-8");
    response.end(x instanceof CharSequence ? x.toString() : Json.encode(x));
};
```

Void类型就直接return，不做处理。

再判断是否为字符串类型，字符串类型可以直接作为返回内容，其余类型走JSON序列化。

### 参数处理

vertx请求参数的内容都可以通过RoutingContext对象获取到，无论是在URI里的query还是body里面的formdata或者JSON形式数据，都是可以的。

```java
MultiMap params = ctx.request().params();
```

一个典型的请求处理函数如下所示：

```java
@RequestMapping("echo")
public Map<String, Object> echo(String message, Long token, int code, RoutingContext ctx) {
    log.info("uri={}", ctx.request().absoluteURI());

    log.info("message={}, token={}, code={}", message, token, code);
    HashMap<String, Object> map = new HashMap<>();
    map.put("message", message);
    map.put("token", token);
    map.put("code", code);
    return map;
}
```

echo 函数有4个参数，我们需要对它进行反射获取4个参数的类型和名字。获取类型可以方便我们对数据进行类型转换，对于一些特殊的类型还有进行专门的处理。参数类型比较好获取，反射API可以直接获取到

```java
Method[] methods = clazz.getDeclaredMethods();
for (Method method : methods) {
	Class<?>[] paramTypes = method.getParameterTypes();
}
```

参数名字就相对复杂一点，虽然JDK8里面提供了对方法形参名反射的API，但它存在限制

```java
Parameter[] parameters = method.getParameters();
for (Parameter p: Parameters){
    String name = p.getName;
    // ...
}
```

如果在编译代码的时候没有加上`–parameters`参数，那么`Parameter#getName`拿到的是`arg0`这样的占位符，毕竟java还是要向前兼容的嘛。

正常途径还真不好拿到形参名，所以像MyBatis这类的框架是让用户通过注解进行形参名标注

```java
public interface DemoMapper {
    List<Card> getCardList(@Param("cardIds") List<Integer> cardIds);
    Card getCard(@Param("cardId") int cardId);
}
```

`@Param`允许用户使用和形参名不一样的值，类似别名，功能上更加灵活，但是大部分情况下，我们就希望直接使用形参名，重复的内容写两遍还是比较不友好的。

但是spring就支持直接获取方法形参名。在强大的搜索引擎帮助下，可以发现spring里面使用了字节码方式去获取方法形参（具体内容请自行google），这里我们使用javassist来实现这个功能，这样就不需要自己去处理字节码了。

```java
// javassist获取反射类
ClassPool classPool = ClassPool.getDefault();
classPool.insertClassPath(new ClassClassPath(clazz));
CtClass cc = classPool.get(clazz.getName());

// 反射获取方法实体
CtMethod ctMethod = cc.getDeclaredMethod(method.getName());
MethodInfo methodInfo = ctMethod.getMethodInfo();
CodeAttribute codeAttribute = methodInfo.getCodeAttribute();
// 获取本地变量表，里面有形参信息
LocalVariableAttribute attribute = 
    (LocalVariableAttribute) codeAttribute.getAttribute(LocalVariableAttribute.tag);

Class<?>[] paramTypes = method.getParameterTypes();
String[] paramNames = new String[ctMethod.getParameterTypes().length];
if (attribute != null) {
    // 通过javassist获取方法形参，成员方法 0位变量是this
    int pos = Modifier.isStatic(ctMethod.getModifiers()) ? 0 : 1;
    for (int i = 0; i < paramNames.length; i++) {
        paramNames[i] = attribute.variableName(i + pos);
    }
}
```

`javassist`可以获取到方法的`LocalVariableAttribute`，这里面有我们需要的形参信息，包括形参名。

获取到对应的形参名后，我们可以靠这个获取请求的参数：

```java
// 单个值
String value = ctx.request().params().get(paramName);
// 数组或集合
List<String> values = ctx.request().params().getAll(paramName);
```

接着我们来注入参数值，需要处理的参数类型可以分为下列几种:

1. 简单类型，基础类型和它们的包装类，再加上字符串
2. 集合类型，`java.util.Collection<T>`及其子类，泛型参数T支持简单类型
3. 数组类型，数组的组件类型(`ComponentType`)支持简单类型
4. 数据类型，类似POJO的数据载体类，里面可以再嵌套数据类型或其他类型
5. 特殊类型，比如FileUpload和RoutingContext，需要特殊处理

   

简单类型处理只需要把基础类型都转换为包装类型，再反射调用`valueOf`方法就可以了

```java
 private <T> T parseSimpleType(String value, Class<T> targetClass) throws Throwable {
     if (StringUtils.isBlank(value)) return null;

     Class<?> wrapType = Primitives.wrap(targetClass);
     if (Primitives.allWrapperTypes().contains(wrapType)) {
         MethodHandle valueOf = MethodHandles.lookup().unreflect(wrapType.getMethod("valueOf", String.class));
         return (T) valueOf.invoke(value);
     } else if (targetClass == String.class) {
         return (T) value;
     }

     return null;
 }
```

集合类型，反射获取到泛型类型后按简单类型处理

```java
private Collection parseCollectionType(List<String> values, Type genericParameterType) throws Throwable {
    Class<?> actualTypeArgument = String.class; // 无泛型参数默认用String类型
    Class<?> rawType;
    // 参数带泛型
    if (genericParameterType instanceof ParameterizedType) {
        ParameterizedType parameterType = (ParameterizedType) genericParameterType;
        actualTypeArgument = (Class<?>) parameterType.getActualTypeArguments()[0];
        rawType = (Class<?>) parameterType.getRawType();
    } else {
        rawType = (Class<?>) genericParameterType;
    }

    Collection coll;
    if (rawType == List.class) {
        coll = new ArrayList<>();
    } else if (rawType == Set.class) {
        coll = new HashSet<>();
    } else {
        coll = (Collection) rawType.newInstance();
    }

    for (String value : values) {
        coll.add(parseSimpleType(value, actualTypeArgument));
    }
    return coll;
}
```

数组类型，反射获取到组件类型后按简单类型处理

```java
if (paramType.isArray()) {
    // 数组元素类型
    Class<?> componentType = paramType.getComponentType();

    List<String> values = allParams.getAll(paramName);
    Object array = Array.newInstance(componentType, values.size());
    for (int j = 0; j < values.size(); j++) {
        Array.set(array, j, parseSimpleType(values.get(j), componentType));
    }
    return array;
}
```

数据类型，反射获取类型中字段的类型，递归处理

```java
private Object parseBeanType(MultiMap allParams, Class<?> paramType) throws Throwable {
    Object bean = paramType.newInstance();
    Field[] fields = paramType.getDeclaredFields();
    for (Field field : fields) {
        Object value = parseSimpleTypeOrArrayOrCollection(
            allParams, field.getType(), field.getName(), field.getGenericType());

        field.setAccessible(true);
        field.set(bean, value);
    }
    return bean;
}
```

特殊类型，其中`RoutingContext`类型不会作为请求参数，而是由route中获取并注入。

```java
if (paramType == RoutingContext.class) {
    return ctx;
} else if (paramType == FileUpload.class) {
    Set<FileUpload> uploads = ctx.fileUploads();
    Map<String, FileUpload> uploadMap = uploads
        .stream()
        .collect(Collectors.toMap(FileUpload::name, x -> x));
    return uploadMap.get(paramNames[i]);
}
```

好了，现在`@RequestMapping`的功能已经实现了。

## @RequestBody

默认情况下用户进行POST请求，数据是以表单形式提交的，有些时候我们需要以JSON形式提交，那么本注解就是实现这样的功能。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.PARAMETER})
public @interface RequestBody {
}
```

这里处理方式比较简单，把整个body以字符串类型读取并进行反序列化

```java
List<? extends Class<? extends Annotation>> parameterAnnotation = Arrays
	.stream(parameterAnnotations[i])
    .map(Annotation::annotationType)
    .collect(Collectors.toList());

if (parameterAnnotation.contains(RequestBody.class)) {
    String bodyAsString = ctx.getBodyAsString();
    argValues[i] = Json.decodeValue(bodyAsString, paramType);
}
```

















