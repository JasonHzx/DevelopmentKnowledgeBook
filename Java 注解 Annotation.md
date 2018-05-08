# Java 注解 Annotation

不少开源库以及Android系统自身都用到了注解的方式来简化代码提高开发效率。
本文简单介绍下 Annotation 示例、概念及作用、分类、自定义、解析，并对几个 Android 开源库 Annotation 原理进行简析。


# 注解案例

Android Activity Override Annotation

```java
@Override
public void onCreate(Bundle savedInstanceState);
```

Retrofit Annotation （一个符合 RESTful 规范的网络请求框架）

```java
@GET("/users/{username}")
User getUser(@Path("username") String username);
```

Butter Knife Annotation  (一个View 及事件等依赖注入框架)

```java
@InjectView(R.id.user) EditText username;
```

# 注解概念

Annotation 中文常译为“注解”。
注解是插入到Java源代码中的一种元数据（meta data）。类、方法、变量、参数、包都可以被注解。
何为元数据，即描述数据的数据。
我的理解是注解就是一种标签，类似给人打标签一样给代码打标签

# 注解作用

注解信息可以在编译期使用预编译工具进行处理（pre-compiler tools），也可以在运行期使用 Java 反射机制进行处理。

```
1. 标记，用于告诉编译器一些信息
2. 编译时动态处理，如动态生成代码  
3. 运行时动态处理，如得到注解信息
```

这里的三个作用实际对应着后面自定义 Annotation 时说的 @Retention 三种值分别表示的 Annotation

# 注解的分类

### 按照运行机制划分：

```
1. 源码注解
2. 编译时注解
3. 运行时注解
```

源码注解：只在源码中存在，编译成.class文件就不存在了。

编译时注解：在源码和.class文件中都存在。像前面的@Override、@Deprecated、@SuppressWarnings，他们都属于编译时注解。

运行时注解：在运行阶段还起作用，甚至会影响运行逻辑的注解。它会在程序运行的时候把你的成员变量自动的注入进来。

### 按照来源划分：

```
1. java内置注解（包含 元注解）
2. 第三方注解 （比如Retrofit）
3. 自定义注解
```


# java内置注解

代码注解：

```java
	•	@Override - 检查该方法是否是重载方法。如果发现其父类，或者是引用的接口中并没有该方法时，会报编译错误。
	•	@Deprecated - 标记过时方法。如果使用该方法，会报编译警告。
	•	@SuppressWarnings - 指示编译器去忽略注解中声明的警告。
```

## 元注解(用于定义注解的注解):

```java
	•	@Retention - 标记这个注解的保留时间
	•	@Documented - 是否包含在用户文档中
	•	@Target - 标记这个注解应该是哪种 Java 成员，未标注则表示可修饰所有
	•	@Inherited - 是否可以被继承，默认为 false
```

### @Retention：注解的作用时间

```java

//SOURCE（源码时）,注解仅存在于源码中，在class字节码文件中不包含,这类注解大都用来校验，比如 Override, SuppressWarnings
@Retention(RetentionPolicy.SOURCE)   

// CLASS（编译时），默认的保留策略，注解会在class字节码文件中存在，但运行时无法获得
@Retention(RetentionPolicy.CLASS)     

// RUNTIME（运行时），注解会在class字节码文件中存在，在运行时可以通过反射获取到
@Retention(RetentionPolicy.RUNTIME)  


```
### @Target：注解的作用目标

```java
@Target(ElementType.TYPE)   //接口、类、枚举、注解
@Target(ElementType.FIELD) //字段、枚举的常量
@Target(ElementType.METHOD) //方法
@Target(ElementType.PARAMETER) //方法参数
@Target(ElementType.CONSTRUCTOR)  //构造函数
@Target(ElementType.LOCAL_VARIABLE)//局部变量
@Target(ElementType.ANNOTATION_TYPE)//注解
@Target(ElementType.PACKAGE) ///包   
```

# 自定义注解

## 调用的例子

```java
public class App {

    @MethodInfo(
        author = “jason”,
        date = "2018/01/23",
        version = 2)
    public String getAppName() {
        return "jasonDemo";
    }
}

```
这里是调用自定义 Annotation——MethodInfo 的示例。
MethodInfo Annotation 作用是给方法添加相关信息，包括 author、date、version。

