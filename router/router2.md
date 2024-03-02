# 自定义路由框架详解（二）- 标记页面、收集页面

## 为什么需要路由框架？

在Android开发中，路由框架的引入主要是为了解决应用内页面跳转、组件间通信和模块化开发中的复杂性和灵活性问题。路由框架为开发者提供了一种更简洁、高效和解耦的方式来处理这些问题。以下是路由框架的几个主要优点：

1. 解耦: 在传统的Android开发中，页面间的跳转通常需要显式地声明Intent和目标Activity。这种方式使得组件之间紧密耦合，难以管理和维护。路由框架通过统一的路由表和动态绑定，允许开发者用URL-like的字符串来指代目标页面，减少了组件间的直接依赖。
2. 简化页面跳转: 使用路由框架，开发者可以轻松实现页面跳转和参数传递，只需一行代码即可完成跳转逻辑，无需手动处理Intent和Bundle，使代码更加简洁。
3. 模块化和组件化支持: 随着项目规模的扩大，模块化和组件化开发变得越来越重要。路由框架支持跨模块的页面跳转和通信，有助于实现真正的模块化开发，每个模块可以独立开发、编译、测试，最终通过路由框架动态集成。
4. 动态功能加载: 高级路由框架支持动态加载模块或功能，这对于需要按需加载资源或实现热更新功能的应用尤其重要。
5. 统一拦截和预处理: 路由框架通常提供拦截器（Interceptor）功能，允许开发者在跳转流程中插入自定义逻辑，如登录检查、权限验证、数据预加载等，增强了应用的安全性和用户体验。
6. 支持跨平台跳转: 一些路由框架设计之初就考虑到了跨平台的需求，支持从Web页面跳转到原生页面，或者反过来，这对于混合开发（Hybrid Development）尤其有用。

总之，路由框架通过提供一套中心化、灵活的机制来处理应用内的页面跳转和组件间通信，极大地提升了开发效率，降低了维护成本，并有助于实现高质量的应用架构。

## 整体方案

自定义路由框架整体上可以分为以下5步：

1. 标记页面
2. 收集页面
3. 生成文档
4. 注册映射
5. 打开页面

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MWYxYTFkYTQ5ZWFhYzZjODY0MWM4ZDJmMGNkYTljMzlfNVA3V3B0ZEg3UlRSZXlvZktzeGwxT0NEb0pReGJBQXpfVG9rZW46TmF1amJLZnAwbzlhWEV4UE1aaGNrTkhDbkFjXzE3MDkzNjU0NzY6MTcwOTM2OTA3Nl9WNA)

来详细解释一下上面五步

1. 标记页面，要求我们提供一个注解，使用这个注解标记就代表这个类将会被注册，后续会通过注解中的相关信息跳转到这个类。我们将提供一个Destination 的注解，来供使用者标记
2. 收集页面   通过注解处理器将通过Destination 注册的activity信息收集起来
3. 生成文档   在第二步中，多项目的页面被分散在不同的项目中，需要有一个文档，涵盖了所有子项目注册的页面，这样方便程序员查找activity对应的Router路径
4. 注册映射  内存中需要记录所有注册了的activity的信息，这样当业务方通过Router跳转时，才能获取到最终要跳转到的类
5. 打开页面  Router可以携带参数信息，这一步需要解析参数信息，并根据第四步中的所有的注册信息跳转到对应的activity

## 标记页面

自定义了Destination注解

```Java
@Target({ElementType.TYPE})

@Retention(RetentionPolicy.CLASS)
public @interface Destination {

    String url();

    String description();
}
```

使用方法如下：

```Java
@Destination(
        url = "router://page-home",
        description = "主页"
)
public class MainActivity extends Activity {
    ...
}
```

## 收集页面

### **APT介绍**

#### **APT的理解**

现在有很多主流库都用上了 APT，比如 Dagger2, ButterKnife, EventBus3 等

代表框架：

- DataBinding
- Dagger2
- ButterKnife
- EventBus3
- DBFlow

**APT**(Annotation Processing Tool)，即 **注解处理器**，是javac中提供的**编译时扫描和处理注解**的工具，它对源代码文件进行检测找出其中的注解，然后使用注解进行额外的处理。

