---
layout: post
categories: java
---
- table
{:toc}

# Reading and writing bytecode

Javassist is a class library for dealing with java bytecode. Java bytecode is stored in a binary file called a class file. Each class file contains one Java class or interface.

In javassist, `CtClass` is an abstract representation of a class file. A CtClass object is a handle for dealing with a class file.

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("test.Rectangle");
cc.setSuperClass(pool.get("test.Point"));
cc.writeFile();
```

The `ClassPool` object is a container of CtClass object representation a class file. It reads a class file on demand for constructing a CtClass object and records the constructed object for response later accesses.

To modify the definition of a class, the users must first obtain a reference to a CtClass object representing that class from a ClassPool object. `get()` in ClassPool is used for this purpose. 

The ClassPool object returned by `getDefault()` searches the default system search path.

From the implementation viewpoint, ClassPool is a hash table of CtClass objects, which uses the class name as keys. get() in ClassPool searches this hash table to find a CtClass object associated with the specified key. If usch a CtClass object isn't found, get() reads a class file to construct a new CtClass object, which is recorded in the hash table and then returned as the resulting value of get().

The CtClass object obtained from a ClassPool object can be modified.`writeFile()` translates the CtClass object into a class file and writes it on a local disk. `toByteCode()` is used to obtain the bytecode implied by  the CtClass.You can directy load the CtClass with `toClass()` method defined in CtClass.toClass() requests the context class loader for the current thread to load the class file represented by the CtClass. It returns a `java.lang.Class` object representing the  loaded class.

## Defining a new class

To define a new class from scratch, `makeClass()` must be called on a ClassPool.

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.makeClass("Point");
```

## Frozen classes

If a CtClass object is converted into a class file by writeFile(), toClass() or toBytecode(), javassist freezes that CtClass object. Further modifications of that CtClass object are not permitted. This is for warning the developers when they attempt to modify a class file that has been already loaded since the JVM does not allow reloading a class.

A frozen CtClass can be defrost so that modifications of the class definition will be permitted.

```java
CtClass cc = ...;
...
cc.writeFile();
cc.defrost();
cc.setSuperclass(...); //Ok since the class is not frozen
```

If `ClassPool.doPruning` is set to true, then Javassist prunes the data structure contained in a CtClass object when Javassist freezes that object. To reduce memory consumption, pruning discards unnecessary attributes(`attribute_info` structures) in that object. For example, `Code_attribute` structures(method bodies) are discarded. Thus, after a CtClass object is pruned, the byte code of a method is not accessible except method names, signatures, and annotations. The pruned CtClass object cannot be defrost again. The default value of ClassPool.doPruning is false.

To disallow pruning a particular CtClass, `stopPruning()` must be called on that object in advance:

```java
CtClass cc = ...;
cc.stopPruning(true);
...
cc.writeFile();
```

The CtClass object cc is not pruned.Thus it can be defrost after writeFile() is called.

## Class search path



The default ClassPool returned by a static method ClassPool.getDefault() searches the same path that the underlying JVM has. If a program is running on a web application server such as JBoss and Tomcat, the ClassPool object may not be able to find user classes since such a web application server uses multiple class loaders as well as the system class loader. In that case, an additional class path must be registered to the ClassPool. 

```java
pool.insertClassPath(new ClassClassPath(this.getClass()));
```

The statement registers the class path that was used for loading the class of the object that `this` refers to.You can use any Class object as an argument instead of this.getClass(). The class path used for loading the class represented by that Class object is registered.

You can register a directory name as the class search path. For example, the following code adds a directory /usr/local/javalib to the search path:

```java
ClassPool pool = ClassPool.getDefault();
pool.insertClassPath("/usr/local/javalib");
```

The search path that the users can add is not only a directory but also a URL:

```java
ClassPool pool = ClassPool.getDefault();
ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
pool.insertClassPath(cp);
```

Futhermore, you can directly give a byte array to a ClassPool object and construct a CtClass object from that array. To do this, use `ByteArrayClassPath`.

```java
ClassPool cp = ClassPool.getDefault();
byte[] b = ...;
String name = ...;
cp.insertClassPath(new ByteArrayClassPath(name, b));
CtClass cc = cp.get(name);
```

If you don't know the fully-qualified name of the class, then you can use makeClass() in ClassPool:

