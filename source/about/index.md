---
title: 关于我 | 曹飞 - 高级Java开发工程师 & 技术博客作者
date: 2020-03-15 14:22:21
layout: about
---

// Code tells the truth.

```java
public class AboutMe {
    public static void main(String[] args) {
        Person me = Person.builder()
                .name("曹飞")
                .role("Java Developer")
                .skills(Arrays.asList("Java", "Spring Cloud", "K8s", "MySQL", "Redis"))
                .passion("Building scalable & resilient systems")
                .currentFocus("云原生架构 & JVM深度优化")
                .hobby("Coding, 阅读, 徒步")
                .build();
        System.out.println("欢迎来到我的技术博客!");
    }
}
```
