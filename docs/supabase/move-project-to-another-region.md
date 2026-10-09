# Move a Supabase project to another region

*Last updated: 9 October 2026*

To move a Supabase project's database to another region, create a new project in that region and copy the database into it. This page dumps the old database to SQL files with the Supabase CLI, then restores them into the new project with psql.

The commands are for Command Prompt (cmd) on Windows.

## What you need

- **Node.js 20 or later**, to run the Supabase CLI with `npx`.
- **Docker Desktop**, running. The Supabase CLI makes the dumps with `pg_dump` inside a Docker container. On Windows, Docker Desktop uses WSL 2.
- **psql**, installed with the [PostgreSQL installer for Windows](https://www.postgresql.org/download/windows/). If Command Prompt says `psql` is not recognized, add PostgreSQL's `bin` folder, for example `C:\Program Files\PostgreSQL\<version>\bin`, to your `Path` environment variable and open a new Command Prompt.

!!! note "What this does not move"
    The dumps only copy the database. Supabase's guide has separate steps for:

    - files in Storage buckets
    - Edge Functions
    - Vault secrets and encrypted columns: the new project cannot decrypt them until you copy the old project's root encryption key across, so get the key before you pause or delete the old project

    See [Backup and Restore using the CLI](https://supabase.com/docs/guides/platform/migrating-within-supabase/backup-restore).

## Step 1: Install the Supabase CLI

Create an empty folder, open Command Prompt in it and install the CLI there:

```bat
npm install supabase --save-dev
```

Run all the following commands from this folder. `npx supabase` runs the CLI installed here, and the dump files are saved here.

## Step 2: Get the old project's connection string

On the old project's dashboard, click **Connect** and copy the **Session pooler** connection string. Supabase recommends the session pooler by default: the direct connection only works if your network supports IPv6 or the project has the IPv4 add-on. The string looks like this:

```text
postgresql://postgres.[PROJECT-REF]:[YOUR-PASSWORD]@aws-0-[REGION].pooler.supabase.com:5432/postgres
```

Replace `[YOUR-PASSWORD]` with the old project's database password. The Supabase CLI requires the connection string to be percent-encoded, so if the password contains special characters, replace them with their codes, for example `@` with `%40` and `#` with `%23`.

The commands below call the finished string `[OLD_CONNECTION_STRING]`.

!!! warning "Keep connection strings private"
    Anyone who has a connection string with the password filled in can connect to the database. Keep it out of pages, repositories and screenshots.

## Step 3: Dump the old database

Make sure Docker Desktop is running, then run these three commands. Each one saves part of the old database as a SQL file:

```bat
npx supabase db dump --db-url "[OLD_CONNECTION_STRING]" -f roles.sql --role-only
npx supabase db dump --db-url "[OLD_CONNECTION_STRING]" -f schema.sql
npx supabase db dump --db-url "[OLD_CONNECTION_STRING]" -f data.sql --use-copy --data-only -x "storage.buckets_vectors" -x "storage.vector_indexes"
```

- `roles.sql`: the database roles (`--role-only`).
- `schema.sql`: the structure, such as tables and functions. The CLI leaves out the schemas that Supabase manages, such as `auth` and `storage`.
- `data.sql`: the rows in the tables (`--data-only`), written as `COPY` statements instead of `INSERT` statements (`--use-copy`). `-x` leaves out the tables `storage.buckets_vectors` and `storage.vector_indexes`, as in Supabase's own command.

The connection string is in double quotes so that characters such as `&` in it do not break the command in Command Prompt.

## Step 4: Comment out the lines the restore cannot run

Some lines in `roles.sql` change roles that Supabase manages itself, and the restore stops with an error on each of them. Open `roles.sql` in a text editor, search for `supabase_admin` and `log_min_messages`, and comment out the lines you find by putting `--` at the start. They look like this once commented out:

```sql
-- ALTER ROLE "supabase_admin" SET "statement_timeout" TO '0';

-- GRANT SET ON PARAMETER "log_min_messages" TO "supabase_realtime_admin";
```

If the restore stops on a line in `schema.sql` like `ALTER ... OWNER TO "supabase_admin";`, comment out those lines in the same way. Supabase's guide lists them as a cause of permission errors.

## Step 5: Create the new project

[Create a new project](https://database.new) and choose the region you are moving to. Save the database password you set for it.

Get its **Session pooler** connection string the same way as in [step 2](#step-2-get-the-old-projects-connection-string), with the new password filled in. The next command calls it `[NEW_CONNECTION_STRING]`.

## Step 6: Restore into the new project

Run psql with the three files, in this order:

```bat
psql ^
  --single-transaction ^
  --variable ON_ERROR_STOP=1 ^
  --file roles.sql ^
  --file schema.sql ^
  --command "SET session_replication_role = replica" ^
  --file data.sql ^
  --dbname "[NEW_CONNECTION_STRING]"
```

In Command Prompt, `^` at the end of a line continues the command on the next line, like `\` in Bash.

- `--single-transaction` runs everything as one transaction, and `ON_ERROR_STOP=1` stops at the first error. If anything fails, nothing is saved in the new project, so you can fix the file and run the command again.
- `SET session_replication_role = replica` turns off triggers while `data.sql` loads. Supabase's guide sets it so that encrypted columns are not encrypted a second time.

## Step 7: Point your app at the new project

Replace the old connection strings in your app's secrets, for example `DIRECT_URL`, with the new project's. If your app also uses the project URL and API keys, for example with `supabase-js`, update those too: the new project has its own.

## References

- [Supabase: Backup and Restore using the CLI](https://supabase.com/docs/guides/platform/migrating-within-supabase/backup-restore)
- [Supabase CLI: `supabase db dump`](https://supabase.com/docs/reference/cli/supabase-db-dump)
- [Supabase CLI: Getting started](https://supabase.com/docs/guides/local-development/cli/getting-started)
