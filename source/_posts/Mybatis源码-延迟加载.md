---
title: Mybatis源码-延迟加载
date: 2019-06-24 16:31:32
tags:
- 源码
- Mybatis
---
# Description
摘自Mybatis3官网：

> 我们有两个 select 查询语句：一个用来加载博客（Blog），另外一个用来加载作者（Author），而且博客的结果映射描述了应该使用 selectAuthor 语句加载它的 author 属性。

> 其它所有的属性将会被自动加载，只要它们的列名和属性名相匹配。

> 这种方式虽然很简单，但在大型数据集或大型数据表上表现不佳。这个问题被称为“N+1 查询问题”。 概括地讲，N+1 查询问题是这样子的：

> **你执行了一个单独的 SQL 语句来获取结果的一个列表（就是“+1”）。**
> **对列表返回的每条记录，你执行一个 select 查询语句来为每条记录加载详细信息（就是“N”）。**
> **这个问题会导致成百上千的 SQL 语句被执行。有时候，我们不希望产生这样的后果。**

> MyBatis 能够对这样的查询进行延迟加载，因此可以将大量语句同时运行的开销分散开来。

延迟加载指按照指定的延迟规则来推迟关联查询的执行时间以达到提升数据库查询性能的目的；也可以理解为将多表关联查询语句拆分为多个单表查询语句，在需要返回数据时执行对应的语句。
![][image-1]

开启aggressiveLazyLoading会在任何方法调用时触发加载；而开启lazyLoadingEnabled会使关联对象按需加载，这个配置会在后续的源码中体现。
# Source Code
## DEBUG
Mybatis的源码中有专门基于LazyLoader的测试类来帮助我们进行DEBUG。
\@: org.apache.ibatis.submitted.lazyload\_common\_property
```html
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="org.apache.ibatis.submitted.lazyload_common_property.ChildMapper">
    
    <resultMap id="ChildMap" type="Child">
        <id property="id" column="id"/>
        <result property="name" column="name"/>
        <association property="father" column="father_id" select="org.apache.ibatis.submitted.lazyload_common_property.FatherMapper.selectById"/>
        <association property="grandFather" column="grand_father_id" select="org.apache.ibatis.submitted.lazyload_common_property.GrandFatherMapper.selectById"/>
    </resultMap>
    
    
    <select id="selectById" resultMap="ChildMap" parameterType="int">
    	SELECT id, name, father_id, grand_father_id
    	FROM Child
    	WHERE id = #{id}
    </select>
</mapper>
```
## Code Analysis
```js
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      boolean foundValues = this.useConstructorMappings;
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
      }
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
  }
```
从getRowValue开始跟，只看和延迟加载相关的部分：
1. createResultObject初始化Mapper配置的对应Bean

```js
  private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new ArrayList<>();
    final List<Object> constructorArgs = new ArrayList<>();
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
      for (ResultMapping propertyMapping : propertyMappings) {
        // issue gcode #109 && issue #149
        if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
          resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
          break;
        }
      }
    }
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
  }
```
当resultMap的propertyMappings的任一propertyMapping存在关联查询且开启延迟加载，Mybatis会为该resultObject创建动态代理拦截调取该Bean的方法进行延迟加载处理。
```js
public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
  final String methodName = method.getName();
  try {
    synchronized (lazyLoader) {
      if (WRITE_REPLACE_METHOD.equals(methodName)) {
        Object original;
        if (constructorArgTypes.isEmpty()) {
          original = objectFactory.create(type);
        } else {
          original = objectFactory.create(type, constructorArgTypes, constructorArgs);
        }
        PropertyCopier.copyBeanProperties(type, enhanced, original);
        if (lazyLoader.size() > 0) {
          return new JavassistSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
        } else {
          return original;
        }
      } else {
        if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
          System.out.println("Size of LazyLoader Map:" + lazyLoader.size());
          if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
            lazyLoader.loadAll();
          } else if (PropertyNamer.isSetter(methodName)) {
            final String property = PropertyNamer.methodToProperty(methodName);
            lazyLoader.remove(property);
          } else if (PropertyNamer.isGetter(methodName)) {
            final String property = PropertyNamer.methodToProperty(methodName);
            if (lazyLoader.hasLoader(property)) {
              lazyLoader.load(property);
            }
          }
        }
      }
    }
    return methodProxy.invoke(enhanced, args);
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
  }
```
invoke时会同步代码块，因为lazyLoader里面是HashMap,线程不安全；
判断lazyLoader的size，如果开启agressiveLazyLoading，则方法被调用时直接load lazyLoaderMap里面所有的延迟加载项；
如果是Setter方法则与延迟加载无关，直接删掉；
如果是Getter则根据指定的方法调用加载对应的关联查询；

