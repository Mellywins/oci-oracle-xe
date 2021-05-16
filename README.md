# oci-oracle-xe
Oracle Database Express Edition Container / Docker images.

**The images are compatible with `podman` and `docker`. You can use `podman` or `docker` interchangeably.**

# Supported tags and respective `Dockerfile` links

* [`18.4.0`, `18`, `latest`](https://github.com/gvenzl/oci-oracle-xe/blob/main/Dockerfile.1840)
* [`18.4.0-full`, `18-full`, `full`](https://github.com/gvenzl/oci-oracle-xe/blob/main/Dockerfile.1840)
* [`11.2.0.2`, `11`](https://github.com/gvenzl/oci-oracle-xe/blob/main/Dockerfile.11202)
* [`11.2.0.2-slim`, `11-slim`](https://github.com/gvenzl/oci-oracle-xe/blob/main/Dockerfile.11202)
* [`11.2.0.2-full`, `11-full`](https://github.com/gvenzl/oci-oracle-xe/blob/main/Dockerfile.11202)

# Quick Start

Run a new database container:

```shell
docker run -d -p 1521:1521 -e ORACLE_PASSWORD=<your password> gvenzl/oracle-xe
```

Run a new persistent **18c** database container:

```shell
docker run -d -p 1521:1521 -e ORACLE_PASSWORD=<your password> -v oracle-volume:/opt/oracle/oradata gvenzl/oracle-xe
```

Run a new persistent **11g R2** database container:

```shell
docker run -d -p 1521:1521 -e ORACLE_PASSWORD=<your password> -v oracle-volume:/u01/app/oracle/oradata gvenzl/oracle-xe:11
```

Reset database `SYS` and `SYSTEM` passwords:

```shell
docker exec <container name|id> resetPassword <your password>
```

# How to use this image

## Subtle differences between versions

The 11gR2 (11.2.0.2) Oracle Database version stores the database data files under `/u01/app/oracle/oradata/XE`.  
**A volume for 11gR2 has to be pointed at `/u01/app/oradata`!**

## Environment variables

### `ORACLE_PASSWORD`
This variable is mandatory for the first container startup and specifies the password for the Oracle Database `SYS` and `SYSTEM` users.

### `ORACLE_RANDOM_PASSWORD`
This is an optional variable. Set this variable to a non-empty value, like `yes`, to generate a random initial password for the `SYS` and `SYSTEM` users. The generated password will be printed to stdout (`ORACLE PASSWORD FOR SYS AND SYSTEM: ...`).

### `APP_USER`
This is an optional variable. Set this variable to a non-empty string to create a new database schema user with the name specified in this variable. This variable requires `APP_USER_PASSWORD` or `APP_USER_PASSWORD_FILE` to be specified as well.

### `APP_USER_PASSWORD`
This is an optional variable. Set this variable to a non-empty string to define a password for the database schema user specified by `APP_USER`. This variable requires `APP_USER` to be specified as well.

## Container secrets

As an alternative to passing sensitive information via environment variables, `_FILE` may be appended to some of the previously listed environment variables, causing the initialization script to load the values for those variables from files present in the container. In particular, this can be used to load passwords from Container/Docker secrets stored in `/run/secrets/<secret_name>` files. For example:

```shell
docker run --name some-oracle -e ORACLE_PASSWORD_FILE=/run/secrets/oracle-passwd -d gvenzl/oracle-xe
```

This mechanism is supported for:

* `ORACLE_PASSWORD`
* `APP_USER_PASSWORD`

## Initialization scripts
If you would like to perform additional initialization of the database running in a container, you can add one or more `*.sql`, `*.sql.gz`, `*.sql.zip` or `*.sh` files under `/container-entrypoint-initdb.d` (creating the directory if necessary). After the database setup is completed, these files will be executed automatically in alphabetical order.

The `*.sql`, `*.sql.gz` and `*.sql.zip` files will be executed in Sql*Plus as the `SYS` user connected to the Oracle instance (`XE`). Compressed files will be uncompressed on the fly, allowing for e.g. bigger data loading scripts to save space.

Executable `*.sh` files will be run in a new shell process while non-executable `*.sh` files (files that do not have the Linux e`x`ecutable permission set) will be sourced into the current shell process. The main difference between these methods is that sourced shell scripts can influence the environment of the current process and should generally be avoided. However, sourcing scripts allows for execution of these scripts even if the executable flag is not set for the files containing them. This basically avoids the "why did my script not get executed" confusion.

***Note:*** scripts in `/container-entrypoint-initdb.d` are only run the first time the database is initialized; any pre-existing database will be left untouched on container startup.

***Note:*** you can also put files under the `/docker-entrypoint-initdb.d` directory. This is kept for backwards compatibility with other widely used container images but should generally be avoided. Do not put files under `/container-entrypoint-initdb.d` **and** `/docker-entrypoint-initdb.d` as this would cause the same files to be executed twice!

***Warning:*** if a command within the sourced `/container-entrypoint-initdb.d` scripts fails, it will cause the main entrypoint script to exit and stop the container. It also may leave the database in an incomplete initialized state. Make sure that shell scripts handle error situations gracefully and ideally do not source them!

***Warning:*** do not exit executable `/container-entrypoint-initdb.d` scripts with a non-zero value (using e.g. `exit 1;`) unless it is desired for a container to be stopped! A non-zero return value will tell the main entrypoint script that something has gone wrong and that the container should be stopped.

### Example

The following example installs the [countries, cities and currencies sample data set](https://github.com/gvenzl/sample-data/tree/master/countries-cities-currencies) under a new user `TEST` into the database:

```shell
[gvenzl@localhost init_scripts]$ pwd
/home/gvenzl/init_scripts

[gvenzl@localhost init_scripts]$ ls -al
total 12
drwxrwxr-x   2 gvenzl gvenzl   61 Mar  7 11:51 .
drwx------. 19 gvenzl gvenzl 4096 Mar  7 11:51 ..
-rw-rw-r--   1 gvenzl gvenzl  134 Mar  7 11:50 1_create_user.sql
-rwxrwxr-x   1 gvenzl gvenzl  164 Mar  7 11:51 2_create_data_model.sh

[gvenzl@localhost init_scripts]$ cat 1_create_user.sql
ALTER SESSION SET CONTAINER=XEPDB1;

CREATE USER TEST IDENTIFIED BY test QUOTA UNLIMITED ON USERS;

GRANT CONNECT, RESOURCE TO TEST;

[gvenzl@localhost init_scripts]$ cat 2_create_data_model.sh
curl -LJO https://raw.githubusercontent.com/gvenzl/sample-data/master/countries-cities-currencies/install.sql

sqlplus -s test/test@//localhost/XEPDB1 @install.sql

rm install.sql

```

As the execution happens in alphabetical order, numbering the files will guarantee the execution order. A new container started up with `/home/gvenzl/init_scripts` pointing to `/container-entrypoint-initdb.d` will then execute the files above:

```shell
podman run --name test \
>          -p 1521:1521 \
>          -e ORACLE_RANDOM_PASSWORD="y" \
>          -v /home/gvenzl/init_scripts:/container-entrypoint-initdb.d \
>      gvenzl/oracle-xe:18.4.0-full
CONTAINER: starting up...
CONTAINER: first database startup, initializing...
...
CONTAINER: Executing user defined scripts...
CONTAINER: running /container-entrypoint-initdb.d/1_create_user.sql ...

Session altered.


User created.


Grant succeeded.

CONTAINER: DONE: running /container-entrypoint-initdb.d/1_create_user.sql

CONTAINER: running /container-entrypoint-initdb.d/2_create_data_model.sh ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  115k  100  115k    0     0   460k      0 --:--:-- --:--:-- --:--:--  460k

Table created.
...
Table                provided actual
-------------------- -------- ------
regions                     7      7
countries                 196    196
cities                    204    204
currencies                146    146
currencies_countries      203    203


Thank you!
--------------------------------------------------------------------------------
The installation is finished, please check the verification output above!
If the 'provided' and 'actual' row counts match, the installation was successful
.

If the row counts do not match, please check the above output for error messages
.


CONTAINER: DONE: running /container-entrypoint-initdb.d/2_create_data_model.sh

CONTAINER: DONE: Executing user defined scripts.


#########################
DATABASE IS READY TO USE!
#########################
...
```

As a result, one can then connect to the new schema directly:

```shell
[gvenzl@localhost init_scripts]$  sql test/test@//localhost/XEPDB1

SQLcl: Release 20.3 Production on Sun Mar 07 12:05:06 2021

Copyright (c) 1982, 2021, Oracle.  All rights reserved.

Connected to:
Oracle Database 18c Express Edition Release 18.0.0.0.0 - Production
Version 18.4.0.0.0


SQL> select * from countries where name = 'Austria';

COUNTRY_ID COUNTRY_CODE NAME    OFFICIAL_NAME       POPULATION AREA_SQ_KM LATITUDE LONGITUDE TIMEZONE      REGION_ID
---------- ------------ ------- ------------------- ---------- ---------- -------- --------- ------------- ---------
AUT        AT           Austria Republic of Austria    8793000      83871 47.33333  13.33333 Europe/Vienna EU

SQL>
```

## Startup scripts

If you would like to perform additional action after the database running in a container has been started, you can add one or more `*.sql`, `*.sql.gz`, `*.sql.zip` or `*.sh` files under `/container-entrypoint-startdb.d` (creating the directory if necessary). After the database is up and ready for requests, these files will be executed automatically in alphabetical order.

The execution order and implications are the same as with the [Initialization scripts](#initialization-scripts) described above.

***Note:*** you can also put files under the `/docker-entrypoint-startdb.d` directory. This is kept for backwards compatibility with other widely used container images but should generally be avoided. Do not put files under `/container-entrypoint-startdb.d` **and** `/docker-entrypoint-startdb.d` as this would cause the same files to be executed twice!

***Note:*** if the database inside the container is initialized (started for the first time), startup scripts are executed after the setup scripts.

***Warning:*** files placed in `/container-entrypoint-startdb.d` are always executed after the database in a container is started, including pre-created databases. Use this mechanism only if you wish to perform a certain task always after the database has been (re)started by the container.

# Image flavors

| Flavor  | Extension | Description                                                                                 | Use cases                                                                                              |
| --------| --------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------|
| Slim    | `-slim`   | An image focussed on smallest possible image size instead of additional functionality.      | Wherever small images sizes are important but advanced functionality of Oracle Database is not needed. |
| Regular | [None]    | A well-balanced image between image size and functionality. Recommended for most use cases. | Recommended for most use cases.                                                                        |
| Full    | `-full`   | An image containing all functionality as provided by the Oracle Database installation.      | Best for extensions and/or customizations.                                                             |

## 18c XE

### Full image flavor (`18-full`)

The full image provides an Oracle Database XE installation "as is", meaning as provided by the RPM install file.
A couple of modifications have been performed to make the installation more suitable for running inside a container.

#### Database settings

* `DBMS_XDB.SETLISTENERLOCALACCESS(FALSE)`
* An `OPS$ORACLE` externally identified user has been created and granted `CONNECT` and `SELECT_CATALOG_ROLE` (this is used for health check and other operations)
* `LOCAL_LISTENER=''`
* `COMMON_USER_PREFIX=''`

### Regular image flavor (`18`)

The regular image strives to balance between the functionality required by most users and image size. It has all customizations that the full image has and removes additional components to further decrease the image size:

#### Database components

* The `HR` schema has been removed
* `Oracle Workspace Manager` has been removed
* `Oracle Multimedia` has been removed
* `Oracle Database Java Packages` have been removed
* `Oracle XDK` has been removed
* `JServer JAVA Virtual Machine` has been removed
* `Oracle OLAP API` has been removed
* `OLAP Analytic Workspace` has been removed
* `OPatch` utility has been removed (`$ORACLE_HOME/OPatch`)
* `QOpatch` utility has been removed (`$ORACLE_HOME/QOpatch`)
* `Oracle Database Assistants` have been removed (`$ORACLE_HOME/assistants`)
* `Oracle Database Migration Assistant for Unicode` has been removed (`$ORACLE_HOME/dmu`)
* The `inventory` directory has been removed (`$ORACLE_HOME/inventory`)
* `JDBC` drivers have been removed (`$ORACLE_HOME/jdbc`, `$ORACLE_HOME/jlib`)
* `Universal Connection Pool` driver has been removed (`$ORACLE_HOME/ucp`)
* Intel Math Kernel libraries have been removed (`$ORACLE_HOME/lib/libmkl_*`)
* Other utilities have been removed (`$ORACLE_HOME/lib/*.zip`)
* Additional Java libraries have been removed (`$ORACLE_HOME/rdbms/jlib`)

#### Database settings

* The `DEFAULT` profile has the following set:
  * `FAILED_LOGIN_ATTEMPTS=UNLIMITED`
  * `PASSWORD_LIFE_TIME=UNLIMITED`

#### Operating system

* The following Linux packages are not installed: 
  * `glibc-devel`
  * `glibc-headers`
  * `kernel-headers`
  * `libpkgconf`
  * `libxcrypt-devel`
  * `pkgconf`
  * `pkgconf-m4`
  * `pkgconf-pkg-config`

## 11g XE

### Full image flavor (`11-full`)

The full image provides an Oracle Database XE installation "as is", meaning as provided by the RPM install file.
A couple of modifications have been performed to make the installation more suitable for running inside a container.

#### Database settings

* Automatic Memory Management has been disables (`MEMORY_TARGET`)
* `DBMS_XDB.SETLISTENERLOCALACCESS()` has been set to `FALSE`
* An `OPS$ORACLE` externally identified user has been created and granted `CONNECT` and `SELECT_CATALOG_ROLE` (this is used for health check and other operations)
* The `REDO` logs have been located into `$ORACLE_BASE/oradata/$ORACLE_SID/`
* The fast recovery area has been removed (`DB_RECOVERY_FILE_DEST=''`)

### Regular image flavor (`11`)

The regular image strives to balance between the functionality required by most users and image size. It has all customizations that the full image has and removes additional components to further decrease the image size:

#### Database components

* Oracle APEX has been removed (you can download and install the latest and greatest from [apex.oracle.com](https://apex.oracle.com)
* The `HR` schema has been removed
* `JDBC` drivers have been removed (`$ORACLE_HOME/jdbc`, `$ORACLE_HOME/jlib`)

#### Database settings

* The `DEFAULT` profile has the following set:
  * `FAILED_LOGIN_ATTEMPTS=UNLIMITED`
  * `PASSWORD_LIFE_TIME=UNLIMITED`

#### Operating system

* The following Linux packages are not installed:
  * `binutils`
  * `gcc`
  * `glibc`
  * `make`

### Slim image flavor (`11-slim`)

The slim images aims for smallest possible image size with only the Oracle Database relational components. It has all customizations that the regular image has and removes all non-relational components (where possible) to further decrease the image size:

#### Database components

* `Oracle Text` has been uninstalled and removed (`$ORACLE_HOME/ctx`)
* `XML DB` has been uninstalled
* `XDK` has been removed (`$ORACLE_HOME/xdk`)
* `Oracle Spatial` has been uninstalled and removed (`$ORACLE_HOME/md`)
* The demo samples directory has been removed (`$ORACLE_HOME/demo`)
* `ODBC` driver samples have been removed (`$ORACLE_HOME/odbc`)
* `TNS` demo samples have been removed (`$ORACLE_HOME/network/admin/samples`)
* `NLS` demo samples have been removed (`$ORACLE_HOME/nls/demo`)
* The hs directory has been removed (`$ORACLE_HOME/hs`)
* The ldap directory has been removed (`$ORACLE_HOME/ldap`)
* The precomp directory has been removed (`$ORACLE_HOME/precomp`)
* The slax directory has been removed (`$ORACLE_HOME/slax`)
* The rdbms/demo directory has been removed (`$ORACLE_HOME/rdbms/demo`)
* The rdbms/jlib directory has been removed (`$ORACLE_HOME/rdbms/jlib`)
* The rdbms/public directory has been removed (`$ORACLE_HOME/rdbms/public`)
* The rdbms/xml directory has been removed (`$ORACLE_HOME/rdbms/xml`)
