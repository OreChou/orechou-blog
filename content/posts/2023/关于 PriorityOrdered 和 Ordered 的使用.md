---
title: 关于 PriorityOrdered 和 Ordered 的使用
date: 2023-02-09 23:00:00
tags: [Spring, Java]
---

## 背景
在调试某个 Springboot 项目业务代码的时候发现了一个现象：Bean 通过实现 Ordered 或 PriorityOrdered 接口来定义优先级，然后用 AnnotationAwareOrderComparator 对些 Bean 进行排序的时候，其顺序并没有按照 order 的大小进行排序。相关的代码如下：

```java
public class A implements PriorityOrdered {
	
    @Override  
    public int getOrder() {  
        return 0;  
    }
	
}

public class B implements Ordered {
    @Override  
    public int getOrder() {  
        return -1;  
    }
}

public class C implements Ordered {

    @Override  
    public int getOrder() {  
        return -2;  
    }
	
}

public class Test {

    public void sort() {
        List<Object> list = new ArrayList();
        list.add(new A());
        list.add(new B());
        list.add(new C());

        // 排序的结果为: A->C->B，而不是第一反应认为的 C->B->A
        AnnotationAwareOrderComparator.sort(list);
    }
	
}
```

## 原因
跟一下源码可以很快查到原因，AnnotationAwareOrderComparator 是继承的 OrderComparator，而 OrderComparator 的比较是这样的：
```Java
@Override  
public int compare(@Nullable Object o1, @Nullable Object o2) {  
    return doCompare(o1, o2, null);  
}  
  
private int doCompare(@Nullable Object o1, @Nullable Object o2, @Nullable OrderSourceProvider sourceProvider) {
    // 当比较 PriorityOrdered 和 Ordered 的时候，默认 PriorityOrdered 的优先级是比 Ordered 高的，不会去看设置的 order 数值。
    boolean p1 = (o1 instanceof PriorityOrdered);  
    boolean p2 = (o2 instanceof PriorityOrdered);  
    if (p1 && !p2) {  
        return -1;  
    }  
    else if (p2 && !p1) {  
        return 1;  
    }
    int i1 = getOrder(o1, sourceProvider);  
    int i2 = getOrder(o2, sourceProvider);  
    return Integer.compare(i1, i2);  
}
```

其实 Spring 的文档里面也说得很清楚：

> Extension of the [`Ordered`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/Ordered.html "interface in org.springframework.core") interface, expressing a _priority_ ordering: `PriorityOrdered` objects are always applied before _plain_ [`Ordered`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/Ordered.html "interface in org.springframework.core") objects regardless of their order values. 
When sorting a set of `Ordered` objects, `PriorityOrdered` objects and _plain_ `Ordered` objects are effectively treated as two separate subsets, with the set of `PriorityOrdered` objects preceding the set of _plain_ `Ordered` objects and with relative ordering applied within those subsets.

另外使用 ChatGPT ，它也可以很好的回答这个问题：
![](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/chatgpt_answer_priorityordered_ordered.png)

## 启发
如果我们要让某个 Bean 具有最高优先级，可以通过给这个 Bean 设置 PriorityOrdered，而其它的 Bean 设置 Ordered。这样即使这个 Bean 的 order 数值很小，也不会被其他的 Bean 所影响。