OK，其实大致的逻辑就是这样，那么LazyLoader这个东西是什么时候初始化的呢？现在跳回getRowValue，分析另一个比较重要的方法applyPropertyMappings。
2. applyPropertyMappings为对应属性赋值

```js
  private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
        column = null;
      }
      if (propertyMapping.isCompositeResult()
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
          || propertyMapping.getResultSet() != null) {
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // issue #541 make property optional
        final String property = propertyMapping.getProperty();
        if (property == null) {
          continue;
        } else if (value == DEFERED) {
          foundValues = true;
          continue;
        }
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          metaObject.setValue(property, value);
        }
      }
    }
    return foundValues;
  }
```
直接跳进getPropertyMappingValue；
```js
  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    if (propertyMapping.getNestedQueryId() != null) {
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERED;
    } else {
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }
```
继续跳进getNestedQueryMappingValue；
```js
  private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    final String property = propertyMapping.getProperty();
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
      final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
      final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
      final Class<?> targetType = propertyMapping.getJavaType();
      if (executor.isCached(nestedQuery, key)) {
        executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
        value = DEFERED;
      } else {
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        if (propertyMapping.isLazy()) {
          lazyLoader.addLoader(property, metaResultObject, resultLoader);
          value = DEFERED;
        } else {
          value = resultLoader.loadResult();
        }
      }
    }
    return value;
  }
```
看到了我们在上面引用到的LazyLoader，当延迟加载开启时，LazyLoader会把对应的属性meta信息存到内部的HashMap中，在方法被invoke时执行对应的Load操作；
```js
public void load() throws SQLException {
  /* These field should not be null unless the loadpair was serialized.
   * Yet in that case this method should not be called. */
  if (this.metaResultObject == null) {
    throw new IllegalArgumentException("metaResultObject is null");
  }
  if (this.resultLoader == null) {
    throw new IllegalArgumentException("resultLoader is null");
  }

  this.load(null);
}

public void load(final Object userObject) throws SQLException {
  if (this.metaResultObject == null || this.resultLoader == null) {
    if (this.mappedParameter == null) {
      throw new ExecutorException("Property [" + this.property + "] cannot be loaded because "
              + "required parameter of mapped statement ["
              + this.mappedStatement + "] is not serializable.");
    }

    final Configuration config = this.getConfiguration();
    final MappedStatement ms = config.getMappedStatement(this.mappedStatement);
    if (ms == null) {
      throw new ExecutorException("Cannot lazy load property [" + this.property
              + "] of deserialized object [" + userObject.getClass()
              + "] because configuration does not contain statement ["
              + this.mappedStatement + "]");
    }

    this.metaResultObject = config.newMetaObject(userObject);
    this.resultLoader = new ResultLoader(config, new ClosedExecutor(), ms, this.mappedParameter,
            metaResultObject.getSetterType(this.property), null, null);
  }

  /* We are using a new executor because we may be (and likely are) on a new thread
   * and executors aren't thread safe. (Is this sufficient?)
   *
   * A better approach would be making executors thread safe. */
  if (this.serializationCheck == null) {
    final ResultLoader old = this.resultLoader;
    this.resultLoader = new ResultLoader(old.configuration, new ClosedExecutor(), old.mappedStatement,
            old.parameterObject, old.targetType, old.cacheKey, old.boundSql);
  }

  this.metaResultObject.setValue(property, this.resultLoader.loadResult());
}
```
LazyLoader(ResultLoaderMap)会把load操作委托给内部类LoadPair，LoadPair又会委托给ResultLoader；
整个调用链其实非常长。。但逻辑并不复杂，这里抽取了一个ResultLoader Depth=2的时序图：
![][image-2]
ResultLoaderMap会判断当前线程是否是Executor的创建线程，如果不是则创建一个Executor来执行对应的Query（同样线程不安全），完成后调用extractObjectFromList将执行结果fetchInto到resultObject
# Tips
其实这块内容没多复杂，但是为什么要记下来呢，因为有一个很坑的地方是这块内容如果你在IDEA进行DEBUG的话，是永远DEBUG不到你的断点里面的。因为你进行了延迟加载配置，但你在DEBUG里面Watch这个延迟加载的变量时，其实已经触发了这个变量的加载，所以你是永远也进不到这个断点里面的。由于这个原因我一度怀疑自己看错了代码。。。哈哈哈

[image-1]:	http://pt1160j8s.bkt.clouddn.com/lazyloading_property.png
[image-2]:	http://pt1160j8s.bkt.clouddn.com/mybatis_lazyloader_resultloader_sequence.png "ResultLoader时序图"