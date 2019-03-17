---
layout: post
comments: false
categories: springframework
date:   2019-03-16 10:30:54
title: Spring源码阅读 - 配置加载与类定义注册
---

<div id="toc"></div>

Spring Framework的源码下载：https://github.com/spring-projects/spring-framework

```
git clone https://github.com/spring-projects/spring-framework.git
git checkout 4.3.x
```

打开源码库里的`import-into-idea.md`，根据其步骤将源码库导入Intellij IDEA。注意第二步中可以使用`./gradlew idea`来下载依赖并生成IDEA的的Project文件`spring.ipr`，而后使用IDEA打开此项目文件。

## 第一个源码分析入口 - XmlBeanFactory

Spring框架中很重要的一个功能就是对项目中类的配置化管理。Spring能够读取配置文件并实例化配置中的类。

配置文件在`test.xml`如:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "http://www.springframework.org/dtd/spring-beans-2.0.dtd">
<beans>
  <bean id="testBean" class="TestBean"/>
</beans>
```

可以使用XmlBeanFactory来处理这个配置文件，如:

```
XmlBeanFactory factory = new XmlBeanFactory(new ClassPathResource("test.xml"));
TestBean emma = (TestBean) factory.getBean("testBean");
```

目前XmlBeanFactory虽然可用但已经被Spring废弃了，但是其单元测试类[XmlBeanFactoryTests](https://github.com/spring-projects/spring-framework/blob/777b4c809d27575334abb4b954dd0089ceb8e1d4/spring-context/src/test/java/org/springframework/beans/factory/xml/XmlBeanFactoryTests.java
)却保留了下来，其定义了大量的XML配置相关的测试用例`Miscellaneous tests for XML bean definitions`。

在XmlBeanFactoryTests中，是没有任何`new XmlBeanFactory()`语句，因为XmlBeanFactory并不适合去做测试，最主要的一个原因是XmlBeanFactory是需要对XML进行有效性验证，而测试所用到的一些XML文件会在验证时出错。而XmlBeanFactory本身只是继承了DefaultListableBeanFactory，并在类中实例化XmlBeanDefinitionReader来解析传入的XML，约等于对以下的代码的封装：

```
DefaultListableBeanFactory xbf = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(xbf);
reader.loadBeanDefinitions(new ClassPathResource("ab.xml");
TestBean emma = (TestBean) factory.getBean("testBean");
```

这样写就可以关闭对XML文件的验证。

```
reader.setValidationMode(XmlBeanDefinitionReader.VALIDATION_NONE);
```

在XmlBeanFactoryTests每个单元测试中都可以看到使用了`DefaultListableBeanFactory`类和`XmlBeanDefinitionReader`类。

DefaultListableBeanFactory类继承结构层次是比较深的，我们暂时先不需要去了解那么多，只需要知道其能够`getBean`即可。

XmlBeanDefinitionReader类比较简单，功能也相对比较单一，首先关注`loadBeanDefinitions`方法即可。

## 第二个源码分析入口 - XmlBeanDefinitionReaderTests

通过单元测试类往往可以更快速的了解相应的实现类。[XmlBeanDefinitionReaderTests](https://github.com/spring-projects/spring-framework/blob/8aa6e5bfea2c7314deaa1b432554e9e914b09ee7/spring-beans/src/test/java/org/springframework/beans/factory/xml/XmlBeanDefinitionReaderTests.java)中的只有12个单元测试。

单元测试中实例化XmlBeanDefinitionReader时，传入的参数是SimpleBeanDefinitionRegistry。

```
@Test
public void withOpenInputStreamAndExplicitValidationMode() {
	SimpleBeanDefinitionRegistry registry = new SimpleBeanDefinitionRegistry();
	Resource resource = new InputStreamResource(getClass().getResourceAsStream("test.xml"));
	XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(registry);
	reader.setValidationMode(XmlBeanDefinitionReader.VALIDATION_DTD);
	reader.loadBeanDefinitions(resource);
	testBeanDefinitions(registry);
}
```

### XmlBeanDefinitionReader - doLoadBeanDefinitions

loadBeanDefinitions作为XmlBeanDefinitionReader的公有方法，返回注册的类的个数。其调用了私有的doLoadBeanDefinitions方法。这个方法主要做了两件事情：

- 调用doLoadDocument函数根据配置文件生成Document(org.w3c.dom.Document)对象

> JAXP:(Java API for XML Processing)开发包是JavaSE的一部分，它由`org.w3c.dom`,`org.xml.sax`,`javax.xml`几个子包组成

> 常用XML的解析方式：DOM和SAX。DOM将整个XML加载内存中，形成文档对象，所以对XML操作都对内存中文档对象进行。SAX一边解析，一边处理，一边释放内存资源

- 调用registerBeanDefinitions函数来把Document对象的结果注册到SimpleBeanDefinitionRegistry中去

#### doLoadDocument函数

先来看一下这个函数的定义：

```
Document doLoadDocument(InputSource inputSource, Resource resource)
```

- inputSource的类型是org.xml.sax.InputSource，可以使用org.xml.sax.XMLReader来解析InputSource

- resource是Spring自定义的org.springframework.core.io.Resource接口，通过doLoadBeanDefinitions传入的实际上就是此章节开始时代码块里实例化的`new ClassPathResource("import.xml", getClass())`

函数中的唯一一行代码是将数据的处理交给`this.documentBuilder`，并返回构建好的Document对象。

```
return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
```

- documentLoader默认值是DefaultDocumentLoader， 当然也可以通过XmlBeanDefinitionReader的setDocumentLoader来设置

- getEntityResolver()获取EntityResolver，XML可以使用dtd或schema校验，而dtd文件或用来校验的xml文件(schema)从哪里加载呢？xml文件中并没有指明去哪里加载，怎么加载，这个时候就需要EntityResolver了。如是从互联网连接去拿取呢，还是ClassPath中查找，还是META-INF/Spring.schemas去查找。

- errorHandler是org.springframework.util.xml.SimpleSaxErrorHandler的实例，SimpleSaxErrorHandler又是org.xml.sax.ErrorHandler接口的实现。

- getValidationModeForResource(resource)，获取验证模式，0不验证，1 auto-guessed, 2 DTD验证，3 XSD验证。最重要的一个逻辑是如果XmlBeanDefinitionReader没有明确定义验证模式，则从resource中获取验证方式，如果获取失败则抛出BeanDefinitionStoreException异常。

- isNamespaceAware(), 默认为false, XML 命名空间提供避免元素命名冲突的方法。如`<h:table>`和`<f:table>`命名不冲突，也可以在元素中使用XML Namespace (xmlns) 属性。

传递这些属性给DefaultDocumentLoader，而DefaultDocumentLoader主要是对如下代码的封装：

```
DocumentBuilderFactory builderFactory = DocumentBuilderFactory.newInstance();
builderFactory.setNamespaceAware(false);
builderFactory.setValidating(true);

DocumentBuilder builder = builderFactory.newDocumentBuilder();
builder.setEntityResolver(new DelegatingEntityResolver(this.getClass().getClassLoader()));
builder.setErrorHandler(new SimpleSaxErrorHandler(LogFactory.getLog(getClass())));

InputSource source = new InputSource(getClass().getResourceAsStream("test.xml"));
Document document = builder.parse(source);
```

至此，可以看出最终是通过javax.xml.parsers.DocumentBuilder来解析XML文件并返回Document对象。

#### registerBeanDefinitions函数

这个函数的定义：

```
int registerBeanDefinitions(Document doc, Resource resource)
```

- doc为doLoadDocument函数从XML配置文件中解析出来的Document

- resource是Spring自定义的org.springframework.core.io.Resource接口，通过doLoadBeanDefinitions传入的实际上就是此章节开始时代码块里实例化的`new ClassPathResource("import.xml", getClass())`

- 函数返回的是此函数注册的Bean的个数

函数中的处理逻辑：

```
BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
int countBefore = getRegistry().getBeanDefinitionCount();
documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
return getRegistry().getBeanDefinitionCount() - countBefore;
```

- documentReader默认为DefaultBeanDefinitionDocumentReader， 其实现了BeanDefinitionDocumentReader接口

  ```
  void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) throws BeanDefinitionStoreException;
  ```

  这里XmlReaderContext是一个类而非接口，其明确定义了`getRegistry`方法。基本上可以得到此接口的隐喻是将文档doc中的类定义注册到XmlReaderContext的registry中。

  > XmlReaderContext在程序中是不可被替换的，因为createReaderContext方法中是直接调用的`new XmlReaderContext(...)`

  XmlReaderContext的reader成员变量将持有XmlBeanDefinitionReader对象的指针，这个很重要，使得XmlReaderContext中能获取到reader中的registry, resourceLoader, beanClassLoader等。

- 这里获取的getRegistry()与中XmlReaderContext的getRegistry获取的Registry是一致的，获取到的是接口`BeanDefinitionRegistry`的实现，可以是测试用的SimpleBeanDefinitionRegistry，也可以是DefaultListableBeanFactory（实现了BeanDefinitionRegistry接口)


### DefaultBeanDefinitionDocumentReader - registerBeanDefinitions

在XmlBeanDefinitionReaderTests中，关于其成员变量`documentReaderClass`有三个单元测试用例，可见对其重视程度。

```
@Test
public void setParserClassSunnyDay() {
	SimpleBeanDefinitionRegistry registry = new SimpleBeanDefinitionRegistry();
	new XmlBeanDefinitionReader(registry).setDocumentReaderClass(DefaultBeanDefinitionDocumentReader.class);
}

