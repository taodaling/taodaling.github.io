---
categories: format
layout: post
---

- Table
{:toc}

# 数据类型

## 列表

```yaml
- Mark McGwire
- Sammy Sosa
- Ken Griffey
```

## 对象

```yaml
name: Mark McGwire
hr:   65
avg:  0.278
```

如果对象的关键字特别复杂（比如说是一个列表），那么可以用?字符进行标注：

```yaml
?
 - cat
 - dog
: 
 - young
 - old
```

## 字符串

yaml支持多行字符串，此时你需要以"|"或">"作为字符串字面量的第一行，二者区别在于"|"认为字符串字面量中的换行对应一个字符串中的换行，而">"认为字符串字面量的换行对应字符串中的一个空格符，但是一个空行被视作一个换行符。

```yaml
---
|
Hello,
Do you know
who I am!
---
"Hello,\n Do you know \nwho I am!"
```

```yaml
---
>
Hello,

Do you know 
who I am!
---
'Hello,\nDo you know who I am!'
```

出了"|"和">"外，还存在对应的"|-"和">-"，它们的作用仅是在原来的基础上删除尾部的换行符号：

```yaml
---
code: >
      1
      2
---
code: '1 2\n'
```

```yaml
---
code: >-
      1
      2
---
code: '1 2'
```

## 注释

```yaml
- Mark McGwire #line 1
- Sammy Sosa #line 2
- Ken Griffey #line 3
```

## 引用

我们可以引用某个值，并在解析时自动替换。

```yaml
a: &type cat
b: *type
```

上面内容等价于：

```yaml
a: cat
b: cat
```

## 类型推断

在YAML中，数据类型会被自动推断：

```yaml
canonical: 
 - 12345
 - 1.2345e+5
decimal: 
 - +12345
octal: 0o14
hexadecimal: 0xC
negative infinity: -.inf
not a number: .NaN
null: 
booleans: [true, false]
string: '12345'
spaced: 2001-12-14 21:59:43.10 -8
date: 2002-12-14
```

出了隐式推断外，你还可以显式指定类型，用法就是!符号：

```yaml
not-date: !!str 2002-04-28

picture: !!binary |
 R0lGODlhDAAMAIQAAP//9/X
 17unp5WZmZgAAAOfn515eXv
 Pz7Y6OjuDg4J+fn5OTk6enp
 56enmleECcgggoBADs=
```



# JAVA编程

## 依赖

```xml
<dependency>
	<groupId>com.fasterxml.jackson.dataformat</groupId>
	<artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```

## 解码

yaml文件:

```yml
age: 18
name: son
contact: 
    mobile: 123456
    email: 123456@123.com
roles:
    - "son"
    - "student"
```

实体类:

```java
@Data
class Contact {
	private String mobile;
	private String email;
}
@Data
class User {
	private int age;
	private String name;
	private Contact contact;
	private List<String> roles;
}
```

java代码：

```java
User user = new ObjectMapper(new YAMLFactory()).readValue(new File(/*location*/), User.class);
```

## 多文档解码

```yaml
---
age: 18
name: son
contact: 
    mobile: 123456
    email: 123456@123.com
roles:
    - "son"
    - "student"
...
---
age: 38
name: father
contact: 
    mobile: 123456
    email: 123456@123.com
roles:
    - "father"
    - "worker"
...
```

java代码：

```java
MappingIterator<User> iterator = new ObjectMapper(new YAMLFactory()).readerFor(User.class).readValues(new File("D:/TEMP/temp.yml"));
while (iterator.hasNext()) {
	System.out.println(iterator.next());
}
```

## 编码

java代码：

```java
User user = new User();
user.setAge(18);
user.setName("xx");
Contact contact = new Contact();
contact.setEmail("123456@123.com");
contact.setMobile("123456");
user.setContact(contact);
user.setRoles(Arrays.asList("a", "b"));
new ObjectMapper(new YAMLFactory()).writeValue(new File("D:/TEMP/temp2.yml"), user);
```

