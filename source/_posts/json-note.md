---
layout:     post
title:      "浅谈JSON类库"
date:       2018-11-23
author:     "Kido"
header-img: "json-note-bg01.jpg"
tags:
    - JSON

---

# 1.JSON介绍

**JSON**(JavaScript Object Notation) 是一种**轻量级的数据交换格式**，易于人阅读和编写，同时也易于机器解析和生成，它基于[JavaScript Programming Language](http://www.crockford.com/javascript "JavaScript Programming Language"), [Standard ECMA-262 3rd Edition - December 1999](http://www.ecma-international.org/publications/files/ecma-st/ECMA-262.pdf "Standard ECMA-262 3rd Edition - December 1999")的一个子集。 **JSON采用完全独立于语言的文本格式**， **这些特性使JSON成为理想的数据交换语言**。


**JSON值可以是：**

*   数字（整数或浮点数）
*   字符串（在双引号中）
*   逻辑值（true 或 false）
*   数组（在方括号中）
*   对象（在花括号中）
*   null

**范例：**

```javascript
{
  "name": "小强",
  "age": 17,
  "gender": true,
  "height": 1.77,
  "grade": null,
  "middle-school": "\"W3C\" Middle School",
  "skills": [
    "JavaScript",
    "Java",
    "Python",
    "Lisp"
  ]
}
```

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
*   Jackson也发生过少量的安全漏洞，比如：[jackson-databind 反序列化漏洞](https://www.openwall.com/lists/oss-security/2017/11/02/3)

#### 5. 方案优点分析

*   较高的准确性和兼容性
*   jackson 的强项是灵活可定制, 并且具有了一个生态
*   支持非常多JSON以外的数据格式，比如：[Avro](https://github.com/FasterXML/jackson-dataformat-avro "Avro")、[CBOR](https://github.com/FasterXML/jackson-dataformat-cbor "CBOR")、[CSV](https://github.com/FasterXML/jackson-dataformat-csv "CSV")、[Ion](https://github.com/FasterXML/jackson-dataformats-binary/tree/master/ion "Ion")、[(Java) Properties](https://github.com/FasterXML/jackson-dataformat-properties "(Java) Properties") 、[Protobuf](https://github.com/FasterXML/jackson-dataformat-protobuf "Protobuf") 、[Smile](https://github.com/FasterXML/jackson-dataformat-smile "Smile")、[XML](https://github.com/FasterXML/jackson-dataformat-xml "XML")、[YAML](https://github.com/FasterXML/jackson-dataformat-yaml "YAML")等

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

*   翻阅fastjson的源码，你会发现有很多写死的代码，比如：针对spring之类的框架的各种处理，都是用classload判断是否存在这种类名[fastjson/SerializeConfig.java at master · alibaba/fastjson · GitHub](https://link.zhihu.com/?target=https%3A//github.com/alibaba/fastjson/blob/master/src/main/java/com/alibaba/fastjson/serializer/SerializeConfig.java "fastjson/SerializeConfig.java at master · alibaba/fastjson · GitHub")；那么这是什么意思呢？意思就是如果你用spring的那种思想，自己写了个类似的功能，因为你这个项目里没有spring的那个类，那么用起来就有一堆bug；当然不仅限于这些，还有很多，比如ASM字节码织入部分[fastjson/ASMSerializerFactory.java at master · alibaba/fastjson · GitHub](https://link.zhihu.com/?target=https%3A//github.com/alibaba/fastjson/blob/master/src/main/java/com/alibaba/fastjson/serializer/ASMSerializerFactory.java "fastjson/ASMSerializerFactory.java at master · alibaba/fastjson · GitHub")（温少的ASM方面水平一般），看源码的话，能发现的缺点数不胜数。

*   属性里分别有包含_（下划线开头、#开头）之类的属性，序列化为json时，出现属性丢失，1.2.14后修复了这个bug（[fastjson-1.2.14 修复BUG功能增强](https://github.com/alibaba/fastjson/releases/tag/1.2.14 "fastjson-1.2.14 修复BUG功能增强")）

*   发生了多次安全漏洞, 比如：[Fastjson远程代码执行漏洞](https://github.com/alibaba/fastjson/wiki/incompatible_change_list)、[Fastjson拒绝服务漏洞](https://github.com/alibaba/fastjson/wiki/1_2_60_incompatible)

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
*   暂没有发现严重的安全漏洞（截止:2019-09-24）

## 附

> From: [ JSON最佳实践](http://kimmking.github.io/2017/06/06/json-best-practice/)


# 3.参考文档
*   [JSON官网](http://www.json.org/json-zh.html "JSON官网")
*   [https://github.com/google/gson](https://github.com/google/gson)
*   [https://github.com/FasterXML/jackson](https://github.com/FasterXML/jackson)
*   [https://github.com/alibaba/fastjson](https://github.com/alibaba/fastjson)
*   [eishay/jvm-serializers JVM序列化测评](https://link.zhihu.com/?target=https%3A//github.com/eishay/jvm-serializers/wiki)
*   [知乎：fastjson这么快老外为啥还 jackson?](https://www.zhihu.com/question/44199956)
*   [Google-JSON风格指南](https://github.com/darcyliu/google-styleguide/blob/master/JSONStyleGuide.md)
*   [JSON最佳实践](http://kimmking.github.io/2017/06/06/json-best-practice/)
*   [你真的会用Gson吗?Gson使用指南](https://www.jianshu.com/p/e740196225a4)