@Test(expected = IllegalArgumentException.class)
public void setParserClassToNull() {
	SimpleBeanDefinitionRegistry registry = new SimpleBeanDefinitionRegistry();
	new XmlBeanDefinitionReader(registry).setDocumentReaderClass(null);
}

@Test(expected = IllegalArgumentException.class)
public void setParserClassToUnsupportedParserType() throws Exception {
	SimpleBeanDefinitionRegistry registry = new SimpleBeanDefinitionRegistry();
	new XmlBeanDefinitionReader(registry).setDocumentReaderClass(String.class);
}
```

三个测试表明，setDocumentReaderClass函数是允许多种参数输入的，但是如果输入参数类型不对，则会抛出异常。从代码中也可以验证这一点：

```
public void setDocumentReaderClass(Class<?> documentReaderClass) {
	if (documentReaderClass == null || !BeanDefinitionDocumentReader.class.isAssignableFrom(documentReaderClass)) {
		throw new IllegalArgumentException(
				"documentReaderClass must be an implementation of the BeanDefinitionDocumentReader interface");
	}
	this.documentReaderClass = documentReaderClass;
}
```


DefaultBeanDefinitionDocumentReader实现了BeanDefinitionDocumentReader接口。

```
void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) throws BeanDefinitionStoreException;
```

其实现：

```
this.readerContext = readerContext;
Element root = doc.getDocumentElement();
doRegisterBeanDefinitions(root);
```

#### doRegisterBeanDefinitions

doRegisterBeanDefinitions函数定义了一个对org.w3c.dom.Element处理的模板流程：

```
获取Element节点profile属性，profile定义的目前配置是属于什么样的环境，readerContext中的Environmentk可以基于profile的值判断来确定目前是否忽略对这个节点的处理
preProcessXml(root);
parseBeanDefinitions(root, this.delegate);
postProcessXml(root);
```

- profile属性使用方法如`<beans profile="dev">`，可以设置多个值`<beans profile="dev,prod">`，可以使用简单的表达式`<beans profile="!dev,!prod">`

- preProcessXml和postProcessXml是public空方法，是为了扩展提供空间。开发人员可以自定义DefaultBeanDefinitionDocumentReader子类并在preProcessXml和postProcessXml添加自己的扩展实现

- delegate是BeanDefinitionParserDelegate，bean节点以及其子节点的处理都定义在其中，其还定义了对自定义标签的处理

- parseBeanDefinitions首先处理传入的Element，如果没有不是Spring自身命名空间中的节点，则使用delegate.parseCustomElement来处理，否则获取传入Element的childNodes，对每个childNode进行判断其是否是Spring自身命名空间中的节点，如果不是使用delegate.parseCustomElement来处理。如果是的话，调用parseDefaultElement来处里

关于parseDefaultElement函数，其根据不同的节点来进行不同的处理

- `import`节点使用`importBeanDefinitionResource`函数来处理，获取其`resource`属性。resource中可以使用程序的系统属性，如`${user.dir}`，在这个函数里会使用XmlReaderContext的`getEnvironment().resolveRequiredPlaceholders`来处理。resource可以是绝对路径（如:"classpath*:/a/bc","classpath:/a/bc","file:///a/b/c"）;也可以是相对路径("a/abc.xml")，这个相对路径当前XML的相对路径。最终不管使用什么路径，都会调用XmlBeanDefinitionReader的loadBeanDefinitions方法，在这里就形成了个递归处理。

> 注意在importBeanDefinitionResource函数最后会触发事件fireImportProcessed，XmlBeanDefinitionReader中默认使用了EmptyReaderEventListener处理这个事件

- `alias`节点使用`processAliasRegistration`函数来处理，获取`name`和`alias`属性。而后注册到Registry中

> 注意在processAliasRegistration函数最后会触发事件fireAliasRegistered，XmlBeanDefinitionReader中默认使用了EmptyReaderEventListener处理这个事件

- `beans`节点会调用`doRegisterBeanDefinitions`方法，这里形成了一次递归处理

- `bean`节点调用`processBeanDefinition`方法，这个方法通过delegate获取BeanDefinitionHolder，并随后处理bean标签下的自定义标签。而后将BeanDefinitionHolder里边的`beanDefinition`，`beanName`, `aliases`注册到Registry中


#### BeanDefinitionParserDelegate

待完成


### SimpleBeanDefinitionRegistry

SimpleBeanDefinitionRegistry是为了单元测试而创建的一个类。其继承自SimpleAliasRegistry，且实现了BeanDefinitionRegistry。而SimpleAliasRegistry类又实现了AliasRegistry，BeanDefinitionRegistry也是AliasRegistry的子接口。

这里重点介绍BeanDefinitionRegistry中定义的方法:

```
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
void removeBeanDefinition(String beanName)
BeanDefinition getBeanDefinition(String beanName)
boolean containsBeanDefinition(String beanName)
String[] getBeanDefinitionNames()
int getBeanDefinitionCount()
boolean isBeanNameInUse(String beanName)
```

## 第三个源码分析入口 - DefaultListableBeanFactoryTests

这里也是选用单元测试列DefaultListableBeanFactoryTests来作为入口，是因为测试类的测试场景更又例如对于类功能的理解。测试类有3000+行代码，对DefaultListableBeanFactory类的各种使用场景都做了测试。



<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
