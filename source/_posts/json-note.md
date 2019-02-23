---
layout:     post
title:      "浅析JSON"
date:       2019-02-23
author:     "Kido"
header-img: "json-note-bg01.jpg"
tags:
    - JSON

---

# 1.JSON介绍

**JSON**(JavaScript Object Notation) 是一种轻量级的数据交换格式。 易于人阅读和编写。同时也易于机器解析和生成。 它基于[JavaScript Programming Language](http://www.crockford.com/javascript "JavaScript Programming Language"), [Standard ECMA-262 3rd Edition - December 1999](http://www.ecma-international.org/publications/files/ecma-st/ECMA-262.pdf "Standard ECMA-262 3rd Edition - December 1999")的一个子集。 JSON采用完全独立于语言的文本格式。 这些特性使JSON成为理想的数据交换语言。

JSON建构于两种结构：

*   “名称/值”对的集合（A collection of name/value pairs）。不同的语言中，它被理解为_对象（object）_，纪录（record），结构（struct），字典（dictionary），哈希表（hash table），有键列表（keyed list），或者关联数组 （associative array）。

*   值的有序列表（An ordered list of values）。在大部分语言中，它被理解为数组（array）。

# 2. 三大JSON类库比较

## 2.1 Jackson 分析

#### 1. 依赖

三个jar包

#### 2. 序列化速度

三者NO.2 [ (From: jvm-serializers)](https://github.com/eishay/jvm-serializers/wiki)

#### 3. 反序列化速度

三者NO.2 [ (From: jvm-serializers)](https://github.com/eishay/jvm-serializers/wiki)

#### 4. 方案缺点分析

*   Jackson的包大小和启动开销在某些领域（如移动电话）可能存在问题，特别是对于轻量使用的情况下（PS: jackson的core+databind+annotations需要超过1.5M，太臃肿）。
    （出于所有这些原因，jackson决定创建一个更简单、更小的库，它支持一个称为Jackson Jr的功能子集。它构建在Jackson流式API上，但不依赖于数据绑定。因此，它的JAR和运行时内存使用量都要小得多。）

#### 5. 方案优点分析

*   较高的准确性和兼容性
*   jackson 的强项是灵活可定制, 并且具有了一个生态
*   Jackson的包大小和启动开销在某些领域（如移动电话）可能存在问题，特别是对于轻量使用的情况下（PS: jackson的core+databind+annotations需要超过1.5M，太臃肿）。
   （出于所有这些原因，jackson决定创建一个更简单、更小的库，一个称为Jackson Jr的功能子集。它构建在Jackson流式API上，但不依赖于数据绑定。因此，它的JAR和运行时内存使用量都要小得多。）

## 2.2 FastJSON 分析

#### 1. 依赖

单个jar

#### 2. 序列化速度

三者NO.1 [ (From: jvm-serializers)](https://github.com/eishay/jvm-serializers/wiki)

#### 3. 反序列化速度

三者NO.1 [ (From: jvm-serializers)](https://github.com/eishay/jvm-serializers/wiki)

#### 4. 方案缺点分析

*   兼容性一般:[Alibaba/fastjson/已知影响兼容的变更](https://github.com/Alibaba/fastjson/wiki/incompatible_change_list "Alibaba/fastjson/已知影响兼容的变更")

*   其解析json主要是用的String类substring这个方法[fastjson/JSONScanner.java at master · alibaba/fastjson · GitHub](https://link.zhihu.com/?target=https%3A//github.com/alibaba/fastjson/blob/master/src/main/java/com/alibaba/fastjson/parser/JSONScanner.java "fastjson/JSONScanner.java at master · alibaba/fastjson · GitHub")，所以解析起来非常“快”，因为申请内存次数很少。但是因为jdk1.7之前substring的实现并没有new一个新对象，在使用的时候，如果解析的json非常多，稍不注意就会出现内存泄漏（比如一个40K的json，你在对象里引用了里边的一个key，即使这个key只有2字节，也会导致这40K的json无法被垃圾回收器回收），这也是“快”带来的负面效果。而且这还不算，在jdk1.7以上版本对string的substring方法做了改写，改成了重新new一个string的方式，于是这个“快”的优势也不存在了。

*   翻阅fastjson的源码，你会发现有很多写死的代码，比如：针对spring之类的框架的各种处理，都是用classload判断是否存在这种类名[fastjson/SerializeConfig.java at master · alibaba/fastjson · GitHub](https://link.zhihu.com/?target=https%3A//github.com/alibaba/fastjson/blob/master/src/main/java/com/alibaba/fastjson/serializer/SerializeConfig.java "fastjson/SerializeConfig.java at master · alibaba/fastjson · GitHub")；那么这是什么意思呢？意思就是如果你用spring的那种思想，自己写了个类似的功能，因为你这个项目里没有spring的那个类，那么用起来就有一堆bug；当然不仅限于这些，还有很多，比如ASM字节码织入部分[fastjson/ASMSerializerFactory.java at master · alibaba/fastjson · GitHub](https://link.zhihu.com/?target=https%3A//github.com/alibaba/fastjson/blob/master/src/main/java/com/alibaba/fastjson/serializer/ASMSerializerFactory.java "fastjson/ASMSerializerFactory.java at master · alibaba/fastjson · GitHub")（温少的ASM方面水平是个半吊子），看源码的话，能发现的缺点数不胜数。

*   属性里分别有包含_（下划线开头、#开头）之类的属性，序列化为json时，出现属性丢失，1.2.14后修复了这个bug（[fastjson-1.2.14 修复BUG功能增强](https://github.com/alibaba/fastjson/releases/tag/1.2.14 "fastjson-1.2.14 修复BUG功能增强")）

#### 5. 方案优点分析

*   轻量级，依赖少

*   稳定性也还好，大规模使用，3441个回归测试，92%测试覆盖率( [Dashboard ⋅ alibaba/fastjson](https://link.zhihu.com/?target=https%3A//codecov.io/gh/alibaba/fastjson/branch/master "Dashboard ⋅ alibaba/fastjson"))

*   目前在Oracle HotSpot JVM上使用标准版本速度还是很快的，在[Home · eishay/jvm-serializers Wiki · GitHub](https://link.zhihu.com/?target=https%3A//github.com/eishay/jvm-serializers/wiki "Home · eishay/jvm-serializers Wiki · GitHub")测试中，fastjson各类json databind中最快的，这个快也能得到一些其他作者的承认的，比如jackson的作者tatu。fastjson-1.1.52.android版本，在android 5/6上性能都很好的，比原生org.json/gson/jackson性能都好很多

*   fastjson也有一些jackson不具备的功能，比如JSONPath ( [JSONPath · alibaba/fastjson Wiki · GitHub](https://link.zhihu.com/?target=https%3A//github.com/alibaba/fastjson/wiki/JSONPath "JSONPath · alibaba/fastjson Wiki · GitHub")) 的支持

## 2.3 GSON 分析

#### 1. 依赖

单个jar

#### 2. 序列化速度

三者NO.3 [ (From: jvm-serializers)](https://github.com/eishay/jvm-serializers/wiki)

#### 3. 反序列化速度

三者NO.3 [ (From: jvm-serializers)](https://github.com/eishay/jvm-serializers/wiki)

#### 4. 方案缺点分析

*   性能略慢于其余两项，不过差距也还好

#### 5. 方案优点分析

*   较高的准确性和兼容性，支持任意复杂的对象（具有深度继承层次结构和广泛使用泛型类型）
*   轻量级，依赖少
*   API 简单易用

# 3.JSON的一些经验

> From: [ JSON最佳实践](http://kimmking.github.io/2017/06/06/json-best-practice/)

#### **3.1 遵循Java Beans规范与JSON规范**

实践告诉我们：遵循beans规范和JSON规范的方式，能减少大部分的问题，比如正确实现setter、getter，用别名就加annotation。注意基本类型的匹配转换，比如在fastjson的issue见到试图把”{“a”:{}}”中的a转换成List的。

#### **3.2 使用正常的key**

尽量不要使用数字等字符开头的key，尽量使用符合Java的class或property命名规范的key，这样会减少不必要的冲突。在jsonpath或js里，a.1可能会被解释成a[1]或a[“1”]，这些都会带来不必要的麻烦。

#### **3.3 关于日期处理**

这一点前面的Google JSON风格指南里也提到了，尽量使用标准的日期格式。或者序列化和反序列化里都是用同样的datePattern格式。

#### **3.4 自定义序列化与反序列化**

对于新手来说，自定义序列化是一切罪恶的根源。
尽量不要使用自定义序列化，除非万不得已，优先考虑使用注解过滤，别名等方式，甚至是重新建一个VO类来组装实际需要的属性。使用自定义序列化时一切要小心，因为这样会导致两个问题：
*   改变了pojo <-> jsonstring 的自然对应关系，从而不利于阅读代码和排查问题，你改变的关系无法简单的从bean和json上看出来了；
*   反序列化可能出错，因为对应不上原来的属性了。
如果只是序列化发出去（响应）的是JSON数据、传过来（请求）的数据格式跟JSON无关或者是标准的，此时自定义序列化就无所谓了，反正是要接收方来处理。

#### **3.5 JSONObject的使用**

JSONObject是JSON字符串与pojo对象转换过程中的中间表达类型，实现了Map接口，可以看做是一个模拟JSON对象键值对再加上多层嵌套的数据集合，对象的每一个基本类型属性是map里的一个key-value，一个非基本类型属性是一个嵌套的JSONObject对象（key是属性名称，value是表示这个属性值的对象的JSONObject）。如果以前用过apache beanutils里的DynamicBean之类的，就知道JSONObject也是一种动态描述Bean的实现，相当于是拆解了Bean本身的结构与数据。这时候由于JSONObject里可能会没有记录全部的Bean类型数据，例如泛型的具体子类型之类的元数据，如果JSONObject与正常的POJO混用，出现问题的概率较高。

下列方式尽量不要使用：

```java
public class TestBean{
    @Setter @Getter
    private TestBean1 testBean1;

    @Setter @Getter
    private JSONObject testBean2; // 尽量不要在POJO里用JSONObject
}
```

应该从设计上改为都用POJO比较合适:

```java
public class TestBean{
    @Setter @Getter
    private TestBean1 testBean1;

    @Setter @Getter
    private TestBean2 testBean2;; // 使用POJO
}
```

相对的，写一些临时性的测试代码，demo代码，可以直接全部用JSONObject先快速run起来。
同理，jsonstring中嵌套jsonstring也尽量不要用，例如：

```javascript
{
    "name":"zhangsan",
    "score":"{\"math\":78,\"history\":82}"
}
```

应该改为全部都是JSON风格的结构：

```javascript
{
    "name":"zhangsan",
    "score":{
        "math":78,
        "history":82
    }
}
```

另外，对于jsonstring转POJO（或POJO转jsonstring），尽量使用直接转的方式，而不是先转成JSONObject过渡的方式。特别是对于Fastjson，由于性能优化的考虑，这两个执行的代码是不一样的，可能导致不一样的结果。

```java
String jsonstring = "{\"a\":12}";

// 不推荐这种方式
// 除非这里需要对jsonObject做一些简单处理
JSONObject jsonObject = JSON.parseObject(jsonstring);
A a = jsonObject.toJavaObject(A.class);

// 推荐方式
A a = JSON.parseObject(jsonstring, A.class);

```

#### **3.6 Hibernate相关问题**

懒加载与级联，可能导致出现问题，例如hibernate，建议封装一层VO类型来序列化。使用VO类还有一个好处，就是可以去掉一些没用的属性，减少数据量，同时可以加上额外的属性。

#### **3.7 深层嵌套与泛型问题**

尽量不要在使用过多的层次嵌套的同时使用泛型（List、Map等），可能导致类型丢失，而且问题比较难查。

#### **3.8 抽象类型与子类型问题**

尽量不要在同一个Bean的层次结构里使用多个子类型对象，可能导致类型丢失，而且问题比较难查。当然我们可以通过代码显示的传递各种正确的类型，但是这样做引入了更多的不确定性。良好的做法应该是一开始设计时就避免出现这些问题。

#### **3.9 避免循环引用**

尽量避免循环引用，这个虽然可以通过序列化特性禁掉，但是如果能避免则避免。

#### **3.10 注意编码和不可见字符**

对于InputStream、OutputStream的处理，有时候会报一些奇怪的错误，not match之类的，这时候也许我们看日志里的json字符串可能很正常，但就是出错。
这时可能就是编码的问题了，可能是导致字符错乱，也可能是因为UTF-8文件的BOM头，这些潜在的问题可能在二进制数据转文本的时候，因为一些不可见字符无法显示，导致日志看起来只有正常字符而是正确的，问题很难排查。
处理办法就是按二进制的方式把Stream保存起来，然后按hex方式查看，看看是否有多余字符，或者其他错误。


# 4.参考文档
*   [JSON官网](http://www.json.org/json-zh.html "JSON官网")
*   [https://github.com/google/gson](https://github.com/google/gson)
*   [https://github.com/FasterXML/jackson](https://github.com/FasterXML/jackson)
*   [https://github.com/alibaba/fastjson](https://github.com/alibaba/fastjson)
*   [eishay/jvm-serializers JVM序列化测评](https://link.zhihu.com/?target=https%3A//github.com/eishay/jvm-serializers/wiki)
*   [知乎：fastjson这么快老外为啥还 jackson?](https://www.zhihu.com/question/44199956)
*   [Google-JSON风格指南](https://github.com/darcyliu/google-styleguide/blob/master/JSONStyleGuide.md)
*   [JSON最佳实践](http://kimmking.github.io/2017/06/06/json-best-practice/)
*   [你真的会用Gson吗?Gson使用指南](https://www.jianshu.com/p/e740196225a4)