**注解**就像是一个[标签](https://cloud.tencent.com/developer/techpedia/1416?from_column=20065&from=20065)，有很多类型，可以贴在某些元素上面进行标记，并且标签上可以写一些信息。APT就是用来处理标签的工具，在编译开始后，可以拿到自己所关心的类型的所有标签，然后根据标签信息和被标记的元素信息，做一些事情。做那些事呢，这就看你如何写APT了，你让他干啥他就干啥，通常都是会生成一些帮助类——帮助完成你的目的的类。后面无论对这种标签的使用是增加、减少了，**每次编译**都会重新走这一过程，而上一次的处理结果会被清空。

宏观上理解，APT就是javac提供给开发者在编译时处理注解的一种技术；微观上，具体到实例中就是指 继承自javax.annotation.processing.**AbstractProcessor** 的实现类，即一个处理特定注解的处理器。（下文提到的APT都是宏观上理解，具体的处理器简称为Processor）

**那不使用APT能否完成目的呢？** 也是可以的，毕竟APT就是为了帮助我完成目的，那我自己肯定也是可以完成目的的，只不过有了APT会很省事。例如，上篇提到的帮助类，目的就是为了收集路由元信息（路由目标类class），我们如果不使用ARouter，那么就需要自己定义一个moduleA、moduelB共同依赖的muduleX，把需要进行跳转的XXXActivity这些类的class手工写代码保存到muduleX的Map中，key可以用XXXActivity的类名，value就是XXXActivity.class，这样moduleA、moduelB之间发起跳转时 就通过想要跳转的Activity的类名 从muduleX的Map中获取目标Activity的class，这样也是能完成 无相互依赖的moduleA、moduelB之前进行页面跳转的。

而使用了APT，只要使用注解进行标记即可，无论使用者怎么标记，**每次编译时都由APT统一处理，不会出错、也不担心有遗漏**。

通常用到APT技术的都是像ARouter这样的通用能力框架：

- 提供定义好的注解，例如@Route
- 提供简洁的API接口，例如ARouter.getInstance().build("/module/1").navigation()

这样上层业务使用极为方便，只需要使用注解进行标记，然后调用API即可。**信息如何收集和存储、如何寻找和使用，框架使用者是不用关心的。**

APT还有两个特点：

- 获取注解及生成代码都是在代码编译时候完成的，相比反射在运行时处理注解大大提高了程序性能。
- 注意APT并不能对源文件进行修改，只能获取注解信息和被注解对象的信息，然后做一些自定义的处理，例如生成java类。

#### **APT的原理**

我们先来看下Java的编译过程：

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTg0Y2QyYTQ1Y2FhOGE4NGZmOGMxNmQ2YWRlNTc2MTlfZ2JnQzFUY2FYV3ZESXFOOXFrSEowMlZSN3hObjExYWxfVG9rZW46R1dlNGJoTXV4b3dwYXd4OGlyZWMxSHg3bmpnXzE3MDkzNjU0NzY6MTcwOTM2OTA3Nl9WNA)

