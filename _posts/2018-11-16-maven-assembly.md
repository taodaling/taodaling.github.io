---
categories: tool
layout: post
---



# 基础插件信息

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
	<version>3.1.0</version>
</plugin>
```

# Descriptor

我们可以提供descriptor来指定具体的打包方式。

```xml
<configuration>
	<descriptors>
		<descriptor</descriptor>
	</descriptors>
</configuration>
```

下面我们描述内置的Descriptor：

- jar-with-dependencies: 打包依赖
- src: 打包源代码
- bin: 打包bin目录
- project: 打包整个项目

内置的descriptor需要使用特殊的标签来包含：

```xml
<configuration>
	<descriptorRefs>
		<descriptorRef>jar-with-dependencies</descriptorRef>
	</descriptorRefs>
</configuration>
```

你可以同时指定多个描述符：

```xml
<configuration>
	<descriptors>
		<descriptor</descriptor>
	</descriptors>
    	<descriptorRefs>
		<descriptorRef>jar-with-dependencies</descriptorRef>
	</descriptorRefs>
</configuration>
```

