## How it works 

PostgreSQL has internal statistics about tables and data - rows, types of values and some info about (statistical) distribution of the data. Mostly based on random sampling. 

In general the commands in this article are based on the PSQL command-line.

## Sample Data

Download the [3-months flight data](https://edu.postgrespro.com/demo-medium-en.zip) database from the [PostgresPro page](https://postgrespro.com/community/demodb). 

    $ wget https://edu.postgrespro.com/demo-medium-en.zip

Extract the zip file:

    $ unzip demo-medium-en.zip

It extracts a file "demo-medium-en-20170815.sql". This file is a script to create a new database and populate it. It creates a new database "demo". Ensure you do not currently have any useful data in a database with the name "demo" - it will be dropped. 

In the Linux command-line, import the SQL script file into Postgres:

    $ psql -f demo-medium-en-20170815.sql -U postgres

It is a large dump file, around 250 MB in size (after extracting). Running the script to populate the database can take some time - a minute, or more. Check the database's [schema diagram](https://postgrespro.com/docs/postgrespro/10/apjs02) and read the [description of the tables and other objects](https://postgrespro.com/docs/postgrespro/10/apjs04).

## Examples

### Index Scans

This example shows how to use EXPLAIN to analyze the performance of a simple index. Run an query on the 'total_amount' column of the 'bookings' table. 

#### Check current indexes

Before running the query, check the current list of indexes and ensure there are no indexes on the column 'total_amount'.

In the PSQL command line, type:

    \d bookings

You can also use the command:

    select * from pg_indexes where tablename = 'bookings'

Run the EXPLAIN query 

    explain (analyze, buffers) select * from bookings where total_amount < 4000

It has a total cost of around and the query takes around __ ms to execute. 

Create a default index on the 'total_amount' column:

    create index bookings_amount on bookings(total_amount)

Re-run the same EXPLAIN query as before. 

### Index Only Scans

    explain (analyze, buffers) select total_amount from bookings where total_amount < 4000

It does an index-only scan. ___



### HashAggregate 

    https://www.depesz.com/2013/05/09/explaining-the-unexplainable-part-3/

Try to make CTEs

Change order of more/less selective parts of the query


### Misc

    (select .. where ..) join (select .. where ..)

    -- change order 
    

left join - change order

not in instead of outer join

    https://www.quora.com/What-are-2-simple-use-cases-for-left-right-inner-and-outer-SQL-joins 

Disable seq scan to force index usage.
Delete fk to insert random rows - 
show diff bw left and inner join using indices

otherwise both get the same plan - seq scan. 
