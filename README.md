# ora2pgWorkshop

Useful links:
Full Windows-based workshop: https://github.com/microsoft/MCW-Migrating-Oracle-to-Azure-SQL-and-PostgreSQL/blob/master/Hands-on%20lab/HOL%20step-by-step%20-%20Migrating%20Oracle%20to%20PostgreSQL.md#task-4-install-ora2pg

Tutorial: https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/dms/tutorial-oracle-azure-postgresql-online.md

ora2pg image: https://hub.docker.com/r/georgmoser/ora2pg-docker

Oracle image: https://hub.docker.com/r/jaspeen/oracle-11g

HR DDLScript - Import the HR sample: https://www.oracle.com/database/technologies/appdev/datamodeler-samples.html

Examples: https://www.darold.net/confs/ora2pg_the_hard_way.pdf


The workshop was checked against Centos 7.

Setting up ora2pg:

```
sudo -i
```
From root user:
```
yum update -y
yum install docker -y

systemctl start docker
systemctl status docker

docker pull georgmoser/ora2pg-docker

mkdir /data
```
## Get inside container:
```
docker run -it --privileged -v /data:/data georgmoser/ora2pg-docker /bin/bash

ora2pg --version

apt-get update -y

apt-get install vim
```

Generate a migration project:

```
ora2pg --project_base /data --init_project myproject
```

change ORACLE_DSN in config file:

```
vi /data/myproject/config/ora2pg.conf

cd data/myproject/

```

### Discovery:

```
ora2pg -c config/ora2pg.conf -t SHOW_VERSION

ora2pg -c config/ora2pg.conf -t SHOW_SCHEMA 
```

Check which tables contain HR schema:

```
ora2pg -c config/ora2pg.conf -t SHOW_TABLE -n HR
```

List columns of table JOBS:

```
ora2pg -c config/ora2pg.conf -t SHOW_COLUMN -a 'TABLE[JOBS]' -n HR
```

### Generate report
```
ora2pg -c config/ora2pg.conf -t SHOW_REPORT --estimate_cost --dump_as_html -n HR > reports/report.html
```

## SCHEMA Migration
### Offline migration
Inside ```/data/myproject``` directory create a new directory:

```
mkdir offline
cd offline
```

Create a new file with following content:

```
vi countries.sql

CREATE TABLE COUNTRIES 
    ( 
     COUNTRY_ID CHAR (2 BYTE)  NOT NULL , 
     COUNTRY_NAME VARCHAR2 (40 BYTE) , 
     REGION_ID NUMBER 
    ) LOGGING 
;
```

now run ora2pg against this file:

```
root@13a8720887da:/data/myproject/offline# ora2pg -i countries.sql -t TABLE -c ../config/ora2pg.conf 
[========================>] 1/1 tables (100.0%) end of table export.

root@13a8720887da:/data/myproject/offline# ls
CONSTRAINTS_output.sql	INDEXES_output.sql  countries.sql  output.sql
```

Investigate the content of 3 files that has been created. 

### Online migration
Create online directory inside your project:
```
root@13a8720887da:/data/myproject/offline# mkdir /data/myproject/online
root@13a8720887da:/data/myproject/offline# cd /data/myproject/online
```

Get the sources of oracle procedures:
```
root@13a8720887da:/data/myproject/online# ora2pg -t PROCEDURE -o procedure.sql -c ../config/ora2pg.conf
[========================>] 2/2 procedures (100.0%) end of procedures export.
root@13a8720887da:/data/myproject/online# ls
ADD_JOB_HISTORY_procedure.sql  SECURE_DML_procedure.sql  procedure.sql
```

Investigate the content of generated files

Add ```-p``` to the command above to convert procedures to plpgsql:

```
root@13a8720887da:/data/myproject/online# ora2pg -p -t PROCEDURE -o procedure.sql -c ../config/ora2pg.conf
[========================>] 2/2 procedures (100.0%) end of procedures export.
root@13a8720887da:/data/myproject/online# ls
ADD_JOB_HISTORY_procedure.sql  SECURE_DML_procedure.sql  procedure.sql
```

The previous content has been overwritten. Check if conversion went well by exploring the content of the files.



### Schema export
Let run the export then!

change the name of the schema in ora2pg.conf file:

```
# Oracle schema/owner to use
SCHEMA  CHANGE_THIS_SCHEMA_NAME
```
So it looks as follows:

```
# Oracle schema/owner to use
SCHEMA  HR
```

Run the export. Make sure you are in the ```/data/myproject``` directory:

```
root@13a8720887da:/data/myproject# pwd
/data/myproject

root@13a8720887da:/data/myproject# ./export_schema.sh
```

Check the content of two directories:
* sources - where oracle sources are kept
* schema - with objects converted to PostgreSQL

## DATA Export
From ```/data/myproject``` run the following command:
```
ora2pg -t COPY -o data.sql -b ./data -c ./config/ora2pg.conf
```

Investigate the content of the ```data``` directory:
```
root@13a8720887da:/data/myproject# cd data/
root@13a8720887da:/data/myproject/data# ls
COUNTRIES_data.sql  DEPARTMENTS_data.sql  EMPLOYEES_data.sql  JOBS_data.sql  JOB_HISTORY_data.sql  LOCATIONS_data.sql  REGIONS_data.sql  data.sql
```

