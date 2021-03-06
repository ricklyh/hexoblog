---
title: myBatis源码分析
date: 2018-04-12 21:12:04
tags: [myBatis]
categories: myBatis
---
  - 在上一篇文章的基础上，我们从代码级别来分析一下MyBatis的执行过程

  - Mybatis的运行分为两大部分，第一部分是读取配置文件到Configuration对象，用以创建SqlSessionFactory，第二部分是SqlSession的执行过程

## 构建SqlSessionFactory的过程
  - 代码如下：
   ```
       String resource = "mybatis-config.xml";
       InputStream inputStream = Resources.getResourceAsStream(resource);
       SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
   ```
  - 通过上面代码，我们看看SqlSessionFactory的构建过程
  - SqlSessionFactoryBuilder是一个builder类，它提供了多种build方法。build方法最终调用的以下两句话：  
   ```
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
      
      // build(parser.parse())调用的下面的方法
      public SqlSessionFactory build(Configuration config) {
          return new DefaultSqlSessionFactory(config);
      }
   ```
  - 可以看到SqlSessionFactoryBuilder的构建分为两步：
   + 通过XMlConfigBuilder来解析XML,并将读取的配置保存在Configuration来中，这个类保存来myBatis涉及到的所有配置
   + 使用Configuration来构建SqlSessionFactory，SqlSessionFactory是一个接口，mybatis默认提供了DefaultSqlSessionFactory实现类

## 构建Configuration
  - 构建SqlSessionFactory过程中，Configuration最重要的。它构建的代码如下：
   ```
      public Configuration parse() {
          if (parsed) {
            throw new BuilderException("Each XMLConfigBuilder can only be used once.");
          }
          parsed = true;
          parseConfiguration(parser.evalNode("/configuration"));
          return configuration;
      }
      
      private void parseConfiguration(XNode root) {
          try {
              propertiesElement(root.evalNode("properties"));
              Properties settings = settingsAsProperties(root.evalNode("settings"));
              loadCustomVfs(settings);
              typeAliasesElement(root.evalNode("typeAliases"));
              pluginElement(root.evalNode("plugins"));
              objectFactoryElement(root.evalNode("objectFactory"));
              objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
              reflectorFactoryElement(root.evalNode("reflectorFactory"));
              settingsElement(settings);
              environmentsElement(root.evalNode("environments"));
              databaseIdProviderElement(root.evalNode("databaseIdProvider"));
              typeHandlerElement(root.evalNode("typeHandlers"));
              mapperElement(root.evalNode("mappers"));
          } catch (Exception e) {
              throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
          }
      }
     
  ```
  - 我们看到parse()方法会判断Configuration是否已经构建过了，它只会构建一次.然后调用parseConfiguration(XNode root)方法解析XML文件中各种配置信息，
     将这些信息保存到Configuraion中，包括：
     +  properties全局参数
     +  settings设置
     +  typeAliases别名
     +  typeHandler类型处理器
     +  ObjectFactory对象
     +  plugin插件
     +  environment环境
     +  DatabaseIdProvider数据库标识
     +  Mapper映射器
  - 来看两个比较重要的配置，environment , Mapper映射器
   
## environment
  - environment的构建在environmentsElement方法中：
   ```
      private void environmentsElement(XNode context) throws Exception {
          if (context != null) {
              if (environment == null) {
                  environment = context.getStringAttribute("default");
              }
          for (XNode child : context.getChildren()) {
              String id = child.getStringAttribute("id");
                  if (isSpecifiedEnvironment(id)) {
                      TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                      DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                      DataSource dataSource = dsFactory.getDataSource();
                      Environment.Builder environmentBuilder = new Environment.Builder(id)
                          .transactionFactory(txFactory)
                          .dataSource(dataSource);
                      configuration.setEnvironment(environmentBuilder.build());
                  }
              }
          }
      }
   ```
  - 可以看到，environment的构建分为4步
     + 获取ID属性，保证xml中environments节点下的environment的唯一
     + 获取transactionManager属性构建TransactionFactory
     + 获取dataSource属性构建DataSource
     + 最后通过前面的id、TransactionFactory、DataSource创建Environment
