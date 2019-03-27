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

- `bean`节点调用`processBeanDefinition`方法，这个方法通过delegate的parseBeanDefinitionElement方法获取BeanDefinitionHolder，并随后处理bean标签下的自定义标签。而后将BeanDefinitionHolder里边的`beanDefinition`，`beanName`, `aliases`注册到Registry中


#### BeanDefinitionParserDelegate

BeanDefinitionParserDelegate有1000+行代码，对于XML配置的bean节点的处理就在此类中。parseBeanDefinitionElement(Element ele)和parseCustomElement是其核心方法。

##### parseBeanDefinitionElement

parseBeanDefinitionElement(Element ele)会调用parseBeanDefinitionElement(ele, null)，后者方法的流程：

- 先获取id, name属性，name属性的值可以用逗号空格分隔成一个alias数组，beanName取Id或alias数组的第一个

- 检查beanName以及alias数组中值的唯一性，不唯一则抛出异常

- parseBeanDefinitionElement(ele, beanName, containingBean)获取到beanDefinition对象

- 如果没有id, name属性，将生成beanName

- 最后返回的是BeanDefinitionHolder对象，其将持有beanDefinition, beanName和aliasesArray

parseBeanDefinitionElement(ele, beanName, containingBean)

- createBeanDefinition将创建一个GenericBeanDefinition（从代码来看是GenericBeanDefinition：BeanDefinitionReaderUtils.createBeanDefinition方法在初始化创建BeanDefinition时，使用的是`GenericBeanDefinition bd = new GenericBeanDefinition();`）；并setParentName，根据XmlBeanDefinitionReader中是否定义了classLoader来确定setBeanClass还是setBeanClassName

- parseBeanDefinitionAttributes用来获取bean定义的属性。依次查看属性scope,abstract，lazy-init，autowire，dependency-check，depends-on，autowire-candidate，primary，init-method，destroy-method,factory-method, factory-bean。最后返回的AbstractBeanDefinition，注意不是BeanDefinition，因为AbstractBeanDefinition中定义了一些BeanDefinition中没有的方法，譬如init-method属性相关的setInitMethodName方法。

- parseMetaElements对meta标签，注意不是属性，进行处理

- parseLookupOverrideSubElements处理lookup-method标签，方法注入可以解决问题： 一个单例模式的bean A需要引用另外一个非单例模式的bean B，为了在我们每次引用的时候都能拿到最新的bean B。

```
<bean class="beanClass">
    <lookup-method name="method" bean="non-singleton-bean"/>
</bean>
```

non-singleton-bean指的是lookup-method中bean属性指向的必须是一个非单例模式的bean，当然如果不是也不会报错，只是每次得到的都是相同引用的bean（同一个实例），这样用lookup-method就没有意义了。

- parseReplacedMethodSubElements处理replaced-method标签，用来替换原有的方法实现

```
<bean id="myBean" class="com.zzr.web.test.MyBean">
    <replaced-method name="display" replacer="replacer"/>
</bean>

<bean id="replacer" class="com.zzr.web.test.MyBeanReplacer"/>
```

MyBeanReplacer类的display方法将替换myBean的display方法实现。在replaced-method中可以使用arg-type标签`<arg-type match="String"/>`或`<arg-type>String</arg-type>`

- parseConstructorArgElements用来解析constructor-arg标签

- parsePropertyElements解析第一级所有的子property标签, 循环使用parsePropertyElement来处理每个property标签，parsePropertyValue和parsePropertySubElement是对于property标签处理的主体函数。
  parsePropertyValue返回是一个Object对象，存储property解析出来的value值，可以是一个ref（RuntimeBeanReference），一个String(TypedStringValue)，list(ManagedList), array(ManagedArray), set(ManagedSet), map(ManagedMap), props(ManagedProperties)

  PropertyValue对象往往存储了<name,value>对，之所以Spring中没有使用map来存储name,value对，是因为PropertyValue有更好扩展性。PropertyValue的集合是MutablePropertyValues。

- parseQualifierElements解析qualifier标签

##### BeanDefinitionTests以及BeanDefinitionBuilderTests

BeanDefinitionTests几个单元测试都是在测试RootBeanDefinition的相等性，不过我们通过测试可以很简单的了解BeanDefinition:

```
RootBeanDefinition bd = new RootBeanDefinition(TestBean.class);
bd.getConstructorArgumentValues().addGenericArgumentValue("test");
bd.getConstructorArgumentValues().addIndexedArgumentValue(1, new Integer(5));
bd.getPropertyValues().add("name", "myName");
bd.getPropertyValues().add("age", "99");
bd.setQualifiedElement(getClass());
```

BeanDefinitionBuilderTests可以用来创建RootBeanDefinition, GenericBeanDefinition或者ChildBeanDefinition。

