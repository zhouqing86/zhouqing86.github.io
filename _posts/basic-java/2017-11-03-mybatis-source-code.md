---
layout: post
comments: false
categories: Java基础
date:   2017-11-03 10:30:54
title: MyBatis源码浅解
---

<div id="toc"></div>

周末学习下MyBatis-3的源码。本文源码阅读的想法时由外至内，即从配置文件的处理至MyBatis的核心处理的过程。

## 配置文件处理

### Resources类

MyBatis 包含一个名叫 Resources 的工具类，它包含一些实用方法，可使从 classpath 或其他位置加载资源文件更加容易
如其提供了如下的方法：

```
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);

String property = "org/apache/ibatis/databases/blog/blog-derby.properties”
Properties props = Resources.getResourceAsProperties(property);

Class<?> boundType = Resources.classForName("org.apache.ibatis.domain.blog.mappers.NestedBlogMapper");
```

### 解析MyBatis的配置文件
Configuration类主要是用来存储对mybatis的配置文件及mapper文件解析后的数据，Configuration对象会贯穿整个myabtis的执行流程，为mybatis的执行过程提供必要的配置信息。那么在MyBatis中，XML是如何转换成一个Configuration类的呢？

#### XPathParser
XPathParser是MyBatis-3自定义的解析XML文件的类，我们可以先查看其单元测试:

```
@Test
public void shouldTestXPathParserMethods() throws Exception {
  String resource = "resources/nodelet_test.xml";
  InputStream inputStream = Resources.getResourceAsStream(resource);
  XPathParser parser = new XPathParser(inputStream, false, null, null);
  assertEquals((Long)1970l, parser.evalLong("/employee/birth_date/year"));
  assertEquals((short) 6, (short) parser.evalShort("/employee/birth_date/month"));
  assertEquals((Integer) 15, parser.evalInteger("/employee/birth_date/day"));
  assertEquals((Float) 5.8f, parser.evalFloat("/employee/height"));
  assertEquals((Double) 5.8d, parser.evalDouble("/employee/height"));
  assertEquals("${id_var}", parser.evalString("/employee/@id"));
  assertEquals(Boolean.TRUE, parser.evalBoolean("/employee/active"));
  assertEquals("<id>${id_var}</id>", parser.evalNode("/employee/@id").toString().trim());
  assertEquals(7, parser.evalNodes("/employee/*").size());
  XNode node = parser.evalNode("/employee/height");
  assertEquals("employee/height", node.getPath());
  assertEquals("employee[${id_var}]_height", node.getValueBasedIdentifier());
  inputStream.close();
}
```

nodelet_test.xml的内容如下:

```
<employee id="${id_var}">
  <blah something="that"/>
  <first_name>Jim</first_name>
  <last_name>Smith</last_name>
  <birth_date>
    <year>1970</year>
    <month>6</month>
    <day>15</day>
  </birth_date>
  <height units="ft">5.8</height>
  <weight units="lbs">200</weight>
  <active>true</active>
</employee>
```

在XPathParser的createDocument方法中获取Document对象，其使用的是javax.xml.parsers包下的类去解析XML，针对DOM的解析器相关类是DocumentBuilderFactory和DocumentBuilder。

```
DocumentBuilder builder = DocumentBuilderFactory.newInstance().newDocumentBuilder();
Document document = builder.parse(ClassLoader.getSystemResourceAsStream("config.xml"));
```

拿到Document对象后，就可以通过XPathFactory.newInstance().newXPath()去获取节点中的值了，如
```
XPath xpath = XPathFactory.newInstance().newXPath();
xpath.evaluate("/employee/active", document, XPathConstants.BOOLEAN);
```

#### XMLConfigBuilder
XPathParser解析了XML文件，XMLConfigBuilder就负责将解析好的document中的Node的值写入到Configuration类中。XMLConfigBuilder类的parse方法：

```
public Configuration parse() {
  if (parsed) {
    throw new BuilderException("Each XMLConfigBuilder can only be used once.");
  }
  parsed = true;
  parseConfiguration(parser.evalNode("/configuration"));
  return configuration;
}
```

此方法中获取XPathParser解析XML文件后获取/configuration节点。而其parseConfiguration方法将configuration节点写入到Configuration类的各个属性中：
```
private void parseConfiguration(XNode root) {
  try {
    //issue #117 read properties first
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    loadCustomVfs(settings);
    typeAliasesElement(root.evalNode("typeAliases"));
    pluginElement(root.evalNode("plugins"));
    objectFactoryElement(root.evalNode("objectFactory"));
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    settingsElement(settings);
    // read it after objectFactory and objectWrapperFactory issue #631
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    typeHandlerElement(root.evalNode("typeHandlers"));
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```