Direct export from Oracle to PostgreSQL
```
ora2pg -c config/ora2pg.conf -t COPY --pg_dsn "dbi:Pg:dbname=mydb;host=192.168.122.1;port=5432" --pg_user postgres
```



## PostgreSQL PaaS

* [Create an instance in Azure Database for PostgreSQL](https://docs.microsoft.com/azure/postgresql/quickstart-create-server-database-portal).
* Connect to the instance and create a database using the instruction in this [document](https://docs.microsoft.com/azure/postgresql/tutorial-design-database-using-azure-portal).

## Import to PostgreSQL PaaS
Create .pg_azure file with the credentials to your PaaS database:
```
vi .pg_azure

export PGDATABASE=ora2pg
export PGHOST=name.postgres.database.azure.com
export PGUSER=user@host
export PGPASSWORD=VeryBadPractice;)
export PGSSLMODE=require #if you are using PostgreSQL PaaS with SSL enabled
```

Save the ```.pg_azure``` file and load it in the session:
```
source .pg_azure
```

Create a new database:
```
createdb ora2pg
```

From the ```/data/myproject``` directory load the files:
```
psql -f schema/tables/table.sql

psql -f schema/sequences/sequence.sql
psql -f schema/views/view.sql 
psql -f schema/procedures/procedure.sql
psql -f schema/triggers/trigger.sql
```

Check if all objects were correctly created in your new postgres database.

Import data:
```
psql -f data/data.sql
```
Create constraints:
```
psql -f schema/tables/INDEXES_table.sql
psql -f schema/tables/CONSTRAINTS_table.sql
psql -f schema/tables/FKEYS_table.sql
```

## Test the migration
In order to lack of DBD::Pg perl module we are not able count rows. If you install the module the command to check migration looks as follows:

```
ora2pg -c config/ora2pg.conf -t TEST
```
## Online data migration
For online data migration approach please refer [here](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/dms/tutorial-oracle-azure-postgresql-online.md#when-the-postgresql-table-schema-already-exists)

# ./export_schema.sh
```
ora2pg -p -t TABLE -o table.sql -b ./schema/tables -c ./config/ora2pg.conf
ora2pg -p -t PACKAGE -o package.sql -b ./schema/packages -c ./config/ora2pg.conf
ora2pg -p -t VIEW -o view.sql -b ./schema/views -c ./config/ora2pg.conf
ora2pg -p -t SEQUENCE -o sequence.sql -b ./schema/sequences -c ./config/ora2pg.conf
ora2pg -p -t TRIGGER -o trigger.sql -b ./schema/triggers -c ./config/ora2pg.conf
ora2pg -p -t FUNCTION -o function.sql -b ./schema/functions -c ./config/ora2pg.conf
ora2pg -p -t PROCEDURE -o procedure.sql -b ./schema/procedures -c ./config/ora2pg.conf
ora2pg -p -t TABLESPACE -o tablespace.sql -b ./schema/tablespaces -c ./config/ora2pg.conf
ora2pg -p -t PARTITION -o partition.sql -b ./schema/partitions -c ./config/ora2pg.conf
ora2pg -p -t TYPE -o type.sql -b ./schema/types -c ./config/ora2pg.conf
ora2pg -p -t MVIEW -o mview.sql -b ./schema/mviews -c ./config/ora2pg.conf
ora2pg -p -t DBLINK -o dblink.sql -b ./schema/dblinks -c ./config/ora2pg.conf
ora2pg -p -t SYNONYM -o synonym.sql -b ./schema/synonyms -c ./config/ora2pg.conf
ora2pg -p -t DIRECTORY -o directorie.sql -b ./schema/directories -c ./config/ora2pg.conf
ora2pg -t PACKAGE -o package.sql -b ./sources/packages -c ./config/ora2pg.conf
ora2pg -t VIEW -o view.sql -b ./sources/views -c ./config/ora2pg.conf
ora2pg -t TRIGGER -o trigger.sql -b ./sources/triggers -c ./config/ora2pg.conf
ora2pg -t FUNCTION -o function.sql -b ./sources/functions -c ./config/ora2pg.conf
ora2pg -t PROCEDURE -o procedure.sql -b ./sources/procedures -c ./config/ora2pg.conf
ora2pg -t PARTITION -o partition.sql -b ./sources/partitions -c ./config/ora2pg.conf
ora2pg -t TYPE -o type.sql -b ./sources/types -c ./config/ora2pg.conf
ora2pg -t MVIEW -o mview.sql -b ./sources/mviews -c ./config/ora2pg.conf
```

# Issues
* DBD::Pg perl module isn't provided in the container. You might want to install it or use psql for import to PostgreSQL
* import_all.sh script isn't working in current version of ora2pg because of following lines: ```  pgdsn_defined=$(grep "^PG_DSN" config/ora2pg.conf | sed 's/.*dbi:Pg/dbi:Pg/')
  if [ "a$pgdsn_defined" = "a" ]; then
    if [ "a$DB_HOST" != "a" ]; then```