```
@Test
public void beanClassWithSimpleProperty() {
	String[] dependsOn = new String[] { "A", "B", "C" };
	BeanDefinitionBuilder bdb = BeanDefinitionBuilder.rootBeanDefinition(TestBean.class);
	bdb.setScope(BeanDefinition.SCOPE_PROTOTYPE);
	bdb.addPropertyReference("age", "15");
	for (int i = 0; i < dependsOn.length; i++) {
		bdb.addDependsOn(dependsOn[i]);
	}

	RootBeanDefinition rbd = (RootBeanDefinition) bdb.getBeanDefinition();
	assertFalse(rbd.isSingleton());
	assertEquals(TestBean.class, rbd.getBeanClass());
	assertTrue("Depends on was added", Arrays.equals(dependsOn, rbd.getDependsOn()));
	assertTrue(rbd.getPropertyValues().contains("age"));
}
```

##### parseCustomElement

对于自定义的标签，parseCustomElement函数会对其进行处理。其处理的过程是

- 根据Element获取到其Namespace URI,如`http://www.springframework.org/schema/util`

- 获取XmlReaderContext中的NamespaceHandlerResolver（源自XmlBeanDefinitionReader），默认为DefaultNamespaceHandlerResolver。

  DefaultNamespaceHandlerResolver默认会读取properties文件META-INF/spring.handlers，获取namespace URI到handler的映射。spring.handlers如:

  ```
  http\://www.springframework.org/schema/c=org.springframework.beans.factory.xml.SimpleConstructorNamespaceHandler
  http\://www.springframework.org/schema/p=org.springframework.beans.factory.xml.SimplePropertyNamespaceHandler
  http\://www.springframework.org/schema/util=org.springframework.beans.factory.xml.UtilNamespaceHandler
  ```

- 调用DefaultNamespaceHandlerResolver的resolve方法来根据Namespace URI解析出具体的NamespaceHandler
  DefaultNamespaceHandlerResolverTests中有一些单元测试用例，如下表示http://www.springframework.org/schema/util的NamespaceHandler应该是UtilNamespaceHandler类型：

  ```
  @Test
  	public void testResolvedMappedHandlerWithNoArgCtor() {
  		DefaultNamespaceHandlerResolver resolver = new DefaultNamespaceHandlerResolver();
  		NamespaceHandler handler = resolver.resolve("http://www.springframework.org/schema/util");
  		assertNotNull("Handler should not be null.", handler);
  		assertEquals("Incorrect handler loaded", UtilNamespaceHandler.class, handler.getClass());
  	}
  ```

