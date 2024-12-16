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