## mappers
  - mappers的构建在mapperElement方法中：
   ```
      private void mapperElement(XNode parent) throws Exception {
          if (parent != null) {
              for (XNode child : parent.getChildren()) {
                  if ("package".equals(child.getName())) {
                      String mapperPackage = child.getStringAttribute("name");
                      configuration.addMappers(mapperPackage);
                  } else {
                      String resource = child.getStringAttribute("resource");
                      String url = child.getStringAttribute("url");
                      String mapperClass = child.getStringAttribute("class");
                      if (resource != null && url == null && mapperClass == null) {
                          ErrorContext.instance().resource(resource);
                          InputStream inputStream = Resources.getResourceAsStream(resource);
                          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                          mapperParser.parse();
                      } else if (resource == null && url != null && mapperClass == null) {
                          ErrorContext.instance().resource(url);
                          InputStream inputStream = Resources.getUrlAsStream(url);
                          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                          mapperParser.parse();
                      } else if (resource == null && url == null && mapperClass != null) {
                          Class<?> mapperInterface = Resources.classForName(mapperClass);
                          configuration.addMapper(mapperInterface);
                      } else {
                          throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                      }
                  }
              }
          }
      }
   ```
  - 遍历mappers下所有的mapper定义，根据mapper不同类型的定义将Mapper加入configuration中
   
     - 如果定义的是package或者mapper定义了class,调用MapperRegistry.addMapper(Class<T> type)，在其中调用MapperAnnotationBuilder.parse()方法解析Mapper类
       - 解析对应的xml文件，这个文件必须和Mapper类处于同一package下
       - 解析cache
       - 解析cache-ref 
       - 解析每个method，生成MappedStatement
       
     - 如果mapper定义了resource或者定义了url,调用XMLMapperBuilder.configurationElement()方法解析xml文件:
       - 解析namespace
       -  解析cache-ref：其他命名空间缓存配置的引用
       -  解析cache：给定命名空间的缓存配置
       -  解析parameterMap：已废弃
       -  解析resultMap：描述如何从数据库结果集中来记载对象
       -  解析sql：可被其他语句引用的可重用语句块
       -  解析insert|update|delete|select语句
       
  - 经过上面的解析，Environment信息放在Configuration的environment变量中。
     节点的映射信息(select|insert|delete|update)封装成MappedStatement放在Configuration的mappedStatements变量中。
     结果的映射信息封装成ResultMap放在Configuration的resultMaps变量中。
    
## SqlSession的执行过程
  - ### openSession
   - SqlSession最佳的作用域是请求或方法作用域。每次使用SqlSession的时候都需要调用SqlSessionFactory.openSession()方法来获取一个新的SqlSession。SqlSession的新建过程如下：
   ```
     final Environment environment = configuration.getEnvironment();
     final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
     tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
     final Executor executor = configuration.newExecutor(tx, execType);
     return new DefaultSqlSession(configuration, executor, autoCommit);
   ```
   - 可以看到使用Configuration和Executor来新建DefaultSqlSession。Executor中包含了Transaction
     
   - ### 获取Mapper
     - 接下去调用getMapper方法从SqlSession中获取Mapper实例,这个在MapperRegistry.getMapper方法中进行，主要代码如下：
   ```
      final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
      return mapperProxyFactory.newInstance(sqlSession);
      
      首先获取MapperProxyFactory，然后调用newInstance方法：
      
      public T newInstance(SqlSession sqlSession) {
          final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
          return newInstance(mapperProxy);
      }
      
      protected T newInstance(MapperProxy<T> mapperProxy) {
          return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
      }
      
      可以看到，这里根据我们自己定义的mapper接口创建了一个动态代理，代理类是MapperProxy。
   ```
 - ### Mapper执行
    - 前面说到，获取mapper时返回的是一个代理类。因此执行mapper方法时调用的是MapperProxy的invoke方法：
  ```
      final MapperMethod mapperMethod = cachedMapperMethod(method);
      return mapperMethod.execute(sqlSession, args);
      
      根据原始的method生成一个MapperMethod，然后调用MapperMethod的execute方法。下面以SELECT类型来看看execute方法的执行过程：
      
      if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
      } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
      } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
      }
      
   ```
   - 首先调用method.convertArgsToSqlCommandParam方法获取参数表ParamMap
   - 接着调用DefaultSqlSession.selectList方法：
   ```
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
      
      说明：
          从configuration中获取selectUser方法对应的MappedStatement
          调用executor的query方法
      
   ```
   - 在进入真正的执行方法之前，先了解一下4个重要的对象：
     + Executor：执行器，由它来调度StatementHandler、ParameterHandler、ResultHandler等来执行对应的SQL
     + StatementHandler：作用是使用数据库的Statement(PreparedStatement)执行操作，它是四大对象的核心，起到承上启下的作用
     + ParameterHandler：用于SQL对参数的处理
     + ResultHandler：用于最后数据集(ResultSet)的封装返回处理
       
   - Mapper执行的过程是通过Executor、StatementHandler、ParameterHandler、ResultHandler来完成数据库操作和结果返回的。
       
