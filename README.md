# Library Roles Activity
## 1. Create the database 
```sql
CREATE DATABASE library;
```

## 2. Revoke access to database and schema
```sql
-- revoke connect on databse
REVOKE CONNECT ON DATABASE library FROM PUBLIC; 

-- Revoke usage the schema public for public roles
REVOKE USAGE ON SCHEMA public FROM public;
```
Now just the **superusers** can access to the database.

## 3. Create schema
Best practices are create a new schema and don't use public.
With this command we can see the actual schema.
```sql
SHOW search_path;
```
If we use it, we can see our database is using *public*.

We need to create a new schema, and notify the database we are going to use it.
```sql
-- Create the schema
CREATE SCHEMA library_management;

-- Use the schema
SET search_path TO library_management;
```

Finally, delete the public schema.
```sql
DROP SCHEMA public;
```

## 4. Create the admin user
```sql
CREATE USER library_admin
SUPERUSER
CREATEROLE
PASSWORD 'admin';
```
We are giving this new admin user privileges of superuser and create roles. This user can't create databases.

At this time, if we try to connect to the database with the library_admin we can because this user is superuser, created with the user postgres.


## 5. Create the library database
Create the database library and insert data using the SQL code of the classroom.


## 6. Create the rest of roles
Let's create the librarians
```sql
-- create the group role for the librarians
CREATE ROLE librarians;

-- create the librarian1
CREATE USER librarian1
WITH PASSWORD '1234';

-- create the librarian2
CREATE USER librarian2
WITH PASSWORD '1234';

-- add the librarians to the group
GRANT librarians TO librarian1, librarian2;

-- create the group role for the readers
CREATE ROLE readers;

-- create the reader1
CREATE USER reader1
WITH PASSWORD '1234';

-- create the reader2
CREATE USER reader2
WITH PASSWORD '1234';

-- add the readers to the group
GRANT readers TO reader1, reader2;
```
At this time, if we try to connect with any of this users we'll get the error:
*User does not have CONNECT privilege.*

## 7. Give access to the roles connect to the database
```sql
-- revoke connect on databse (using superuser)
GRANT CONNECT ON DATABASE library TO librarians, readers;
```
After this, we can connect as **librarian or reader user** and the error has disappeared.


## 8. Give permissions to use the schema
At this time, **using the librarian or reader user** if we try to make a query like
```sql
SELECT * FROM books;
```
We'll get the error: *relation "books" does not exist*. But this is not true. That's because this users doesn't have privileges to use the new schema.

```sql
-- Grant usage the schema public for public roles (using superuser)
GRANT USAGE ON SCHEMA library_management TO librarians, readers;
```
Now the librarians and the readers have access to all the tables in schema.

## 9. Limit access to tables
We can allow all kind of users, access to all the data in the tables.
The main question is **which tables our librarians and readers can have access?**.
It will depend on the final platform, what should readers and librarians do. 

For example, the librarians make the reservations, they insert the data into the reservations table. So, for sure, librarians can do `SELECT, INSERT and UPDATE` into the tables `reservations` and `return status`.
```sql
GRANT 
    SELECT, INSERT, UPDATE
ON
    reservations, returnstatus
TO
    librarians;
```
Now, **using some librarian user**, if we try to make a select into reservations table, everything will be OK.

But, **using some reader user**, if we try to make a select into reservations table, we'll get the error: *permission denied for table reservations*. Because the readers doesn't have permissions.

In this step, give to the roles the privileges they need to use this database.

## 10. Limit access to some columns
First, using the **superuser** role, add a column password to the users.
```sql
ALTER TABLE users
ADD COLUMN password VARCHAR;
```

Now, logically, the librarian should access all the columns of the `users` table excep the column `password`.

For that, specify which columns the role should have access:
```sql
GRANT 
    SELECT
    (userid, name, email)
ON
    users
TO
    librarians;
```
Now, using the **librarian user**, if we execute this query
```sql
SELECT userid FROM users;
```
Everything goes OK but, if we execute this other query
```sql
SELECT * FROM users;
```
We'll get the error *permission denied for table users*.

## 10. RLS (Row Level Security)
The last step is adding some security policies. The objective is the users can't see data for security reasons.

For example, a user can't see the usernames, emails and password of other users. Or, a user can't see the reservations of the other users, can see just their own reservations.

So, let's start with this example. A user (reader) must see just their own reservations.

The first step is enable the row level security in the table.
```sql
ALTER TABLE reservations
ENABLE ROW LEVEL SECURITY;
```
Now we have a little problem, we just can know the current user name. We can see our actual username with:
```sql
SELECT current_user;
```
This returns the **username** but, in the table `reservations` we don't have the user name, we have the column `userid`. So, what we should do in order to get the `userid` if the just know the `name`? With a query:
```sql
SELECT userid FROM users WHERE name = current_user;
```

Let's apply the policy:
```sql
CREATE POLICY own_readers_reservations
ON reservations 
TO readers
USING (userid = (SELECT userid FROM users WHERE name = current_user));
```

If we execute the query, using a **reader** user:
```sql
SELECT * FROM reservations;
```
This user should see their own reservations.

If this doesen't work, make sure the role name is equal to the user name linked to this reservation. And make sure also this user has reservations.

**What happens is this policy applies to all users?**
In the superuser, we can enable an option that policies don't affect to the superuser.

```sql
ALTER ROLE library_admin
BYPASSRLS;
```


## Drop Roles
This is an example of how can delete roles. If the role isn't owner of anything is easier but, if the role is owner of tables on databases:
```sql
-- replace the owner
REASSIGN OWNED BY library_admin TO postgres;

-- drop the owner attribute
DROP OWNED BY library_admin;

-- drop the role
DROP ROLE library_admin;
```
