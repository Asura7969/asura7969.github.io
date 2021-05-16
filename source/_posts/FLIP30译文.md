---
title: FLP-30 Unified Catalog APIs译文
date: 2021-03-07 08:31:57
tags: flink
categories: flink
cover: https://i.loli.net/2021/05/16/Qfg2vpysuoJx69A.png
---


# 背景

随着其在流处理中的广泛采用，Flink也显示了其在批处理中的潜力。 改进Flink的批处理，尤其是在SQL方面，将使Flink在流处理之外得到更大的采用，并为用户提供一套完整的解决方案，以满足他们的流和批处理需求。
另一方面，Hive已将重点放在大数据技术及其完整的生态系统上。 对于大多数大数据用户而言，Hive不仅是用于大数据分析和ETL的SQL引擎，还是一个数据管理平台，可以在该平台上查询，定义和演化数据。 换句话说，Hive是Hadoop上大数据的事实上的标准。
因此，Flink必须与Hive生态系统集成，以进一步将其扩展到批处理和SQL用户。 为此，必须与Hive元数据和数据集成。

Hive Metastore集成有两个方面：1.使Hive的元对象（如表和视图）可供Flink使用，并且Flink也能够为Hive和在Hive中创建此类元对象； 2.使用Hive Metastore作为持久性存储，使Flink的元对象（表，视图和UDF）持久化。
本文档涵盖Flink和Hive生态系统集成的三个部分之一。 它不仅涉及Hive集成，还涉及重新构建目录界面以及TableEnvironment目录和外部目录的统一，其长期目标是能够在目录中存储批处理和流式连接器信息（不仅是Hive，而且是Kafka， Elasticsearch等）。

# 变更
在当前的Flink代码库中，已经为外部目录定义了一组接口。 但是，API尚不稳定，需要完善我们的工作。
更改在以下类层次结构中得到最好的体现：

