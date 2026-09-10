# MongoDB Connectivity in Helical Insight

## How Helical Insight connects to MongoDB

Helical Insight does not talk to MongoDB with a plain JDBC driver. Like its other
"No SQL & Big Data" sources (DynamoDB, Elasticsearch, etc.), it connects through
**Apache Drill**: Drill ships a native MongoDB storage plugin, so Helical Insight
registers a Mongo connection as a Drill storage plugin and then queries it with
ordinary SQL through Drill's JDBC driver, exactly the same way it queries Hive or
any other Drill-backed source. MongoDB databases/collections simply show up as
Drill schemas/tables.

The full pipeline (all pre-existing in this codebase) is:

1. **UI** – The "Mongodb" data source in the "No SQL & Big Data" category
   (`server/hi-repository/System/Admin/Static/DataSourcesList.groovy`) collects
   host, port, database, collection, and credentials.
2. **Save/Test** – `NoSqlDataSourceProperties` / `NoSqlDataSourcePropertiesDB`
   (`server/core/.../efw/components/`) persist the connection and look up the
   Spring bean whose name equals the driver id `com.helicalinsight.nosql.mongo`
   via `NoSqlUtils.getNoSqlImplementation(subType)`.
3. **Bridge to Drill** – That bean is
   `MongoDrillLoader` (`server/adhoc/.../services/MongoDrillLoader.java`), which:
   - builds a `mongodb://` connection string from the form data,
   - `POST`s a Drill storage-plugin config to Drill's REST API
     (`DrillCsvDataSourceCreator`, the same helper used for CSV/flat-file
     sources), and
   - implements "Test Connection" using the MongoDB Java driver
     (`org.mongodb:mongo-java-driver`, already a dependency in `server/pom.xml`).
4. **Querying/metadata** – Once the storage plugin is registered, the existing
   Drill JDBC pathway (`org.apache.drill.jdbc.Driver`,
   `server/hi-repository/System/Admin/DbConfig/drill.efwd`) lists Mongo
   databases/collections as schemas/tables and runs SQL against them like any
   other Drill data source.

This mechanism was already present but had two bugs that prevented it from
working correctly.

## Changes made

1. **`server/adhoc/src/main/java/com/helicalinsight/adhoc/services/MongoDrillLoader.java`**
   - **Fixed "Test Connection" always failing for unauthenticated MongoDB
     instances.** In `MongoModel.testConnection()`, the no-credentials branch
     opened a `MongoClient` and never returned a result — control fell through
     to a hard-coded `return false;` at the end of the method, so testing a
     connection to a MongoDB server without a username/password always
     reported failure even when the connection succeeded. Added the missing
     `return this.mongoDb != null;` for that branch and removed the now-dead
     trailing `return false;`.
   - Removed the `@Deprecated` annotation on the class. It is the live,
     wired-in implementation for the `com.helicalinsight.nosql.mongo`
     connection type (see `DataSourcesList.groovy`, `NoSqlDataSourceProperties`,
     `NoSqlUtils`) and was misleadingly marked for removal; documented its role
     in a class-level Javadoc instead.

2. **`server/adhoc/src/main/java/com/helicalinsight/adhoc/metadata/ExtendedWorkflowMetadataProducer.java`**
   - Fixed `setDatabaseType(Metadata)`, which set the metadata's database type
     to the concatenated literal `"mongoDb" + "mongo"` (i.e. the nonsensical
     string `"mongoDbmongo"`). This is the code path used when building
     Workflow metadata for a MongoDB source without a live JDBC connection.
     The value now written is `"Mongodb"`, consistent with the display name
     Helical Insight already uses for Mongo elsewhere (`DataSourcesList.groovy`,
     the `Mongodb` / `Helical Mongodb` icon cases in
     `client/src/components/common/custom-icons/CustomIcon.jsx`).

Both changes are minimal, targeted bug fixes to existing code — no new
dependencies, no schema/API changes, and no impact on any other data source
type.

## Configuring and using a MongoDB connection

MongoDB support depends on Apache Drill being enabled, since Drill is the
engine that actually executes SQL against Mongo collections.

1. **Enable Apache Drill**
   - In the Helical Insight Admin console, open the Drill/Middleware
     configuration screen (backed by
     `server/hi-repository/System/Admin/drillConfig.xml`).
   - Set it enabled, and point it at a running Apache Drill cluster/instance
     (`host`, `port` — REST port 8047 by default, `dbPort` — JDBC port 31010 by
     default). Apache Drill's standard distribution bundles the MongoDB
     storage plugin, so no extra Drill-side installation is required.
2. **Create the MongoDB connection**
   - In Helical Insight, go to **Data Sources → Add Data Source**.
   - Under **No SQL & Big Data**, select **Mongodb** (only shown once Drill is
     enabled).
   - Fill in `hostName`, `port` (default `27017`), `database`, `collection`,
     and, if the MongoDB server requires authentication, `userName`/`password`.
   - Click **Test Connection** to verify connectivity, then **Save**. Saving
     registers a Drill storage plugin named `<connectionName>_<id>` pointing at
     that MongoDB instance.
3. **Use it**
   - The new connection appears wherever other data sources do (Ad-hoc
     reporting, Metadata/Cube builder, Workflow). Browsing catalogs/schemas
     lists MongoDB databases, and tables list MongoDB collections; queries are
     translated to Mongo's aggregation framework by Drill under the hood.

### Notes / limitations (pre-existing, unchanged by this work)

- MongoDB connectivity requires an external Apache Drill instance — Helical
  Insight does not talk to MongoDB directly.
- `databaseDrivers.properties` / `sqlDialects.properties` also contain
  entries for a proprietary `com.helical.mongodb.MongoJdbcDriver` /
  `cdata.jdbc.mongodb` JDBC driver. Those are optional commercial add-ons (the
  jar isn't part of this repository) and are unrelated to the Drill-based
  connector described above; they were left untouched.
