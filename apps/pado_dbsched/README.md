# App: Pado

The pado app provides a Hazelcast `Portable` class generator and CSV file import tools for Hazelcast. This bundle includes step-by-step instructions for ingesting mock data and UCI Machine Learning datasets into Hazelcast. It also includes a Pado scheduler demo that automates scheduled job executions for exporting and importing data from databases.

## Installing Pado App

The `pado` app currently comes in the form of a bundle. To install the pado bundle, Run the `install_bundle` command as follows:

```console
install_bundle -download bundle-imdg-3.12.4-app-pado.tar.gz
```

## Building Pado

```console
cd_app pado
cd bin_sh
./build_app
```

## Pado CSV `data` Directory

The Pado CSV `data` directory structure includes the `import` directory where you place the CSV files to import and the `schema` directory in which you provide schema files that define how to parse the CSV files. Pado automatically moves the successfully imported files from the `import` directory to the `processed` directory. It moves the unsuccessful ones in the `error` directory.

```console
data
├── error
├── import
├── processed
└── schema
```

## Running Pado CSV Importer

The Pado CSV importer facility automatically generates schema files, generates and compiles `VersionedPortable` classes, and imports CSV file contents into Hazelcast in the form of `VersionedPortable` objects. The imported data can then be viewed using the `desktop` app. These steps are shown in sequence below.

1. Place CSV files in the `data/import/` directory.
2. Generate schema files using the CSV files in `data/import/`.
3. Generate `VersionedPortable` source code.
4. Compile and create a `VersionedPortable` jar file.
5. Deploy the generated jar file to a Hazelcast cluster and add the `Portable` factory class in hazelcast.xml.
6. Start a Hazelcast cluster.
7. Import CSV files.
8. View imported data using the `desktop` app.

## NW Demo

For our demo, let's import the NW sample data included in the Pado distribution into Hazelcast. To import data in CSV files, you need to generate schema files. Pado provides the `generate_schema` command that auto-generates schema files based on CSV file contents. Once you have schema files ready, then you can generate Hazelcast `VersionedPortable` classes by executing the `generate_versioned_portable` command.

1. Change directory to the `pado` directory and copy the NW CSV files to the import directory. 

```console
cd_app pado
cd pado_<version>

# Copy CSV files into data/import/
cp -r data/nw/import data/
```

2. Generate schema files.

First, edit `bin_sh/setenv.sh` file and set the correct path to `JAVA_HOME`.

```console
vi bin_sh/setenv.sh
```

Generate schema files for the `nw` data

```console
# Generate schema files. The following command generates schema files in the
# data/schema/generated directory.
cd bin_sh/hazelcast
./generate_schema

# Move the generated schema files to data/schema.
mv ../../data/schema/generated/* ../../data/schema/
```

3. Generate `VersionedPortable` source code. The following command reads schema files located in data/schema/ and generates the corresponding `VersionedPortable Java source code.

```console
# Generate VersionedPortable classes with the factory ID of 30000 and the
# start class ID of 30000.
./generate_versioned_portable  -fid 30000 -cid 30000
```

4. Compile and create jar file.

```console
./compile_generated_code
```

5. Deploy the generated jar file to Hazelcast cluster and add the Portable factory class ID in hazelcast.xml.

```console
# Copy the jar file to the hazelcast-addon workspace plugins directory
cp ../../../pado_<vesion>/dropins/generated.jar $PADOGRID_WORKSPACE/plugins/

# Add the Portable factory class ID in hazelcast.xml
switch_cluster myhz

# In hazelcast.xml, add the serialization configuration outputted by
# the generate_versioned_portable command in step 3.
vi etc/hazelcast.xml

<!-- Find the serialization element in ect/hazelast.xml and add the portable-factory
     element shown below. -->
             <serialization>
                 <portable-factories>
                     <portable-factory factory-id="30000">
                          org.hazelcast.data.PortableFactoryImpl
                     </portable-factory>
                 </portable-factories>
             </serialization>
```

6. Start Hazelcast cluster

```console
start_cluster
```

7. Import CSV files.

```console
cd_app pado
cd pado_<version>/bin_sh/hazelcast
./import_csv
```

8. View imported data using the `desktop` app.

```console
# If you haven't installed the desktop app then install and build it.
create_app -app desktop
cd_app desktop
cd bin_sh
./build_app

# Change directory to hazelcast-desktop
cd ../hazelcast-desktop_<verson>

# Add serialization configuration in etc/pado.properties
vi etc/pado.properties

hazelcast.client.config.serialization.portable.factories=1:org.hazelcast.demo.nw.data.PortableFactoryImpl,\
10000:org.hazelcast.addon.hql.impl.PortableFactoryImpl,\
30000:org.hazelcast.data.PortableFactoryImpl

