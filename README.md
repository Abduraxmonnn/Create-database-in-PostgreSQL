# PostgreSQL: Creating a Database, Managing User Permissions and Resolving Authentication Issues

This guide explains how to create a PostgreSQL database, configure a dedicated database user, grant permissions, and resolve common authentication errors.

## 1. Connect to PostgreSQL

Connect to PostgreSQL as the default administrator:

```bash
sudo -u postgres psql
```

The `postgres` user has administrative privileges to manage databases, roles, and permissions.

## 2. Create a User

```sql
CREATE USER fbox_user WITH PASSWORD 'your_strong_password';
```

This creates a new PostgreSQL role with login privileges.

To verify that the user exists:

```sql
\du
```

## 3. Create a Database

```sql
CREATE DATABASE fbox OWNER fbox_user;
```

This creates a database named `fbox` and assigns `fbox_user` as its owner.

If the database already exists, you can change its owner:

```sql
ALTER DATABASE fbox OWNER TO fbox_user;
```

## 4. Grant Database Permissions

```sql
GRANT ALL PRIVILEGES ON DATABASE fbox TO fbox_user;
```

This grants database-level privileges, including:

- `CONNECT` — connect to the database.
- `CREATE` — create schemas within the database.
- `TEMPORARY` — create temporary tables.

Since `fbox_user` is already the database owner, this command is technically redundant but useful when configuring non-owner users.

## 5. Connect to the Database

```sql
\c fbox
```

Schema, table, sequence, and function permissions must be configured inside the target database.

## 6. Grant Schema Permissions

```sql
ALTER SCHEMA public OWNER TO fbox_user;

GRANT ALL PRIVILEGES ON SCHEMA public TO fbox_user;
```

Schema-level privileges include:

- `USAGE` — access objects within the schema, subject to their permissions.
- `CREATE` — create new objects within the schema.

## 7. Grant Permissions on Existing Objects

### Tables

```sql
GRANT ALL PRIVILEGES ON ALL TABLES
IN SCHEMA public TO fbox_user;
```

Allows the user to read, insert, update, delete, and truncate records, among other table-level privileges.

### Sequences

```sql
GRANT ALL PRIVILEGES ON ALL SEQUENCES
IN SCHEMA public TO fbox_user;
```

Allows the user to access and manage sequence values, commonly used for auto-incrementing primary keys.

### Functions

```sql
GRANT ALL PRIVILEGES ON ALL FUNCTIONS
IN SCHEMA public TO fbox_user;
```

Allows the user to execute existing PostgreSQL functions.

**Note:** Granting privileges does not transfer object ownership. Operations such as altering or dropping existing tables generally require ownership.

## 8. Grant Permissions on Future Objects

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE postgres
IN SCHEMA public
GRANT ALL PRIVILEGES ON TABLES TO fbox_user;

ALTER DEFAULT PRIVILEGES FOR ROLE postgres
IN SCHEMA public
GRANT ALL PRIVILEGES ON SEQUENCES TO fbox_user;

ALTER DEFAULT PRIVILEGES FOR ROLE postgres
IN SCHEMA public
GRANT ALL PRIVILEGES ON FUNCTIONS TO fbox_user;
```

These commands automatically grant permissions on future objects created by the `postgres` role within the `public` schema.

Objects created by `fbox_user` itself will already belong to that user.

Default privileges do not affect existing objects or objects created by other roles.

## 9. Configure User Security Restrictions

Execute the following command as the `postgres` administrator:

```sql
ALTER ROLE fbox_user WITH
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE
    NOREPLICATION
    NOBYPASSRLS
    CONNECTION LIMIT 100;
