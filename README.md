# Evaluating Flyway

In this project I want to evaluate _Flyway_ for automatic schema migrations.
My goal is to find out if and how automatic migrations can be used having some real world requirements in mind (like not using a local H2 database, but an _Oracle_ database).
Non-obvious requirements are listed in the [Requirements and Questions section](#requirements-and-questions).

# Environment

I want to evaluate a real world environment where you have multiple stages (DEV, TST, INT, PROD, maybe more).
Each stage has one main schema, but especially on DEV level there are additional ones (one for each developer). 
Furthermore, I investigate dynamic object creation, e.g. creating users depending on the schemas name, as it's a best practice that each component uses an own, specialized user with individual rights.
Where I work, the names of those application users are aligned to the schema, e.g. `<schema><application>`.  

I did **not** investigated SSL connections, as I'm not aware of the requirements at my work.
However, the [_Flayway_ documentation](https://flywaydb.org/documentation/ssl) states, that SSL connections are support in general.

For the evaluation I've created a local _Oracle_ database using the official [_Oracle_ 18 XE](https://www.oracle.com/database/technologies/xe-downloads.html) installer.
Because I didn't want to spend too much time creating _Oracle_ databases, I simulated different schemas by using different users (as each user gets an own schema automatically).

The connection string of my local database is `jdbc:oracle:thin:@localhost:1521/xe`.

## Basic script
Up to now the basic schema creation, including defining the user `c##schemaowner` as a simulation of a main schema, is done manually before starting with _Flyway_.

```sql

CREATE USER c##schemauser
   IDENTIFIED BY _Oracle_;
 
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


# _Maven_ setup
My goal was to define and store only the general, non-critical database information (e.g. the connection URL) inside the project.

**Note**: Of course, the user and password should not be stored inside the repository at all!
Call me lazy that I've done it during the evaluation, after I tried it (successfully) using the CLI.

# Usage
To migrate a database using _Flyway_ and _Maven_ on the command line: 

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

It can also be created by defining a baseline.

# Requirements and questions
The following requirements (next to the one listed in the [Environment section](#environment) and questions were investigated during the evaluation.

**How to support multiple schemas?**

The easiest (and I think most comfortable) way is to define different _Maven_ profiles.
Each profile defines the non-critical (in terms of SCM) values, like the URL, but of course not any credentials.
So when working on a schema only the profile's name, and the credentials must be passed.
The credentials can even be stored in a personal `settings.xml`, but of course it then should be encrypted using [_Maven_'s password encryption](https://maven.apache.org/guides/mini/guide-encryption.html).

The three default variation only work, when the same credentials are used everywhere, see [_Flyway_ FAQ: Multiple schemas](https://flywaydb.org/documentation/faq#multiple-schemas).
Of course, the schema name(s) must be defined (e.g. by using the `schemas` property), when working on another user's schema. 
In this example the schema owner is used for migration, resulting in using the schema owner's schema.


**How to connect to _Oracle_ databases?**

As the API is plain JAVA, it is possible to use a regular JDBC connection: `jdbc:oracle:thin:@//<host>:<port>/<service>`

It's also possible to use an entry of the `tnsames.ora` file: `jdbc:oracle:thin:@<tns_entry>`.
For this the value of `TNS_ADMIN` environment variable must point to the folder containing the `tnsames.ora` file.

[Source: _Flyway_ documentation: configfiles](https://flywaydb.org/documentation/configfiles)

The driver for the connection can be defined in the _Maven_ project and automatically downloaded from _Maven_ central.
So there's no driver handling needed. 

**What's the default encoding?**

The default encoding is `UTF-8` for migration scripts.

[Source: _Flyway_ documentation: command `migrate`](https://flywaydb.org/documentation/commandline/migrate)

**Is it possible to use placeholders to create dynamic names / values?**

By using placeholders it is possible to create dynamic values inside the scripts.
The value for those can be passed from the commandline, using this option:

`-Dflyway.placeholders.<name_of_the_placeholder>=<value_to_be_passed>`

_Flyway_ also provides some default placeholder, e.g. `${flyway:defaultSchema}`, which can perfectly used to create schema-related objects, e.g. users for applications.

[Source: _Flyway_ documentation: placeholders](https://flywaydb.org/documentation/placeholders)


**How is the migrating user identified (for storage in history table)?**

By default, the user which is used to open the connection and executes the migration is stored inside the history table.
This means that a personalized user shall be used for connections and not a technical one.

**Are migrations done in transactions and are rollbacks possible?**

In general _Flyway_ executes all migrations in transactions.
However, _Oracle_ does not support executing DDL statements in transactions.

> Other databases such as _Oracle_ will implicitly sneak in a commit before and after each DDL statement, drastically reducing the effectiveness of this roll back.
> One alternative if you want to work around this, is to include only a single DDL statement per migration.
> This solution however has the drawback of being quite cumbersome. 

[Source: _Flyway_ FAQ: Rollback](https://flywaydb.org/documentation/faq#rollback)

Depending on the changes made, migrations can be undone using the [`undo` command](https://flywaydb.org/documentation/command/undo) (in combination with an undo-script).
The command needs a _Flyway Pro_ version.

**Is _Oracle_ SQL\*Plus Syntax supported?**

When using the _Flyway Pro_ version, _Flyway_ supports several commands, see [_Flyway_ documentation: SQL*Plus commands](https://flywaydb.org/documentation/database/oracle#sqlplus-commands)

To activate SQL*Plus support the following setting must be placed: `<OracleSqlplus>true<OracleSqlplus>`

**Warning**: Interactions, which are possible in SQL*Plus, are not supported!
Such statements just take `null` as input which can result in heavy problems.
Scripts must not contain such interactions to work probably.
This may effort work if a project already has interactive SQL*Plus scripts.

**Is a clean database needed to start using _Flyway_?**

No.
An already existing database can be used to, using the [`baseline` command](https://flywaydb.org/documentation/command/baseline).
**Caution**: All scripts that resides inside the project's migration folder are ignored!
Only scripts added **after* the baseline was created are taken into migration!

If there's the need to clean a database, e.g. to rebuild a developer's one, the  [`clean` command](https://flywaydb.org/documentation/command/clean) can be used, which deletes all objects inside the schema.
**Note**: To use the `clean` command, the `cleanDisabled` setting must be set to `false`.
In this project I set it to `true` to not allow (unwanted) deletions.
Of course the setting can be overwritten from the command line.

**What _Oracle_ databases does _Flyway_ support?**

[_Flyway_ supports all _Oracle_ databases](https://flywaydb.org/documentation/database/oracle) since version `10.1`.
However, the not all _Flyway_ versions support all versions.
Only the _Flyway Enterprise_ version supports older databases up to (including) `12.1`.

**Which licence does _Flyway_ required for enterprise usage?**

According to the [editions overview](https://flywaydb.org/download/) the _Pro_ and _Enterprise_ version "only" differ in Java 7 compatibility, support (including payment methods) and pricing.
The big difference when using _Oracle_ databases lies in the supported database versions (see above), where only the _Enterprise_ version supports `12.1` databases.