#### MapperRegistry
在XMLConfigBuilder的parseConfiguration方法中，最后调用了mapperElement方法，其实对XML中的mappers节点做解析，将其存储到MapperRegistry中。

```
<mappers>
  <mapper resource="org/apache/ibatis/builder/BlogMapper.xml"/>
  <mapper url="file:./src/test/java/org/apache/ibatis/builder/NestedBlogMapper.xml"/>
  <mapper class="org.apache.ibatis.builder.CachedAuthorMapper"/>
  <package name="org.apache.ibatis.builder.mapper"/>
</mappers>
```

这四中方式的mapper最终都会在MapperRegistry中增加到`knowMappers`中，以`NestedBlogMapper.xml`为例子，将有类似如下的步骤：

```
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();

Class<?> boundType = Resources.classForName("org.apache.ibatis.domain.blog.mappers.NestedBlogMapper");
knownMappers.put(boundType, new MapperProxyFactory<T>(boundType))
```

#### XMLMapperBuilder
XMLMapperBuilder是用来解析mapper.xml的，其底层用的也是XPathParser，并且将解析后的结果存储到Configuration中。

```
Configuration configuration = new Configuration();
String resource = "org/apache/ibatis/builder/AuthorMapper.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
XMLMapperBuilder builder = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
builder.parse();

MappedStatement mappedStatement = configuration.getMappedStatement("selectWithOptions");
assertThat(mappedStatement.getFetchSize()).isEqualTo(200);
assertThat(mappedStatement.getTimeout()).isEqualTo(10);
```

XMLMapperBuilder中的configurationElement方法定义了mapper.xml的解析的基本流程:

```
private void configurationElement(XNode context) {
  try {
    String namespace = context.getStringAttribute("namespace");
    if (namespace == null || namespace.equals("")) {
      throw new BuilderException("Mapper's namespace cannot be empty");
    }
    builderAssistant.setCurrentNamespace(namespace);
    cacheRefElement(context.evalNode("cache-ref"));
    cacheElement(context.evalNode("cache"));
    parameterMapElement(context.evalNodes("/mapper/parameterMap"));
    resultMapElements(context.evalNodes("/mapper/resultMap"));
    sqlElement(context.evalNodes("/mapper/sql"));
    buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
  }
}
```

#### XMLStatementBuilder
XMLMapperBuilder中buildStatementFromContext解析select/update/insert/delete的元素时会创建一个XMLStatementBuilder对象，解析的工作交由其方法parseStatementNode()方法完成。

在parseStatementNode中会创建SqlSource等，SqlSource中会存储SQL语句。如`select * from author`。最后调用的是builderAssistant.addMappedStatement方法，其会常见一个MappedStatement对象，注入SqlSource等，最终将这个MapperStatement添加到Configuration类的mappedStatements中。


## 数据库的连接管理

### DataSource
DataSource是一个java提供的一个接口，MyBatis-3中的实现类是UnpooledDataSource和PooledDataSource。MyBatis-3中配置一个数据源:

```
<dataSource type="UNPOOLED">
    <property name="driver" value="org.hsqldb.jdbcDriver"/>
    <property name="url" value="jdbc:hsqldb:mem:automapping"/>
    <property name="username" value="sa"/>
</dataSource>
```

在UnpooledDataSource中，获取与数据库的连接方式如下，每次都会获取一个新的连接:

```
private Connection doGetConnection(Properties properties) throws SQLException {
  initializeDriver();
  Connection connection = DriverManager.getConnection(url, properties);
  configureConnection(connection);
  return connection;
}
```

在PooledDataSource中，实现方式要复杂很多，这里不做具体说明。其主要逻辑在popConnection方法中。其使用了UnpooledDataSource来创建新连接，而使用PoolState类来存储空闲的数据库连接。

## 执行SQL语句
使用java.sql来做数据库查询时，其一般步骤如下：

```
Class.forName("com.mysql.jdbc.Driver");    
String url = "jdbc:mysql://localhost:3306/demo";    
String user = "root";    
String pass = "123456";    
conn = DriverManager.getConnection(url,user,pass);   
String sql="select * from users";    
PreparedStatement stmt = conn.prepareStatement(sql);  
ResultSet rs = stmt.execute();
while(rs.next()) {
  String result = rs.getString(columnName);
}
```

### MappedStatement
MappedStatement类在Mybatis框架中用于表示XML文件中一个sql语句节点，即一个<select />、<update />或者<insert />标签。Mybatis框架在初始化阶段会对XML配置文件进行读取，将其中的sql语句节点对象化为一个个MappedStatement对象。

