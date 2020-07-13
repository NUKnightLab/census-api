# Using Docker and docker-compose for local development

A docker-compose.yml file is provided which builds a development stack including
both the API application and the Postgresql database.


## Preliminaries

If you wish to initialize the database with some data, download the appropriate
data file(s) from the [SQL dumps](https://censusreporter.tumblr.com/post/73727555158/easier-access-to-acs-data).

It is recommended that you only download what you need for development.

Place any downloaded sql.gz files in the postgres/data directory. These will
be copied to the docker pg instance and will be executed upon initialization
of the database. This initial data load will take some time!

There is information further below about how to subsequently load additional
data dump files into the database (See **Re-initializing the Database** below).

## Getting started

### Bring up the compose stack

```
 $ docker-compose build
 $ docker-compose run
```

If you have included any data dumps, these will take time to load (see the
**Preliminaries** section above.

After initialization, you may choose to delete the dump files from the docker
image:

```
 $ docker-compose run pg sh
 # rm /docker-entrypoint-initdb.d/*.sql.gz
 # exit
```

### Create the s3 bucket

You will need to have the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) installed.

```
 $ aws --endpoint-url=http://localhost:4566 s3 mb s3://embed.censusreporter.org
``` 

### Supporting full-text search

If you want to use full text search during development:

```
 $ cat full-text-search/metadata_script.sql | docker-compose run -e PGPASSWORD=censuspassword pg psql -h pg -p 5433 -U census```


### 5-year data

You will probably want to load at least one of the 1-year data dumps to do
development work. Some functionality, however, will still be broken if 5-year
data has not been loaded. This is a substantial amount of data, with a single
5-year dump being in the 100s of millions of rows, and is too much for many
systems.

I've had some initial success by loading a partial dump of 5-year data. E.g.:

```
 $ gzcat acs2018_5yr_backup.sql.gz|head -n 1000000 > acs2018_5yr_backup.1M.sql
 $ cat ~/Downloads/acs2018_5yr_backup.1M.sql| docker-compose run -e PGPASSWORD=censuspassword pg psql -h pg -p 5433 -U census
```

This will, of course, result in missing data for most locations, but the
general functionality of the application should be intact with data for some
municipalities.


### Connect to the API

You should now be able to connect to the API at localhost:5000. E.g.:

[http://localhost:5000/1.0/tabulation/01001](http://localhost:5000/1.0/tabulation/01001)


## Connecting to postgres with psql

```
 $ docker-compose run -e PGPASSWORD=censuspassword pg psql -h pg -p 5433 -U census
```

## Re-initializing the dB

Note that the dB initialization process is rather finicky. See the
**Iniitalizationn scripts** section of the documentation for the
[postgres Docker image](https://hub.docker.com/_/postgres) for more details.

The gist of troubleshooting the init process tends to come down to deleting
the existing database volume in order to get the initdb to re-execute, since
the **initializations scripts are only run a first time when the data volume is
empty**.

If you want to re-init the database, bring down the stack, remove the
data volume, and bring everything back up:

```
 $ docker-compose down
 $ rm -rf .pgdata
```

You may need to do this, e.g., if you want to load additional data from the
data dumps. Alternatively, you should be able to load the data dump file
directly, with something like the following:

```
 $ gzcat local/path/to/file.sql.gz | docker-compose run -e PGPASSWORD=censuspassword pg psql -h pg -p 5433 -U census
```

Note that the dump files tend to be very large. Overall, this development
strategy has not been tested with more than a single 1-year dump file for
initialization, followed by a subsequent load of a single year of tiger data.

## Gotchas

### initdb execution

The database init scripts are executed only if the data volume is empty. See
the process in **Re-initializing the dB** above for deleting the data volume.

### Order of initdb execution

The database init scripts are executed in their listed sort order (see the
[postgres Docker image docs](https://hub.docker.com/_/postgres) for details).
We have not tried to explicitly manipulate the order of the scripts in the
postgres/Dockerfile build copy process. However, it is easy to imagine a
scenario that might arise where the init order would matter.

### Environment variables

Some effort is made here to use the postgres environment variables for managing
connectivity configurations, at least for the docker-compose stack bootstrapping
process if not (yet) for the application itself. Some clarification is in order
about the nature of these variables:

 1. The PGxxx variables as set on the api container

These are client-side variables that indicate what configuration the psql
client would use when connecting to the database. These include:

  * PGDATABASE
  * PGUSER
  * PGPASSWORD
  * PGPORT
  * PGHOST

Note that the host must be the alias of the container, ie. **pg**, which is the
name of the service that is exposed by the Postgresql container.

 2. The POSTGRES_xxx variables set on the pg service

These variables are specific to the Docker image and indicate how the image
bootstrap should setup the initial container configuration, including the
default database, the user, and the password.

  * POSTGRES_USER
  * POSTGRES_DB
  * POSTGRES_PASSWORD

**Note:** when troubleshooting database initialization, these are the variables
that are being used to initialize the database.