```java
ClassPool cp = ClassPool.getDefault();
InputStream is = ...;
CtClass cc = cp.makeClass(ins);
```

# ClassPool

A ClassPool obejct is a container of CtClass objects. Once a CtClass object is created, it is recorded in a ClassPool forever. 

If you add a method in class A, and invoke this method in class B, then A's CtClass must not be lost, else the compiler will fail to compile the source code of B because no such method in A's original definition.

## Avoid out of memory

This specification of ClassPool may cause huge memory consumption if the number of CtClass objects becomes amazingly large. To avoid this problem, you can explicitly remove an unnecessary CtClass object from the ClassPool. If you call detach on a CtClass object, then that CtClass object is removed from the ClassPool.

```java
CtClass cc = ...;
cc.writeFile();
cc.detach();
```

Another idea is to occasionally replace a ClassPool with a new one and discard the old one. So that garbage collector could release the memory occupied by the old one.

```
cp = new ClassPool();
```

This creates a ClassPool object that behaves as the default ClassPool returned by ClassPool.getDefault() dose. Note that ClassPool.getDefault() is a singleton factory method provided for convenience.

## Cascaded ClassPools

If a program is running on a web application server, creating multiple instances of ClassPool might be necessary; an instance of ClassPool should be created for each class loader.

Multiple ClassPool objects can be cascaded like java.lang.ClassLoader. 

```java
ClassPool parent = ClassPool.getDefault();
ClassPool child = new ClassPool(parent);
child.insertClassPath("./classes");
```

If child.get() is called, the child ClassPool first delegates to the parent ClassPool. If the parent ClassPool fails to find a class file, then the child ClassPool attempts to find a class file under the "./classes" directory.

If child.childFirstLookup is true, the child ClassPool attempts to find a class file before delegating to the parent ClassPool.

```java
ClassPool parent = ClassPool.getDefault();
ClassPool child = new ClassPool(parent);
child.appendSystemPath();
child.childFirstLookup = true.
```

## Changing a class name for defining a new class

A new class can be defined as a copy of an existing class.

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("point");
cc.setName("Pair");
```

Note that setName() in CtClass changes a record in the ClassPool object. setName() will change the key associated to the CtClass object in the hash table. The key is changed from the original class name to the new class name.

Therefore, if get("Point") is later called on the ClassPool object again, then it will read a class file again and constructs a new CtClass object for class Point. 

## Renaming a frozen class for defining a new class



Once a CtClass object is converted into a class file by writeFile() or toBytecode(), Javassist rejects further modifications of that CtClass object. Hence, after the CtClass object representing Point class is converted into a class file, you cannot define Pair class as a copy of Point since executing setName() on Point is rejected.

To avoid this restriction, you should call getAndRename() in ClassPool.

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
cc.writeFile();
CtClass cc2 = pool.getAndRename("Point", "Pair");
```

If getAndRename() is called, the ClassPool first reads Point.class for creating a new CtClass object representing Point class. However, it renames that CtClass object from Point to  Pair before it records that CtClass object in a hash table. Thus getAndRename() can be executed after writeFile() or toBytecode() is called on the CtClass object representing Point  class.

# Class loader

If what classes must be modified is known in advance, the easiest way for modifying the classes is as follows:

1. Get a CtClass object by calling ClassPool.get()
2. Modify it, and
3. Call writeFile() or toBytecode() on that CtClass object to obtain a modified class file.

If whether a class is modified or not is determined at load time, the users must make Javassist collaborate with a class loader. Javassist can be used with a class loader so that bytecode can be modified at load time. The users of Javassist can define their own version of class loader but they can also use a class loader provided by Javassist.

## The toClass method in CtClass

The CtClass provides a convenience method toClass(), which requests the context class loader for the current thread to load the class represented by the CtClass object.To call this method, the caller must have appropriate permission; otherwise, a SecurityException may be thrown.

toClass() is provided for convenience. If you need more complex functionality, you should write your own class loader.

```java
CtClass cc = ...;
Class c = cc.toClass(bean.getClass().getClassLoader());
```

## Class loading in Java

In Java, multiple class loaders can coexist and each class loader creates its own name space. Different class loaders can load different class files with the same class name. The loaded two classes are regarded as different ones. 