在代码中获取一个查询语句的MappedStatement对象:

```
public static MappedStatement prepareSelectOneAuthorMappedStatement(final Configuration config) {
  final TypeHandlerRegistry registry = config.getTypeHandlerRegistry();

  final ResultMap rm = new ResultMap.Builder(config, "defaultResultMap", Author.class, new
      ArrayList<ResultMapping>() {
        {
          add(new ResultMapping.Builder(config, "id", "id", registry.getTypeHandler(int.class)).build());
          add(new ResultMapping.Builder(config, "username", "username", registry.getTypeHandler(String.class)).build());
          add(new ResultMapping.Builder(config, "password", "password", registry.getTypeHandler(String.class)).build());
          add(new ResultMapping.Builder(config, "email", "email", registry.getTypeHandler(String.class)).build());
          add(new ResultMapping.Builder(config, "bio", "bio", registry.getTypeHandler(String.class)).build());
          add(new ResultMapping.Builder(config, "favouriteSection", "favourite_section", registry.getTypeHandler(Section.class)).build());
        }
      }).build();

  MappedStatement ms = new MappedStatement.Builder(config, "selectAuthor", new StaticSqlSource(config,"SELECT * FROM author WHERE id = ?"), SqlCommandType.SELECT)
      .parameterMap(new ParameterMap.Builder(config, "defaultParameterMap", Author.class,
          new ArrayList<ParameterMapping>() {
            {
              add(new ParameterMapping.Builder(config, "id", registry.getTypeHandler(int.class)).build());
            }
          }).build())
      .resultMaps(new ArrayList<ResultMap>() {
        {
          add(rm);
        }
      })
      .cache(authorCache).build();
  return ms;
}
```

### SimpleExecutor/BaseExecutor

在BaseExecutorTest中，可以看到查询的过程：

```
@Test
  public void shouldSelectAllAuthorsAutoMapped() throws Exception {

    Executor executor = createExecutor(new JdbcTransaction(ds, null, false));
    try {
      MappedStatement selectStatement = ExecutorTestHelper.prepareSelectAllAuthorsAutoMappedStatement(config);
      List<Author> authors = executor.query(selectStatement, null, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER);
      assertEquals(2, authors.size());
      Author author = authors.get(0);
      // id,username, password, email, bio, favourite_section
      // (101,'jim','********','jim@ibatis.apache.org','','NEWS');
      assertEquals(101, author.getId());
      assertEquals("jim", author.getUsername());
      assertEquals("jim@ibatis.apache.org", author.getEmail());
      assertEquals("", author.getBio());
      assertEquals(Section.NEWS, author.getFavouriteSection());
    } finally {
      executor.rollback(true);
      executor.close(false);
    }
  }
```

createExecutor方法创建了一个SimpleExecutor类，注入了JdbcTransaction。而后 ExecutorTestHelper.prepareSelectAllAuthorsAutoMappedStatement创建了一个MappedStatement对象，调用executor.query方法就能从数据库中获取到Author列表了。

而在SimpleExecutor类中，执行查询的方法为doQuery：

