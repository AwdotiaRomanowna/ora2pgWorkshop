# ora2pgWorkshop

Workshop: https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/dms/tutorial-oracle-azure-postgresql-online.md

ora2pg image: https://hub.docker.com/r/georgmoser/ora2pg-docker

Oracle image: https://hub.docker.com/r/jaspeen/oracle-11g

HR DDLScript - Import the HR sample: https://www.oracle.com/database/technologies/appdev/datamodeler-samples.html



setenforce 0
docker run -it -v /root/migration/config/:/etc/ora2pg/ georgmoser/ora2pg-docker bash -c 'ora2pg  -t SHOW_VERSION'
