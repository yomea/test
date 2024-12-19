- FieldEncryptInterceptor

```java
package com.example.demo.sulaoban;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.math.BigDecimal;
import java.sql.PreparedStatement;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import org.apache.ibatis.executor.parameter.ParameterHandler;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Signature;

@Intercepts({@Signature(type = ParameterHandler.class, method = "setParameters", args = {
    PreparedStatement.class})})
public class FieldEncryptInterceptor implements Interceptor {

    private static final List<Class<?>> CLASS_LIST = Arrays
        .asList(Object.class, String.class, BigDecimal.class, double.class, Double.class
            , int.class, Integer.class, long.class, Long.class, float.class, Float.class, short.class, Short.class
            , byte.class, Byte.class, boolean.class, Boolean.class);

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        if (SystemConditionControl.matchMarket(MarketConstant.EU)) {
            ParameterHandler parameterHandler = (ParameterHandler) invocation.getTarget();
            Object parameter = parameterHandler.getParameterObject();
            this.execEncrypt(parameter);
        }
        return invocation.proceed();
    }

    private void execEncrypt(Object parameter) throws IllegalAccessException {
        if (Objects.isNull(parameter)) {
            return;
        }
        if(parameter instanceof Map) {
            Map map = (Map)parameter;
            map.values().stream().distinct()
                .forEach(item -> {
                    this.doGetEncryptVal(item, null);
//                    this.process(item);
                });
        } else {
            this.doGetEncryptVal(parameter, null);
//                    this.process(item);
        }
    }

    //只处理bean
    private void process(Object parameter) {
        if (Objects.isNull(parameter)) {
            return;
        }
        Class<?> clazz = parameter.getClass();
        if (CLASS_LIST.contains(clazz) || clazz.getName().startsWith("java.lang")) {
            return;
        }
        // 只处理java bean，这种字段上才可能存在注解
        List<Field> fieldList = this.getFieldList(clazz);
        this.doProcess(parameter, fieldList);

    }

    public List<Field> getFieldList(Class<?> clazz) {
        List<Field> fieldList = new ArrayList<>();
        this.getFields(fieldList, clazz);
        return fieldList;
    }

    private void doProcess(Object bean, List<Field> fieldList) {

        fieldList.stream().forEach(field -> {

            field.setAccessible(true);
            try {
                field.set(bean, this.getEncryptVal(bean, field));
            } catch (IllegalAccessException e) {
                throw new RuntimeException("方法不允许访问！");
            }
        });
    }

    private Object getEncryptVal(Object parameter, Field field) throws IllegalAccessException {
        Object fieldBean = field.get(parameter);
        if (fieldBean == null) {
            // 没有值，不需要操作
            return null;
        }
        return this.doGetEncryptVal(fieldBean, field);
    }

    private Object doGetEncryptVal(Object fieldBean, Field field) {
        if(Objects.isNull(fieldBean)) {
            return null;
        }
        // 字段类型
        Class<?> clazz = fieldBean.getClass();
        // 只对标有注解的String类型java bean字段做加密
        if (clazz.isArray()) {
            Object[] c = (Object[]) fieldBean;
            for(Object item : c) {
                this.doGetEncryptVal(item, null);
            }
        } else if (Iterable.class.isAssignableFrom(clazz)) {
            Iterable c = (Iterable) fieldBean;
            for(Object item : c) {
                this.doGetEncryptVal(item, null);
            }
        } else if (Map.class.isAssignableFrom(clazz)) {
            Map map = (Map)fieldBean;
            map.values().stream().forEach(item -> {
                this.doGetEncryptVal(item, null);
            });
        } else if (String.class.isAssignableFrom(clazz)) {
            boolean encrypt = this.isEncrypt(field);
            return this.encryptNess( (String) fieldBean, encrypt);
        } else {
            this.process(fieldBean);
        }
        return fieldBean;
    }

    private Object encryptNess(String fieldBean, boolean encrypt) {
        if(!encrypt) {
            return fieldBean;
        }
        String encryptedValue = AesSecureHelper.encryptBase64(fieldBean);
        return encryptedValue;
    }

    private boolean isEncrypt(Field field) {
        if(Objects.isNull(field)) {
            return false;
        }
        Crypto cryptoAnnotation = field.getAnnotation(Crypto.class);
        if (cryptoAnnotation != null) {
            return cryptoAnnotation.encrypt();
        }
        return false;
    }

    private void getFields(List<Field> fieldList, Class<?> tClass) {

        if (tClass == null || tClass == Object.class || tClass.getName().startsWith("java.lang")) {
            return;
        }
        Field[] fields = tClass.getDeclaredFields();
        if (fields != null && fields.length > 0) {
            for (Field field : fields) {
                if (!Modifier.isStatic(field.getModifiers()) && !Modifier.isFinal(field.getModifiers())) {
                    fieldList.add(field);
                }

            }
        }
        this.getFields(fieldList, tClass.getSuperclass());
    }
}
```


- CustomInterceptor

```java

package com.banksteel.openerp.test;

import com.example.demo.plugin.CustomResultSetHandler;
import java.sql.PreparedStatement;
import java.util.List;
import java.util.Properties;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.executor.parameter.ParameterHandler;
import org.apache.ibatis.executor.resultset.DefaultResultSetHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.ParameterMapping;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;
import org.apache.ibatis.reflection.DefaultReflectorFactory;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.factory.DefaultObjectFactory;
import org.apache.ibatis.reflection.wrapper.DefaultObjectWrapperFactory;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

@Intercepts(
    {
        @Signature(type = ParameterHandler.class, method = "setParameters", args = {PreparedStatement.class}),
    }
)
public class CustomInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        try {
            ParameterHandler parameterHandler = (ParameterHandler) invocation.getTarget();
            MetaObject metaObject = MetaObject.forObject(parameterHandler, new DefaultObjectFactory(),
                new DefaultObjectWrapperFactory(), new DefaultReflectorFactory());
            MappedStatement ms = (MappedStatement)metaObject.getValue("mappedStatement");
            SqlCommandType sqlCommandType = ms.getSqlCommandType();
            if(SqlCommandType.INSERT == sqlCommandType
                || SqlCommandType.UPDATE == sqlCommandType) {
                BoundSql boundSql = (BoundSql)metaObject.getValue("boundSql");
                Configuration configuration = (Configuration)metaObject.getValue("configuration");
                List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
                Object parameterObject = boundSql.getParameterObject();
                MetaObject metaParameterObject = configuration.newMetaObject(parameterObject);
                parameterMappings.forEach(e -> {
                    Object val = metaParameterObject.getValue(e.getProperty());
                    //TODO：判断是否是需要加密的字段，如果是加密，这里的逻辑你先想好怎么操作
                    metaParameterObject.setValue(e.getProperty(), val);
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return invocation.proceed();
    }


    @Override
    public Object plugin(Object target) {
        if (target instanceof DefaultResultSetHandler) {
            DefaultResultSetHandler setHandler = (DefaultResultSetHandler) target;
            MetaObject metaObject = MetaObject.forObject(setHandler, new DefaultObjectFactory(),
                new DefaultObjectWrapperFactory(), new DefaultReflectorFactory());
            Executor executor = (Executor) metaObject.getValue("executor");
            MappedStatement mappedStatement = (MappedStatement) metaObject.getValue("mappedStatement");
            RowBounds rowBounds = (RowBounds) metaObject.getValue("rowBounds");
            ParameterHandler parameterHandler = (ParameterHandler) metaObject.getValue("parameterHandler");
            BoundSql boundSql = (BoundSql) metaObject.getValue("boundSql");
            ResultHandler<?> resultHandler = (ResultHandler) metaObject.getValue("resultHandler");
            // Executor executor, MappedStatement mappedStatement, ParameterHandler parameterHandler, ResultHandler<?> resultHandler, BoundSql boundSql,
            //                                 RowBounds rowBounds
            CustomResultSetHandler handler = new CustomResultSetHandler(
                executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds
            );
            return handler;
        }
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        // 把你的配置加载进来
    }
}

```

