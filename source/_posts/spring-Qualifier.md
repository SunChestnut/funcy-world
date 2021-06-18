---
title: 探讨的问题：@Qualifier 注解解决了什么问题？如何使用？
date: 2021-06-11 15:42:57
tags:
---

# `@Autowired`需要用`@Qualifier`注解消除歧义

在某些情况下，比如一个接口有多个实现类，且每个实现类都被声明成组件时，`@Autowired`注解单独使用时就不足以让 Spring 区分需要注入哪个组件（PS：`@Autowired`只能类型注入）。

> If more than one bean of the same type is available in the container, the framework will throw NoUniqueBeanDefinitionException. 

证实如下：

现有一个接口`Activity`，表示各个学校可以组织的活动类型，具体信息如下：

```Java
public interface Activity {

    void ballGame();

    void sing();
}
``` 

学校 A 和学校 B 分别根据活动类型，组织了相应的活动：

```Java
@Component("schoolA")
public class SchoolA implements Activity {

    @Override
    public void ballGame() {
        System.out.println("School A is playing football.");
    }

    @Override
    public void sing() {
        System.out.println("School A is sing...");
    }
}
```

```Java
@Component("schoolB")
public class SchoolB implements Activity {

    @Override
    public void ballGame() {
        System.out.println("School B is playing basketball.");
    }

    @Override
    public void sing() {
        System.out.println("School B is sing 😊");
    }
}
```

现在，直接在测试类中使用`@Autowired`注入`Activity`：

```Java
@SpringBootTest(classes = PracticeDemoApplication.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class WhatQualifierDoTest {

    @Autowired
    private Activity activity;

    @Test
    public void test() {
        activity.ballGame();
        activity.sing();
    }
}
```

其实从 IDE 中，就可以看到`activity`下有警告信息，我们来截图看一下警告信息的内容：

![CleanShot 2021-05-12 at 16.20.33@2x](http://192.168.33.10:8899/test/2021/JUNE/THURSDAY/CleanShot%202021-05-12%20at%2016.20.33@2x.png)


警告信息上描述了：“无法注入。有超过一个的`Activity`类型“。下面推荐添加`@Qualifier`注解。

现在我们先忽略警告信息，直接运行该测试方法，观察结果会不会如上述所说，抛出`NoUniqueBeanDefinitionException`异常：

![CleanShot 2021-05-12 at 16.35.30@2x](http://192.168.33.10:8899/test/2021/JUNE/THURSDAY/CleanShot%202021-05-12%20at%2016.35.30@2x.png)


…… 真的是很明显很详细的异常信息呢！

下面我们在`@Autowired`下加上`@Qualifier("schoolB")`，再次运行测试方法：

![CleanShot 2021-05-12 at 16.39.16@2x](http://192.168.33.10:8899/test/2021/JUNE/THURSDAY/CleanShot%202021-05-12%20at%2016.39.16@2x.png)


非常流畅，顺利通过！

通过使用`@Qualifier`注解，可以将问题缩小到决定哪个`bean`需要被注入即可。

`@Qualifier`注解也可以直接用在实现类上，取代了直接在`@Component`组件上指定名称，这种写法可以达到相同的效果：

```Java
@Component
@Qualifier("schoolA")
public class SchoolA implements Activity {

    // ......
}
```

```Java
@Component
@Qualifier("schoolB")
public class SchoolB implements Activity {

     // ......
}
```

「 ❓🌰猜想： `@Component`注解如果不标识出名称的话，貌似会默认取当前类名，并把首字母小写。`@Service`注解也有相同的功能。 **--> 有待确认**」

# `@Qualifier` VS `@Primary`

当出现同类型的`bean`时，`@Primary`注解也能解决 Spring 无法决定注入哪个`bean`的问题。

「 此处盲猜下，见名猜意，primary 这个单词有“主要的”的意思，放在这个场景下，就相当于从多个`bean`中选了个默认值，当无法确定使用哪个`bean`时，就是用被`@Primary`注解标识为默认值的`bean`。」

> `@Primary` annotation defines a preference when multiple beans of same type are present.

还是刚才的🌰，现在既然`@Primary`给了我们偏心的机会，那我就选择偏向`schoolA`吧：

```Java
@Component
@Primary
public class SchoolA implements Activity {

    // ......
}
```

```Java
@Component
public class SchoolB implements Activity {

   // ......
}
```

执行测试方法：

```Java
@SpringBootTest(classes = PracticeDemoApplication.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class WhatQualifierDoTest {

    @Autowired
    private Activity activity;

    @Test
    public void test() {
        activity.ballGame();
        activity.sing();
    }
}
```

![CleanShot 2021-05-12 at 18.23.41@2x](http://192.168.33.10:8899/test/2021/JUNE/THURSDAY/CleanShot%202021-05-12%20at%2018.23.41@2x.png)

哇！！！掌声、鲜花、煎饼果子麻烦各来一套！！！

> `@Primary` annotation is useful when we want to specify which bean of a certain type should be injected by default.

那如果我现在想改为注入`SchoolB`的`bean`，难道还得去把`SchoolA`上的`@Primary`搬到`SchoolB`上么？

当然不需要。这时我们只需要在注入的代码上加回`@Qualifier`注解即可。

> It's worth nothing that if both the `@Primary` and `@Qualifier` annotations are present, then the `@Qualifier` annotation will have precedence.

「 😶友情提示： It's worth nothing than -> 值得注意的是」

综合来说，`@Primary`声明默认值，`@Qualifier`指定特殊值。

> Basically,`@Primary`defines a default, while `@Qualifier` is very specific.

# `@Qualifier` VS 使用名字自动注入

通过在注入语句中写名待注入的`bean`的名字，也可以明确注入的是哪个`bean`。


🌰 如下：

```Java
@SpringBootTest(classes = PracticeDemoApplication.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class WhatQualifierDoTest {

    @Autowired
    private Activity schoolB;

    @Test
    public void test() {
        schoolB.ballGame();
        schoolB.sing();
    }
}
```

「❓🌰猜想：所以在查找可注入的`bean`时，`@Autowired`注解是先根据类型查找，查找不到的话，再根据名称查找 **--> 有待确认**」

那么这种方式和`@Qualifier`注解的优先级，哪个更高呢？现在我们将测试类中的注入语句加上`@Qualifier`注解，并设置其`value`值为`schoolA`：

```Java
@SpringBootTest(classes = PracticeDemoApplication.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class WhatQualifierDoTest {

    @Autowired
    @Qualifier("schoolA")
    private Activity schoolB;

    // ......
}
```

结果如下：

![CleanShot 2021-05-12 at 18.48.16@2x](http://192.168.33.10:8899/test/2021/JUNE/THURSDAY/CleanShot%202021-05-12%20at%2018.48.16@2x.png)

结果显而易见，`@Qualifier`注解的优先级 > 使用名称自动注入。回到上面的猜想中，是不是可以确定，在查找可注入的`bean`时，Spring 是先根据类型查找，再根据名称查找的呢？！


「参考链接：https://www.baeldung.com/spring-qualifier-annotation」