In general the commands in this article are based on the PSQL command-line.

## Sample Data

Download the [one year flight data](https://edu.postgrespro.com/demo-big-en.zip) database from the [PostgresPro page](https://postgrespro.com/community/demodb). 

    $ wget https://edu.postgrespro.com/demo-big-en.zip

Extract the zip file:

    $ unzip demo-big-en.zip

It extracts a file "demo-big-en-20170815.sql". Import this file into PostgreSQL. 

In the PSQL terminal (or using your favorite GUI), create a new database:
    
    CREATE DATABASE airlines;

As the 'postgres' user, in the Linux command-line, import the dump file into this database:

    psql airlines < demo-big-en-20170815.sql 

It is a large dump file, around 900 MB in size (after extracting). Restoring the dump into the database can take some time - a minute, or more. 