- CustomResultSetHandler

```java
/**
 *    Copyright 2009-2021 the original author or authors.
 *
 *    Licensed under the Apache License, Version 2.0 (the "License");
 *    you may not use this file except in compliance with the License.
 *    You may obtain a copy of the License at
 *
 *       http://www.apache.org/licenses/LICENSE-2.0
 *
 *    Unless required by applicable law or agreed to in writing, software
 *    distributed under the License is distributed on an "AS IS" BASIS,
 *    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *    See the License for the specific language governing permissions and
 *    limitations under the License.
 */
package com.example.demo.plugin;

import java.lang.reflect.Constructor;
import java.sql.CallableStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import org.apache.ibatis.annotations.AutomapConstructor;
import org.apache.ibatis.binding.MapperMethod.ParamMap;
import org.apache.ibatis.cache.CacheKey;
import org.apache.ibatis.cursor.Cursor;
import org.apache.ibatis.cursor.defaults.DefaultCursor;
import org.apache.ibatis.executor.ErrorContext;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.executor.ExecutorException;
import org.apache.ibatis.executor.loader.ResultLoader;
import org.apache.ibatis.executor.loader.ResultLoaderMap;
import org.apache.ibatis.executor.parameter.ParameterHandler;
import org.apache.ibatis.executor.result.DefaultResultContext;
import org.apache.ibatis.executor.result.DefaultResultHandler;
import org.apache.ibatis.executor.result.ResultMapException;
import org.apache.ibatis.executor.resultset.DefaultResultSetHandler;
import org.apache.ibatis.executor.resultset.ResultSetHandler;
import org.apache.ibatis.executor.resultset.ResultSetWrapper;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.Discriminator;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.ParameterMapping;
import org.apache.ibatis.mapping.ParameterMode;
import org.apache.ibatis.mapping.ResultMap;
import org.apache.ibatis.mapping.ResultMapping;
import org.apache.ibatis.reflection.MetaClass;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.ReflectorFactory;
import org.apache.ibatis.reflection.factory.ObjectFactory;
import org.apache.ibatis.session.AutoMappingBehavior;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.ResultContext;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.TypeHandler;
import org.apache.ibatis.type.TypeHandlerRegistry;
import org.apache.ibatis.util.MapUtil;

/**
 * @author Clinton Begin
 * @author Eduardo Macarron
 * @author Iwao AVE!
 * @author Kazuki Shimizu
 */
public class CustomResultSetHandler extends DefaultResultSetHandler {

  private static final Object DEFERRED = new Object();

  private final Executor executor;
  private final Configuration configuration;
  private final MappedStatement mappedStatement;
  private final RowBounds rowBounds;
  private final ParameterHandler parameterHandler;
  private final ResultHandler<?> resultHandler;
  private final BoundSql boundSql;
  private final TypeHandlerRegistry typeHandlerRegistry;
  private final ObjectFactory objectFactory;
  private final ReflectorFactory reflectorFactory;

  // nested resultmaps
  private final Map<CacheKey, Object> nestedResultObjects = new HashMap<>();
  private final Map<String, Object> ancestorObjects = new HashMap<>();
  private Object previousRowValue;

  // multiple resultsets
  private final Map<String, ResultMapping> nextResultMaps = new HashMap<>();
  private final Map<CacheKey, List<PendingRelation>> pendingRelations = new HashMap<>();

  // Cached Automappings
  private final Map<String, List<UnMappedColumnAutoMapping>> autoMappingsCache = new HashMap<>();

  // temporary marking flag that indicate using constructor mapping (use field to reduce memory usage)
  private boolean useConstructorMappings;

  private static class PendingRelation {
    public MetaObject metaObject;
    public ResultMapping propertyMapping;
  }

  private static class UnMappedColumnAutoMapping {
    private final String column;
    private final String property;
    private final TypeHandler<?> typeHandler;
    private final boolean primitive;

    public UnMappedColumnAutoMapping(String column, String property, TypeHandler<?> typeHandler, boolean primitive) {
      this.column = column;
      this.property = property;
      this.typeHandler = typeHandler;
      this.primitive = primitive;
    }


  }

  public CustomResultSetHandler(Executor executor, MappedStatement mappedStatement, ParameterHandler parameterHandler, ResultHandler<?> resultHandler, BoundSql boundSql,
                                 RowBounds rowBounds) {
    super(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    this.executor = executor;
    this.configuration = mappedStatement.getConfiguration();
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;
    this.parameterHandler = parameterHandler;
    this.boundSql = boundSql;
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();
    this.reflectorFactory = configuration.getReflectorFactory();
    this.resultHandler = resultHandler;
  }

  //
  // HANDLE OUTPUT PARAMETER
  //

  @Override
  public void handleOutputParameters(CallableStatement cs) throws SQLException {
    final Object parameterObject = parameterHandler.getParameterObject();
    final MetaObject metaParam = configuration.newMetaObject(parameterObject);
    final List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    for (int i = 0; i < parameterMappings.size(); i++) {
      final ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() == ParameterMode.OUT || parameterMapping.getMode() == ParameterMode.INOUT) {
        if (ResultSet.class.equals(parameterMapping.getJavaType())) {
          handleRefCursorOutputParameter((ResultSet) cs.getObject(i + 1), parameterMapping, metaParam);
        } else {
          final TypeHandler<?> typeHandler = parameterMapping.getTypeHandler();
          metaParam.setValue(parameterMapping.getProperty(), typeHandler.getResult(cs, i + 1));
        }
      }
    }
  }

  private void handleRefCursorOutputParameter(ResultSet rs, ParameterMapping parameterMapping, MetaObject metaParam) throws SQLException {
    if (rs == null) {
      return;
    }
    try {
      final String resultMapId = parameterMapping.getResultMapId();
      final ResultMap resultMap = configuration.getResultMap(resultMapId);
      final ResultSetWrapper rsw = new ResultSetWrapper(rs, configuration);
      if (this.resultHandler == null) {
        final DefaultResultHandler resultHandler = new DefaultResultHandler(objectFactory);
        handleRowValues(rsw, resultMap, resultHandler, new RowBounds(), null);
        metaParam.setValue(parameterMapping.getProperty(), resultHandler.getResultList());
      } else {
        handleRowValues(rsw, resultMap, resultHandler, new RowBounds(), null);
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rs);
    }
  }

  //
  // HANDLE RESULT SETS
  //
  @Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
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

  @Override
  public <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling cursor results").object(mappedStatement.getId());

    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();

    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    if (resultMapCount != 1) {
      throw new ExecutorException("Cursor results cannot be mapped to multiple resultMaps");
    }

    ResultMap resultMap = resultMaps.get(0);
    return new DefaultCursor<>(this, resultMap, rsw, rowBounds);
  }

  private ResultSetWrapper getFirstResultSet(Statement stmt) throws SQLException {
    ResultSet rs = stmt.getResultSet();
    while (rs == null) {
      // move forward to get the first resultset in case the driver
      // doesn't return the resultset as the first result (HSQLDB 2.1)
      if (stmt.getMoreResults()) {
        rs = stmt.getResultSet();
      } else {
        if (stmt.getUpdateCount() == -1) {
          // no more results. Must be no resultset
          break;
        }
      }
    }
    return rs != null ? new ResultSetWrapper(rs, configuration) : null;
  }

  private ResultSetWrapper getNextResultSet(Statement stmt) {
    // Making this method tolerant of bad JDBC drivers
    try {
      if (stmt.getConnection().getMetaData().supportsMultipleResultSets()) {
        // Crazy Standard JDBC way of determining if there are more results
        if (!(!stmt.getMoreResults() && stmt.getUpdateCount() == -1)) {
          ResultSet rs = stmt.getResultSet();
          if (rs == null) {
            return getNextResultSet(stmt);
          } else {
            return new ResultSetWrapper(rs, configuration);
          }
        }
      }
    } catch (Exception e) {
      // Intentionally ignored.
    }
    return null;
  }

  private void closeResultSet(ResultSet rs) {
    try {
      if (rs != null) {
        rs.close();
      }
    } catch (SQLException e) {
      // ignore
    }
  }

  private void cleanUpAfterHandlingResultSet() {
    nestedResultObjects.clear();
  }

  private void validateResultMapsCount(ResultSetWrapper rsw, int resultMapCount) {
    if (rsw != null && resultMapCount < 1) {
      throw new ExecutorException("A query was run and no Result Maps were found for the Mapped Statement '" + mappedStatement.getId()
          + "'.  It's likely that neither a Result Type nor a Result Map was specified.");
    }
  }

  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }

  @SuppressWarnings("unchecked")
  private List<Object> collapseSingleResultList(List<Object> multipleResults) {
    return multipleResults.size() == 1 ? (List<Object>) multipleResults.get(0) : multipleResults;
  }

  //
  // HANDLE ROWS FOR SIMPLE RESULTMAP
  //

  public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }

  private void ensureNoRowBounds() {
    if (configuration.isSafeRowBoundsEnabled() && rowBounds != null && (rowBounds.getLimit() < RowBounds.NO_ROW_LIMIT || rowBounds.getOffset() > RowBounds.NO_ROW_OFFSET)) {
      throw new ExecutorException("Mapped Statements with nested result mappings cannot be safely constrained by RowBounds. "
          + "Use safeRowBoundsEnabled=false setting to bypass this check.");
    }
  }

  protected void checkResultHandler() {
    if (resultHandler != null && configuration.isSafeResultHandlerEnabled() && !mappedStatement.isResultOrdered()) {
      throw new ExecutorException("Mapped Statements with nested result mappings cannot be safely used with a custom ResultHandler. "
          + "Use safeResultHandlerEnabled=false setting to bypass this check "
          + "or ensure your statement returns ordered data and set resultOrdered=true on it.");
    }
  }

  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }

  private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    if (parentMapping != null) {
      linkToParents(rs, parentMapping, rowValue);
    } else {
      callResultHandler(resultHandler, resultContext, rowValue);
    }
  }

  @SuppressWarnings("unchecked" /* because ResultHandler<?> is always ResultHandler<Object>*/)
  private void callResultHandler(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue) {
    resultContext.nextResultObject(rowValue);
    ((ResultHandler<Object>) resultHandler).handleResult(resultContext);
  }

  private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) {
    return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
  }

  private void skipRows(ResultSet rs, RowBounds rowBounds) throws SQLException {
    if (rs.getType() != ResultSet.TYPE_FORWARD_ONLY) {
      if (rowBounds.getOffset() != RowBounds.NO_ROW_OFFSET) {
        rs.absolute(rowBounds.getOffset());
      }
    } else {
      for (int i = 0; i < rowBounds.getOffset(); i++) {
        if (!rs.next()) {
          break;
        }
      }
    }
  }

  //
  // GET VALUE FROM ROW FOR SIMPLE RESULT MAP
  //

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

  //
  // GET VALUE FROM ROW FOR NESTED RESULT MAP
  //

  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, CacheKey combinedKey, String columnPrefix, Object partialObject) throws SQLException {
    final String resultMapId = resultMap.getId();
    Object rowValue = partialObject;
    if (rowValue != null) {
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      putAncestor(rowValue, resultMapId);
      applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
      ancestorObjects.remove(resultMapId);
    } else {
      final ResultLoaderMap lazyLoader = new ResultLoaderMap();
      rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
      if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        boolean foundValues = this.useConstructorMappings;
        if (shouldApplyAutomaticMappings(resultMap, true)) {
          foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
        }
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
        putAncestor(rowValue, resultMapId);
        foundValues = applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, true) || foundValues;
        ancestorObjects.remove(resultMapId);
        foundValues = lazyLoader.size() > 0 || foundValues;
        rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
      }
      if (combinedKey != CacheKey.NULL_CACHE_KEY) {
        nestedResultObjects.put(combinedKey, rowValue);
      }
    }
    return rowValue;
  }

  private void putAncestor(Object resultObject, String resultMapId) {
    ancestorObjects.put(resultMapId, resultObject);
  }

  private boolean shouldApplyAutomaticMappings(ResultMap resultMap, boolean isNested) {
    if (resultMap.getAutoMapping() != null) {
      return resultMap.getAutoMapping();
    } else {
      if (isNested) {
        return AutoMappingBehavior.FULL == configuration.getAutoMappingBehavior();
      } else {
        return AutoMappingBehavior.NONE != configuration.getAutoMappingBehavior();
      }
    }
  }

  //
  // PROPERTY MAPPINGS
  //

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
        } else if (value == DEFERRED) {
          foundValues = true;
          continue;
        }
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          // TODO：这里做解密操作
          String sql = boundSql.getSql();
          // 该sql没有查询该表
          List<String> finalNameList = JSqlParserUtil.parse(sql, "tbl_fm_cashier_receivables_apply", "applicant_name");
          if(finalNameList.contains(column)) {
            value = value + "***";
          }
          metaObject.setValue(property, value);
        }
      }
    }
    return foundValues;
  }

  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    if (propertyMapping.getNestedQueryId() != null) {
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERRED;
    } else {
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }

  private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    final String mapKey = resultMap.getId() + ":" + columnPrefix;
    List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
    if (autoMapping == null) {
      autoMapping = new ArrayList<>();
      final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
      for (String columnName : unmappedColumnNames) {
        String propertyName = columnName;
        if (columnPrefix != null && !columnPrefix.isEmpty()) {
          // When columnPrefix is specified,
          // ignore columns without the prefix.
          if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
            propertyName = columnName.substring(columnPrefix.length());
          } else {
            continue;
          }
        }
        final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
        if (property != null && metaObject.hasSetter(property)) {
          if (resultMap.getMappedProperties().contains(property)) {
            continue;
          }
          final Class<?> propertyType = metaObject.getSetterType(property);
          if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
            final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
            autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
          } else {
            configuration.getAutoMappingUnknownColumnBehavior()
                .doAction(mappedStatement, columnName, property, propertyType);
          }
        } else {
          configuration.getAutoMappingUnknownColumnBehavior()
              .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
        }
      }
      autoMappingsCache.put(mapKey, autoMapping);
    }
    return autoMapping;
  }

  private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
      for (UnMappedColumnAutoMapping mapping : autoMapping) {
        final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          // TODO：这里做解密操作
          String sql = boundSql.getSql();
          // 该sql没有查询该表
          List<String> finalNameList = JSqlParserUtil.parse(sql, "tbl_fm_cashier_receivables_apply", "applicant_name");
          String column = mapping.column;
          Object val = value;
          if(finalNameList.contains(column)) {
            val = val + "***";
          }
          metaObject.setValue(mapping.property, val);
        }
      }
    }
    return foundValues;
  }

  // MULTIPLE RESULT SETS

  private void linkToParents(ResultSet rs, ResultMapping parentMapping, Object rowValue) throws SQLException {
    CacheKey parentKey = createKeyForMultipleResults(rs, parentMapping, parentMapping.getColumn(), parentMapping.getForeignColumn());
    List<PendingRelation> parents = pendingRelations.get(parentKey);
    if (parents != null) {
      for (PendingRelation parent : parents) {
        if (parent != null && rowValue != null) {
          linkObjects(parent.metaObject, parent.propertyMapping, rowValue);
        }
      }
    }
  }

  private void addPendingChildRelation(ResultSet rs, MetaObject metaResultObject, ResultMapping parentMapping) throws SQLException {
    CacheKey cacheKey = createKeyForMultipleResults(rs, parentMapping, parentMapping.getColumn(), parentMapping.getColumn());
    PendingRelation deferLoad = new PendingRelation();
    deferLoad.metaObject = metaResultObject;
    deferLoad.propertyMapping = parentMapping;
    List<PendingRelation> relations = MapUtil.computeIfAbsent(pendingRelations, cacheKey, k -> new ArrayList<>());
    // issue #255
    relations.add(deferLoad);
    ResultMapping previous = nextResultMaps.get(parentMapping.getResultSet());
    if (previous == null) {
      nextResultMaps.put(parentMapping.getResultSet(), parentMapping);
    } else {
      if (!previous.equals(parentMapping)) {
        throw new ExecutorException("Two different properties are mapped to the same resultSet");
      }
    }
  }

  private CacheKey createKeyForMultipleResults(ResultSet rs, ResultMapping resultMapping, String names, String columns) throws SQLException {
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(resultMapping);
    if (columns != null && names != null) {
      String[] columnsArray = columns.split(",");
      String[] namesArray = names.split(",");
      for (int i = 0; i < columnsArray.length; i++) {
        Object value = rs.getString(columnsArray[i]);
        if (value != null) {
          cacheKey.update(namesArray[i]);
          cacheKey.update(value);
        }
      }
    }
    return cacheKey;
  }

  //
  // INSTANTIATION & CONSTRUCTOR MAPPING
  //

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

  private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
      throws SQLException {
    final Class<?> resultType = resultMap.getType();
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
      return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    } else if (!constructorMappings.isEmpty()) {
      return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
      return objectFactory.create(resultType);
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
      return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs);
    }
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
  }

  Object createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings,
                                         List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) {
    boolean foundValues = false;
    for (ResultMapping constructorMapping : constructorMappings) {
      final Class<?> parameterType = constructorMapping.getJavaType();
      final String column = constructorMapping.getColumn();
      final Object value;
      try {
        if (constructorMapping.getNestedQueryId() != null) {
          value = getNestedQueryConstructorValue(rsw.getResultSet(), constructorMapping, columnPrefix);
        } else if (constructorMapping.getNestedResultMapId() != null) {
          final ResultMap resultMap = configuration.getResultMap(constructorMapping.getNestedResultMapId());
          value = getRowValue(rsw, resultMap, getColumnPrefix(columnPrefix, constructorMapping));
        } else {
          final TypeHandler<?> typeHandler = constructorMapping.getTypeHandler();
          value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(column, columnPrefix));
        }
      } catch (ResultMapException | SQLException e) {
        throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
      }
      constructorArgTypes.add(parameterType);
      constructorArgs.add(value);
      foundValues = value != null || foundValues;
    }
    return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }

  private Object createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) throws SQLException {
    final Constructor<?>[] constructors = resultType.getDeclaredConstructors();
    final Constructor<?> defaultConstructor = findDefaultConstructor(constructors);
    if (defaultConstructor != null) {
      return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, defaultConstructor);
    } else {
      for (Constructor<?> constructor : constructors) {
        if (allowedConstructorUsingTypeHandlers(constructor, rsw.getJdbcTypes())) {
          return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, constructor);
        }
      }
    }
    throw new ExecutorException("No constructor found in " + resultType.getName() + " matching " + rsw.getClassNames());
  }

  private Object createUsingConstructor(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, Constructor<?> constructor) throws SQLException {
    boolean foundValues = false;
    for (int i = 0; i < constructor.getParameterTypes().length; i++) {
      Class<?> parameterType = constructor.getParameterTypes()[i];
      String columnName = rsw.getColumnNames().get(i);
      TypeHandler<?> typeHandler = rsw.getTypeHandler(parameterType, columnName);
      Object value = typeHandler.getResult(rsw.getResultSet(), columnName);
      constructorArgTypes.add(parameterType);
      constructorArgs.add(value);
      foundValues = value != null || foundValues;
    }
    return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }

  private Constructor<?> findDefaultConstructor(final Constructor<?>[] constructors) {
    if (constructors.length == 1) {
      return constructors[0];
    }

    for (final Constructor<?> constructor : constructors) {
      if (constructor.isAnnotationPresent(AutomapConstructor.class)) {
        return constructor;
      }
    }
    return null;
  }

  private boolean allowedConstructorUsingTypeHandlers(final Constructor<?> constructor, final List<JdbcType> jdbcTypes) {
    final Class<?>[] parameterTypes = constructor.getParameterTypes();
    if (parameterTypes.length != jdbcTypes.size()) {
      return false;
    }
    for (int i = 0; i < parameterTypes.length; i++) {
      if (!typeHandlerRegistry.hasTypeHandler(parameterTypes[i], jdbcTypes.get(i))) {
        return false;
      }
    }
    return true;
  }

  private Object createPrimitiveResultObject(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final Class<?> resultType = resultMap.getType();
    final String columnName;
    if (!resultMap.getResultMappings().isEmpty()) {
      final List<ResultMapping> resultMappingList = resultMap.getResultMappings();
      final ResultMapping mapping = resultMappingList.get(0);
      columnName = prependPrefix(mapping.getColumn(), columnPrefix);
    } else {
      columnName = rsw.getColumnNames().get(0);
    }
    final TypeHandler<?> typeHandler = rsw.getTypeHandler(resultType, columnName);
    return typeHandler.getResult(rsw.getResultSet(), columnName);
  }

  //
  // NESTED QUERY
  //

  private Object getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix) throws SQLException {
    final String nestedQueryId = constructorMapping.getNestedQueryId();
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, constructorMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
      final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
      final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
      final Class<?> targetType = constructorMapping.getJavaType();
      final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
      value = resultLoader.loadResult();
    }
    return value;
  }

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
        value = DEFERRED;
      } else {
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        if (propertyMapping.isLazy()) {
          lazyLoader.addLoader(property, metaResultObject, resultLoader);
          value = DEFERRED;
        } else {
          value = resultLoader.loadResult();
        }
      }
    }
    return value;
  }

  private Object prepareParameterForNestedQuery(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
    if (resultMapping.isCompositeResult()) {
      return prepareCompositeKeyParameter(rs, resultMapping, parameterType, columnPrefix);
    } else {
      return prepareSimpleKeyParameter(rs, resultMapping, parameterType, columnPrefix);
    }
  }

  private Object prepareSimpleKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
    final TypeHandler<?> typeHandler;
    if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
      typeHandler = typeHandlerRegistry.getTypeHandler(parameterType);
    } else {
      typeHandler = typeHandlerRegistry.getUnknownTypeHandler();
    }
    return typeHandler.getResult(rs, prependPrefix(resultMapping.getColumn(), columnPrefix));
  }

  private Object prepareCompositeKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
    final Object parameterObject = instantiateParameterObject(parameterType);
    final MetaObject metaObject = configuration.newMetaObject(parameterObject);
    boolean foundValues = false;
    for (ResultMapping innerResultMapping : resultMapping.getComposites()) {
      final Class<?> propType = metaObject.getSetterType(innerResultMapping.getProperty());
      final TypeHandler<?> typeHandler = typeHandlerRegistry.getTypeHandler(propType);
      final Object propValue = typeHandler.getResult(rs, prependPrefix(innerResultMapping.getColumn(), columnPrefix));
      // issue #353 & #560 do not execute nested query if key is null
      if (propValue != null) {
        metaObject.setValue(innerResultMapping.getProperty(), propValue);
        foundValues = true;
      }
    }
    return foundValues ? parameterObject : null;
  }

  private Object instantiateParameterObject(Class<?> parameterType) {
    if (parameterType == null) {
      return new HashMap<>();
    } else if (ParamMap.class.equals(parameterType)) {
      return new HashMap<>(); // issue #649
    } else {
      return objectFactory.create(parameterType);
    }
  }

  //
  // DISCRIMINATOR
  //

  public ResultMap resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix) throws SQLException {
    Set<String> pastDiscriminators = new HashSet<>();
    Discriminator discriminator = resultMap.getDiscriminator();
    while (discriminator != null) {
      final Object value = getDiscriminatorValue(rs, discriminator, columnPrefix);
      final String discriminatedMapId = discriminator.getMapIdFor(String.valueOf(value));
      if (configuration.hasResultMap(discriminatedMapId)) {
        resultMap = configuration.getResultMap(discriminatedMapId);
        Discriminator lastDiscriminator = discriminator;
        discriminator = resultMap.getDiscriminator();
        if (discriminator == lastDiscriminator || !pastDiscriminators.add(discriminatedMapId)) {
          break;
        }
      } else {
        break;
      }
    }
    return resultMap;
  }

  private Object getDiscriminatorValue(ResultSet rs, Discriminator discriminator, String columnPrefix) throws SQLException {
    final ResultMapping resultMapping = discriminator.getResultMapping();
    final TypeHandler<?> typeHandler = resultMapping.getTypeHandler();
    return typeHandler.getResult(rs, prependPrefix(resultMapping.getColumn(), columnPrefix));
  }

  private String prependPrefix(String columnName, String prefix) {
    if (columnName == null || columnName.length() == 0 || prefix == null || prefix.length() == 0) {
      return columnName;
    }
    return prefix + columnName;
  }

  //
  // HANDLE NESTED RESULT MAPS
  //

  private void handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    final DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    Object rowValue = previousRowValue;
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
      Object partialObject = nestedResultObjects.get(rowKey);
      // issue #577 && #542
      if (mappedStatement.isResultOrdered()) {
        if (partialObject == null && rowValue != null) {
          nestedResultObjects.clear();
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
      } else {
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
        if (partialObject == null) {
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
      }
    }
    if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
      previousRowValue = null;
    } else if (rowValue != null) {
      previousRowValue = rowValue;
    }
  }

  //
  // NESTED RESULT MAP (JOIN MAPPING)
  //

  private boolean applyNestedResultMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String parentPrefix, CacheKey parentRowKey, boolean newObject) {
    boolean foundValues = false;
    for (ResultMapping resultMapping : resultMap.getPropertyResultMappings()) {
      final String nestedResultMapId = resultMapping.getNestedResultMapId();
      if (nestedResultMapId != null && resultMapping.getResultSet() == null) {
        try {
          final String columnPrefix = getColumnPrefix(parentPrefix, resultMapping);
          final ResultMap nestedResultMap = getNestedResultMap(rsw.getResultSet(), nestedResultMapId, columnPrefix);
          if (resultMapping.getColumnPrefix() == null) {
            // try to fill circular reference only when columnPrefix
            // is not specified for the nested result map (issue #215)
            Object ancestorObject = ancestorObjects.get(nestedResultMapId);
            if (ancestorObject != null) {
              if (newObject) {
                linkObjects(metaObject, resultMapping, ancestorObject); // issue #385
              }
              continue;
            }
          }
          final CacheKey rowKey = createRowKey(nestedResultMap, rsw, columnPrefix);
          final CacheKey combinedKey = combineKeys(rowKey, parentRowKey);
          Object rowValue = nestedResultObjects.get(combinedKey);
          boolean knownValue = rowValue != null;
          instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject); // mandatory
          if (anyNotNullColumnHasValue(resultMapping, columnPrefix, rsw)) {
            rowValue = getRowValue(rsw, nestedResultMap, combinedKey, columnPrefix, rowValue);
            if (rowValue != null && !knownValue) {
              linkObjects(metaObject, resultMapping, rowValue);
              foundValues = true;
            }
          }
        } catch (SQLException e) {
          throw new ExecutorException("Error getting nested result map values for '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
        }
      }
    }
    return foundValues;
  }

  private String getColumnPrefix(String parentPrefix, ResultMapping resultMapping) {
    final StringBuilder columnPrefixBuilder = new StringBuilder();
    if (parentPrefix != null) {
      columnPrefixBuilder.append(parentPrefix);
    }
    if (resultMapping.getColumnPrefix() != null) {
      columnPrefixBuilder.append(resultMapping.getColumnPrefix());
    }
    return columnPrefixBuilder.length() == 0 ? null : columnPrefixBuilder.toString().toUpperCase(Locale.ENGLISH);
  }

  private boolean anyNotNullColumnHasValue(ResultMapping resultMapping, String columnPrefix, ResultSetWrapper rsw) throws SQLException {
    Set<String> notNullColumns = resultMapping.getNotNullColumns();
    if (notNullColumns != null && !notNullColumns.isEmpty()) {
      ResultSet rs = rsw.getResultSet();
      for (String column : notNullColumns) {
        rs.getObject(prependPrefix(column, columnPrefix));
        if (!rs.wasNull()) {
          return true;
        }
      }
      return false;
    } else if (columnPrefix != null) {
      for (String columnName : rsw.getColumnNames()) {
        if (columnName.toUpperCase().startsWith(columnPrefix.toUpperCase())) {
          return true;
        }
      }
      return false;
    }
    return true;
  }

  private ResultMap getNestedResultMap(ResultSet rs, String nestedResultMapId, String columnPrefix) throws SQLException {
    ResultMap nestedResultMap = configuration.getResultMap(nestedResultMapId);
    return resolveDiscriminatedResultMap(rs, nestedResultMap, columnPrefix);
  }

  //
  // UNIQUE RESULT KEY
  //

  private CacheKey createRowKey(ResultMap resultMap, ResultSetWrapper rsw, String columnPrefix) throws SQLException {
    final CacheKey cacheKey = new CacheKey();
    cacheKey.update(resultMap.getId());
    List<ResultMapping> resultMappings = getResultMappingsForRowKey(resultMap);
    if (resultMappings.isEmpty()) {
      if (Map.class.isAssignableFrom(resultMap.getType())) {
        createRowKeyForMap(rsw, cacheKey);
      } else {
        createRowKeyForUnmappedProperties(resultMap, rsw, cacheKey, columnPrefix);
      }
    } else {
      createRowKeyForMappedProperties(resultMap, rsw, cacheKey, resultMappings, columnPrefix);
    }
    if (cacheKey.getUpdateCount() < 2) {
      return CacheKey.NULL_CACHE_KEY;
    }
    return cacheKey;
  }

  private CacheKey combineKeys(CacheKey rowKey, CacheKey parentRowKey) {
    if (rowKey.getUpdateCount() > 1 && parentRowKey.getUpdateCount() > 1) {
      CacheKey combinedKey;
      try {
        combinedKey = rowKey.clone();
      } catch (CloneNotSupportedException e) {
        throw new ExecutorException("Error cloning cache key.  Cause: " + e, e);
      }
      combinedKey.update(parentRowKey);
      return combinedKey;
    }
    return CacheKey.NULL_CACHE_KEY;
  }

  private List<ResultMapping> getResultMappingsForRowKey(ResultMap resultMap) {
    List<ResultMapping> resultMappings = resultMap.getIdResultMappings();
    if (resultMappings.isEmpty()) {
      resultMappings = resultMap.getPropertyResultMappings();
    }
    return resultMappings;
  }

  private void createRowKeyForMappedProperties(ResultMap resultMap, ResultSetWrapper rsw, CacheKey cacheKey, List<ResultMapping> resultMappings, String columnPrefix) throws SQLException {
    for (ResultMapping resultMapping : resultMappings) {
      if (resultMapping.isSimple()) {
        final String column = prependPrefix(resultMapping.getColumn(), columnPrefix);
        final TypeHandler<?> th = resultMapping.getTypeHandler();
        List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
        // Issue #114
        if (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH))) {
          final Object value = th.getResult(rsw.getResultSet(), column);
          if (value != null || configuration.isReturnInstanceForEmptyRow()) {
            cacheKey.update(column);
            cacheKey.update(value);
          }
        }
      }
    }
  }

  private void createRowKeyForUnmappedProperties(ResultMap resultMap, ResultSetWrapper rsw, CacheKey cacheKey, String columnPrefix) throws SQLException {
    final MetaClass metaType = MetaClass.forClass(resultMap.getType(), reflectorFactory);
    List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
    for (String column : unmappedColumnNames) {
      String property = column;
      if (columnPrefix != null && !columnPrefix.isEmpty()) {
        // When columnPrefix is specified, ignore columns without the prefix.
        if (column.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
          property = column.substring(columnPrefix.length());
        } else {
          continue;
        }
      }
      if (metaType.findProperty(property, configuration.isMapUnderscoreToCamelCase()) != null) {
        String value = rsw.getResultSet().getString(column);
        if (value != null) {
          cacheKey.update(column);
          cacheKey.update(value);
        }
      }
    }
  }

  private void createRowKeyForMap(ResultSetWrapper rsw, CacheKey cacheKey) throws SQLException {
    List<String> columnNames = rsw.getColumnNames();
    for (String columnName : columnNames) {
      final String value = rsw.getResultSet().getString(columnName);
      if (value != null) {
        cacheKey.update(columnName);
        cacheKey.update(value);
      }
    }
  }

  private void linkObjects(MetaObject metaObject, ResultMapping resultMapping, Object rowValue) {
    final Object collectionProperty = instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject);
    if (collectionProperty != null) {
      final MetaObject targetMetaObject = configuration.newMetaObject(collectionProperty);
      targetMetaObject.add(rowValue);
    } else {
      metaObject.setValue(resultMapping.getProperty(), rowValue);
    }
  }

  private Object instantiateCollectionPropertyIfAppropriate(ResultMapping resultMapping, MetaObject metaObject) {
    final String propertyName = resultMapping.getProperty();
    Object propertyValue = metaObject.getValue(propertyName);
    if (propertyValue == null) {
      Class<?> type = resultMapping.getJavaType();
      if (type == null) {
        type = metaObject.getSetterType(propertyName);
      }
      try {
        if (objectFactory.isCollection(type)) {
          propertyValue = objectFactory.create(type);
          metaObject.setValue(propertyName, propertyValue);
          return propertyValue;
        }
      } catch (Exception e) {
        throw new ExecutorException("Error instantiating collection property for result '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
      }
    } else if (objectFactory.isCollection(propertyValue.getClass())) {
      return propertyValue;
    }
    return null;
  }

  private boolean hasTypeHandlerForResultObject(ResultSetWrapper rsw, Class<?> resultType) {
    if (rsw.getColumnNames().size() == 1) {
      return typeHandlerRegistry.hasTypeHandler(resultType, rsw.getJdbcType(rsw.getColumnNames().get(0)));
    }
    return typeHandlerRegistry.hasTypeHandler(resultType);
  }

}

```
  