## Executor.query
  - 执行器(Executor)起到了至关重要的作用。它是一个真正执行Java和数据库交互的东西。在MyBatis中存在三种执行器。我们可以在Mybatis的配置文件中进行选择（setting元素的属性defaultExecutorType）:
     + SIMPLE，简易执行器，不配置它就是默认执行器
     + REUSE，会重用预处理语句(prepared statements)
     + BATCH，执行器将重用语句并执行批量更新
  - BaseExecutor.query方法：
     ```
       public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
           BoundSql boundSql = ms.getBoundSql(parameter);
           CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
           return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
       }
     ```
   - 首先获取BoundSql实例，它是建立SQL和参数的地方，有3个常用属性：
     + SQL：我们书写在映射器里面的一条SQL
     + parameterObject：参数表
     + parameterMappings：一个List，每个元素都是一个ParameterMapping的对象。这个对象会描述我们的参数，包括：属性、名称、表达式、javaType、jdbcType、typeHandler等重要信息。通过它可以实现参数和SQL的结合，以便PreparedStatement能够通过它找到parameterObject对象的属性并设置参数，使得程序准确运行
   
   - 创建CacheKey
   
   - 调用另一个重载的query方法，query方法中调用BaseExecutor.queryFromDatabase方法，queryFromDatabase方法中调用Executor.doQuery方法
       