```

| Attribute | Description |
|---|---|
| `NOSUPERUSER` | Prevents server-level administrative access |
| `NOCREATEDB` | Prevents creating additional databases |
| `NOCREATEROLE` | Prevents administrative role management |
| `NOREPLICATION` | Prevents replication access |
| `NOBYPASSRLS` | Does not grant the ability to bypass row-level security |
| `CONNECTION LIMIT 100` | Limits concurrent connections to 100 |

These restrictions allow the application user to manage its own database objects without unnecessary server-level privileges.

The connection limit should be adjusted according to the server's resources and application requirements.

## 10. Test the Database Connection

Exit PostgreSQL:

```sql
\q
```

Connect using the newly created user:

```bash
psql -h localhost -p 5432 -U fbox_user -d fbox -W
```

Parameters:

| Parameter | Description |
|---|---|
| `-h` | Database host |
| `-p` | Database port |
| `-U` | Username |
| `-d` | Database name |
| `-W` | Force a password prompt |

---

# Troubleshooting PostgreSQL Authentication

## Problem 1: Ident Authentication Failed

When connecting to PostgreSQL, you may encounter the following error:

```text
psql: error: FATAL: Ident authentication failed
for user "fbox_user"
```

### Why does this happen?

PostgreSQL uses the `pg_hba.conf` configuration file to determine how clients are authenticated.

On some Linux distributions, local TCP connections use `ident` authentication by default.

Ident authentication attempts to identify the operating system user instead of validating the PostgreSQL user's password.

Therefore, even if the password is correct, authentication can still fail.

### Step 1. Find the Active Configuration File

```bash
sudo -u postgres psql -c "SHOW hba_file;"
```

Example output:

```text
/var/lib/pgsql/data/pg_hba.conf
```

The location may vary depending on the PostgreSQL version and operating system.

### Step 2. Edit the Configuration File

```bash
sudo nano /var/lib/pgsql/data/pg_hba.conf
```

Alternatively, open the active configuration file directly:

```bash
sudo nano "$(sudo -u postgres psql -Atc 'SHOW hba_file;')"
```

Find the following configuration:

```conf
host    all    all    127.0.0.1/32    ident
host    all    all    ::1/128         ident
```

Add password authentication rules for the application user above the existing rules:

```conf
# Local Unix socket connections
local   all     all         peer

# Application database authentication
host    fbox    fbox_user   127.0.0.1/32    scram-sha-256
host    fbox    fbox_user   ::1/128         scram-sha-256

# Other local TCP connections
host    all     all         127.0.0.1/32    ident
host    all     all         ::1/128         ident
```

- `127.0.0.1/32` — local IPv4 connections.
- `::1/128` — local IPv6 connections.
- `scram-sha-256` — secure password-based authentication.
- `peer` — operating system authentication through Unix sockets.

**Important:** PostgreSQL evaluates authentication rules from top to bottom and uses the first matching rule.

### Step 3. Validate the Configuration

```bash
sudo -u postgres psql -c "
SELECT
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
ORDER BY line_number;
"
```

This displays authentication rules and helps identify configuration syntax errors.

### Step 4. Reload PostgreSQL

```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

Expected output:

```text
 pg_reload_conf
----------------
 t
```

Reloading applies configuration changes without restarting the PostgreSQL server. Check the server logs if the configuration fails to load.

### Step 5. Test the Connection

```bash
psql -h 127.0.0.1 -p 5432 -U fbox_user -d fbox -W
```

You can also test IPv6:

```bash
psql -h ::1 -p 5432 -U fbox_user -d fbox -W
```

---

## Problem 2: Password Authentication Failed Despite a Correct Password

After switching to `scram-sha-256`, you may encounter another error:

```text
psql: error: FATAL: password authentication failed
for user "fbox_user"
```

### Why does this happen?

One possible reason is that the user's password was originally stored using an MD5 password verifier, while the authentication configuration now requires SCRAM-SHA-256.

An MD5 password verifier cannot be used directly for SCRAM authentication.

### Step 1. Check the Password Format

Connect as the PostgreSQL administrator:

```bash
sudo -u postgres psql
```

Execute:

