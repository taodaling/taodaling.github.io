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

# 文件配置

```xml
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
	
  <id>release</id> <!--指定打包后缀-->
  <formats> <!--打包格式，可以指定多个-->
    <format>tar.gz</format>
    <format>dir</format>
  </formats>

  <dependencySets> <!--依赖管理-->
    <dependencySet>
      <outputDirectory>/lib</outputDirectory> <!--输出到压缩包的/lib目录下-->
    </dependencySet>
  </dependencySets>

  <fileSets>
    <fileSet>
      <includes>
        <include>bin/**</include>
      </includes>
      <fileMode>0755</fileMode>
    </fileSet>

    <fileSet>
      <includes>
        <include>/conf/**</include>
        <include>logs</include>
      </includes>
    </fileSet>

  </fileSets>

  <files>
    <file>
      <source>README.txt</source>
      <outputDirectory>/</outputDirectory>
    </file>
  </files>

</assembly>
```

## include和exclude

include和exclude用于指定包含文件和排除文件，当一个文件满足所有的include条件和不满足exclude条件下，才会被包含。

在使用depedencySet和moduleSet的时候，include和exclude使用artifacts而非文件名，这样就可以避免我们需要得知artifact在本地的实际存储名字。格式必须为以下三种之一：

- groupId:artifactId:type:classifier
- groupId:artifactId
- groupId:artifactId:type:classifier:version

其中type默认为jar，而classifier在为空时会被忽略。

如果在格式中包含了通配符，那么将会分别计算文件的三种格式的信息，如果任意其一包含了剩余的子串，那么将认为匹配。

比如要匹配所有的war文件，则可以使用\*:war:\*。

如果没有指定通配符，那么将会拿模式串与三种格式做等值比较。

## fileSet

fileSet用于包含文件，由于文件中可能会出现一些元文件（比如.git等），assembly会默认帮我们将其剔除，但是如果希望包含它们，我们需要先指定useDefaultExcludes为false。

```xml
<fileSets>
    <fileSet>
      <useDefaultExcludes>false</useDefaultExcludes>
      <excludes>
        <exclude>**/target/**</exclude>
      </excludes>
    </fileSet>
  </fileSets>
```

fileSet拥有两个重要属性，outputDirectory和directory。两者默认值均为`/`(输出目录的根目录对应`输出文件名-${配置的id}`，输入目录的默认值为`${basedir}`)。前者表示输出目录，后者表示来源目录，看下面例子：

```xml
<fileSets>
    <fileSet>
        <outputDirectory>/conf</outputDirectory>
        <directory>/</directory>
        <includes>
        	<include>/src/main/conf/*</include>
        </includes>
    </fileSet>
</fileSets>
```

上面会在target下建立`输出文件名-${配置的id}/输出文件名/conf/src/main/conf`。而如果指定了directory：

```xml
<fileSets>
    <fileSet>
        <outputDirectory>/conf</outputDirectory>
        <directory>/src/main/conf</directory>
        <includes>
        	<include>*</include>
        </includes>
    </fileSet>
</fileSets>
```

则会将/src/main/conf下的所有文件（仅一级）拷贝到`输出文件名-${配置的id}/输出文件名/conf`下。

## dependencySets

dependencySets用于操作依赖。

```xml
<dependencySets>
    <dependencySet>
        <outputDirectory>/lib</outputDirectory>
        <scope>runtime</scope> <!--仅包含runtime所需包-->
        <fileMode>0444</fileMode>
    </dependencySet>
</dependencySets>
```

- fileMode：指定文件的权限，由4位数字决定，第二位为当前用户权限，第三位为同组用户权限，第四位为其他用户权限。

# Archive

Assembly插件提供了archive元素，该元素由maven-archiver处理。只有jar和war格式允许由archive元素。

## manifest

打包的时候可以指定manifest文件中的包含内容。

```xml
<configuration>
    <archive>
        <manifest>
        	<mainClass>org.sample.App</mainClass>
        </manifest>
    </archive>
</configuration>
```

# 附录

详细文件配置模板：

[模板地址](https://maven.apache.org/plugins/maven-assembly-plugin/assembly.html)