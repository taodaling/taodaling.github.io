---
categories: framework
layout: post
---

- table
{:toc}
# Introduction

Byte Buddy is a code generation and manipulation library for creating and modifying Java classes during the runtime of a java application and without the help of a compiler.

# Concept



# Example

## New type

```java
DynamicType.Unloaded<?> dynamicType = new ByteBuddy()
  .subclass(Object.class)
  .name("example.Type")
  .make();
```

## NamingStrategy

It's not necessary to name each class we created by byte-buddy, the default Byte Buddy configuration provides a NamingStrategy with randomly creates a class name based on a dynamic type's superclass name. The  default behavior might not be convenient for you, and thanks to the convention over configuration principle, you can always alter the default behavior by your needs. 

```java
DynamicType.Unloaded<?> dynamicType = new ByteBuddy()
  .with(new NamingStrategy.AbstractBase() {
    @Override
    public String subclass(TypeDescription superClass) {
        return "i.love.ByteBuddy." + superClass.getSimpleName();
    }
  })
  .subclass(Object.class)
  .make();
```