- JSqlParserUtil
```java
package com.example.demo.plugin;

import com.google.common.collect.Sets;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.Optional;
import java.util.Set;
import java.util.stream.Collectors;
import net.sf.jsqlparser.JSQLParserException;
import net.sf.jsqlparser.expression.Alias;
import net.sf.jsqlparser.expression.Expression;
import net.sf.jsqlparser.parser.CCJSqlParserUtil;
import net.sf.jsqlparser.schema.Column;
import net.sf.jsqlparser.schema.Table;
import net.sf.jsqlparser.statement.Statement;
import net.sf.jsqlparser.statement.StatementVisitorAdapter;
import net.sf.jsqlparser.statement.select.FromItem;
import net.sf.jsqlparser.statement.select.Join;
import net.sf.jsqlparser.statement.select.ParenthesedFromItem;
import net.sf.jsqlparser.statement.select.ParenthesedSelect;
import net.sf.jsqlparser.statement.select.PlainSelect;
import net.sf.jsqlparser.statement.select.Select;
import net.sf.jsqlparser.statement.select.SelectItem;
import net.sf.jsqlparser.statement.select.SelectVisitorAdapter;
import org.apache.commons.lang3.StringUtils;
import org.springframework.util.CollectionUtils;

public class JSqlParserUtil {

    public static void main(String[] args) {
//        String sql = "SELECT c.name2 as xxx, t1.id as 正满   FROM t1 join t2 on t1.id = t2.id "
//            + " left join t3 on t2.id = t3.id right join t4 on t4.id = t3.id "
//            + " join (select t6.id, t6.name1 as n, t5.name1 as name2 from (select  bbb.c as name1 from a,(select c from j ) as bbb where a.id = 1) t5 join (select id, name1 from uu)t6 on t5.id = t6.id where id = 10) c on c.id = t4.id WHERE id = ?";
// 暂时不考虑这种，不过构建树的时候已考虑这种，但是解析获取对应最顶层字段名是没有考虑，等有空再写
//            String sql = "select (select name from a where id = 0) as name from t1";
//            String sql = "select name from a where id = 0";
//            String sql = "select name from a as bb where id = 0";
//            String sql = "select bb.name from a as bb where id = 0";
//            String sql = "select a.name,b.name as name2 from a left join b on a.id = b.id";
            String sql = "select name from (select name from a) ab where ab.id = ?";
        // 获取指定表某个字段，最终显示到最顶层 select的字段名
        List<String> rootColNameList = parse(sql, "a", "name");
        System.out.println(rootColNameList);
    }

    // 返回最终的 select xx 这个xx里的别名
    public static List<String> parse(String sql, String tableName, String columnName) {
        SelectBO rootSelectBO = JSqlParserUtil.parse(sql);
        Set<String> factTbNameSet = Sets.newHashSet();
        JSqlParserUtil.digui(rootSelectBO, factTbNameSet);
        // 该sql没有查询该表
        if(!factTbNameSet.contains(tableName)) {
            return Collections.emptyList();
        }
        // 如果存在要操作的表，接下来要解析对应需要操作的字段名是否和提供的字段名一致
        List<String> rootColNameList = new ArrayList<>();
        JSqlParserUtil.digui2(rootSelectBO, tableName, columnName, rootColNameList);
        return rootColNameList;
    }

    private static ColumnBO mergeColName(String tbAlias, String colName, String colAlias) {
        ColumnBO columnBO = new ColumnBO();
        columnBO.setName(colName);
        if(StringUtils.isNotBlank(colAlias)) {
            columnBO.setName(colAlias);
        }
        if(StringUtils.isNotBlank(tbAlias)) {
            columnBO.setTbAlias(tbAlias);
        }
        return columnBO;
    }

    private static void digui2(SelectBO rootSelectBO, String tableName, String columnName, List<String> rootColNameList) {
        List<TableBO> tableList = rootSelectBO.getTableList();
        Set<String> tableNameSet = Optional.ofNullable(tableList).orElse(Collections.emptyList())
            .stream().map(TableBO::getName).collect(Collectors.toSet());
        if(!tableNameSet.contains(tableName)) {
            List<SelectBO> childrenList = rootSelectBO.getChildrenList();
            Optional.ofNullable(childrenList).orElse(Collections.emptyList())
                .forEach(selectBO -> {
                    JSqlParserUtil.digui2(selectBO, tableName, columnName, rootColNameList);
                });
        } else {
            matchColumn(rootSelectBO, tableName, columnName, rootColNameList);
        }
    }

    private static void matchColumn(SelectBO rootSelectBO, String tableName, String columnName, List<String> rootColNameList) {

        // 目标表
        List<ColumnBO> columnList = rootSelectBO.getColumnList();
        Set<String> colNameSet = columnList.stream().map(ColumnBO::getName).collect(Collectors.toSet());
        if(colNameSet.contains(columnName)) {
            doMatchColumn(rootSelectBO, tableName, rootSelectBO.getTableList(), columnList, columnName, rootColNameList);

            List<SelectBO> childrenList = rootSelectBO.getChildrenList();
            Optional.ofNullable(childrenList).orElse(Collections.emptyList())
                .forEach(selectBO -> {
                    JSqlParserUtil.digui2(selectBO, tableName, columnName, rootColNameList);
                });
        } else {
            List<SelectBO> childrenList = rootSelectBO.getChildrenList();
            Optional.ofNullable(childrenList).orElse(Collections.emptyList())
                .forEach(selectBO -> {
                    JSqlParserUtil.digui2(selectBO, tableName, columnName, rootColNameList);
                });
        }
    }

    private static void doMatchColumn(SelectBO rootSelectBO, String tableName, List<TableBO> tableList, List<ColumnBO> columnList, String columnName,
        List<String> rootColNameList) {
        List<ColumnBO> finalNameList = new ArrayList<>();
        for (ColumnBO columnBO : columnList) {
            String name = columnBO.getName();
            if(Objects.nonNull(name) && name.equals(columnName)) {
                String colTbAlas = columnBO.getTbAlias();
                if(tableList.size() == 1) {
                    String temAlias = rootSelectBO.getAlias();
                    String finalAlias = StringUtils.isBlank(temAlias) ? colTbAlas : temAlias;
                    ColumnBO fianlName = mergeColName(finalAlias, name, columnBO.getAlias());
                    finalNameList.add(fianlName);
                } else {
                    // 多表关联的
                    tableList.stream().filter(e -> e.getName().equals(tableName)).forEach(tableBO -> {
                        String temAlias = rootSelectBO.getAlias();
                        String tbAlias = tableBO.getAlias();
                        if(StringUtils.isBlank(tbAlias) && StringUtils.isNotBlank(colTbAlas)
                        && !tableName.equals(colTbAlas)) {
                            return;
                        }
                        if(StringUtils.isNotBlank(tbAlias) && StringUtils.isNotBlank(colTbAlas)
                            && !tbAlias.equals(colTbAlas)) {
                            return;
                        }

                        if(StringUtils.isBlank(tbAlias) && StringUtils.isBlank(colTbAlas)) {
                            // 这种无法确定是哪张表的，比如有人这么写 （select name from a,b），其中
                            // 字段a只有表a有，b没有，这个时候sql解析不出来的，得结合数据库表元信息
                            // 如果你想优化的话，可以在这里增加代码，去查询数据库这些表的字段，以确定是哪张表的
                            // 为了性能，最好不要写这样的sql
                            return;
                        }
                        String finalAlias = StringUtils.isBlank(temAlias) ? colTbAlas : temAlias;
                        ColumnBO fianlName = mergeColName(finalAlias, name, columnBO.getAlias());
                        finalNameList.add(fianlName);

                    });
                }
            }
        }
        findRoot(rootSelectBO.getParent(), finalNameList, rootColNameList);
    }

    private static void doFindRoot(SelectBO rootSelectBO, List<ColumnBO> columnList, ColumnBO targetColumnBO,
        List<String> rootColNameList) {
        List<ColumnBO> finalNameList = new ArrayList<>();
        String targeTbAlias = StringUtils.trimToEmpty(targetColumnBO.getTbAlias());
        String columnName = targetColumnBO.getName();
        for (ColumnBO columnBO : columnList) {
            String name = columnBO.getName();
            String willMatchTbAlias = StringUtils.trimToEmpty(columnBO.getTbAlias());
            if(Objects.nonNull(name) && name.equals(columnName)) {

                if(StringUtils.isNotBlank(willMatchTbAlias)
                && StringUtils.isNotBlank(targeTbAlias)
                && !Objects.equals(willMatchTbAlias, targeTbAlias)) {
                    continue;
                }

                // select上没有写别名，但是临时表有别名，此时名字已匹配上，那么这个名字绝对是多个临时表
                // 临时表在sql语法上是必须要有别名的，否则数据库执行报错，比如这样的 （select name from (select name from b)）
                // 独一无二的，所以允许匹配
                // select上有别名，但是临时表没有别名是错误的语法，所以直接返回
                // 都没有设置别名的情况下，允许匹配
                if(StringUtils.isNotBlank(willMatchTbAlias)
                    && StringUtils.isBlank(targeTbAlias)) {
                    continue;
                }

                String colTbAlas = columnBO.getTbAlias();
                String temAlias = rootSelectBO.getAlias();
                String finalAlias = StringUtils.isBlank(temAlias) ? colTbAlas : temAlias;
                ColumnBO fianlName = mergeColName(finalAlias, name, columnBO.getAlias());
                finalNameList.add(fianlName);
            }
        }
        if(CollectionUtils.isEmpty(finalNameList)) {
            System.out.println();
        }
        findRoot(rootSelectBO.getParent(), finalNameList, rootColNameList);
    }

    private static void findRoot(SelectBO selectBO, List<ColumnBO> columnList, List<String> rootColNameList) {
        if(Objects.isNull(selectBO)) {
            rootColNameList.addAll(columnList.stream()
                .map(ColumnBO::getName).distinct().collect(Collectors.toList()));
            return;
        }
        columnList.stream().forEach(columnBO -> {
            doFindRoot(selectBO, selectBO.getColumnList(),
                columnBO, rootColNameList);
        });

    }

    private static void digui(SelectBO rootSelectBO, Set<String> factTbNameSet) {
        List<TableBO> tableBOList = rootSelectBO.getTableList();
        if(!CollectionUtils.isEmpty(tableBOList)) {
            for (TableBO tableBO : tableBOList) {
                factTbNameSet.add(tableBO.getName());
            }
        }
        List<SelectBO> childrenList = rootSelectBO.getChildrenList();
        if(!CollectionUtils.isEmpty(childrenList)) {
            for (SelectBO selectBO : childrenList) {
                digui(selectBO, factTbNameSet);
            }

        }
    }

    public static SelectBO parse(String sql) {
        SelectBO[] ROOT = new SelectBO[1];
        Statement statement = null;
        try {
            statement = CCJSqlParserUtil.parse(sql);
        } catch (JSQLParserException e) {
            throw new RuntimeException(e);
        }
        statement.accept(new StatementVisitorAdapter() {
            @Override
            public void visit(Select select) {
                select.accept(new SelectVisitorAdapter() {
                    @Override
                    public void visit(PlainSelect plainSelect) {
                        ROOT[0] = JSqlParserUtil.doVisit(plainSelect, null);
                    }

                    @Override
                    public void visit(ParenthesedSelect parenthesedSelect) {
                        PlainSelect plainSelect = parenthesedSelect.getPlainSelect();
                        ROOT[0] = JSqlParserUtil.doVisit(plainSelect, null);
                    }

                });
            }
        });
        return ROOT[0];
    }

    private static SelectBO doVisit(PlainSelect plainSelect, SelectBO parent) {
        SelectBO selectBO = new SelectBO();
        selectBO.setParent(parent);
        if(Objects.nonNull(parent)) {
            parent.getChildrenList().add(selectBO);
        }
        FromItem fromItem = plainSelect.getFromItem();
        JSqlParserUtil.fromItem(fromItem, selectBO);
        List<Join> joins = plainSelect.getJoins();
        Optional.ofNullable(joins).orElse(Collections.emptyList()).stream().forEach(join -> {
            JSqlParserUtil.fromItem(join.getFromItem(), selectBO);
        });
        List<SelectItem<?>> selectItems = plainSelect.getSelectItems();
        Optional.ofNullable(selectItems).orElse(Collections.emptyList())
            .forEach(selectItem -> {
                JSqlParserUtil.selectItem(selectItem, selectBO);
            });
        return selectBO;
    }

    private static void fromItem(FromItem fromItem, SelectBO selectBO) {
        if (fromItem instanceof Table) {
            Table table = (Table) fromItem;
            String tableName = table.getName();
            Alias alias = table.getAlias();
            TableBO tableBO = new TableBO();
            tableBO.setName(tableName);
            tableBO.setAlias("");
            if(Objects.nonNull(alias)) {
                tableBO.setAlias(alias.getName());
            }
            selectBO.getTableList().add(tableBO);
        } else if (fromItem instanceof ParenthesedFromItem) {
            ParenthesedFromItem parenthesedFromItem = (ParenthesedFromItem) fromItem;
            FromItem f = parenthesedFromItem.getFromItem();
            fromItem(f, selectBO);
            List<Join> joins = parenthesedFromItem.getJoins();
            Optional.ofNullable(joins)
                .orElse(Collections.emptyList()).forEach(join -> {
                    fromItem(join.getFromItem(), selectBO);
                });

        } else if (fromItem instanceof ParenthesedSelect) {
            ParenthesedSelect parenthesedSelect = (ParenthesedSelect) fromItem;
            PlainSelect plainSelect = parenthesedSelect.getPlainSelect();
            SelectBO currentSelectBO = JSqlParserUtil.doVisit(plainSelect, selectBO);
            Alias alias = parenthesedSelect.getAlias();
            if(Objects.nonNull(alias)) {
                currentSelectBO.setAlias(alias.getName());
            }

        }
    }

    private static void selectItem(SelectItem selectItem, SelectBO selectBO) {
        Alias alias = selectItem.getAlias();
        if(Objects.nonNull(alias)) {
            String name = selectItem.getAlias().getName();
            expression(selectItem.getExpression(), name, selectBO);
        } else {
            Expression expression = selectItem.getExpression();
            expression(expression, "", selectBO);

        }
    }

    private static void expression(Expression expression, String alias, SelectBO selectBO) {
        if(expression instanceof Column) {
            Column column = (Column)expression;
            Table table = column.getTable();
            String columnName = column.getColumnName();
            ColumnBO columnBO = new ColumnBO();
            columnBO.setName(columnName);
            columnBO.setAlias(alias);
            if(Objects.nonNull(table)) {
                columnBO.setTbAlias(table.getName());
            }
            selectBO.getColumnList().add(columnBO);
            // 这种就是  类似这样的 sql （ select (select name from a where id = 0) as name from t1 ）
        } else if( expression instanceof ParenthesedSelect) {
            ParenthesedSelect parenthesedSelect = (ParenthesedSelect) expression;
            PlainSelect plainSelect = parenthesedSelect.getPlainSelect();
            SelectBO columnSelect = JSqlParserUtil.doVisit(plainSelect, null);
            ColumnBO columnBO = new ColumnBO();
            columnBO.setName(null);
            columnBO.setChild(columnSelect);
            Alias tbAlias = parenthesedSelect.getAlias();
            if(Objects.nonNull(tbAlias)) {
                columnBO.setAlias(tbAlias.getName());
            }
            selectBO.getColumnList().add(columnBO);
        }
    }

    //该对象不支持序列化
    public static class SelectBO {
        private SelectBO parent;
        private String alias;
        private List<TableBO> tableList = new ArrayList<>();
        private List<ColumnBO> columnList = new ArrayList<>();
        private List<SelectBO> childrenList = new ArrayList<>();

        public List<TableBO> getTableList() {
            return tableList;
        }

        public void setTableList(List<TableBO> tableList) {
            this.tableList = tableList;
        }

        public List<ColumnBO> getColumnList() {
            return columnList;
        }

        public void setColumnList(List<ColumnBO> columnList) {
            this.columnList = columnList;
        }

        public SelectBO getParent() {
            return parent;
        }

        public void setParent(SelectBO parent) {
            this.parent = parent;
        }

        public List<SelectBO> getChildrenList() {
            return childrenList;
        }

        public void setChildrenList(List<SelectBO> childrenList) {
            this.childrenList = childrenList;
        }

        public String getAlias() {
            return alias;
        }

        public void setAlias(String alias) {
            this.alias = alias;
        }
    }

    public static class ColumnBO {
        // 可能是别名，或者没有值（比如 select age from tb）
        private String tbAlias;
        private String name;
        private String alias;
        private SelectBO child;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getAlias() {
            return alias;
        }

        public void setAlias(String alias) {
            this.alias = alias;
        }

        public String getTbAlias() {
            return tbAlias;
        }

        public void setTbAlias(String tbAlias) {
            this.tbAlias = tbAlias;
        }

        public SelectBO getChild() {
            return child;
        }

        public void setChild(SelectBO child) {
            this.child = child;
        }
    }

    public static class TableBO {
        private String name;
        private String alias;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getAlias() {
            return alias;
        }

        public void setAlias(String alias) {
            this.alias = alias;
        }
    }

}
```