- 调用NamespaceHandler的parse方法解析出BeanDefinition，如UtilNamespaceHandler继承自NamespaceHandlerSupport，NamespaceHandlerSupport中的parse方法会将其委托给相应的BeanDefinitionParser处理而返回BeanDefinition, 所以tilNamespaceHandler只需要在init方法中定义好tag与BeanDefinitionParser之间的关系，并独立实现各个BeanDefinitionParser。

  ```
  @Override
	public void init() {
		registerBeanDefinitionParser("constant", new ConstantBeanDefinitionParser());
		registerBeanDefinitionParser("property-path", new PropertyPathBeanDefinitionParser());
		registerBeanDefinitionParser("list", new ListBeanDefinitionParser());
		registerBeanDefinitionParser("set", new SetBeanDefinitionParser());
		registerBeanDefinitionParser("map", new MapBeanDefinitionParser());
		registerBeanDefinitionParser("properties", new PropertiesBeanDefinitionParser());
	}
  ```

  [UtilNamespaceHandlerTests](https://github.com/spring-projects/spring-framework/blob/8aa6e5bfea2c7314deaa1b432554e9e914b09ee7/spring-beans/src/test/java/org/springframework/beans/factory/xml/UtilNamespaceHandlerTests.java)定义了多个单元测试，而[testUtilNamespace.xml](https://github.com/spring-projects/spring-framework/blob/8aa6e5bfea2c7314deaa1b432554e9e914b09ee7/spring-beans/src/test/resources/org/springframework/beans/factory/xml/testUtilNamespace.xml)给出了util标签的使用样例。

  ```
  <util:constant id="min" static-field="
			java.lang.Integer.
			MIN_VALUE
 	"/>
  ```

  对应的单元测试

  ```
  @Before
	public void setUp() {
		this.beanFactory = new DefaultListableBeanFactory();
		XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this.beanFactory);
		reader.setEventListener(this.listener);
		reader.loadBeanDefinitions(new ClassPathResource("testUtilNamespace.xml", getClass()));
	}


	@Test
	public void testConstant() {
		Integer min = (Integer) this.beanFactory.getBean("min");
		assertEquals(Integer.MIN_VALUE, min.intValue());
	}
  ```

  Spring中如需要自定义标签可以参考UtilNamespaceHandler的实现。当然Spring中不仅仅可以自定义标签，也可以自定义属性，这些都是可以通过实现自己的UtilNamespaceHandler来达到目的。

### SimpleBeanDefinitionRegistry

SimpleBeanDefinitionRegistry是为了XmlBeanDefinitionReader单元测试而创建的一个类。其继承自SimpleAliasRegistry，且实现了BeanDefinitionRegistry。而SimpleAliasRegistry类又实现了AliasRegistry，BeanDefinitionRegistry也是AliasRegistry的子接口。

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

SimpleBeanDefinitionRegistry中实现的registerBeanDefinition：

```
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(64);

@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {

	Assert.hasText(beanName, "'beanName' must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");
	this.beanDefinitionMap.put(beanName, beanDefinition);
}
```

SimpleBeanDefinitionRegistry没有实现BeanFactory接口，所以无法执行getBean等方法。只能够测试是否解析出来的BeanDefinition是期望的，如：

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

private void testBeanDefinitions(BeanDefinitionRegistry registry) {
	assertEquals(24, registry.getBeanDefinitionCount());
	assertEquals(24, registry.getBeanDefinitionNames().length);
	assertTrue(Arrays.asList(registry.getBeanDefinitionNames()).contains("rod"));
	assertTrue(Arrays.asList(registry.getBeanDefinitionNames()).contains("aliased"));
	assertTrue(registry.containsBeanDefinition("rod"));
	assertTrue(registry.containsBeanDefinition("aliased"));
	assertEquals(TestBean.class.getName(), registry.getBeanDefinition("rod").getBeanClassName());
	assertEquals(TestBean.class.getName(), registry.getBeanDefinition("aliased").getBeanClassName());
	assertTrue(registry.isAlias("youralias"));
	String[] aliases = registry.getAliases("aliased");
	assertEquals(2, aliases.length);
	assertTrue(ObjectUtils.containsElement(aliases, "myalias"));
	assertTrue(ObjectUtils.containsElement(aliases, "youralias"));
}
```

## 第三个源码分析入口 - DefaultListableBeanFactoryTests

这里也是选用单元测试列DefaultListableBeanFactoryTests来作为入口，是因为测试类的测试场景更又例如对于类功能的理解。测试类有3000+行代码150个单元测试，对DefaultListableBeanFactory类的各种使用场景都做了单元测试。

### PropertiesBeanDefinitionReader

单元测试中都使用的是PropertiesBeanDefinitionReader而不是XmlBeanDefinitionReader，是因为PropertiesBeanDefinitionReader可以很方便的在代码中构造Properties作为输入，且其与XmlBeanDefinitionReader一样都继承自AbstractBeanDefinitionReader。

PropertiesBeanDefinitionReaderTest中的一个单元测试

```
private final DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

private final PropertiesBeanDefinitionReader reader = new PropertiesBeanDefinitionReader(
		beanFactory);

@Test
public void withSimpleConstructorArg() {
	this.reader.loadBeanDefinitions(new ClassPathResource("simpleConstructorArg.properties", getClass()));
	TestBean bean = (TestBean)this.beanFactory.getBean("testBean");
	assertEquals("Rob Harrop", bean.getName());
}
```

而在`simpleConstructorArg.properties`中定义的是:

```
testBean.(class)=org.springframework.tests.sample.beans.TestBean
testBean.$0=Rob Harrop
```

### DefaultListableBeanFactory的方法

#### preInstantiateSingletons

preInstantiateSingletons方法是`ConfigurableListableBeanFactory`接口的一个方法。将实例化非懒加载的单例。

preInstantiateSingletons主要有两个步骤：

- 获取非懒加载的单例，分为两种情况，类是FactoryBean的情况

- 对已经加载的单例，进行后初始化操作

  getSingleton方法去获取单例，在 getSingleton的时候，spring的默认实现是，先从singletonObjects寻找，如果找不到，再从earlySingletonObjects寻找，仍然找不到，那就从singletonFactories寻找对应的制造singleton的工厂，然后调用工厂的getObject方法，造出对应的SingletonBean，并放入earlySingletonObjects中。

初始化类的单元测试：

```
@Test
public void testUnreferencedSingletonWasInstantiated() {
	KnowsIfInstantiated.clearInstantiationRecord();
	DefaultListableBeanFactory lbf = new DefaultListableBeanFactory();
	Properties p = new Properties();
	p.setProperty("x1.(class)", KnowsIfInstantiated.class.getName());
	assertTrue("singleton not instantiated", !KnowsIfInstantiated.wasInstantiated());
	(new PropertiesBeanDefinitionReader(lbf)).registerBeanDefinitions(p);
	lbf.preInstantiateSingletons();
	assertTrue("singleton was instantiated", KnowsIfInstantiated.wasInstantiated());
}
```

在这里，我们换种方式去跟踪代码，以上面的这个单元测试作为入口，进行Debug。

- 断点位置： DefaultListableBeanFactory的registerBeanDefinition

  回顾一下，XmlBeanDefinitionReader中的registerBeanDefinitions函数把生成的GenericBeanDefinition注册进入DefaultListableBeanFactory。简单来说，就是将其放入DefaultListableBeanFactory的beanDefinitionMap中。

  ```
  this.beanDefinitionMap.put(beanName, beanDefinition);
  this.beanDefinitionNames.add(beanName);
  this.manualSingletonNames.remove(beanName);
  ```

- 断点位置：DefaultListableBeanFactory的preInstantiateSingletons

  循环this.beanDefinitionNames，根据beanName获取RootBeanDefinition，注意这里并不是GenericBeanDefinition。关于GenericBeanDefinition怎么转换成RootBeanDefinition，具体可以查看getMergedBeanDefinition方法。


#### getMergedBeanDefinition

这个方法里用了同步关键字synchronized来保证对this.mergedBeanDefinitions的多线程保护。

- 如果bean的parentName为空，就根据beanDefinition生成RootBeanDefinition，并缓存至`this.mergedBeanDefinitions`

- 如果parentName不为空，则需要与父节点的BeanDefinition进行合并

- 还需要考虑当前BeanFactory有parent的情况

#### doGetBean

- doGetBean方法刚开始的会调用getSingleton(beanName)方法去查看bean是否在缓存`singletonObjects`或`earlySingletonObjects`中

- 如果getSingleton获取到的Instance非空时，则会调用getObjectForBeanInstance

- 如果beanName在当前的BeanFactory中找不到，则到parent的BeanFactory中去查找并实例化bean并返回

- 循环所有的依赖Bean，根据依赖情况添加到dependentBeanMap和dependenciesForBeanMap中，并先实例化依赖bean

- 如果是Scope是单例，调用getSingleton(String beanName, ObjectFactory<?> singletonFactory)方法，这个方法会先从缓存`singletonObjects`中尝试读取，如果不存在，则创建并将其加入到缓存中，处理流程简写如下：

  ```
  singletonObject = this.singletonObjects.get(beanName)
  beforeSingletonCreation(beanName);
  singletonObject = singletonFactory.getObject();
  afterSingletonCreation(beanName);
  addSingleton(beanName, singletonObject);
  ```
  这里`singletonFactory.getObject()`实际上是调用了`createBean(beanName, mbd, args)`

- 如果Scope是Prototype，则会创建新的实例

  ```
  beforePrototypeCreation(beanName)
  prototypeInstance = createBean(beanName, mbd, args);
  afterPrototypeCreation(beanName);
  ```

- 如果Scope是其他，则会通过Scope去获取创建实例

  ```
  Scope scope = this.scopes.get(scopeName);
  scope.get(String name, ObjectFactory<?> objectFactory)
  ```

  scope.get函数的第二个参数是一个匿名类，其定义的getObject方法时间上做的事:

  ```
  beforePrototypeCreation(beanName)
  prototypeInstance = createBean(beanName, mbd, args);
  afterPrototypeCreation(beanName);
  ```

所以大体上不管Scope是什么，其创建类的核心逻辑都是`createBean(beanName, mbd, args)`，且在最后都会调用`getObjectForBeanInstance(scopedInstance, name, beanName, mbd)`来获取实例。


##### getSingleton

doGetBean方法刚开始的会调用getSingleton(beanName)方法：

```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

这个方法涉及到循环依赖的检测：

singletonObjects：用于保存BeanName和创建bean实例之间的关系

singletonFactories：用于保存BeanName和创建bean的工厂之间的关系

earlySingletonObjects：也是保存BeanName和创建bean实例之间的关系，与singletonObjects的不同之处在于，当一个单例bean被放到这里面后，那么当bean还在创建过程中，就可以通过getBean方法获取到了，其目的是用来检测循环引用。

registeredSingletons：用来保存当前所有已注册的bean


##### getObjectForBeanInstance

`getObjectForBeanInstance(scopedInstance, name, beanName, mbd)`是创建实例的必经步骤。其主要针对传入的scopedInstance是否是FactoryBean。

- 如果不是FactoryBean或者是FactoryBean但name是"&"为前缀，则直接返回scopedInstance

- 如果是FactoryBean且name不是"&"为前缀，则先尝试从缓存`factoryBeanObjectCache`中获取，如果不存在，调用getObjectFromFactoryBean来获取实例





<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
