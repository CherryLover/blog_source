---
title: 从注解到 ButterKnife
---

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/butterKnife20191028135145.jpg)

# 从注解到 ButterKnife

一直好奇 BufferKnife 内部是怎么做到 findViewById 的，今天就好好看看它。

```java
@BindView(R.id.tv_hello) TextView tvHello;
@Override public void onCreate(Bundle savedInstanceState) {
    //......
    ButterKnife.bind(this);
}
```

其实，重点就是那个注解，利用 `BindView` 这个注解，可以把 tvHello 这个成员变量跟 `R.id.tv_hello` 进行绑定，下面就来一步步实现一下。

先来认识一下注解吧。声明注解和声明一个借口类似，不同的是注解声明时在 interface 这个关键字前加一个 `@` 符号，就像这样：`public @interface BindView` ，而且注解还可以增加各种限制，比如限制当前注解的适用范围（`Target`），是只能用于方法（函数）还是可以用在成员变量上，亦或是可以用在类型上。当然了，最重要的还是可以设置当前注解的保存期限，有三个选项可供选择，分别是：

- SOURCE：仅保留在源代码时；
- CLASS：保留到 java 代码编译到 class 文件时；
- RUNTIME：保留到实际运行时 Class 所对应的对象上；

在声明注解内部的方法时也和接口有所不用，在注解中，可以为声明的方法设置一个默认值，这个是接口没有的，其他的也是像接口一样的声明。

> 在 Java 后续的版本中提供了设置默认值的方式，这里只讨论 Java 本身的接口设计不包含其他后续版本的新特性。

注解内部也可以定义多个方法，使用时需要显式声明出多个方法，就像下面这样。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface BindView {

    int value();

    String name() default "";
}

@BindView(value = R.id.tv_hello, name = "jiang") TextView tvHello;
```

如果注解里面只有一个方法，且方法名为 value，在使用时可以省去方法名。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface BindView {
    int value();
}

@BindView(R.id.tv_hello) TextView tvHello;
```

简单的了解一下注解，之后我们可以继续往下了。注意，在这里我们对 BindView 这个注解加了两个限制，一个指明了当前的生命周期是 RUNTIME，另一个是限制只用在成员变量上，避免误用。

注解定义好之后，我们还需要再实现一个类，干什么的呢？就是做跟  `ButterKnife.bind(this);` 一样的事情。那简简单单的一行代码里，具体做的什么呢？简单来说，利用传入的 this 配合着注解，在运行时通过反射进行 findViewById 的操作。

```java
public static void bind(Activity activity) {
    for (Field field : activity.getClass().getDeclaredFields()) {
        BindView annotation = field.getAnnotation(BindView.class);
        if (annotation != null) {
            try {
                field.setAccessible(true);
                field.set(activity, activity.findViewById(annotation.value()));
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
    }
}
```

这样好是好，不过呢，就一点不好，反射是有时间成本的，所以，ButterKnife 换了另一种方式，变成了利用注解动态生成代码的方式。

生成什么代码呢？其实就是生成 findViewById 这行代码，并且我们在生成之后还需要让代码能够执行，自动执行好办，可以利用反射，根据生成代码的路径，通过 `Class#forName` 来完成对生成代码后的调用。

```java
public static void bind(Activity activity) {
        // $Binding 是自动生成代码的后缀，用于与开发者声明的类做区分
        Class bindClass = Class.forName(activity.getClass().getCanonicalName() + "$Binding");
        Class activityClass = Class.forName(activity.getClass().getCanonicalName());
        Constructor constructor = bindClass.getDeclaredConstructor(activityClass);
        constructor.newInstance(activity);
    }
```

看这部分代码这么少就知道重头戏不在这，而在于动态生成代码，其实动态生成我们 Android 开发者一点也不陌生，就是 AnnotationProcessor。

要实现 AnnotationProcessor 我们需要创建一个 Java Module，之后再稍加配置，然后就可以写出让我们能自动生成代码的代码了。

> 这里我创建的 module 名为 lib-processor

配置：

