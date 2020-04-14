# Bundle: dbsched

The `dbsched` bundle is preconfigured with the Pado scheduler to periodically execute jobs that dump database tables to CSV files from which it automatically extracts column information to generate the corresponding `VersionedPortable` classes. It then transforms the CSV records to objects using the generated classes before ingesting them into Hazelcast.

## Installing Bundle

```console
install_bundle -download bundle-hazelcast-3-app-pado_dbsched-perf_test_dbsched-cluster-dbsched
```

## Use Case

In this use case, to control the system load, the IT department has put restrictions on applications from accessing the database in real time, especially during the business hours. Only select applications are allowed to read the database in a batch mode. Writes are not permitted. Our task is to download the data from the database and feed it to Hazelcast as follows:

1. Dump table contents to CSV files
2. Create `VersionedPortable` classes based on CSV file contents
3. Transform CSV records to objects
4. Ingest the transformed data into Hazelcast
5. Automate and schedule jobs to periodically repeat the above steps

![DB Sched Screenshot](/images/db-sched.png)

## Job Scheduler and ETL

This bundle includes the Pado scheduler to simplify the ETL process. The Pado scheduler offers the following:

1. Schedule jobs that execute SQL queries and dump the results to CSV files.
2. Automatically generate Hazelcast `VersionedPortable` classes based on CSV file contents.
3. Automatically generate schema (metadata) files for ingesting `VersionedPortable` objects into Hazelcast maps.
4. Transform CSV file contents to `VersionedPortable` objects.
5. Ingest `VersionedPortable` objects into Hazelcast.

We could also use the popular schedulers like Spring Cloud Data Flow and Apache Nifi but we would need to manually perform data transformation.

## Database: `nw` Schema

Open MySQL Workbench and create the `nw` schema. When you run the test_group script (see below), the `customers` and `orders` tables will automatically be created by Hibernate.

## Building `perf_test_dbsched`

We need to download the MySQL binary files by building `perf_test_dbsched` as follows.

```console
cd_app perf_test_dbsched; cd bin_sh
./build_app
```

## Configuring Hibernate

The `perf_test_dbsched` app has been preconfigured to connect to MySQL on localhost with the user name `root` and the password `MySql123`. You can change the user name and password in `etc/hibernate.cfg-mysql.xml`.
We will be using the `perf_test` app to load data directly into the database tables. After the database has been loaded with data, we will execute the use case by first dumping database tables to CSV files.

```console
# Change database user name and password in the Hibernate config file.
cd_app perf_test_dbsched
vi etc/hibernate.cfg-mysql.xml
```

## Loading Data into Database

Let's load mock data into the `nw.customers` and `nw.oders` tables by executing the `test_group -db` command. The `-db` option directly loads data into the database instead of Hazelcast.

```console
cd_app perf_test_dbsched; cd bin_sh
./test_group -db -run -prop ../etc/group-factory.properties
```

## Building Pado App

```console
cd_app pado_dbsched; cd bin_sh
./build_app
```

## Configuring and Running Pado Scheduler

1. Encrypt the database password. Copy the encrypted password.

```console
cd_app pado_dbsched
cd pado_<version>/bin_sh/tools
./encryptor
```

2. Copy the included scheduler job to the scheduler's data directory.

```console
cd_app pado_dbsched
cp -r scheduler pado_<version>/data/
```

3. Edit the job file and enter the user name and encrypted password.

```console
cd pado_<version>/data/scehduler/etc
vi mysql.json
```

The `mysql.json` file contents are shown below.

```json
{
        "Driver": "com.mysql.cj.jdbc.Driver",
        "Url": "jdbc:mysql://localhost:3306/nw?allowPublicKeyRetrieval=true&serverTimezone=EST",
        "User": "root",
        "Password": "yMgF43JvHM0fWSHDCA1GmQ==",
        "Delimiter": ",",
        "Null": "'\\N'",
        "GridId": "dbsched",
        "Paths": [
                {
                        "Path": "nw/customers",
                        "Columns": "customerId, address, city, companyName, contactName, contactTitle, country, fax, phone, postalCode, region",
                        "Query": "select * from nw.customers",
                        "Day": "Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday",                                "Time": "00:00:00"
                },
                {
                        "Path": "nw/orders",
                        "Columns": "orderId, customerId, employeeId, freight, orderDate, requiredDate, shipAddress, shipCity, shipCountry, shipName, shipPostalCode, shipRegion, shipVia, shippedDate",
                        "Query": "select * from nw.orders",
                        "Day": "Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday",                                "Time": "00:00:00, 01:00:00, 02:00:00, 03:00:00, 04:00:00, 05:00:00, 06:00:00, 07:00:00, 08:00:00, 09:00:00, 10:00:00, 11:00:00, 12:00:00, 13:00:00, 14:00:00, 15:00:00, 16:00:00, 17:00:00, 18:00:00, 19:00:00, 20:00:00, 21:00:00, 22:00:00, 23:00:00"
                }
        ]
}
```

:exclamation: Note that `serverTimezone` is set to `EST` for the JDBC URL. Without it, you may see the following exception if your MySQL uses the system timezone and unable to calculate the dates due to the leap year.

```console
com.mysql.cj.exceptions.WrongArgumentException: HOUR_OF_DAY: 2 -> 3
```

4. We need to first execute the `mysql.json` job without scheduling it to generate the schema files. Run the following command to download data into CSV files in the default `data/scheduler/import` directory.

```console
cd_app pado_dbsched
cd pado_<version>/bin_sh/hazelcast
./import_scheduler -now
```

5. Generate schema files using the downloaded data files.

```console
./generate_schema -schemaDir data/scheduler/schema -dataDir data/scheduler/import -package org.hazelcast.data.demo.nw
```

6. Generate the corresponding `VersionedPortable` source code in the default `src/generated` directory.

```console
./generate_versioned_portable -schemaDir data/scheduler/schema -fid 20000 -cid 20000
```

The factory class with the ID 20000 has already been configured in the Hazelcast member configuration file, `hazelcast.xml`, as shown below. If you specified the `-fid` value other than 20000 then you must also change the value in the file.

```console
switch_cluster dbsched
vi etc/hazelcast.xml
```

Change the `factory-id` value to a new value if you changed it.

```xml
   <serialization>
   ...
      <portable-factory factory-id="20000">
       org.hazelcast.data.demo.nw.PortableFactoryImpl
      </portable-factory>
   </serialization>
```

7. Compile the generated code and deploy the generated jar file to the workspace `plugins` directory so that it will be included in the cluster class path.

```console
cd_app pado_dbsched
cd pado_<version>/bin_sh/hazelcast
./compile_generated_code
cp ../../dropins/generated.jar $PADOGRID_WORKSPACE/plugins/
```

8. Start cluster.

```console
switch_cluster dbsched

# First add two (2) members
add_member; add_member

# Stat cluster
start_cluster
```

9. Import the downloaded data into the cluster.

```console
cd_app pado_dbsched
cd pado_<version>/bin_sh/hazelcast
./import_scheduler -import
```

Execute the following SQL statements on your database to verify the data.

```sql
select * from nw.customers;
select * from nw.orders;
```

10. Once you are satisfied with the results, you can schedule the job by executing the following.

```console
./import_scheduler -sched
```

## Tearing Down

```console
stop_cluster
```