![image1.png](http://ww1.sinaimg.cn/large/b3b57085gy1gob5usr9a1j20hb0avmxo.jpg)

在图1中，ReadableCatalog，ReadableWritableCatalog和CatalogManager是我们正在定义的主要接口。 其他仅仅是实现或接口调用程序。 

## ReadableCatalog 接口
此类来自重命名现有的ExternalCatalog类。 删除“外部”关键字的原因是内部和外部之间没有明显的区别，因为外部目录也可以用于存储Flink的元对象。
我们需要调整现有的API，以容纳Flink中存在的并且在典型数据库目录中也很常见的其他元对象，例如表和视图。 我们还恢复了架构/数据库概念，而不是非标准的子目录术语。 

```java
    public interface ReadableCatalog {
    
        void open();
    
        void close();
        
        // Get a table factory instance that can convert catalog tables in this catalog
        // to sources or sinks. Null can be returned if table factory isn’t provided,
        // in which case, the current discovery mechanism will be used.
        Optional<TableFactory> getTableFactory();
    
        List<String> listDatabases();
    
        CatalogDatabase getDatabase(String databaseName);
    
        /**
         * List table names under the given database. Throw an exception if database isn’t
         * found.
         */
    
        List<String> listTables(String databaseName);
        
        /**
         * List view names under the given database. Throw an exception if database isn’t
         * found.
         */
        List<String> listViews(String databaseName);
    
        // getTable() can return view as well.
        CommonTable getTable(ObjectPath tableOrViewName);
    
        /**
         * List function names under the given database. Throw an exception if
         * database isn’t found.
         */
        List<String> listFunctions(String databaseName);
        
        CatalogFunction getFunction(ObjectPath functionName);
    }
```
更改内容：
* 添加open（）和close（）。 它们被添加到ReadableCatalog接口中以照顾外部连接。 他们可能需要一些运行时上下文，但是我暂时不做介绍。
* 添加了与视图/ UDF相关的读取 
* 定义表和视图之间的关系（见图2） 

![image2.png](http://ww1.sinaimg.cn/large/b3b57085gy1gob5yaau1sj20hc0bjdg3.jpg)

视图是表的一种特定类型。 更具体地说，视图是通过SQL语句在其他表和/或视图之上定义的虚拟表。

## CatalogDatabase 类
这表示描述/数据库对象。 它目前被建模为子目录，来自FLINK-6574。 有关更多讨论，请参见“其他说明”部分。 

请注意，许多元对象类，包括CatalogDatabase，CatalogTable和CatalogView，都有一个称为属性的成员变量。 它们之所以出现，是因为外部目录可能允许用户指定任何通用属性，例如所有者，creation_time，last_access_time等。 

```java
    public class CatalogDatabase {
        private Map<String, String> properties;
        
        public CatalogDatabase(Map<String, String> properties) {
                this.properties = properties;
        }
    
        public Map<String, String> getProperties() {
                return properties;
        }
    }
```

## ObjectPath 类

```java
    /**
     * A database name and object (table/function) name combo in a catalog
     */
    
    public class ObjectPath {
        private final String databaseName;
        private final String objectName;
    
        public ObjectPath(String databaseName, String objectName) {
                this.databaseName = databaseName;
                this.objectName = objectName;
        }
    }
```

## CatalogTable 接口
通过以下更改，从ExternalCatalogTable重命名了类CatalogTable。 它可以通过属性映射来实现，在该属性映射中，关于表的所有内容都被编码为编码表状态和模式的键值对（描述符），或者仅通过具有表模式，表状态和表属性的POJO类来实现。 

```java
    public interface CommonTable {
        public Map<String, String> getProperties();
    }
    
    
    public interface CatalogTable extends CommonTable{
    
        public TableSchema getSchema();
        
        public TableStatistics getTableStatistics();
    }
```

## HiveTable 类

```java
    Public class HiveTable implements CatalogTable {
    
        Public TableSchema getSchema() {
                // get from Hive megastore
        }
    
        Public TableStats getStats() {
                // get from Hive megastore
        }
    
        /**
          * Hive table properties (not contain schema or stats)
          */
        Public TableStats getProperties() {
                // get from Hive megastore
        }
    }
```

## GenericCatalogTable 类

此类表示Flink中当前定义的表，这些表没有外部定义。 此类表当前存储在内存中，但可以存储在永久性存储中，以实现跨用户会话的持久性。 

```java
    public class GenericCatalogTable implements CatalogTable {
    
        // All table info (schema, stat, and table properties) is encoded as properties
    
        private Map<String, String> properties;
    
        private TableSchema tableSchema;
    
        prviate TableStats tableStats;
        
        public TableSchema getSchema() {
                return tableSchema
        }
    
        Public TableStats getStats() {
                return tableStats;
        }
        
        public Map<String, String> getProperties() {
                return properties;
        }
    }
```

## CatalogView 接口

目录视图是CommonTable的一种特定类型。 视图是用查询语句定义的，还需要存储其扩展形式以记住查询上下文（例如当前数据库）。 

```java
    public interface CatalogView extends CommonTable {
    
        // Original text of the view definition.
    
        String getOriginalQuery();
    
        // Expanded text of the original view definition
    
        // This is needed because the context such as current DB is
    
        // lost after the session, in which view is defined, is gone.
    
        // Expanded query text takes care of the this, as an example.
    
        String getExpandedQuery();
    }
```

## CatalogFunction Class/Interface

此类表示在目录中定义的函数（UDF）。 它现在用作占位符，因为需要清除许多细节。 但是，它需要涵盖Flink功能和Hive功能。 

```java
    /**
      * The detailed definition of this class needs to be further sorted out
      */
    
    public class CatalogFunction {
    
        private Enum from; // source of the function (can only be "CLASS" for now)
    
        private String clazz; // fully qualified class name of the function
    
         ...
    
        private Map<String, String> properties;
         
        public CatalogFunction(String from, String clazz, Map<String, String> properties) {
                this.properties = properties;
        }
        
        public Map<String, String> getProperties() {
                return properties;
        }
    }
```

## ReadableWritableCatalog Interface

该接口来自重命名CrudExternalCatalog类。 我们添加了与视图和功能相关的方法。 

```java
    public interface ReadableWritableCatalog extends ReadableCatalog {
    
        void createDatabase(String databaseName, CatalogDatabase database, boolean ignoreIfExists);
    
        void alterDatabase(String databaseName, CatalogDatabase database, boolean ignoreIfNotExists);
    
        void renameDatabase(String databaseName,String newDatabaseName, boolean ignoreIfNotExists);
    
        void dropDatabase(String databaseName, boolean ignoreIfNotExists);
        
        void createTable(ObjectPath tableName, CatalogTable table, boolean ignoreIfExists);
    
        /**
         * dropTable() also covers views.
         * @param tableName
         * @param ignoreIfNotExists
         */
        void dropTable(ObjectPath tableName, boolean ignoreIfNotExists);
    
        void renameTable(ObjectPath tableName, String newTableName, boolean ignoreIfNotExists);
    
        void alterTable(ObjectPath tableName, CatalogTable table, boolean ignoreIfNotExists):
    
        void createView(ObjectPath viewName, CatalogView view, boolean ignoreIfExists);
    
        void alterView(ObjectPath viewName, CatalogView view, boolean ignoreIfNotExists);
    
        void createFunction(ObjectPath funcName, CatalogFunction function, boolean ignoreIfExists);
    
        void renameFunction(ObjectPath funcName, String newFuncName, boolean ignoreIfNotExists);
    
        void dropFunction(ObjectPath funcName, boolean ignoreIfNotExists);
    
        void alterFunction(ObjectPath funcName, CatalogFunction function, boolean ignoreIfNotExists);
    }
```

## HiveCatalogBase Class
```java
abstract class HiveCatalogBase implements ReadableWritableCatalog {

    Private HiveMetastoreClient hmsClient;
    // implementation for reading metadata from or writing metadata to

    // Hive metastore
    
    // Any utility methods that are common to both HiveCatalog and

    // FlinkHmsCatalog

}
```

## HiveCatalog Class
```java
class HiveCatalog extends HiveCatalogBase {

    public TableFactory getTableFactory() {
        return new HiveTableFactory();
    }
    // Implementation of other methods that are not implemented yet.
}
```

## GenericHiveMetastoreCatalog Class
这是目录的实现类，用于保存Flink当前定义的表（视图/函数）。 此实现利用Hive元存储作为持久性存储。 

```java
class GenericHiveMetastoreCatalog extends HiveCatalogBase {

    public TableFactory getTableFactory() {
        return null; // Use table factory discovery mechanism
    }

    // Implementation of other methods that are not implemented yet.
}
```

## CatalogManager Class
我们引入CatalogManager类来管理表环境中所有已注册的ReadableCatalog实例。 它还具有默认目录和默认数据库的概念，当在元对象引用中未提供目录名称时，将选择默认目录和默认数据库。 

```java
public class CatalogManager {

    // The catalog to hold all registered and translated tables
    // We disable caching here to prevent side effects
    private CalciteSchema internalSchema = CalciteSchema.createRootSchema(true, false);

    private SchemaPlus rootSchema = internalSchema.plus();
    
    // A list of named catalogs.
    private Map<String, ReadableCatalog> catalogs;

    // The name of the default catalog
    private String defaultCatalog = null;
    
    public CatalogManager(Map<String, ReadableCatalog> catalogs, String defaultCatalog) {
            // make sure that defaultCatalog is in catalogs.keySet().
            this.catalogs = catalogs;
            this.defaultCatalog = defaultCatalog;
    }
    
    public void registerCatalog(String catalogName, ReadableCatalog catalog) {
            catalogs.put(catalogName, catalog);
    }
    
    public ReadableCatalog getCatalog(String catalogName) {
            return catalogs.get(catalogName);
    }

    public Set<String> getCatalogs() {
            return this.catalogs.keySet();
    }

    public void setDefaultCatalog(String catName) {
            // validate
            this.defaultCatalog = catqName;
    }

    public ReadableCatalog getDefaultCatalog() {
            return this.catalogs.get(defaultCatalog);
    }
}
```

除了列出ReadableCatalog之外，CatalogManger还封装了Calcite的架构框架，这样，除需要所有目录的解析器外，CatalogManager之外的任何代码都无需与Calcite的架构进行交互。 （所有目录都将添加到Calcite架构中，以便在查询解析和分析过程中Calcite可以解析所有外部表和表。） 

## TableEnvironment Class

这是表API中的现有类，现在具有对CatalogManager实例的引用，该引用将添加以替换内存中的元对象和已注册的目录。 

```java
abstract class TableEnvironment(val config: TableConfig) {

  private val catalogManager:CatalogManager;
  
  // This is an existing class with only argument type change
  def registerCatalog(name: String, catalog: ReadableCatalog): Unit

  // Set the default catalog
  def setDefaultCatalog(catName: String);
  
  // Set the default database
  Def setDefaultDatabase(catName: String, databaseName: String): unit
}
```
TableEnvironment类当前具有一些采用不同表定义的registerTable方法，例如TableSource，TableSink和非公共类，例如Table RelTable和Table InlineTable。 这些API将保持不变。 但是，为了利用持久性目录，可能会更改其实现。 详细信息将在本设计系列的第2部分中提供。 

## YAML Configuration for Catalogs

以下是Flink中目录配置的演示。 可用的目录类型包括：内存中的flink，generic-hive-metastore和hive。 每个实现类和相应的工厂类的详细信息将在后续的设计文档中提供。 在这里，我们仅关注如何在YAML文件中指定目录。 

```yaml
catalogs:
-  name: hive1
    catalog:
     type: hive
     is-default: false
     default-db: default
     connection-params:
            hive.metastore.uris: “thrift://host1:10000,thrift://host2:10000”
            hive.metastore.username: “flink”

- name: flink1
   catalog:
     type: generic-hive-metastore
     is-default: true
     Default-db: default
     connection-params:
            hive.metastore.uris: “thrift://host1:10000,thrift://host2:10000”
            hive.metastore.username: “flink”
       Hive.metastore.db: flink
```

## TableFactory Interfaces

```java
// This is the existing interface.
interface TableFactory {

  Map<String, String> requiredContext();
  
  List<String> supportedProperties[j][k]();

}

// Utility classes providing implementations for conversions between CatallogTable
// and property map.
public class TableFactoryUtils {
    
  public static CatalogTable convertToCatalogTable(Map<String,String> properties) {
    // default implementation
  }

  Public static Map<String, String> convertToProperties(CatalogTable table) {
    // implementation
  }
}


Interface StreamTableSourceFactory extends TableFactory {

    // this one is existing one, which will be deprecated.
    @Deprecated
    StreamTableSource createStreamTableSource(Map<String, String> properties);
    
    // This one is new with default implementation.
    Default StreamTableSource createStreamTableSource(CatalogTable table) {
      return createStreamTableSource(
        TableFactoryUtils.convertToProperties(table) );
    }
}

Interface StreamTableSinkFactory extends TableFactory {

    // this one is existing one
    StreamTableSink createStreamSinkSource(Map<String, String> properties);
    
    // This one is new.
    Default StreamTableSink createStreamSinkSource(CatalogTable table) {
      return createStreamTableSink(
        TableFactoryUtils.convertToProperties(table) );
    }
}


Interface BatchTableSourceFactory extends TableFactory {
    
    // this one is existing one
    BatchTableSource createBatchTableSource(Map<String, String> properties);
    
    // This one is new.
    Default BatchTableSource createBatchTableSource(CatalogTable table) {
      return createBatchTableSource(
        TableFactoryUtils.convertToProperties(table) );
    }
}

Interface BatchTableSinkFactory extends TableFactory {
    // this one is existing one
    BatchTableSink createBatchTableSink(Map<String, String> properties);
    
    // This one is new.
    BatchTableSink createBatchTableSink(CatalogTable table) {
      return createBatchTableSink(
        TableFactoryUtils.convertToProperties(table) );
    }

}
```

## HiveTableFactory Class

```java
/**
  * Batch only for now.
  */
Public class HiveTableFactory implements BatchTableSourceFactory, BatchTableSinkFactory {

    Map<String, String> requiredContext() {
        // return an empty map to indicate that auto discovery is not needed.
        return new HashMap<>();
    }

    List<String> supportedProperties() {
        // Return an empty list to indicate that no check is needed.
        Return new ArrayList<>();
    }

    BatchTableSource createBatchTableSource(Map<String, String> properties) {
        // convert properties to catalogtable and call the other version of this method.
        // It’s fine not to support this method.
    }
    
    BatchTableSource createBatchTableSink(Map<String, String> properties) {
        // convert properties to catalogtable and call the other version of this method.
        // It’s fine not to support this method.
    }

    BatchTableSource createBatchTableSource(CatalogTable table) {
        Assert (table instanceof HiveTable);
               HiveTable hiveTable = (HiveTable)table;
               // create a table source based on HiveTable
               // This is specific implementation for Hive tables.
    }
    
    BatchTableSource createBatchTableSink(CatalogTable table) {
        Assert (table instanceof HiveTable);
               HiveTable hiveTable = (HiveTable)table;
        // create a table sink based on HiveTable
        // This is specific implementation for Hive tables.
        }
}
```
# 表工厂自动发现 
如果目录（例如上面的GenericHiveMetastoreCatalog）从其getTableFactory（）实现中返回null，则该框架将利用Java服务提供者接口（SPI）自动发现实际的表工厂。 这是Flink中定义的所有当前表的现有机制。 

# 附加条款
在一个系统中具有多个目录的情况下，必须通过目录名称，模式/数据库名称和表名称来标识表。因此，表引用需要包括目录名称，架构名称和表名称，例如hive1.risk_db.user_events。如果缺少目录名称，则表示默认目录（无论设置为默认目录）和默认数据库。

我们在Flink SQL中引入了默认数据库概念。这对应于SQL“使用xxx”，其中将架构（数据库）设置为当前架构，而没有数据库/架构前缀的任何表都引用默认架构。由于Flink具有多个目录，因此语法将为“ use cat1.db1”，其中cat1将是默认目录，而db1将是默认数据库。给定一个表名，目录管理器必须将其解析为全名，以便正确识别该表。

这与FLINK-6574中所做的更改形成对比，后者试图减少指定目录名称的需要。从理论上讲，这是不可行的，因为支持多个目录。初步测试表明，FLINK-6574未能达到预期的效果。相反，它造成了极大的概念混乱。因此，我们将审查并调整此工作中的更改。 


[原文](https://cwiki.apache.org/confluence/display/FLINK/FLIP-30%3A+Unified+Catalog+APIs)