可以看到在Java源码到class文件之间，需要经过**注解处理器**的处理，注解处理器生成的代码也同样会经过这一过程，最终一起生成class文件。在[Android](https://cloud.tencent.com/developer/techpedia/1819?from_column=20065&from=20065)中，class文件还会被打进Dex文件中，最后生成APK文件。

注解处理器的执行是在编译的初始阶段，并且会有多个processor（查看所有注册的processor：intermediates/annotation_processor_list/debug/annotationProcessors.json）。

那么我们自定义的注解处理器，如何注册进去呢？又如何实现一个注解处理器呢？

来看一个简单的实例：

```Java
@AutoService(Processor.class) //把 TestProcessor注册到编译器中
@SupportedAnnotationTypes({"com.hfy.test_annotations.TestAnnotation"})//TestProcessor 要处理的注解
@SupportedSourceVersion(SourceVersion.RELEASE_8)//设置jdk环境为java8
//@SupportedOptions() //一些支持的可选配置项
public class TestProcessor extends AbstractProcessor {
    public static final String ACTIVITY = "android.app.Activity";
    //Filer 就是文件流输出路径，当我们用AbstractProcess生成一个java类的时候，我们需要保存在Filer指定的目录下
    Filer mFiler;
    //类型 相关的工具类。当process执行的时候，由于并没有加载类信息，所以java文件中的类信息都是用element来代替了。
    //类型相关的都被转化成了一个叫TypeMirror，其getKind方法返回类型信息，其中包含了基础类型以及引用类型。
    Types types;
    //Elements 获取元素信息的工具，比如说一些类信息继承关系等。
    Elements elementUtils;
    //来报告错误、警告以及提示信息
    //用来写一些信息给使用此注解库的第三方开发者的
    Messager messager;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mFiler = processingEnv.getFiler();
        types = processingEnv.getTypeUtils();
        elementUtils = processingEnv.getElementUtils();
        messager = processingEnv.getMessager();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnvironment) {
        if (annotations == null || annotations.size() == ){
            return false;
        }
        //获取所有包含 @TestAnnotation 注解的元素
        Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(TestAnnotation.class);

        //每个Processor的独自的逻辑，其他的写法一般都是固定的
        parseAnnotation(elements)

        return true;
    }

    //解析注解并生成java文件
    private boolean parseAnnotation(Set<? extends Element> elements) {
       ...
    }

}
```

上面的TestProcessor就是一个典型的注解处理器，继承自javax.annotation.processing.**AbstractProcessor**。注意到TestProcessor重写了init()和process()两个方法，并且添加了几个注解。

几个注解是**处理器的注册和配置** ：

1. 使用注解`@AutoService`进行注册，这样在编译这段就会执行这个处理器了。需要依赖com.google.auto.service:auto-service:1.0-rc4' 才能使用@AutoService
2. 使用注解`@SupportedAnnotationTypes` 设置 TestProcessor 要处理的注解，要使用目标注解全类名
3. 使用注解`@SupportedSourceVersion`设置支持的Java版本、使用注解@SupportedOptions()配置一些支持的可选配置项

重写的两个方法：init()、process()是Processor的初始化和处理过程:

1. init()
2. 方法是初始化处理器，每个注解处理器被初始化的时候都会被调用，通常是这里使用 处理环境-
3. ProcessingEnvironment
4. 获取一些工具实例，如上所示是一般通用的写法：
   1. **mFiler**，就是文件流输出路径，当我们用AbstractProcess生成一个java类的时候，我们需要保存在Filer指定的目录下
   2. **types**，类型 相关的工具类。当process执行的时候，由于并没有加载类信息，所以java文件中的类信息都是用element来代替了。类型相关的都被转化成了一个叫TypeMirror，其getKind方法返回类型信息，其中包含了基础类型以及引用类型。
   3. **elementUtils**，获取元素信息的工具，比如说一些类信息继承关系等。
   4. **messager**，来报告错误、警告以及提示信息，用来写一些信息给使用此注解库的第三方开发者的
5. process()
6. 方法，注解处理器实际处理方法，在这里写处理注解的代码，以及生成Java文件。它有两个参数：
   1. **annotations**，是@SupportedAnnotationTypes声明的注解 和 未被其他Processor消费的注解的子集，也就是剩下的且是本Processor关注的注解，类型是**TypeElement**
   2. **roundEnvironment**，有关当前和之前处理器的环境信息，可以让你查询出包含特定注解的被注解元素
   3. 返回值，true表示@SupportedAnnotationTypes中声明的注解 由此 Processor 消费掉，不会传给下个Processor。注解不断向下分发，每个processor都可以决定是否消费掉自己声明的注解。

**在编译流程进入Processor前，APT会对整个Java源文件进行扫描，这样就会获取到 所有添加了的注解和对应被注解的类。注解和被注解的类，一起被视为一个元素，即TypeElement，就是process()方法参数annotations的数据类型。通过TypeElement，我们可以获取注解的所有信息、被注解类的所有信息，这样就可以根据这些信息来生成 我们需要的帮助类了**。

> 这里重点是理解Processor的原理，关于注解和Element的知识这里不过多介绍，可自行了解。

到这里，Processor的工作流程我们了解了，并且其 定义、配置、注册 这些都是固定的写法，一般无需过多关注。**最重要的就是 process()方法的实现：拿到所有关注的注解元素后，就是每个Processor的独自的逻辑——解析注解并生成需要的java文件。**

### 实现

我们希望在这一步达成的目标是，对于每个应用插件的模块，都生成一个类似下面的java文件，该文件中记录了所有在这个模块中注册了的类

```Java
package com.imooc.router.mapping;

import java.util.HashMap;
import java.util.Map;

public class RouterMapping_1708842731648 {

    public static Map<String, String> get() {

        Map<String, String> mapping = new HashMap<>();

        mapping.put("router://page-home", "com.imooc.router.demo.MainActivity");
        mapping.put("router://page-kotlin", "com.imooc.router.demo.KtMainActivity");
        mapping.put("router://imooc/profile", "com.imooc.router.demo.ProfileActivity");
        return mapping;
    }
}
```

下面是我们自定义的注解处理器的详细代码

```Java
@AutoService(Processor.class)
public class DestinationProcessor extends AbstractProcessor {

    private static final String TAG = "DestinationProcessor";

    /**
     *
     * @param set
     * @param roundEnvironment
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> set,
                           RoundEnvironment roundEnvironment) {

        //1. 有什么作用？为什么要写这个if？
        if (roundEnvironment.processingOver()) {
            return false;
        }

        System.out.println(TAG + " >>> process start ...");

        String rootDir = processingEnv.getOptions().get("root_project_dir");

        // 2. 过滤当前project获取到的被Destination注解的Element
        Set<Element> allDestinationElements = (Set<Element>)
                roundEnvironment.getElementsAnnotatedWith(Destination.class);

        System.out.println(TAG + " >>> all Destination elements count = "
                + allDestinationElements.size());


        if (allDestinationElements.size() < 1) {
            return false;
        }


        String className = "RouterMapping_" + System.currentTimeMillis();

        StringBuilder builder = new StringBuilder();
        // 3. 构建一个java类，这个java类中有一个get方法，通过get方法可以获取到该
        // project中的所有注册信息
        builder.append("package com.imooc.router.mapping;\n\n");
        builder.append("import java.util.HashMap;\n");
        builder.append("import java.util.Map;\n\n");
        builder.append("public class ").append(className).append(" {\n\n");
        builder.append("    public static Map<String, String> get() {\n\n");
        builder.append("        Map<String, String> mapping = new HashMap<>();\n\n");

        final JsonArray destinationJsonArray = new JsonArray();

        
        for (Element element : allDestinationElements) {
            // 4. 对于每个注册的类，把它放入到一个json文件中，这个json文件记录类每个注册类的
            // url、description、realPath
            final TypeElement typeElement = (TypeElement) element;


            final Destination destination =
                    typeElement.getAnnotation(Destination.class);

            if (destination == null) {
                continue;
            }

            final String url = destination.url();
            final String description = destination.description();
            final String realPath = typeElement.getQualifiedName().toString();

            System.out.println(TAG + " >>> url = " + url);
            System.out.println(TAG + " >>> description = " + description);
            System.out.println(TAG + " >>> realPath = " + realPath);

            builder.append("        ")
                    .append("mapping.put(")
                    .append("\"" + url + "\"")
                    .append(", ")
                    .append("\"" + realPath + "\"")
                    .append(");\n");

            JsonObject item = new JsonObject();
            item.addProperty("url", url);
            item.addProperty("description", description);
            item.addProperty("realPath", realPath);

            destinationJsonArray.add(item);
        }
        
```

看到上面代码的注释 1 - 4，我们一步一步解释process方法中的代码：

1. 同一个project会调用process方法两遍（不确定是不是所有project都会调用两遍，但app 模块是调用了两遍），这个if 可以在第二遍调用process方法的时候，及时推出，否则可能会导致最终生成的文件中有两遍内容。
2. 过滤当前project 获取到的被 Destination 注解的Element
3. 构建最开始说的RouterMapping_xxx.java类，这个java类中有一个get方法，通过get方法可以获取到该project中的所有注册信息
4. 对于每个注册的类，把它放入到一个json文件中，这个json文件记录类每个注册类的 url、description、realPath

上面的代码为了方便阅读只显示了一半，下面是另一半

```Java
        // 5. 接着第3步，构建java类结束
        builder.append("        return mapping;\n");
        builder.append("    }\n");
        builder.append("}\n");

        String mappingFullClassName = "com.imooc.router.mapping." + className;

        System.out.println(TAG + " >>> mappingFullClassName = "
                + mappingFullClassName);

        System.out.println(TAG + " >>> class content = \n" + builder);


        try {
            // 6.写入java类
            JavaFileObject source = processingEnv.getFiler()
                    .createSourceFile(mappingFullClassName);
            Writer writer = source.openWriter();
            writer.write(builder.toString());
            writer.flush();
            writer.close();
        } catch (Exception ex) {
            throw new RuntimeException("Error while create file", ex);
        }


        File rootDirFile = new File(rootDir);
        if (!rootDirFile.exists()) {
            throw new RuntimeException("root_project_dir not exist!");
        }


        File routerFileDir = new File(rootDirFile, "router_mapping");
        if (!routerFileDir.exists()) {
            routerFileDir.mkdir();
        }

        File mappingFile = new File(routerFileDir,
                "mapping_" + System.currentTimeMillis() + ".json");


        try {
            // 7. 写入json文件
            BufferedWriter out = new BufferedWriter(new FileWriter(mappingFile));
            String jsonStr = destinationJsonArray.toString();
            out.write(jsonStr);
            out.flush();
            out.close();
        } catch (Throwable throwable) {
            throw new RuntimeException("Error while writing json", throwable);
        }

        System.out.println(TAG + " >>> process finish.");

        return false;
    }

    /**
     *
     * @return
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(
                Destination.class.getCanonicalName()
        );
    }
}
```

1. 接着第3步，构建java类结束
2. 写入java文件
3. 写入json文件