The JVM doesn't allow dynamically reloading a class. Once a class loader loads a class, it can't reload a modified version of that class during runtime.Thus, you can't alter the definition of a class after the JVM loads it.

If the same class file is loaded by two distinct class loaders, the JVM makes two distinct classes with the same name and definition. The two classes are regarded as different ones. Since the two classes are not identical, an instance of one class is not assignable to a variable of the other class. The cast operation between the two classes fails and throws a ClassCastException.

## Using javassist.Loader

Javassist provides a class loader javassist.Loader. This class loader uses a ClassPool object for reading a class file.

For example, javassist.Loader can be used for loading a particular class modified with Javassist.

```java
ClassPool pool = ClassPool.getDefault();
Loader loader = new Loader(pool);
Class c1 = pool.get("Point").toClass();
Class c2 = pool.get("Point").toClass(loader);
```

If the users want to modify a class on demand when it is loaded, the users can add an event listener to a Loader. The added event listener is notified when the class loader loads a class. The event-listener class must implement the following interface.

```java
public interface Translator {
    public void start(ClassPool pool)
        throws NotFoundException, CannotCompileException;
    public void onLoad(ClassPool pool, String classname)
        throws NotFoundException, CannotCompileException;
}
```

The method start() is called when this event listener is added to a javassist.Loader object by addTranslator() in javassist.Loader. The method onLoad() is called before javassist.Loader loads a class. onLoader() can modify the definition of the loaded class.Note that onload() dosen't have to call toBytecode() or writeFile() since Loader calls these methods to obtain a class file.

Loader searches for classes in a different order from ClassLoader. ClassLoader firt delegates the loading operations to the parent class loader and then attempts to load the classes only if the parent class loadeer cannot find them. On the other hand, Loader attemps to load the classes before delegating to the parent class loader.It delegates only if:

- The classes are not found by calling get() on a ClassPool object, or
- The classes have been specified by using delegateLoadingOf() to be loaded by the parent class loader.

This search order allows loading modified classes by Javassist. However, it delegates to the parent loader if it fails to find modified classes for some reason. Once a class is loaded by the parent class loader, the other classes referred to in that class will be also loaded by the parent class loader and thus they are never modified.If your program fails to load a modified class, you should make sure whether all the classes using that class have been loaded by javassist.Loader.

## Modifying a system class

The system classes like String can't be loaded by a class loader other than the system class loader. Therefore, Loader cannot modify the system classes at loading time.

