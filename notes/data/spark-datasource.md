
# Spark Data source

Apache Spark provides the ability to plug in custom data sources into its compute engine, through the Data Source API. Apache Spark version 2.3.0 introduced a revised data source API (*Jira: <https://issues.apache.org/jira/browse/SPARK-15689>, and design doc: <https://docs.google.com/document/d/1n_vUVbF4KD3gxTmkNEon5qdQ-Z8qU5Frf6WMQZ6jJVM/>*), this v2 API is continually evolving, especially in Spark version 3.0.\*. There is still backward compatibility support for V1 API in latest Spark versions (*at least for 3.0.\**). The following sections shows the main interfaces/traits needed to create a custom data source.

## Data source V1

* Git - <https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/sources/interfaces.scala>
* core - `org.apache.spark.sql.sources`

name| type| since| note
-|-|-|-
`DataSourceRegister`| trait| 1.5.0| A data source should implement this trait to register an alias used for referencing it, over the fully qualified class name.
`RelationProvider`| trait| 1.3.0| Produce relations for a specific kind of data source.
`SchemaRelationProvider`| trait| 1.3.0| Produce relations for a specific kind of data source with a given schema.
`StreamSourceProvider`| trait| 2.0.0| Produce a streaming `Source` for a specific format or system.
`StreamSinkProvider`| trait| 2.0.0| Produce a streaming `Sink` for a specific format or system.
`CreatableRelationProvider`| trait| 1.3.0| Saves a DataFrame to a destination (using data source-specific parameters).
`BaseRelation`| trait| 1.3.0| Represents a collection of tuples with a known schema. Concrete implementation should inherit from one of the descendant `Scan` classes.
`TableScan`| trait| 1.3.0| A `BaseRelation` that can produce all of its tuples as an RDD of Row objects.
`PrunedScan`| trait| 1.3.0| A `BaseRelation` that can eliminate unneeded columns before producing an RDD containing all of its tuples as Row objects.
`PrunedFilteredScan`| trait| 1.3.0| A `BaseRelation` that can eliminate unneeded columns and filter using selected predicates before producing an RDD containing all matching tuples as Row objects.
`InsertableRelation`| trait| 1.3.0| A `BaseRelation` that can be used to insert data into it through the insert method.
`CatalystScan`| trait| 1.3.0| An interface for experimenting with a more direct connection to the query planner. Compared to `PrunedFilteredScan`, this operator receives the raw expressions from the `org.apache.spark.sql.catalyst.plans.logical.LogicalPlan`.

> The difference between a `RelationProvider` and a `SchemaRelationProvider` is that users need to provide a schema when using a `SchemaRelationProvider`.  
A relation provider can inherit both `RelationProvider` and `SchemaRelationProvider` if it can support both schema inference and user-specified schemas.

## Data source V2 - Spark 2.4.*

* Git - <https://github.com/apache/spark/tree/branch-2.4/sql/core/src/main/java/org/apache/spark/sql/sources/v2>
* core - `org.apache.spark.sql.sources.v2`
* reader - `org.apache.spark.sql.sources.v2.reader`
* writer - `org.apache.spark.sql.sources.v2.writer`

name| type| note
-|-|-
`DataSourceV2`| | Marker interface to indicate support for V2 data source.
`ReadSupport`| reader| Indicate support for reading.
`DataSourceReader`| reader| Return actual data schema, and input partitions; each partition outputs one RDD partition.
`InputPartition`| reader| An input partition responsible for creating the actual data reader of one RDD partition.
`InputPartitionReader`| reader| An input partition reader responsible for outputting data for a RDD partition.
`SupportsPushDownFilters`| reader| Implement this interface to push down filters to the data source.
`SupportsPushDownRequiredColumns`| reader| Implement this interface to push down required columns to the data source and only read these columns during scan.
`SupportsReportPartitioning`| reader| Implement this interface to report data partitioning and try to avoid shuffle at Spark side.
`SupportsReportStatistics`| reader| Implement this interface to report statistics to Spark.
`SupportsScanColumnarBatch`| reader| Implement this interface to output `ColumnarBatch` and make the scan faster.
`WriteSupport`| writer| Indicate support for writing.
`DataSourceWriter`| writer| Data source writer. It can mix in various writing optimization interfaces to speed up the data saving. The actual writing logic is delegated to `DataWriter`.
`DataWriter`| writer| A data writer responsible for writing data for an input RDD partition.

## Data source V2 - Spark 3.0.*

* Git - <https://github.com/apache/spark/tree/master/sql/catalyst/src/main/java/org/apache/spark/sql/connector>
* core - `org.apache.spark.sql.connector.catalog`
* reader - `org.apache.spark.sql.connector.read`
* writer - `org.apache.spark.sql.connector.write`

path| name|  note
-|-|-
core| `CatalogExtension`| An API to extend the Spark built-in session catalog.
core| `CatalogPlugin`| A marker interface to provide a catalog implementation for Spark.
core| `DelegatingCatalogExtension`| A simple implementation of `CatalogExtension`, which implements all the catalog functions by calling the built-in session catalog directly.
core| `SessionConfigSupport`| Implement to propagate session configs with the specified *key-prefix* to all data source operations in this session.
core| `StagingTableCatalog`| Support staging creation of a table before committing the table's metadata along with its contents. It is highly recommended to implement this trait whenever possible so that `CREATE TABLE AS SELECT` (*CTAS*) and `REPLACE TABLE AS SELECT` (*RTAS*) operations are atomic.
core| `SupportsAtomicPartitionManagement`| An atomic partition interface of `Table` to operate multiple partitions atomically.
core| `SupportsCatalogOptions`| An interface, which `TableProvider` can implement, to support table existence checks and creation through a catalog, without having to use table identifiers.
core| `SupportsDelete`| Data sources can implement this interface to provide the ability to delete data from tables that matches filter expressions.
core| `SupportsNamespaces`| Catalog methods for working with namespaces.
core| `SupportsPartitionManagement`| These APIs are used to modify table partition identifier or partition metadata. In some cases, they will change the table data as well.
core| `SupportsRead`| Indicate a `Table` is readable.
core| `SupportsWrite`| Indicate a `Table` is writable.
core| `Table`| A logical structured data set of a data source. For example, the implementation can be a directory on the file system, a topic of Kafka, or a table in the catalog, etc.
core| `TableCapability`| Capabilities that can be provided by a `Table` implementation.
core| `TableCatalog`| Catalog methods for working with Tables.
core| `TableProvider`| The base interface for v2 data sources *which don't have a real catalog*. The major responsibility of this interface is to return a `Table` for read/write.
read| `ScanBuilder`| An interface for building the `Scan`.
read| `Scan`| A logical representation of a data source scan.
read| `Batch`| A physical representation of a data source scan for batch queries. Contains information like how many partitions the scanned data has, and how to read records from the partitions.
read| `PartitionReader`| Responsible for outputting data for a RDD partition.
read| `SupportsPushDownFilters`| Implement this interface to push down filters to the data source.
read| `SupportsPushDownRequiredColumns`| Implement this interface to push down required columns to the data source and only read these columns during scan.
read| `SupportsReportPartitioning`| Implement this interface to report data partitioning and try to avoid shuffle at Spark side.
read| `SupportsReportStatistics`| Implement this interface to report statistics to Spark.
write| `WriteBuilder`| An interface for building the `BatchWrite`.
write| `BatchWrite`| Defines how to write the data to data source for batch processing.
write| `DataWriter`| Responsible for writing data for an input RDD partition.
write| `SupportsDynamicOverwrite`| For tables that support dynamic partition overwrite. A write that dynamically overwrites partitions removes all existing data in each logical partition for which the write will commit new data.
write| `SupportsOverwrite`| For tables that support overwrite by filter. Overwriting data by filter will delete any data that matches the filter and replace it with data that is committed in the write.
write| `SupportsTruncate`| For tables that support truncation. Truncation removes all data in a table and replaces it with data that is committed in the write.

1 key improvement for V2 in spark 3.0.\* is the support of custom catalogs. This opens up a lot more possibilities on how people manage data in a data lake, or some other data store. Projects like Apache Iceberg (<https://iceberg.apache.org/>), Delta Lake (<https://delta.io/>) provides good alternatives to Hive metastore. For me, this also opens up a better way to audit, and auto-document data assets.

## Data source Fileformat

TBD

## Create and register a data source

When spark encounters `spark.read.format("<SOME DATA SOURCE FORMAT>").load()`, the `load` method in the `DataFrameReader` tries to locate the implementation that can handle the specified format. Spark uses the Java ServiceLoader (<https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/ServiceLoader.html>) to discover and dynamically load all user defined data sources. These are the steps for creating and registering a bare data source:

1. Create class that implements the interface `org.apache.spark.sql.sources.DataSourceRegister`.
1. Create folder `META-INF/services/`; if maven project, this would be within `src/main/resources/META-INF/services/`.
1. Insider that folder, create file `org.apache.spark.sql.sources.DataSourceRegister`.
1. The file content should contain the full class path defined in step 1.

> Data sources should implement this trait so that they can register an alias to their data source. This allows users to give the data source alias as the format type over the fully qualified class name.<br />  
> A new instance of this class will be instantiated each time a DDL call is made. - <https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/sources/DataSourceRegister.html>

Once you have a bare data source, you are ready to hack on with the remaining dat source API mentioned prior.

## References

<https://www.river-of-bytes.com/2018/04/revisiting-spark-external-data-sources.html>

<https://www.programmersought.com/article/1042437394/>

<https://www.shangmayuan.com/a/7d6731ff00d84971b577b6f1.html>

<http://shzhangji.com/blog/2018/12/08/spark-datasource-api-v2/>

<https://zhuanlan.zhihu.com/p/83006243>

<http://blog.madhukaraphatak.com/categories/datasource-v2-series/>

<http://blog.madhukaraphatak.com/categories/datasource-v2-spark-three/>

<https://youtu.be/YKkgVEgn2JE>

<https://youtu.be/vfd83ELlMfc>

<https://issues.apache.org/jira/browse/SPARK-15689>
