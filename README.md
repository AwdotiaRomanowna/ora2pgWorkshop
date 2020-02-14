# ora2pgWorkshop

Workshop: https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/dms/tutorial-oracle-azure-postgresql-online.md

ora2pg image: https://hub.docker.com/r/georgmoser/ora2pg-docker

Oracle image: https://hub.docker.com/r/jaspeen/oracle-11g

HR DDLScript - Import the HR sample: https://www.oracle.com/database/technologies/appdev/datamodeler-samples.html

Examples: https://www.darold.net/confs/ora2pg_the_hard_way.pdf


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

mkdir /data /config
```
## Get inside container:
```
docker run -it --privileged -v /config/:/config -v /data:/data georgmoser/ora2pg-docker /bin/bash

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

## PostgreSQL PaaS

* [Create an instance in Azure Database for PostgreSQL](https://docs.microsoft.com/azure/postgresql/quickstart-create-server-database-portal).
* Connect to the instance and create a database using the instruction in this [document](https://docs.microsoft.com/azure/postgresql/tutorial-design-database-using-azure-portal).







docker run -it -v /root/migration/config/:/etc/ora2pg/ -v /root/migration/data:/data georgmoser/ora2pg-docker bash -c 'ora2pg -i /etc/ora2pg/ora_table.sql -t TABLE -b /data -o output.sql'


Copy data:
ora2pg -t COPY -o data.sql -b ./data -c ./config/ora2pg.conf
