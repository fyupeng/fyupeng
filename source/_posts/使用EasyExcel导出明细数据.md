---
title: 使用EasyExcel导出明细数据
top: false
cover: false
toc: true
mathjax: false
date: 2026-03-05 15:12:18
author: fyupeng
img:
coverImg:
password:
summary: 使用阿里巴巴开源轻量工具EasyExcel导出Excel明细数据
tags:
- MongoDB
- 数据库
categories:
- Java笔记

---

> 作者：**fyupeng**
> 技术专栏：☞ [https://github.com/fyupeng](https://github.com/fyupeng)
> 分布式博客项目地址：☞ [https://github.com/fyupeng/distributed-blog-system-api](https://github.com/fyupeng/distributed-blog-system-api)
---

> 留给读者

工作中难免会遇到数据导出的问题，最简单的解决方案就是引入阿里的`EasyExcel`来解决。

## 一、介绍

EasyExcel是alibaba开源的一个Excel工具，容易上手，基本能够满足企业的需求。

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>2.2.6</version>
</dependency>
```
## 二、代码

### 1. 实体注解
```java
@NoArgsConstructor
@AllArgsConstructor
@ToString
@Data
@Builder(toBuilder = true)
public class Entity implements Serializable {
	// 导出value对应表格头，index对应排序（从0下标开始）
	@ExcelProperties(value = "编码", index = 0)
	private String code;

	@ExcelProperties(value = "名称", index = 1)
	private String name;
}
```

### 2. 核心代码
```java
public void export(HttpServletRequest request, HttpServletResponse response) {
	String fileName = "导出明细.xlsx";
	OutputStream out = null;
	try {
		response.setContentType("application/octet-stream;charset=UTF-8");
		response.setHeader("Content-Type", "application/vnd.ms-excel");
		fileName = URLEncoder.encode(fileName, "UTF-8");
		response.setHeader("Content-Disposition", "attachment;filename=", fileName);
		response.addHeader("Cache-Control", "no-cache");
		out = response.getOutputStream();
		
		// 创建 Exce 文件写入流
		ExcelWriter excelWriter = EasyExcel.write(out).build();
		
		// 新增 Sheet 表格
		WriteSheet sheet = EasyExcel.writeSheet(0, "基本信息").head(Entity.class).build();
		List<Entity> list = new ArrayList<>();
		excelWriter.write(list, sheet);
	
		// 关闭 excel 写入流
		excelWriter.finish();
		// 内存刷入响应流并清空内存
		response.flushBuffer();
	} catch (IOException e) {
		throw new Exception("明细导出失败！");
	}
}

```
## 三、总结

每日总结归档，学习会遗忘，但学习后总结不会，它能让你变得越来越强！