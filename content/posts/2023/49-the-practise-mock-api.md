---
title: 关于接口 mock 的技术实现
date: 2023-09-11 20:50:00
tags: [OpenAPI, JSON Schema, mockjs]
---

最近在做接口 mock 的功能，用到了一些新的技术，主要有 OpenAPI 规范、JsonSchema 和 mockjs。这里简单的记录下技术方案，以及过程中曾踩到的坑。

# 原理
整个的原理比较简单，首先 OpenAPI 定义了一套接口规范，按照这个规范我们可以准确的描述一个接口，比如接口的方法类型、出入参格式和内容描述。这是 OpenAPI 规范中文版的[文档地址](https://openapi.apifox.cn/)。Apifox、Postman 等接口管理工具对接口的定义也是以这个规范进行存储的。我们可以通过这个[地址](https://editor.swagger.io/)很直观的观察一份 OpenAPI 规范的接口数据是什么样的。

当我们拥有了接口定义之后，我们就知道了接口的返回体的结构，大部分的服务接口通常将数据以 JSON 的格式进行返回，而描述这个 JSON 格式的就是 JSON Schema。JSON Schema 是 JSON 数据的模式描述语言，用来定义 JSON 数据结构和格式约束。JSON Schema提供了一套验证 JSON 数据有效性的规则集。这是 JSON Schema 规范中文版的[文档地址](https://json-schema.apifox.cn/)。如果你仔细看的话，就会发现 OpenAPI 规范里面对 JSON 类型的请求体或返回体的就是用 JSON Schema 进行描述的。

最后要做的事情，就是解析 JSON Schema 的定义，将其转换成 mockjs 的模板，让 mockjs 能够 mock 出对应的 JSON。当然我们 Java 微服务要使用 mockjs，需要使用到 ScriptEngine。

# 踩坑
## OpenAPI 扩展字段的存储
为了前端能够直接使用 mockjs 的预制变量，我们对接口定义增加了一个名为 mock 的属性，用以存储预制变量。后面联调的时候发现 mock 属性虽然能够转换到 OpenAPI 格式中，但是读取不出来。最后排查是因为 OpenAPI 的扩展字段需要增加 x- 前缀，且通过 OpenAPIV3Parser.java 解析出来的时候，结构会有变化，不会放在存储的层级结构下，而是放在 extensions 属性字段下面。

## mockjs 无法直接 mock JSON 数组
这个其实是 mockjs 一直没有支持的功能，Github 上的 [Issue 链接](https://github.com/nuysoft/Mock/issues/6)已明确了不会支持。因为这个 feature 不支持，最终我们不得不在项目代码里面做一些兼容，因为我们需要 mockjs 能够动态生成 JSON 数组里面元素个数的能力。最后的 mock JSON 数组，其实是 mock 一个带属性的 JSON 对象，这个属性值就是实际需要的 JSON 数组，最后将这个提取这个属性的值进行返回。

# 总结
做接口 mock 大体是一件比较有意思的意思。当大家在系统中定义好接口之后，使用者可以直接调用 mock 服务的地址，再加上他们的接口 url，就可以得到他们想要的内容。在一个比较大的公司里面，有这样统一工具对大家工作上面的对接效率提升也是比较大的，不仅体现在前后短联调，也体现在微服务之间的相互调用。