## 定义的例子

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface MethodInfo {

    String author() default "jason";

    String date();

    int version() default 1;
}
```

这里是 MethodInfo 的实现部分
(1). 通过 @interface 定义，注解名即为自定义注解名
(2). 注解配置参数名为注解类的方法名，且：
a. 所有方法没有方法体，没有参数没有修饰符，实际只允许 public & abstract 修饰符，默认为 public，不允许抛异常
b. 方法返回值只能是基本类型，String, Class, annotation, enumeration 或者是他们的一维数组
c. 若只有一个默认属性，可直接用 value() 函数。一个属性都没有表示该 Annotation 为 Mark Annotation
(3). 可以加 default 表示默认值

## Annotation 解析

### 运行时 Annotation 解析
(1) 运行时 Annotation 指 @Retention 为 RUNTIME 的 Annotation，可手动调用下面常用 API 解析

```java
method.getAnnotation(AnnotationName.class);
method.getAnnotations();
method.isAnnotationPresent(AnnotationName.class);
```
其他 @Target 如 Field，Class 方法类似
getAnnotation(AnnotationName.class) 表示得到该 Target 某个 Annotation 的信息，因为一个 Target 可以被多个 Annotation 修饰
getAnnotations() 则表示得到该 Target 所有 Annotation
isAnnotationPresent(AnnotationName.class) 表示该 Target 是否被某个 Annotation 修饰

(2) 解析示例如下：

```java
public static void main(String[] args) {
    try {
        Class cls = Class.forName("com.xxx.java.test.annotation.App");
        for (Method method : cls.getMethods()) {
            MethodInfo methodInfo = method.getAnnotation(
MethodInfo.class);
            if (methodInfo != null) {
                System.out.println("method name:" + method.getName());
                System.out.println("method author:" + methodInfo.author());
                System.out.println("method version:" + methodInfo.version());
                System.out.println("method date:" + methodInfo.date());
            }
        }
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

以之前自定义的 MethodInfo 为例，利用 Target（这里是 Method）getAnnotation 函数得到 Annotation 信息，然后就可以调用 Annotation 的方法得到响应属性值

### 编译时 Annotation 解析
(1) 编译时 Annotation 指 @Retention 为 CLASS 的 Annotation，甴编译器自动解析。需要做的
a. 自定义类集成自 AbstractProcessor
b. 重写其中的 process 函数
这块很多同学不理解，实际是编译器在编译时自动查找所有继承自 AbstractProcessor 的类，然后调用他们的 process 方法去处理
(2) 假设 MethodInfo 的 @Retention 为 CLASS，解析示例如下：

```java
@SupportedAnnotationTypes({ "com.xxx.java.test.annotation.MethodInfo" })
public class MethodInfoProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        HashMap<String, String> map = new HashMap<String, String>();
        for (TypeElement te : annotations) {
            for (Element element : env.getElementsAnnotatedWith(te)) {
                MethodInfo methodInfo = element.getAnnotation(MethodInfo.class);
                map.put(element.getEnclosingElement().toString(), methodInfo.author());
            }
        }
        return false;
    }
}
```

SupportedAnnotationTypes 表示这个 Processor 要处理的 Annotation 名字。
process 函数中参数 annotations 表示待处理的 Annotations，参数 env 表示当前或是之前的运行环境
process 函数返回值表示这组 annotations 是否被这个 Processor 接受，如果接受后续子的 processor 不会再对这个 Annotations 进行处理

# 几个 Android 开源库 Annotation 原理简析

## Annotation — Retrofit
(1) 调用

```java
@GET("/users/{username}")
User getUser(@Path("username") String username);
```

(2) 定义

```java
@Documented
@Target(METHOD)
@Retention(RUNTIME)
@RestMethod("GET")
public @interface GET {
  String value();
}
```

从定义可看出 Retrofit 的 Get Annotation 是运行时 Annotation，并且只能用于修饰 Method

(3) 原理

```java
private void parseMethodAnnotations() {
    for (Annotation methodAnnotation : method.getAnnotations()) {
    Class<? extends Annotation> annotationType = methodAnnotation.annotationType();
    RestMethod methodInfo = null;

    for (Annotation innerAnnotation : annotationType.getAnnotations()) {
        if (RestMethod.class == innerAnnotation.annotationType()) {
            methodInfo = (RestMethod) innerAnnotation;
            break;
        }
    }
    ……
    }
} 
```
RestMethodInfo.java 的 parseMethodAnnotations 方法如上，会检查每个方法的每个 Annotation， 看是否被 RestMethod 这个 Annotation 修饰的 Annotation 修饰，这个有点绕，就是是否被 GET、DELETE、POST、PUT、HEAD、PATCH 这些 Annotation 修饰，然后得到 Annotation 信息，在对接口进行动态代理时会掉用到这些 Annotation 信息从而完成调用。
Retrofit 原理涉及到动态代理，这里原理都只介绍 Annotation

## Annotation — Butter Knife

(1) 调用

```java
@InjectView(R.id.user) 
EditText username;
```

(2) 定义

```java
@Retention(CLASS) 
@Target(FIELD)
public @interface InjectView {
  int value();
}
```
可看出 Butter Knife 的 InjectView Annotation 是编译时 Annotation，并且只能用于修饰属性

(3) 原理

```java
@Override 
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, ViewInjector> targetClassMap = findAndParseTargets(env);

    for (Map.Entry<TypeElement, ViewInjector> entry : targetClassMap.entrySet()) {
        TypeElement typeElement = entry.getKey();
        ViewInjector viewInjector = entry.getValue();

        try {
            JavaFileObject jfo = filer.createSourceFile(viewInjector.getFqcn(), typeElement);
            Writer writer = jfo.openWriter();
            writer.write(viewInjector.brewJava());
            writer.flush();
            writer.close();
        } catch (IOException e) {
            error(typeElement, "Unable to write injector for type %s: %s", typeElement, e.getMessage());
        }
    }

    return true;
}
```
ButterKnifeProcessor.java 的 process 方法如上，编译时，在此方法中过滤 InjectView 这个 Annotation 到 targetClassMap 后，会根据 targetClassMap 中元素生成不同的 class 文件到最终的 APK 中，然后在运行时调用 ButterKnife.inject(x) 函数时会到之前编译时生成的类中去找。