```sql
SELECT
    rolname,
    rolcanlogin,
    CASE
        WHEN rolpassword IS NULL THEN 'NO PASSWORD'
        WHEN rolpassword LIKE 'SCRAM-SHA-256$%'
            THEN 'SCRAM-SHA-256'
        WHEN rolpassword LIKE 'md5%'
            THEN 'MD5'
        ELSE 'UNKNOWN'
    END AS password_format,
    rolvaliduntil
FROM pg_authid
WHERE rolname = 'fbox_user';
```

This checks the password storage format without exposing the password verifier.

### Step 2. Configure SCRAM-SHA-256

```sql
SET password_encryption = 'scram-sha-256';
```

This tells PostgreSQL to generate SCRAM-SHA-256 password verifiers when setting passwords within the current session.

### Step 3. Reset the User's Password

```sql
\password fbox_user
```

Enter and confirm the password.

You can reuse the existing password if it is sufficiently strong. PostgreSQL will generate a new SCRAM-SHA-256 verifier.

Using `\password` is preferable to entering a plaintext password directly into an SQL command.

### Step 4. Verify the Password Format

```sql
SELECT
    rolname,
    rolpassword LIKE 'SCRAM-SHA-256$%' AS is_scram
FROM pg_authid
WHERE rolname = 'fbox_user';
```

Expected output:

```text
  rolname  | is_scram
-----------+----------
 fbox_user | t
```

### Step 5. Reconnect

Exit PostgreSQL:

```sql
\q
```

Connect again:

```bash
psql -h localhost -p 5432 -U fbox_user -d fbox -W
```

Expected result:

```text
fbox=>
```

Password changes take effect immediately. PostgreSQL does not need to be restarted.

---

## PostgreSQL Authentication Methods

| Method | Description | Recommendation |
|---|---|---|
| `trust` | Allows connections without passwords | Avoid |
| `ident` | Authenticates using operating system identity via Ident | Specialized use |
| `peer` | Authenticates local operating system users | Local administration |
| `md5` | Legacy password authentication | Legacy compatibility |
| `scram-sha-256` | Modern password-based authentication | Recommended |

SCRAM-SHA-256 requires PostgreSQL 10 or newer and a compatible client library.

---

## Useful PostgreSQL Commands

| Command | Description |
|---|---|
| `\du` | List users and roles |
| `\l` | List databases |
| `\dn+` | List schemas and permissions |
| `\dt+` | List tables |
| `\dp` | Display object permissions |
| `\ddp` | Display default privileges |
| `\conninfo` | Display current connection information |
| `SHOW hba_file;` | Show authentication configuration path |
| `SHOW config_file;` | Show PostgreSQL configuration path |
| `SHOW port;` | Show PostgreSQL port |
| `SHOW password_encryption;` | Show default password storage method |

## Final Verification

Connect as the application user and execute:

```sql
SELECT current_user, current_database();

CREATE TABLE permission_test (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

INSERT INTO permission_test (name)
VALUES ('PostgreSQL connection successful');

SELECT * FROM permission_test;

UPDATE permission_test
SET name = 'Permissions verified'
WHERE id = 1;

DELETE FROM permission_test WHERE id = 1;

DROP TABLE permission_test;
```

If all commands execute successfully, the application user has the necessary permissions to create and manage its own database tables.

## Security Best Practices

- Use a dedicated database user for each application.
- Use SCRAM-SHA-256 instead of legacy MD5 authentication.
- Never grant unnecessary superuser privileges to application users.
- Store database passwords in environment variables or a secrets manager.
- Avoid exposing PostgreSQL port `5432` publicly when the database and application run on the same server.
- Apply the principle of least privilege.
- Verify authentication and permissions after configuration changes.

A properly configured PostgreSQL database provides a secure and reliable foundation for backend applications, including Django, Spring Boot, and other frameworks.

#PostgreSQL #Database #BackendDevelopment #Security #Linux #DevOps