## Executor.doQuery
   ```
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
   - doQuery方法执行分为以下几个步骤：
     + 获取Configuration
     + 根据Configuration来创建StatementHandler
     + 调用prepareStatement方法创建Statement
     + 调用StatementHandler的query方法。
   
   - StatementHandler，数据库会话器(StatementHandler)是专门处理数据库会话的
   
   - 创建StatementHandler，StatementHandler在Configuration中创建：
   ```
      public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
          StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
          statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
          return statementHandler;
      }
   ```
   - 可以看到，创建的真实对象是RoutingStatementHandler对象，它实现了接口StatementHandler
   - RoutingStatementHandler不是我们真实的服务对象，它是通过适配模式找到对应的StatementHandler来执行的。
     在MyBatis中，StatementHandler和Executor一样分为三种：SimpleStatementHandler、PreparedStatementHandler、CallableStatementHandler，他们分别对应三种Executor
   - RoutingStatementHandler定义了一个对象的适配器delegate，它是一个StatementHandler接口对象，构造方法根据配置来适配对应的StatementHandler对象。
     它的作用是给实现类对象的使用提供一个统一、简易的使用适配器。此为对象的适配模式，可以让我们使用现有的类和方法对外提供服务，也可以根据实际的需求对外屏蔽一些方法，甚至是加入新的服务
   
   - 下面我们以最常用的PreparedStatementHandler为例，看看它三个主要的方法：prepare、parameterize、query
   
   - prepare
   ```
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
      
      1 instantiateStatement()方法对SQL进行预编译，获得Statement
      2 超时设置
      3 获取的最大行数等的设置
   ```
   - parameterize
     + parameterize方法用于设置参数，它调用ParameterHandler的setParameters方法来完成。
     + 参数处理器(ParameterHandler)用于对预编译语句进行参数设置。它有两个方法：getParameterObject方法作用是返回参数对象，setParameters方法作用是设置预编译SQL语句的参数。
     + setParameters方法步骤如下：
                                从parameterObject对象中获取参数
                                然后使用typeHandler进行参数处理
   
   - query
     + 由于在执行前参数和SQL都被prepare()方法预编译，参数在parameterize()方法已经进行了设置。所以这里只要执行SQL，然后返回结果就可以了。
     ```
        public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
            PreparedStatement ps = (PreparedStatement) statement;
            ps.execute();
            return resultSetHandler.<E> handleResultSets(ps);
        }
      说明：
          1 首先调用Statement的execute方法，Statement是一个动态代理类，实际执行的是PreparedStatementLogger.invoke方法，在invoke方法中执行PreparedStatement的execute方法。
        
          2 然后调用DefaultResultSetHandler.handleResultSets来解析结果集
     ```
## 结果处理器
 - 结果处理器接口ResultSetHandler有一个默认实现类DefaultResultSetHandler。重点关注handleResultSets方法：
  ```
       public List<Object> handleResultSets(Statement stmt) throws SQLException {
           ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
       
           final List<Object> multipleResults = new ArrayList<Object>();
       
           int resultSetCount = 0;
           // 将ResultSet包装成ResultSetWrapper
           ResultSetWrapper rsw = getFirstResultSet(stmt);
       
           List<ResultMap> resultMaps = mappedStatement.getResultMaps();
           int resultMapCount = resultMaps.size();
           validateResultMapsCount(rsw, resultMapCount);
           while (rsw != null && resultMapCount > resultSetCount) {
               ResultMap resultMap = resultMaps.get(resultSetCount);
               // handleResultSet方法是关键：根据resultMap将rsw中的结果数据设置到multipleResults列表中
               handleResultSet(rsw, resultMap, multipleResults, null);
               rsw = getNextResultSet(stmt);
               cleanUpAfterHandlingResultSet();
               resultSetCount++;
           }
       
           String[] resultSets = mappedStatement.getResultSets();
           if (resultSets != null) {
               while (rsw != null && resultSetCount < resultSets.length) {
                   ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
                   if (parentMapping != null) {
                       String nestedResultMapId = parentMapping.getNestedResultMapId();
                       ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                       handleResultSet(rsw, resultMap, null, parentMapping);
                   }
                   rsw = getNextResultSet(stmt);
                   cleanUpAfterHandlingResultSet();
                   resultSetCount++;
               }
           }
       
           return collapseSingleResultList(multipleResults);
       }
       
       
  ```
  - DefaultResultSetHandler.handleResultSet方法的核心是下面的代码：
 ```
    DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
    // 调用handleRowValues解析查询结果，封装后的对象保存在defaultResultHandler中
    handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
    multipleResults.add(defaultResultHandler.getResultList());
 ```                 
  - 如果没有嵌套查询，调用DefaultResultSetHandler.handleRowValuesForSimpleResultMap方法
  - handleRowValuesForSimpleResultMap方法遍历查询返回的ResultSet，
   调用getRowValue(ResultSetWrapper rsw, ResultMap resultMap)方法解析每行数据：
    + 调用createResultObject方法创建结果对象
    + 调用applyAutomaticMappings方法解析自动映射
    + 调用applyPropertyMappings方法解析手动的映射
    + 返回经过处理的结果对象

## SQL执行过程总结
  - Executor会先调用StatementHandler的prepare()方法预编译SQL语句，同时设置一些基本运行的参数。
  - 然后用parameterize()方法启用ParameterHandler设置参数，完成预编译。然后执行查询。
  - 最后使用ResultSetHandler封装结果返回给调用者