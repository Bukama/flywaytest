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

# Requirements / Questions
The following requirements and questions were investigated during the evaluation.

**How to support multiple schemas?**

**Is it possible to use placeholders to create dynamic names / values?**

**How is the migrating user identified (for storage in history table)?**

**Are migrations done in transactions and therefore rollbackable?**

**Is Oracle SQLPlus Syntax supported?**

**Is a clean database needed?**

**What Oracle databases does _Flyway_ support?**

**Which licence does _Flyway_ required for enterprise usage?**