```
@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.<E>query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

但是其还不是最终获取PreparedStatement以及执行查询的地方，其获取了StatementHandler对象并委托StatementHandler来进行查询。

注意这里Configuration类提供的newStatementHandler方法:

```
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}
```

在这里将配置中的plugin与statementHandler结合了起来，生成了代理类，这里也是MyBatis-3提供对插件支持的一个地方。


### RoutingStatementHandler/PreparedStatementHandler

RoutingStatementHandler是一个路由内，在其构造方法中，会根据MappedStatement的statementType来选择StatementHandler。

```
switch (ms.getStatementType()) {
  case STATEMENT:
    delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
    break;
  case PREPARED:
    delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
    break;
  case CALLABLE:
    delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
    break;
  default:
    throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
}
```

RoutingStatementHandler同时也是一个代理类，delegate就是其代理的对象，其实现了所有StatementHandler接口的方法，并转发给delegate成员变量的相应方法。

而PreparedStatementHandler方法RoutingStatementHandler的一个可能的delegate，其定义了获取PreparedStatementHandler的方法:

```
@Override
protected Statement instantiateStatement(Connection connection) throws SQLException {
  String sql = boundSql.getSql();
  if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
    String[] keyColumnNames = mappedStatement.getKeyColumns();
    if (keyColumnNames == null) {
      return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
    } else {
      return connection.prepareStatement(sql, keyColumnNames);
    }
  } else if (mappedStatement.getResultSetType() != null) {
    return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
  } else {
    return connection.prepareStatement(sql);
  }
}
```

其还真正执行了查询操作:

```
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();
  return resultSetHandler.<E> handleResultSets(ps);
}
```


## SQL回话管理
基本上，在MyBatis-3中，当我们拥有了Configuration对象以及Executor(包括statementHandler)等，我们就能进行对数据库的增删改查操作。但是为了让我们的上层代码写的更好看，MyBatis-3定义了SqlSession接口，让我们不必关心使用的Configuration或者哪个具体的Executor。


### SqlsessionFactory/DefaultSqlSessionFactory/SqlSessionFactoryBuilder
SqlSession是由SqlsessionFactory创建的，SqlsessionFactory又是通过SqlSessionFactoryBuilder基于配置文件创建的。

```
createBlogDataSource();
final String resource = "org/apache/ibatis/builder/MapperConfig.xml";
final Reader reader = Resources.getResourceAsReader(resource);
SqlSessionFactory sqlMapper = new SqlSessionFactoryBuilder().build(reader);
```

SqlSessionFactoryBuilder使用的是DefaultSqlSessionFactory来创建SqlSessionFactory。其可以通过openSession来获取到SqlSession对象。

```
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    final Executor executor = configuration.newExecutor(tx, execType);
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    closeTransaction(tx); // may have fetched a connection so lets call close()
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```


### SqlSession/DefaultSqlSession
通过SqlSessionFactory的openSession获取SqlSession对象后，就可以通过session来执行SQL语句了。

```
@Test
public void shouldSelectAllAuthors() throws Exception {
  SqlSession session = sqlMapper.openSession(TransactionIsolationLevel.SERIALIZABLE);
  try {
    List<Author> authors = session.selectList("org.apache.ibatis.domain.blog.mappers.AuthorMapper.selectAllAuthors");
    assertEquals(2, authors.size());
  } finally {
    session.close();
  }
}
```
这里对应的TransactionIsolationLevel有NONE, READ_COMMITTED, READ_UNCOMMITTED, REPEATABLE_READ, SERIALIZABLE。

而SqlSession的selectList方法中是通过Configuration对象获取到MapperStatement对象，而后通过executor去执行SQL。

```
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

## 事务管理

### TransactionFactory
XMLConfigBuilder从配置文件中获取事务相关配置并解析的代码:

```
private TransactionFactory transactionManagerElement(XNode context) throws Exception {
  if (context != null) {
    String type = context.getStringAttribute("type");
    Properties props = context.getChildrenAsProperties();
    TransactionFactory factory = (TransactionFactory) resolveClass(type).newInstance();
    factory.setProperties(props);
    return factory;
  }
  throw new BuilderException("Environment declaration requires a TransactionFactory.");
}
```

这里会根据type去获取JdbcTransactionFactory或ManagedTransactionFactory。


### JdbcTransaction

使用JDBC的事务管理机制：即利用java.sql.Connection对象完成对事务的提交（commit()）、回滚（rollback()）、关闭（close()）等。

JdbcTransaction中主要的三个成员变量是：

```
protected Connection connection;
protected DataSource dataSource;
protected TransactionIsolationLevel level;
```

而关于事务的提交与回滚：

```
@Override
public Connection getConnection() throws SQLException {
  if (connection == null) {
    openConnection();
  }
  return connection;
}

@Override
public void commit() throws SQLException {
  if (connection != null && !connection.getAutoCommit()) {
    if (log.isDebugEnabled()) {
      log.debug("Committing JDBC Connection [" + connection + "]");
    }
    connection.commit();
  }
}
```


### ManagedTransaction
使用MANAGED的事务管理机制：这种机制MyBatis自身不会去实现事务管理，而是让程序的容器如（JBOSS，Weblogic）来实现对事务的管理。

其实现中提交和混滚方法中不做任何处理:

```
@Override
public void commit() throws SQLException {
  // Does nothing
}

@Override
public void rollback() throws SQLException {
  // Does nothing
}
```


## 参数映射和结果映射

此部分内容后续如果有时间再添加到本文中。



## 参考资料
- [MyBatis文档](http://www.mybatis.org/mybatis-3/zh/index.html)

- [MyBatis configuration配置详解](http://blog.csdn.net/u014034854/article/details/47375955)

- [Mybatis数据源与连接池](http://blog.csdn.net/luanlouis/article/details/37671851)

- [MyBatis中的插件原理和应用](http://blog.csdn.net/ykzhen2015/article/details/50349540)

- [MyBatis事务管理机制](http://blog.csdn.net/luanlouis/article/details/37992171)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
