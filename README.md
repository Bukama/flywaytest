# Evaluating Flyway

In this project I want to evaluate _Flyway_ for schema migrations.
Goal is to find out if and how automatic migrations can be used having some real world requirements in mind like not using a local H2 database, but an Oracle database.

# Setup
I've created a local Oracle database using the official [Oracle 18 XE](https://www.oracle.com/database/technologies/xe-downloads.html) installer.

The connection string is `jdbc:oracle:thin:@localhost:1522/xe`.

## Basic script
Up to now the basic schema creation including defining the schema owner is done manually before starting with _Flyway_.

```sql

CREATE USER c##schemauser
   IDENTIFIED BY oracle;
 
 GRANT CONNECT TO c##schemauser;
 GRANT create session TO c##schemauser;
 GRANT create table TO c##schemauser;
 GRANT create view TO c##schemauser;
 GRANT create any trigger TO c##schemauser;
 GRANT create any procedure TO c##schemauser;
 GRANT create sequence TO c##schemauser;
 GRANT create synonym TO c##schemauser;
 
 GRANT UNLIMITED TABLESPACE TO c##schemauser;
```

# Maven setup
My goal was to define only the general, non-critical database information (e.g. the connection URL) inside the project.
The user and password of course should not be stored inside the repository at all. 

# Usage
To use migrate a database using _Flyway_ and maven on the command line: 

`mvn flyway:migrate -Dflyway.url=... -Dflyway.user=... -Dflyway.password=...`

On the first usage, _Flyway_ creates the `flyway_schema_history` table. 

```
[INFO] Flyway Community Edition 6.5.6 by Redgate
[INFO] Database: jdbc:oracle:thin:@localhost:1521/xe (Oracle 18.0)
[INFO] Successfully validated 1 migration (execution time 00:00.018s)
[INFO] Creating Schema History table "C##SCHEMAUSER"."flyway_schema_history" ...
[INFO] Current version of schema "C##SCHEMAUSER": << Empty Schema >>
[INFO] Migrating schema "C##SCHEMAUSER" to version 1 - Create person table
[INFO] Successfully applied 1 migration to schema "C##SCHEMAUSER" (execution time 00:00.415s)
```

It can can also be created by defining a baseline.

# Requirements / Questions
The following requirements and questions were investigated during the evaluation.

**How to support multiple schemas?**

The easiest (and I think most comfortable) way is to define different _Maven_ profiles.
Each profile defines the non-critical (in terms of SCM) values, like the URL and the schemauser, but of course not the password.
So when working on a schema only the profile's name and the password must be passed.
The password can even be stored in a personal `settings.xml`, but of course it then should be encrypted using [Maven's password encryption](https://maven.apache.org/guides/mini/guide-encryption.html).

**How to connect to Oracle databases?**

As the API is plain JAVA, it is possible to use a regular JDBC connection: `jdbc:oracle:thin:@//<host>:<port>/<service>`

It's also possible to use an entry of the `tnsames.ora` file: `jdbc:oracle:thin:@<tns_entry>`.
For this the value of `TNS_ADMIN` environment variable must point to the folder containing the `tnsames.ora` file.

[Source: _Flyway_ documentation: configfiles](https://flywaydb.org/documentation/configfiles)

The driver for the connection can be defined in the maven project and automatically downloaded from _Maven_ central.
So there's no driver handling needed. 

**What's the default encoding?**

The default encoding is `UTF-8` for migration scripts.

[Source: _Flyway_ documentation: command `migrate`](https://flywaydb.org/documentation/commandline/migrate)

**Is it possible to use placeholders to create dynamic names / values?**

**How is the migrating user identified (for storage in history table)?**

By default, the user, used to open the connection, is stored inside the history table.

**Are migrations done in transactions and are rollbacks possible?**

In general _Flyway_ executes all migrations in transactions.
However, Oracle does not support executing DDL statements in transactions.

> Other databases such as Oracle will implicitly sneak in a commit before and after each DDL statement, drastically reducing the effectiveness of this roll back.
> One alternative if you want to work around this, is to include only a single DDL statement per migration.
> This solution however has the drawback of being quite cumbersome. 

[Source: _Flyway_ FAQ: Rollback](https://flywaydb.org/documentation/faq#rollback)

**Is Oracle SQL\*Plus Syntax supported?**

When using the _Flyway Pro_ version, _Flyway_ supports several commands, see [_Flyway_ documentation: SQL*Plus commands](https://flywaydb.org/documentation/database/oracle#sqlplus-commands)

To activate SQL*Plus support the following setting must be placed: `<oracleSqlplus>true</oracleSqlplus>`

**Is a clean database needed?**

No.
An already existing database can be used to, using the [`baseline` command](https://flywaydb.org/documentation/command/baseline).

**What Oracle databases does _Flyway_ support?**

[_Flyway_ supports all Oracle databases](https://flywaydb.org/documentation/database/oracle) since version `10.1`.
However, the not all _Flyway_ versions support all versions.
Only the _Flyway Enterprise_ version supports older databases up to (including) `12.1`.

**Which licence does _Flyway_ required for enterprise usage?**
