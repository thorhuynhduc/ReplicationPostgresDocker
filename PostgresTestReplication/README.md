## Start Primary postgres-1
```
cd postgres-1 && docker compose up -d
```

## Create Replication User

In order to take a backup we will use a new PostgreSQL user account which has the permissions to do replication. </br>

Let's create this user account by logging into `postgres-1`:

```
cd postgres-1 && docker compose exec db bash

# create a new user
createuser -U postgresadmin -P -c 5 --replication replicationUser

exit
```

# Enable Write-Ahead Log and Replication

There is quite a lot to read about PostgreSQL when it comes to high availability. </br>

The first thing we want to take a look at is [WAL](https://www.postgresql.org/docs/current/wal-intro.html) </br>
Basically PostgreSQL has a mechanism of writing transaction logs to file and does not accept the transaction until its been written to the transaction log and flushed to disk. </br>

This ensures that if there is a crash in the system, that the database can be recovered from the transaction log. </br>
Hence it is "writing ahead". </br>

More documentation for configuration [wal_level](https://www.postgresql.org/docs/current/runtime-config-wal.html) and [max_wal_senders](https://www.postgresql.org/docs/current/runtime-config-replication.html)
```
wal_level = replica
max_wal_senders = 3
```

# Enable Archive

More documentation for configuration [archive_mode](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-ARCHIVE-MODE)

```
archive_mode = on
archive_command = 'test ! -f /mnt/server/archive/%f && cp %p /mnt/server/archive/%f'

```


## Take a base backup

To take a database backup, we'll be using the [pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) utility. </br>

The utility is in the PostgreSQL docker image, so let's run it without running a database as all we need is the `pg_basebackup` utility. <br/>
Note that we also mount our blank data directory as we will make a new backup in there:

```
cd /home/savvy/Projects/Test/ABC

docker run -it --rm --net postgres-1_postgres_replica -v ${PWD}/postgres-2/pgdata:/data --entrypoint /bin/bash postgres:15.0
```

Take the backup by logging into `postgres-1` with our `replicationUser` and writing the backup to `/data`.

```
pg_basebackup -h postgres-1 -p 5432 -U replicationUser -D /data/ -Fp -Xs -R
```

## Start standby postgres-2
```
cd postgres-2 && docker compose up -d
```

## Test the replication

Let's test our replication by creating a new table in `postgres-1` </br>
On our primary instance, lets do that:

```
# login to postgres
psql --username=postgresadmin postgresdb

#create a table
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

#show the table
\dt
```

Now lets log into our `postgres-2` instance and view the table:

```
# login to postgres
psql --username=postgresadmin postgresdb

#show the tables
\dt
```