```java
package com.example.demo;

import cn.hutool.json.JSONUtil;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.databind.ser.std.StdSerializer;
import java.io.IOException;
import java.io.OutputStream;
import java.lang.reflect.Field;
import java.util.Arrays;
import java.util.Objects;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

/**
 * @author wuzhenhong
 * @date 2024/12/12 8:22
 */
@ControllerAdvice(basePackageClasses = TestController.class)
public class ResponseBodyAdviceImpl implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
        Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request,
        ServerHttpResponse response) {

        // 这个可以自己封装下成util
        com.fasterxml.jackson.databind.ObjectMapper mapper = new com.fasterxml.jackson.databind.ObjectMapper();
        SimpleModule module = new SimpleModule();
        module.addSerializer(returnType.getParameterType(), new CustomeSerializer(returnType));
        mapper.registerModule(module);

        String json = null;
        try {
            json = mapper.writeValueAsString(body);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }

        response.getHeaders().add("Content-Type",selectedContentType.toString());
        try (OutputStream outputStream = response.getBody()){
            outputStream.write(json.getBytes());
            response.flush();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        return null;
    }


    public static class CustomeSerializer extends StdSerializer<Object> {
        private MethodParameter returnType;
        public CustomeSerializer(MethodParameter returnType) {
            super(Object.class);
            this.returnType = returnType;
        }

        @Override
        public void serialize(Object value, JsonGenerator gen, SerializerProvider provider) throws IOException {

            String name = gen.getOutputContext().getCurrentName();
            try {
                Field field = gen.getOutputContext().getCurrentValue().getClass().getDeclaredField("name");
                在字段上的注解 aaa = field.getAnnotation(在字段上的注解.class);
                if(Objects.nonNull(aaa)) {
                    Handler handler = aaa.newInstance();
                    String str = trimQuotes(value);
                    String mask = handler.mask(str);
                    gen.writeStartObject();
                    gen.writeStringField(name + "脱敏", str);
                    gen.writeStringField(name + "密码", mask);
                    gen.writeEndObject();
                    return;
                }
            } catch (NoSuchFieldException e) {
                throw new RuntimeException(e);
            }
            
            Sensitive sensitiveAnnotation = returnType.getAnnotatedElement().getAnnotation(sensitive.class);
            if(sensitiveAnnotation == null) {
                gen.writeObject(value);
                return;
            }
            FieldHandlerPair[] fieldHandlerPairs = sensitiveAnnotation.value();
            if (fieldHandlerPairs == null || fieldHandlerPairs.length == 0) {
                gen.writeObject(value);
                return;
            }

            for (FieldHandlerPair fieldHandlerPair : fieldHandlerPairs) {
                String fieldName = fieldHandlerPair.fieldName();
                String name = gen.getOutputContext().getCurrentName();
                if(gen.getOutputContext().getCurrentName().equals(fieldName)) {
                    Handler handler = hc.newInstance();
                    String str = trimQuotes(value);
                    String mask = handler.mask(str);
                    gen.writeStartObject();
                    gen.writeStringField(name + "脱敏", str);
                    gen.writeStringField(name + "密码", mask);
                    gen.writeEndObject();
                }
            }
        }
    }
}


```