If your application needs to do that, the system classes must be statically modified.

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("java.lang.String");
CtField f = new CfField(CtClass.intType, "hiddenValue", cc);
f.setModifiers(Modifier.PUBLIC);
cc.addField(f);
cc.writeFile(".");
```

## Reloading a class at runtime

If the JVM is launched with JPDA(Java Platform Debugger Architecture) enabled, a class is dynamically reloadable. After the JVM loads a class, the old version of the class definition of that class can be  unloaded and a new one can be reloaded again. However, the new class definition must be somewhat compatible to  the old one. The JVM doesn't allow schema changes between the two versions. They have the same set of methods and fields.

Javassist provides a convenient class for reloading a class at runtime. For more information, see the javassist.tools.HotSwapper.

# Introspection and customization

CtClass provides methods for introspection. The introspective ability of Javassist is compatible with that of the Java reflection API. CtClass provides getName(), getSuperclass(), getMethods(), and so on. CtClass also provides methods for modifying a class definition. It allows adding a new field, constructor, and method. Instrumenting a method body is also possible.

Methods are represented by CtMethod objects. CtMethod provides several methods for modifying the definition of the method. Note that if a method is inherited from a super class, then the same CtMethod object that represents the inherited method represents the method declared in that super class. A CtMethod object corresponds to every method declaration.

For example, if class Point declares method move() and a subclass ColorPoint of Point does not override move(), the two move() methods declared in Point and inherited in ColorPoint are represented by the identical CtMethod object. If the method definition represented by this CtMethod object is modified, the modification is reflected on both the methods. If you want to modify only the move() method in ColorPoint, you first have to add to ColorPoint a copy of the CtMethod object representing move() in Point. A copy of the CtMethod object can be obtained by `CtNewMethod.copy()`.

Javassist doesn't allow removing a method or field, but it allows changing the name, so if a method is not necessary any more, it should be renamed and changed to be a private method by calling setName() and setModifiers() declared in CtMethod. Javassist doesn't allow adding an extra parameter to an existing method, either. Instead of doing that, a new method receiving the extra parameter as well as the other parameters should be added to the same class. For example, if you want to add an extra int parameter newZ to a method:

```java
void move(int x){...}
void move(int x, int y){...}
```

Javassist also provides low-level API for directly editing a raw class file. For example, getClassFile() in CtClass returns a ClassFile object representing a raw class file. getMethodInfo() in CtMethod returns a MethodInfo object representing a method_info structure included in a class file.

The class files modified by Javassist requires the javassist.runtime package for runtime support only if some special identifiers starting with $ are used.Those special identifiers are described below. The class files modified without those special identifiers do not need the javassist.runtime package or any other Javassist package at runtime. For more details, see the API documentation of the javassist.runtime package.

## Inserting source text at the beginning/end of a method body

CtMethod and CtConstructor provides methods insertBefore(), insertAfter(), and addCatch(). They are used for inserting a code fragment into the body of an existing method. The users can specify those code fragments with source text written in Java. Javassist includes a simple Java compiler for processing source text. It receives source text written in Java and compiles it into Java bytecode, which will be inlined into a method body.

Inserting a code fragment at the position specified by a line number is also possible(if the line number table is contained in the class file). insertAt() in CtMethod and CtConstructor takes source text and a line number in the source file of the original class definition. It compiles the source text and inserts the compiled code at the line number.

The methods insertBefore(), insertAfter() and insertAt() receive a String object representing a statement or a block. A statement is a single control structure like if and while or an expression ending with a semicolon(;). A block is a set of statements surrounded with braces `{}`. Hence each of the following lines is an example of valid statement or block:

```java
System.out.println("Hello");
{System.out.println("Hello");}
if(i < 0){i = -i;}
```

The statement and the block can refer to fields and methods. They can also refer to the parameters to the method that they are inserted into if  that method was compiled with the -g option(to inclue a local variable attribute in the class file). Otherwise, they must access the method parameters through the special variables $0, $1, $2, ... described below. Accessing local variables declared in the method is not allowed although declaring a new local variable in the block is allowed. However, insertAt() allows the statement and the block to access local variables if these variables are available at the specified line number and the target method was compiled with -g option.

The String object passed to the methods insertBefore(), insertAfter(), addCatch(), and insertAt() are compiled by the compiler included in Javassist. Since the compiler supports language extensions, serveral identifiers starting with $ have special meaning:

| identifier     | meaning                                                      |
| -------------- | ------------------------------------------------------------ |
| $0,$1, $2, ... | this and actual parameters                                   |
| $args          | An array of parameters. The type of $args is Object[].       |
| $$, &nbsp      | All actual parameters. For example, m($$) is equivalent to m($1,$2,...) |
| &cflow(...)    | `cflow`variable                                              |
| $r             | The result type. It is used in a cast expression.            |
| $w             | The wrapper type. It is used in a cast expression.           |
| $_             | The resulting value                                          |
| $sig           | An array of java.lang.Class objects representing the formal parameter types |
| $type          | A java.lang.Class object representing the formal result type. |
| $class         | A java.lang.Class object representing the class currently edited. |

### $0, $1, $2..

The parameters passed to the target method are accessible with $1, $2, ... instead of the original parameter names, $1 represents the first parameter, $2 represents the second parameter, and so on. The types of those variables are identical to the parameter type. $0 is equivalent to this. If the method is static, $0 is not available.

### $args

The variable $args represents an array of all the parameters. The type of that variable is an array of class Object. If a parameter type is a primitive type such as int, then the parameter value is converted into a wrapper object such as java.lang.Integer to store in $args. Thus, $args[0] is equivalent to $1 unless the type of the first parameter is a primitive type. Note that $args[0] is not equivalent to $0; $0 represents this.

If an array of Object is assigned to $args, then each element of that array is assigned to each parameter. If a parameter type is a primitive type, the type of corresponding element must be a wrapper type. The value is converted from the wrapper type to the primitive type before it is assigned to the parameter.

### $

The variable $$ is abbreviation of a list of all the parameters separated by commas. For example:

```java
public void setXY(int x, int y){...}