1. 在 java 文件夹中正确的包下创建一个继承自 `AbstractProcessor` 的类；
2. 在 main 文件夹下新建与 java 文件夹同级的名为 resources 文件夹；
3. 在 resources 中按照 `resources/META-INF/services/javax.annotation.processing.Processor ` 新建一个文件，并把刚才新建的类的路径写入 `javax.annotation.processing.Processor`；

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/butterKnife20191028130047.jpg)



现在就需要开始编写能够进行自动生成代码的代码了，不过在此之前，我们需要理一下思路。首先，我们需要注解，其次，需要用到 AnnotationProcessor，然后才能在 Activity 中调用 `@BindView` 的注解以及 `Bind.bind(this)` 方法。

这里我们需要注意一点，就是 AnnotationProcessor 需要用到「依赖」注解「BindView」，Activity 所在的 App 模块也需要用到它，这样一来我们就不能在 app 模块定义注解，需要在另一个模块中单独定义注解，这个模块叫 `lib-annotation`。而且我要做的是像 ButterKnife 一样的依赖库，所以还得把 Bind 这个类也放在一个 module 里，而且还不能放在 `lib-annotaion` 中，所以我们还需要一个叫 `bind-lib` 的模块。

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/butterKnife20191028131931.jpg)

> 1. app 模块和 bind-lib 都需要依赖 lib-annotation，可以在 bind-lib 已传递依赖的形式进行依赖 lib-annotaion，这样 app 模块就不需要单独声明依赖 lib-annotation。
>
>    ```groovy
>    api project(path: ':lib-annotaion')
>    ```
>
> 2. 动态生成代码需要用到一个第三方库
>
>    ```groovy
>    implementation 'com.squareup:javapoet:1.11.1'
>    ```

准备工作一切就绪，下面来看看动态生成的代码吧。

```java
public class BindProcessor extends AbstractProcessor {
    private Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        filer = processingEnvironment.getFiler();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {

        for (Element element : roundEnvironment.getRootElements()) {
            String packageStr = element.getEnclosingElement().toString();
            String classStr = element.getSimpleName().toString();
            ClassName className = ClassName.get(packageStr, classStr + "$Binding");
            MethodSpec.Builder constructorBuilder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC)
                    .addParameter(ClassName.get(packageStr, classStr), "activity");
            boolean hasBinding = false;
            for (Element enclosedElement : element.getEnclosedElements()) {
                BindView bindView = enclosedElement.getAnnotation(BindView.class);
                if (bindView != null) {
                    hasBinding = true;
                    constructorBuilder.addStatement("activity.$N = activity.findViewById($L)", enclosedElement.getSimpleName(), bindView.value());
                }
            }
            TypeSpec buildClass = TypeSpec.classBuilder(className)
                    .addModifiers(Modifier.PUBLIC)
                    .addMethod(constructorBuilder.build())
                    .build();
            if (hasBinding) {
                JavaFile.builder(packageStr, buildClass).build().writeTo(filer);
            }
        }

        return false;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(BindView.class.getCanonicalName());
    }
}
```

先来解释一下这三个方法：

- getSupportedAnnotationTypes

  这个方法真的是见名之意，就是获取当前 AnnotationProcessor 所支持的注解集合而已；

- init

  初始化 AnnotationProcessor 并获取到 Filer，这个 Filer 是我们等下生成代码需要用到的。

- process

  真正做生成代码的部分。

这个类中，process 才是真正做事情的方法，代码也不难懂。先是从根据当前类提取到有用的包名及类名，再利用包名、类名及特殊字符创建出一个类，紧接着为其声明一个参数名为 `activity` 的公开构造方法，在构造方法内部加入一行 `activity.$N = activity.findViewById($L)` 。其中，N 和 L 均是占位符，真正的值是开发者定义的字段的 name 以及注解的 value 也就是 View 的 Id。

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/butterKnife20191028133523.png)

这个就是 ButterKnife 它做的事情，不同的是，它内部还做了其他的优化以及一些其他功能。





本文首发于[个人博客](https://jiangjiwei.site/)，文中全部源代码已上传至 [GitHub](https://github.com/CherryLover/BlogTest)，代码分支为 bindView。喜欢本文的麻烦点个🌟。

本文封面图：Photo by [Wil Stewart](https://unsplash.com/@wilstewart3?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/look?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)