# Run desktop
cd bin_sh
./desktop
```

## Dataset Examples

The following links provide Pado instructions for ingesting downloadable datasets.

- [UCI Machine Learning Repository](UCI-ML.md)

## Scheduler Demo

Pado includes an ETL scheduler that automates exporting data from databases (and other external systems), and importing them into Hazelcast clusters. You create and schedule jobs in JSON form to periodically export data from any databases via JDBC. Each job defines the required JDBC connectivity and driver information and one or more grid paths (map names) with their query strings and scheduled time information.

Once you have created jobs, you can run them immediately without having the scheduler enabled. This allows you to quickly test your configurations but more importantly, generate the required schema files. You would generate the schema files in the same way as you did in the [NW Demo](#NW-Demo) section. The default scheduler directory is `data/scheduler` and has the same hierarchy as the CSV data directory described previously in the [Pado CSV `data` Directory](#Pado-CSV-data-Directory) section.

```console
data/scheduler
├── error
├── import
├── processed
└── schema
```

To run the scheduler demo, you need read/write access to a database. For our demo, we will be using MySQL.
 
1. Get access to a database. You need to encrypt your password as follows. Copy the encrypted password, which we will insert in the job file in step 3.

```console
cd_app pado
cd pado_<version>/bin_sh/tools
./encryptor
```

2. Copy the scheduler template directory and create jobs that dump database tables to CSV files.

```console
# Copy the entire template scheduler directory
cp -r data/template/scheduler data/

# IMPORTANT: Remove the files that came with the template. We don't need them.
rm data/scheduler/etc/*
rm data/scheduler/schema/*
```

Create the `mysql.json` file.

```
cd data/scheduler/etc
vi mysql.json
```

3. Enter query information in the `mysql.json` file as shown below. Copy/paste the encrypted password in the file. Set the `GridId` attribute to the Hazelcast cluster name. Set the `Path` attributes to the map names. 

```json
{
        "Driver": "com.mysql.cj.jdbc.Driver",
        "Url": "jdbc:mysql://localhost:3306/nw?allowPublicKeyRetrieval=true&serverTimezone=EST",
        "User": "root",
        "Password": "yMgF43JvHM0fWSHDCA1GmQ==",
        "Delimiter": ",",
        "Null": "'\\N'",
        "GridId": "myhz",
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

Note that `serverTimezone` is set to `EST` for the JDBC URL. Without it, you may see the following exception if your MySQL uses the system timezone and unable to calculate the dates due to the leap year.

```console
com.mysql.cj.exceptions.WrongArgumentException: HOUR_OF_DAY: 2 -> 3
```

We have configured two jobs in the `mysql.json` file. The first job downloads the `customers` table every midnight and the second job downloads the `orders` table every hour. We could have configured with more practical queries like downloading just the last hour's worth of orders, for example. For the demo purpose, let's keep it simple and fluid. Our main goal is to ingest the database data into Hazelcast.

4. We need to create the schema files for properly reading and transforming CSV file contents to Hazelcast objects. We can manually create the schema files or simply generate them. To generate the schema files, we need CSV files. This is done by executing the `import_scheduler -now` command which generates CSV files without scheduling the jobs in the default directory, `data/scheduler/import`.

```console
cd_app pado
cd pado_<version>/bin_sh/hazelcast
./import_scheduler -now
```

5. Generate schema files using the downloaded data files.

```console
./generate_schema -schemaDir data/scheduler/schema -dataDir data/scheduler/import -package org.hazelcast.data.demo.nw
```

6. Generate the corresponding `VersionedPortable` source code in the default directory, `src/generated`.

```console
./generate_versioned_portable -schemaDir data/scheduler/schema -fid 20000 -cid 20000
```

7. Compile the generated code and deploy the generated jar file to the workspace `plugins` directory so that it will be included in the cluster class path.

```console
./compile_generated_code
cp ../../dropins/generated.jar $PADOGRID_WORKSPACE/plugins/
```

8. Start cluster.

```console
start_cluster
```

9. Import the downloaded data into the cluster.

```console
./import_scheduler -import
```

10. Once you are satisfied with the results, you can schedule the job by executing the following.

```console
./import_scheduler -sched
```

## About Pado

Pado is authored by Dae Song Park (email:dspark@netcrest.com) to bring linear scalability to IMDG for storing Big Data. His architecture achieves this by logically federating data grids and providing an abstract API layer that not only hides the complexity of the underlying IMDG API but introduces new Big Data capabilities that IMDG products lack today. He coined the terms **grids within grid** and **grid of grids** to illustrate his architecture which spans in-memory data across a massive number of clusters with a universal namespace similar to URL for easy data access.

The current implementation of Pado only supports Pivotal GemFire 8.x. Unfortunately, the API transformation of GemFire to Apache Geode is requiring a major overhaul to Pado. Stay tuned.

The `hazelcast-addon` project was inspired by Pado and borrows many architecture and script ideas from Pado.

