---
layout: page
title: BeanUtils踩坑记录
permalink: /BeanUtils_log
date: 2021-07-11 18:18:18 -0000
categories: spring java
---

# BeanUtils 踩坑记录

## 呜呼哀哉

本来想吐槽：“哎 spring 出品，就这？”，后来...... 我是白嫖的，我不配吐槽，哈哈哈哈！

## 结论

1. BeanUtils 不支持范型 copy 
2. BeanUtils 是 浅拷贝 
3. 对象赋值使用MapStruct

## 解决方案

```
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.4.2.Final</version>
</dependency>
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct-processor</artifactId>
  <version>1.4.2.Final</version>
</dependency>
```

要酷，不要解释

## 前提

1. org.springframework.beans.BeanUtils 包下 BeanUtils
2. org.springframework.beans.BeanUtils 是 org.springframework:spring-beans:4.3.25.RELEASE

## 问题复现方式

```
public class CopyTest {

    @Setter
    @Getter
    public static class A {
        private String value;
        private List<AList> listValue;
    }

    @Setter
    @Getter
    public static class AList {
        private String listName;
    }

    // ---------------分隔符---------------------

    @Setter
    @Getter
    public static class B {
        private String value;
        private List<BList> listValue;

    }

    @Setter
    @Getter
    private static class BList {
        private String listName;
    }


    // --------------验证代码-------------------
    public static void main(String[] args) {
        AList aList = new AList();
        aList.setListName("abcd");
        A a = new A();
        a.setValue("a's father");
        a.setListValue(Collections.singletonList(aList));

        B b = new B();
        List<BList> listValue = new ArrayList<>();
        b.setListValue(listValue);
        BeanUtils.copyProperties(a, b);
        System.out.println(JSON.toJSONString(b));

        aList.setListName("gedc");
        System.out.println(JSON.toJSONString(b));
    }

}
```

在上述 45 行  debug ，会发现 b 对象的 listValue 属性 是list，但是list中的数据类型是 AList类 而非 BList类！

## 溯源

1. 对 BeanUtils 进行源码分析，看到核心代码如下：
   ![beanUtils-copy](/Never12581/bocl/blob/gh-pages/picture/20210711/beanUtils-copy.jpg?raw=true)
   因如上图所示，就本质而言，BeanUtils 的拷贝是 指针复制的浅拷贝。浅拷贝与深拷贝的差异不进行赘述，也是一个容易造坑的点！
   当然，仅到这里无法解释我们上述问题，如何将 List<BList> 对象 赋值为 List<AList> 对象，请往下看反射部分。
2. 直接上图
   ![reflect](/Never12581/bocl/blob/gh-pages/picture/20210711/reflect.png?raw=true)
   从图中可以看到，通过反射进行方法获取时，需要传入该方法的参数类型。
   结合我们的例子 B#setListValue(List<BList>) 分析，
   在获取该 method 时，传入的类型是 List.class（不支持范型），故在 invoke 的时候，传入List<AList> ，满足类型校验，调用成功！！！所以得到结果：复制以后的 b 对象的 listValue 属性中的 类型是AList！！！