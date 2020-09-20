# Evaluating Flyway

In this project I want to evaluate _Flyway_ for schema migrations.
Goal is to find out if and how automatic migrations can be used having some real world requirements in mind like not using a local H2 database, but an Oracle database.

# Setup
I've created a local Oracle database using the official Oracle XE installer.

The connection string is `jdbc:oracle:thin:@localhost:1521:container`.

# Maven setup
My goal was to define only the general, non-critical database information (e.g. the connection URL) inside the project.
The user and password of course should not be stored inside the repository at all. 

# Usage
To use migrate a database using _Flyway_ and maven on the command line: 

`mvn flyway:migrate -Dflyway.url=... -Dflyway.user=... -Dflyway.password=...`

# Open todos
- Multiple schemas