public void setPos(int x, int y){
    setXY($$);
}
```

### $cflow

`$cflow` means "control flow". This read-only variable returns the depth of the recursive calls to a specific method.

Suppose that the method shown below is represented by a CtMethod object cm:

```java
int fact(int n){
    return n <= 1 ? 1 : n * fact(n - 1);
}
```

To use $cflow, first declare that $cflow is used for monitoring calls to the method fact.

```java
CtMethod cm = ...;
cm.useCflow("fact");
```

The parameter to useCflow() is the identifier of the declared $cflow variable. Any valid Java name can be used as the identifier. Since the identifier can also include ".", for example, "my.test.fact" also is a valid identifier.

Then, $cflow(fact) represents the depth of the recursive calls to the method specified by cm. The value of $cflow(fact) is 0 when the method is first called whereas it is 1 when the method is recursively called within the method. 

```java
cm.insertBefore("if($cflow(fact) == 0) System.out.println(\"fact \" + $1);");
```

### $r

$r represents the result type of the method. It must be used as the cast type in a cast expression. For example, this is a typical use:

```java
Object result = ...l
$_ = ($r)result;
```

If the result type is a primitive type, then ($r) follows special semantics. First, if the operand type of the cast expression is a primitive type, ($r) works as a normal cast operator to the result type. On the other hand, if the operand type is a wrapper type, ($r) converts from the wrapper type to the result type. For example, if the reuslt type is int, then ($r) converts from java.lang.Integer to int.

If the result type is void, then ($r) doesn't convert a type; it does nothing. However, if the operand is a call to a void method, then ($r) results in null. For example, if the result type is void and foo() is a void method, then

```java
$_ = ($r) foo();
```

is a valid statement.

The cast operator ($r) is also useful in a return statement, even if the result type is void, the following return statement is valid:

```java
return ($r) result;
```

Here, result is some local variable. Since ($r) is specified, the resulting value is discarded, this return statement is regarded as the equivalent of the return statement without a resulting value:

```java
return;
```

### $w

$w represents a wrapper type. It must be used as the cast type in a cast expression.($w) converts from a primitive type to the corresponding wrapper type.

The following code is an example:

```java
Integer i = ($w)5;
```

The selected wrapper type depends on the type of the expression following ($w). If the type of the expression is double, then the wrapper type is java.lang.Double.

If the type of the expression following ($w) isn't a primitive type, then ($w) does nothing.

### $_

The variable $_ represents the resulting value of the method. The type of that variable is the type of the result type of the method. If the result type is void, then the type of $_ is Object and the value of $_ is null.

When $_ is used in statement which is send as argument for insertAfter, and mark asFinally with true, then once an exception is thrown, the compiled code inserted by insertAfter is executed as a finally clause. The value of $_ is 0 or null in the compiled code.

### $sig

The value of $sig is an array of java.lang.Class objects that represent the formal parameter types in declaration order.

### $type

The value of $type is an java.lang.Class object representing formal type of the result value. This variable refers to Void.class if this is a constructor.

### $class

The value of $class is an java.lang.Class object representing the class in which the edited method is declared. This represents the type of $0.

## addCatch()

addCatch() inserts a code fragment into a method body so that the code fragment is executed when the method body throws an exception and the control returns to the caller. In the source text representing the inserted code fragment, the exception value is referred to with the special variable $e.

```java
CtMethod m = ...;
CtClass etype = ClassPool.getDefault().get("java.io.IOException");
m.addCatch("{System.out.println($e);throw $e;}", etype);
```

Note that the inserted code fragment must end with a throw or return statement.

## Altering a method body

CtMethod and CtConstructor provide `setBody()` for substituting a whole method body. They compile the given source text into Java bytecode and substitutes it for the original method body. If the given source text is null, the substituted body includes only a return statement, which returns zero or null unless the result type is void.

### Substituting source text for an existing expression

Javassist allows modifying only an expression included in a method body. `javassist.expr.ExprEditor` is a class for replacing an expression in a method body. The users can define a subclass of `ExprEditor` to specify how an expression is modified.

To run an ExprEditor object, the user must call instrument() in CtMethod or CtClass.

```java
CtMethod cm = ...;
cm.instrument(
    new ExprEditor(){
        public void edit(MethodCall m) throws CannotCompileException
        {
            if(m.getClassName().equals("Point") &&
               m.getMethodName().equals("move")){
                m.replace("{$1 = 0; $_ = $proceed($$);}");
            }
        }
    }
);
```

The method `instrument()` searches a method body. If it finds an expression such as a method call, field access, and object creation, then it calls `call()` is an object representing the found expression. The `edit()` method can inspect and replace the expression through that object.

Calling `replace()` on the parameter to `edit()` substitutes the given statement or block for the expression. If the given block is  an empty block, that is, if `replace("{}"` is executed, then the expression is removed from the method body.

If you want to insert a statement (or a block) before/after the expression, a block like the following should be passed to replace.

```java
{
    before-statements;
    $_ = $proceed($$);
    after-statements;
}
```

whichever the expression is either a method call, field access, object creation, or others. The  second statement could be:

```java
$_ = $proceed();
```

If the expression is read access, or

```java
$proceed($$);
```

if the expression is a write access.

Local variables in the target expression is also available in the source text passed to replace() if the method searched by instrument() was compiled with the -g option (the class file includes a local variable attribute).

### javassist.expr.MethodCall

A `MethodCall` object represents a method call. The method `replace()` in `MethodCall` substitutes a statement or a block for the method call. It receives source text representing the substituted statement or block, in which the identifiers starting with `$` has special meaning as in the source text passed to `insertBefore()`.

| variable    | description                                                  |
| ----------- | ------------------------------------------------------------ |
| $proceed    | The name of the method originally called in the type. $0 is null if the method is static. |
| $0          | The target object of the method call.                        |
| $1, $2, ... | The parameters of the method call                            |
| $_          | The resulting value of the method call                       |
| $class      | A java.lang.Class object representing the class declaring the method. |
| $sig        | An array of java.lang.Class objects representing the formal parameter types |
| $type       | A java.lang.Class object representing the formal result type |

Unless the result type of the method call is `void`, a value must be assigned to `$_` in the source text and the type of `$_` is the result type. If the result type is `void`, the type of `$_` is Object and the value assigned to `$_` is ignored.

### javassist.expr.ConstructorCall

A `ConstructorCall` object represents a constructor call such as `this()` and `super()` included in a constructor body. The method `replace()` in `ConstructorCall` substitutes a statement or a block for the constructor call. It receives source text representing the substituted statement or block, in which the identifiers starting with $ have special meaning as in the source text passed to insertBefore().

| variable  | description                                                  |
| --------- | ------------------------------------------------------------ |
| $0        | The target object of the constructor call. This is equaivalent to this. |
| $1, $2... | The parameters of the constructor call.                      |
| $class    | A java.lang.Class object representing the class declaring the constructor. |
| $sig      | An array of java.lang.Class objects representing the formal parameter types. |
| $proceed  | The name of the constructor originally called in the expression. |



### javassist.expr.FieldAccess

A `FieldAccess` object represents field access. The method edit() in ExprEditor receives this object if field access is found. The method replace() in FieldAccess receives source text representing the substituted statement or block for the field access.

If the expreassion is read access, a value must be assigned to $_ in the source text. The type of $_ is the type of the field.

| variable | description                                                  |
| -------- | ------------------------------------------------------------ |
| $0       | The object containing the field accessed by the expression. This is not equivalent to this. $0 is null if the field is static. |
| $1       | The value that would be stored in the field if the expression is write access. Otherwise, $1 is not available. |
| $_       | The resulting value of the  field access if the expression is read access. Otherwise the value stored in $_ is discarded. |
| $r       | The type of the field if the expression is read access. Otherwise $r is void. |
| $class   | A java.lang.Class object representing the class declaring the field. |
| $type    | A java.lang.Class object representing the field type.        |
| $proceed | The name of a virtual method executing the original field access |

### javassist.expr.NewExpr

A `NewExpr` object represents object creation with the `new` operator (not including array creation). The method edit() in ExprEditor receives this object if object creation is found. The method replace() in NewExpr receives source text representing the substituted statement or block for the object creation.

| variable    | description                                                  |
| ----------- | ------------------------------------------------------------ |
| $0          | n                                                            |
| $1, $2, ... | The parameters to the constructor                            |
| $_          | The resulting value of the object creation.A newly created object must be stored in this variable. |
| $r          | The type of the created object.                              |
| $sig        | An array of java.lang.Class objects representing the formal paramter types. |
| $type       | A java.lang.Class object representing the class of the created object. |
| &proceed    | The name of a virtual method executing the original object creation. |

### javassist.expr.NewArray

A `NewArray` object represents array creation with the new operator. The method `edit()` in `ExprEditor` receives this object if array creation is found. The method `replace()` in `NewArray` receives source text representing the substituted statement or block for the array creation.

| variable    | description                                                  |
| ----------- | ------------------------------------------------------------ |
| $0          | null                                                         |
| $1, $2, ... | The size of each dimension.                                  |
| $_          | The resulting value of the array creation. A newly created aray must be stored in this variable. |
| $r          | The type of the created array.                               |
| $type       | A java.lang.Class object representing the class of the created array. |
| $proceed    | The name of a virtual method executing the original array crea |

```java
String[][] s = new String[3][4];
```

In the previous example, $1 and $2 are 3 and 4.

### javassist.expr.Instanceof

A `Instanceof` object represents an `instanceof` expression. The method edit() in ExprEditor receives this object if an instanceof expression is found. The method replace() in Instanceof receives source text representing the substituted statement or block for the expression.

| variable | description                                                  |
| -------- | ------------------------------------------------------------ |
| $0       | null                                                         |
| $1       | The value on the left hand side of the original instanceof operator |
| $_       | The resulting value of the expression.The type of $_ is boolean. |
| $r       | The type on the rightr hand side of the instanceof  operator |
| $type    | A java.lang.Class object representing the type on the right hand side of the instanceof operator. |
| $proceed | The name of a virtual method executing the original instanceof expresion. It takes one parameter (the type is java.lang.Object) and returns true is the parameter value is an instance of the type on the right hand side of the original instanceof operator. Otherwise, it returns false. |

### javassist.expr.Cast

A `Cast` object represents an expression for explicit type casting. The method edit()  in ExprEditor receives this object if explicit type casting is found. The method replace() in Cast receives source text representing the substituted statement or block for the expression.

| variable | description                                                  |
| -------- | ------------------------------------------------------------ |
| $0       | null                                                         |
| $1       | The value the type of which is explicitly cast.              |
| $_       | The resulting value of the expression. The type of $_ is the same as the type after the explicit casting, that is, the type surrounded by (). |
| $r       | the type after the explicit casting, or the type surrounded by (). |
| $type    | A java.lang.Class object representing the same type as $r.   |
| $proceed | The name of the virtual method executing the original type casting. It takes one parameter of the type java.lang.Object and returns it after the explicit type casting specified by the original expression. |

### javassist.expr.Handler

A `Handler` object represents a catch clause of try-catch statement. The method edit() in ExprEditor receives this object if a catch is found. The method edit() in ExprEditor receives this object if a catch is found. The method insertBefore() in Handler compiles the received source text and inserts it at the beginning of the catch clause.

| variable | description                                                  |
| -------- | ------------------------------------------------------------ |
| $1       | The exception object caught by the catch clause.             |
| $r       | The type of the exception caught by the catch clause. It is used in a cast expression. |
| $w       | The wrapper type. It is used in a cast expression.           |
| $type    | A java.lang.Class object representing the type of the exception caught by the catch clause. |

If a new exception object is assigned to $1, it is passed to the original catch clause as the caught exception.

## Adding a new method or field

### Adding a method

Javassist allows the users to create a new method and constructor from scratch.`CtNewMethod` and `CtNewConstructor` provides several factory methods, which are static methods for creating CtMethod or CtConstructor object. Especially, make() creates a CtMethod or CtConstructor object from the given source text.

```java
CtClass point = ClassPool.getDefault().get("Point");
CtMethod m = CtNewMethod.make("public int xmove(int dx){x += dx;}", point);
point.addMethod(m);
```

adds a public method xmove() to class Point. In this example, x is a int field in the class.

Javassist provides another way to add a new method. You can first create an abstract method and later give it a method body.

```java
CtClass cc = ...;
CtMethod m = new CtMethod(CtClass.intType, "move", new CtClass[]{CtClass.intType}, cc);
cc.addMethod(m);
m.setBody("{x += $1;}");
cc.setModifiers(cc.getModifiers() & ~Modifier.ABSTRACT);
```

Since javassist makes a class abstract if an abstract method is added to the class, you have to explicitly change the class back to a non-abstract one after calling setBody().

### Mutual recursive methods

Javassist cannot compile a method if it calls another method that has not been added to a class.(Javassist can compile a method that calls itself recursively.) To add mutual recursive methods to a class, you need a trick shown below. Suppose that you want to add methods m() and n() to a class represented by cc.

```java
CtClass cc = ...;
CtMethod m = CtNewMethod.make("public abstract int m(int i);", cc);
CtMethod n = CtNewMethod.make("public abstract int n(int i);", cc);
cc.addMethod(m);
cc.addMethod(n);
m.setBody("{return $1 == 0 ? 0 : n($1);}");
n.setBody("{return $1 == 0 ? 0 : m($1 - 1);}");
cc.setModifiers(cc.getModifiers() & ~Modifier.ABSTRACT);
```

### Adding a field

Javassist also allows the users to create a new field.

```java
CtClass point = ClassPool.getDefault().get("Point");
CtField field = new CtField(CtClass.intType, "z", point);
point.addField(f);
```

If the initial value of the added field must be specified

```java
CtField field = new CtField(CtClass.intType, "z", point);
point.addField(field);
point.addField(field, "0"); //initial value is 0
```

Furthermore, the above code can be rewritten into the following simple code:

```java
CtClass point = ClassPool.getDefault().get("Point");
CtField f = CtField.make("public int z = 0;", point);
point.addField(f);
```

### Removing a member

To remove a field or a method, call removeField() or removeMethod() in CtClass. A CtConstructor can be removed by removeConstructor() in CtClass.

## Annotations

CtClass, CtMethod, CtField and CtConstructor provides a convenient method getAnnotations() for reading annotations. It returns an annotation-type object.

For example, suppose the following annotation:

```java
public @interface Author{
    String name();
    int year();
}
```

This annotation is used as the following:

```java
@Autor(name="Chiba", year=2005)
public class Point{
    int x, int y;
}
```

Then the value of the annotation can be obtained by getAnnotation(). It returns an array containing annotation-type objects.

```java
CtClass cc = ClassPool.getDefault().get("Point");
Object[] all = cc.getAnnotations();
Author a = (Author)all[0];
String name = a.name();
int year = a.year();
System.out.println("name: " + name + ", year: " + year);
```

## Runtime support classes

In most cases, a class modified by Javassist doesn't require Javassist to run. However, some kinds of bytecode generated by the Javassist compiler need runtime support classes, which are in the javassist.runtime package is the only package that classes modified by Javassist may need for running.The other Javassist classes are never used at runtime of the modified classes.

## Import

All the class names in source code must be fully qualified(They must include package names). However, the java.lang package is an exception; for example, the Javassist compiler can resolve Object as well as java.lang.Object.

To tell the compiler to search other packages when resolving a class name, call importPackage() in ClassPool. For example, 

```java
ClassPool pool = ClassPool.getDefault();
pool.importPackage("java.awt");
CtClass cc = pool.makeClass("Test");
CtField f = CtField.make("public Point p;", cc);
cc.addField(f);
```

Note that importPackage() doesn't affect the get() method in ClassPool. Only the compiler considers the imported packages. The parameter to get() must be always a fully qualified name.

## Limitations

In the current implementation, the Java compiler included in Javassist has several limitations with respect to the language that the compiler can accept. Those limitations are:

- The new syntax introduced by J2SE5.0(including enums and generics) has not been supported. Annotations are supported by the low level API of Javassist. Generics are also only partly supported.
- Array initializers, a comma-separated list of expressions enclosed by braces { and }, are not available unless the array dimension is one.
- Inner classes or anonymous classes are not supported. Not that this is a limitation of the compiler only. It cannot compile source code including an anonymous-class declaration. Javassist can read and modify a class file or inner/anonymous class.
- Labeled `continue` and `brea` statements are not supported.
- The compiler doesn't correctly implement the java method dispatch algorithm. The compiler may confuse if methods defined in a class have the same name but take different parameter lists.
- The users are recommended to use `#` as the separator between a class name and a static method or field name.



# Bytecode level API

Javassist also provides lower-level API for directly editing a class file. To use this level of API, you need detailed knowledge of the Java bytecode and the class file format while this level of API allows you any kind of modification of class files.

