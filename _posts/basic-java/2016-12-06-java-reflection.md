---
layout: post
comments: false
categories: Java基础
date:   2016-11-29 10:30:54
title: Java 反射
---

<div id="toc"></div>

本文内容参考自(Java Reflection Tutorial)[http://tutorials.jenkov.com/java-reflection/index.html]

文章相关测试代码参考 https://github.com/zhouqing86/three-java/blob/master/src/test/java/medium/ReflectionTest.java

## 在类上的反射

- 获取类名字

```java
assertEquals("java.lang.Object", Object.class.getName());
assertEquals("Object", Object.class.getSimpleName());
```

- Modifier

```java
int modifiers = Object.class.getModifiers();
Modifier.isAbstract(modifiers)
Modifier.isFinal(modifiers)
Modifier.isInterface(modifiers)
Modifier.isNative(modifiers)
Modifier.isPrivate(modifiers)
Modifier.isProtected(modifiers)
Modifier.isPublic(modifiers)
Modifier.isStatic(modifiers)
Modifier.isStrict(modifiers)
Modifier.isSynchronized(modifiers)
Modifier.isTransient(modifiers)
Modifier.isVolatile(modifiers)
```

- 包信息

```java
assertEquals("java.lang", Object.class.getPackage().getName());
```

- 父类

```java
assertEquals("java.lang.Object", String.class.getSuperclass().getName());
assertEquals(null,Object.class.getSuperclass());
```

- 接口

```java
Class [] interfaces = Writer.class.getInterfaces();
assertEquals(3, interfaces.length);
assertEquals("java.lang.Appendable",interfaces[0].getName());
```

- 类构造器

```java
Constructor [] constructors = File.class.getConstructors();
assertEquals(4, constructors.length);

Constructor constructor1 = String.class.getConstructor(new Class[]{String.class});
Constructor constructor2 = String.class.getConstructor(String.class);

String testStr = (String) constructor1.newInstance("hello");
assertEquals("hello", testStr);
testStr = (String) constructor2.newInstance("world");
assertEquals("world", testStr);
```

- 类成员变量

```java
@Test
public void testGetFields() throws Exception {
   Field [] fields = File.class.getFields();
   assertEquals(4, fields.length);
   fields = File.class.getDeclaredFields();
   assertEquals(14, fields.length);
   Stream.of(fields).map(field -> field.getName()).forEach(System.out::println);
}

@Test(expected = IllegalAccessException.class)
public void testSetStaticFinalFieldWillThrowException() throws Exception {
   Field field = File.class.getField("separator");
   File file = new File("test.txt");
   field.setAccessible(true);
   assertEquals("/", field.get(file));
   field.set(file, "/test");
}

@Test
public void testSetPrivateField() throws Exception {
    Field field = File.class.getDeclaredField("path");
    field.setAccessible(true);
    File file = new File("test.txt");
    field.set(file, "test1.txt");
    assertEquals("test1.txt", file.getPath());
}
```

- 类方法
对于如果method为静态方法， method.invoke的第一个参数设置为null。

与私有的类成员变量的访问一样，如果要访问类私有方法，需要使用getDeclaredMethod方法来获取私有方法。getMethods只获取public方法。在invoke方法之前需要setAccessible为true。

```java
@Test
public void testGetMethods() throws Exception {
    Method [] methods = Object.class.getMethods();
    assertEquals(9, methods.length);
    Stream.of(methods).map(method -> method.getName()).forEach(System.out::println);
}

@Test
public void testSetMethods() throws Exception {
    Method method = People.class.getMethod("setName", new Class[]{String.class});
    People people = new People();
    method.invoke(people, "Sam");
    assertEquals("Sam", people.getName());
}

@Test
public void testMethodGetParameters() throws Exception {
    Method method = People.class.getMethod("setName", new Class[]{String.class});
    assertEquals(1, method.getParameterTypes().length);
    assertEquals("java.lang.String", method.getParameterTypes()[0].getName());
}

@Test
public void testMethodGetReturnType() throws Exception {
    Method method = People.class.getMethod("getName", null);
    assertEquals(0, method.getParameterTypes().length);
    assertEquals("java.lang.String", method.getReturnType().getName());
}
```

- 注解

```java
@Test
public void testClassAnnotations() throws Exception {
    TestClassAnnotation annotation = People.class.getAnnotation(TestClassAnnotation.class);
    assertEquals("someClass",annotation.name());
    assertEquals("Hello World",annotation.value());
}

@Test
public void testMethodAnnotations() throws Exception {
    Method method = People.class.getMethod("setName", new Class[]{String.class});
    TestMethodAnnotation annotation = method.getAnnotation(TestMethodAnnotation.class);
    assertEquals("someMethod",annotation.name());
    assertEquals("Hello World",annotation.value());
}

@Test
public void testFieldAnnotations() throws Exception {
    Field field = People.class.getDeclaredField("name");
    TestFieldAnnotation annotation = field.getAnnotation(TestFieldAnnotation.class);
    assertEquals("someField",annotation.name());
    assertEquals("Hello World",annotation.value());
}

@Test
public void testParameterAnnotations() throws Exception {
    Method method = People.class.getMethod("setName", new Class[]{String.class});
    Annotation[][] parameterAnnotations = method.getParameterAnnotations();
    for(Annotation[] annotations : parameterAnnotations){
        for(Annotation annotation : annotations){
            if(annotation instanceof TestParameterAnnotation){
                assertEquals("someParameter", ((TestParameterAnnotation) annotation).name());
                assertEquals("Hello World",((TestParameterAnnotation)annotation).value());
            }
        }
    }
}

@TestClassAnnotation(name="someClass",  value = "Hello World")
class People {

    @TestFieldAnnotation(name="someField",  value = "Hello World")
    private String name;

    private Integer age;

    public String getName() {
        return name;
    }

    @TestMethodAnnotation(name="someMethod",  value = "Hello World")
    public void setName(@TestParameterAnnotation(name="someParameter", value="Hello World") String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface TestClassAnnotation {
    public String name();
    public String value();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface TestMethodAnnotation {
    public String name();
    public String value();
}


@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface TestFieldAnnotation {
    public String name();
    public String value();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@interface TestParameterAnnotation {
    public String name();
    public String value();
}
```

## 泛型与反射
Java的类型信息在编译的时候就被擦除了，所以在运行时不能获取到任何泛型相关信息？这是不完全正确的，在所定义的类型上我们看不到泛型相关的信息，但是在使用泛型类型的成员变量或者方法上我们是可以获得泛型信息的。

举个例子

```
public class MyClass {

  protected List<String> stringList = ...;

  public List<String> getStringList(){
    return this.stringList;
  }
}
```

在运行时是可以获取到getStringList()的返回类型是List<String>而不是List。

```java
@Test
public void testGenericFieldTypes() throws Exception {
    Field field = People.class.getDeclaredField("stringList");
    Type genericFieldType = field.getGenericType();
    ParameterizedType aType = (ParameterizedType) genericFieldType;
    Type[] fieldArgTypes = aType.getActualTypeArguments();
    assertEquals(1, fieldArgTypes.length);
    assertEquals("java.lang.String", fieldArgTypes[0].getTypeName());
}

public List<String> stringList = new ArrayList<String>();
```

## 数组的反射

```java
@Test
public void testArrayReflection() throws Exception {
    int[] intArray = (int[]) Array.newInstance(int.class, 3);
    Array.set(intArray, 0, 123);
    Array.set(intArray, 1, 456);
    Array.set(intArray, 2, 789);
    assertEquals(123, Array.get(intArray, 0));
    assertEquals(456, Array.get(intArray, 1));
    assertEquals(789, Array.get(intArray, 2));
}


@Test
public void testObtainingStringArray() throws Exception {
    Class stringArrayClass = String[].class;
    assertEquals("java.lang.String", stringArrayClass.getComponentType().getName());
    Class intArray = Class.forName("[I");
    assertEquals("int", intArray.getComponentType().getName());
    stringArrayClass = Class.forName("[Ljava.lang.String;");
    assertEquals("java.lang.String", stringArrayClass.getComponentType().getName());
    int[] intArray2 = (int[]) Array.newInstance(int.class, 3);
    assertEquals(3, intArray2.length);
}
```

## 动态代理

动态代理可以用于很多场景： 数据库连接，事务管理，单元测试中动态Mock对象，其他的类AOP的对方法进行切面。

```java
interface MyInterface {
    void method1(int i);
    void method2(int i);
}

class MyInterfaceImpl implements MyInterface{

    @Override
    public void method1(int i) {
        System.out.println("method1 start");
        System.out.println(i);
        System.out.println("method1 end");
    }

    @Override
    public void method2(int i) {
        System.out.println("method2 start");
        System.out.println(i);
        System.out.println("method2 end");
    }
}

class MyInvocationHandler implements InvocationHandler {

    private Object myInterface;

    public MyInvocationHandler(Object myInterface) {
        this.myInterface = myInterface;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Invoke a method start");
        Object object = method.invoke(myInterface, args);
        System.out.println("Invoke a method end");
        return object;
    }
}

@Test
public void testDynamicProxy() throws Exception {
    MyInterfaceImpl myInterfaceImpl = new MyInterfaceImpl();
    InvocationHandler handler = new MyInvocationHandler(myInterfaceImpl);
    MyInterface myInterface = (MyInterface) Proxy.newProxyInstance(myInterfaceImpl.getClass().getClassLoader(), myInterfaceImpl.getClass().getInterfaces(), handler);
    myInterface.method1(5);
    System.out.println();
    myInterface.method2(10);
}


```

## 动态类加载和重新加载
动态编译的class文件不要放到jre的ClassPath中，在jre在ClassPath中找到的类只会加载一次。

使用反射获取类的时，如果需要获取最新生成的，需要重新实例化一个类加载器，因为旧的类加载器已经加载过这个类，再次加载只会加载上次那个。而重新实例化的类加载器没有加载过这个类，所以会重新去定义、链接和加载。



## 参考资料
[Java Reflection Tutorial](http://tutorials.jenkov.com/java-reflection/index.html)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
