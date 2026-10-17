# Back up and restore a Supabase database with Bash scripts

*Last updated: 17 October 2026*

These Bash scripts create a timestamped logical backup of a Supabase database and restore one of those backups with psql. They are useful for keeping manual backups or copying a database into another Supabase project.

The commands are for Linux or another environment with Bash.

!!! warning "This is not a complete Supabase project backup"
    The scripts back up the database, including Storage metadata, but not the files stored through the Storage API. They also do not copy Edge Functions, API keys or project settings.

    A manual restore into another project needs extra work if the database uses Vault secrets or encrypted columns. See [Move a Supabase project to another region](move-project-to-another-region.md#what-you-need) and Supabase's [Backup and Restore using the CLI](https://supabase.com/docs/guides/platform/migrating-within-supabase/backup-restore) before relying on the new project.

## What you need

- **Bash**, to run the scripts.
- **Node.js and npm**, to run the Supabase CLI with `npx`.
- **The Supabase CLI**, installed locally in the working folder. See [step 1 of the project-move guide](move-project-to-another-region.md#step-1-install-the-supabase-cli).
- **Docker**, running. `supabase db dump` runs pg_dump in a container.
- **psql**, available on `PATH`, to restore a backup.
- **A database connection string** for the source database and, when restoring, the target database.

!!! warning "Keep the connection strings and backup files private"
    A connection string contains the database password, and the SQL files can contain private application data. Do not commit them, put them under `docs/` or include them in screenshots. Passing a connection string on the command line can also save it in shell history.

## How the scripts work

`backup.sh` creates a folder under `./backups/` named with the current date and time. It writes three files there:

| File | Contents |
|---|---|
| `roles.sql` | Database roles |
| `schema.sql` | Tables, functions and the rest of the database structure |
| `data.sql` | Table rows, written as `COPY` statements |

The data dump excludes `storage.buckets_vectors` and `storage.vector_indexes`, following Supabase's backup guide.

`restore.sh` loads the files in the order shown above. It uses one transaction and stops at the first error, so a failed restore is rolled back instead of leaving only part of the backup restored. It also disables triggers while loading the data and ignores any local psql startup file.

## Run the commands manually

The scripts automate the following commands. Replace `[CONNECTION_STRING]` with a percent-encoded Session pooler connection string in this format:

```text
postgresql://postgres.[PROJECT_REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:5432/postgres
```

Keep the finished connection string in quotes so that the shell does not interpret any of its characters.

### Backup commands

Run all three commands to back up the roles, schema and data:

```bash
npx supabase db dump --db-url "[CONNECTION_STRING]" -f roles.sql --role-only
npx supabase db dump --db-url "[CONNECTION_STRING]" -f schema.sql
npx supabase db dump --db-url "[CONNECTION_STRING]" -f data.sql --use-copy --data-only -x "storage.buckets_vectors" -x "storage.vector_indexes"
```

### Restore command

The restore command loads the three files in the same order. Use the version for the current shell:

=== "Bash"

    ```bash
    psql \
      --single-transaction \
      --variable ON_ERROR_STOP=1 \
      --file roles.sql \
      --file schema.sql \
      --command "SET session_replication_role = replica" \
      --file data.sql \
      --dbname "[CONNECTION_STRING]"
    ```

=== "Windows Command Prompt"

    ```bat
    psql ^
      --single-transaction ^
      --variable ON_ERROR_STOP=1 ^
      --file roles.sql ^
      --file schema.sql ^
      --command "SET session_replication_role = replica" ^
      --file data.sql ^
      --dbname "[CONNECTION_STRING]"
    ```

## Step 1: Save the scripts

Save this as `backup.sh` in the private working folder where the backups should be created:

```bash
#!/usr/bin/env bash

set -euo pipefail

# Read database URL from first argument
DB_URL="${1:-}"

if [[ -z "$DB_URL" ]]; then
    echo "Usage: ./backup.sh <DB_URL>"
    exit 1
fi

# Create timestamped backup directory
BACKUP_DIR="./backups/$(date +%Y-%m-%d_%H-%M-%S)"
mkdir -p "$BACKUP_DIR"

echo "Starting database backup..."
echo "Backup directory: $BACKUP_DIR"

# Backup roles
echo "Backing up roles..."
npx supabase db dump \
    --db-url "$DB_URL" \
    -f "$BACKUP_DIR/roles.sql" \
    --role-only
echo "Roles done"

# Backup schema
echo "Backing up schema..."
npx supabase db dump \
    --db-url "$DB_URL" \
    -f "$BACKUP_DIR/schema.sql"
echo "Schema done"

# Backup data
echo "Backing up data..."
npx supabase db dump \
    --db-url "$DB_URL" \
    -f "$BACKUP_DIR/data.sql" \
    --use-copy \
    --data-only \
    -x "storage.buckets_vectors" \
    -x "storage.vector_indexes"
echo "Data done"

echo "--------------------------------"
echo "Backup completed successfully!"
echo "Location: $BACKUP_DIR"
```

Save this as `restore.sh` in the same folder:

```bash
#!/usr/bin/env bash

set -euo pipefail

# Read arguments
DB_URL="${1:-}"
BACKUP_DIR="${2:-}"
OPTION="${3:-}"

# Validate arguments
if [[ -z "$DB_URL" || -z "$BACKUP_DIR" ]]; then
    echo "Usage: ./restore.sh <DB_URL> <BACKUP_DIR> [--skip-roles]"
    exit 1
fi

if [[ -n "$OPTION" && "$OPTION" != "--skip-roles" ]]; then
    echo "Error: Unknown option: $OPTION"
    exit 1
fi

# Check PostgreSQL client
if ! command -v psql &> /dev/null; then
    echo "Error: psql is not installed or not in PATH."
    exit 1
fi

# Validate backup directory
if [[ ! -d "$BACKUP_DIR" ]]; then
    echo "Error: Backup directory does not exist: $BACKUP_DIR"
    exit 1
fi

# Check required files
FILES=("schema.sql" "data.sql")

if [[ "$OPTION" != "--skip-roles" ]]; then
    FILES+=("roles.sql")
fi

for file in "${FILES[@]}"; do
    if [[ ! -f "$BACKUP_DIR/$file" ]]; then
        echo "Error: Missing $BACKUP_DIR/$file"
        exit 1
    fi
done

echo "Starting database restore..."
echo "Backup directory: $BACKUP_DIR"

# Build restore command
PSQL_ARGS=(
    --no-psqlrc
    --single-transaction
    --variable ON_ERROR_STOP=1
)

# Restore roles if not skipped
if [[ "$OPTION" != "--skip-roles" ]]; then
    PSQL_ARGS+=(--file "$BACKUP_DIR/roles.sql")
else
    echo "Skipping roles..."
fi

# Restore schema and data
PSQL_ARGS+=(
    --file "$BACKUP_DIR/schema.sql"
    --command "SET session_replication_role = replica"
    --file "$BACKUP_DIR/data.sql"
    --dbname "$DB_URL"
)

# Execute restore
psql "${PSQL_ARGS[@]}"

echo "--------------------------------"
echo "Database restored successfully!"
```

Make both files executable:

```bash
chmod +x backup.sh restore.sh
```

## Step 2: Get the database connection string

In the Supabase dashboard, open the project's **Connect** panel and copy the **Session pooler** connection string. Fill in the database password and percent-encode any special characters in it. [The project-move guide shows the connection-string format and examples](move-project-to-another-region.md#step-2-get-the-old-projects-connection-string).

The commands below use `[SOURCE_CONNECTION_STRING]` and `[TARGET_CONNECTION_STRING]` as placeholders. Keep the single quotes around the real strings so that the shell does not interpret characters such as `&`.

## Step 3: Create a backup

From the folder containing the scripts, run:

```bash
./backup.sh '[SOURCE_CONNECTION_STRING]'
```

If all three dumps succeed, the script prints the new backup folder. For example:

```text
./backups/2026-10-17_14-30-00
```

Check that this folder contains `roles.sql`, `schema.sql` and `data.sql`. Keep a protected copy somewhere other than the source project; a backup stored only beside the system it protects is easy to lose with it.

## Step 4: Prepare the target database

Use a new or otherwise empty target database unless there is a deliberate reason not to. The restore script does not clear existing objects first, so existing tables, roles or data can cause conflicts.

When restoring into a new Supabase project, enable any non-default extensions that the source database uses. Also review Supabase's guide for Vault encryption, custom Auth or Storage schema changes, migration history, Realtime publications and database webhooks before restoring.

## Step 5: Restore a backup

Pass the target connection string and the backup folder to `restore.sh`:

```bash
./restore.sh '[TARGET_CONNECTION_STRING]' './backups/2026-10-17_14-30-00'
```

By default, the script restores roles, then the schema, then the data. To restore only the schema and data, add `--skip-roles`:

```bash
./restore.sh '[TARGET_CONNECTION_STRING]' './backups/2026-10-17_14-30-00' --skip-roles
```

Only skip roles when the target already has every role the schema and grants need, or when the backup deliberately does not include custom roles.

If the restore reports a permission error involving `supabase_admin` or `cli_login_postgres`, follow the matching item under **Troubleshooting notes** in Supabase's backup and restore guide. Because the script uses a single transaction, fix the SQL file and run the restore again.

## Step 6: Verify the restored database

Before relying on the target:

- compare important table row counts with the source
- test the application against the target
- test Auth, Row Level Security policies, database functions and triggers that the application uses
- re-enable and test any Realtime publications, webhooks or project settings that were configured separately
- set new passwords for custom login roles, because role passwords are not included in the dump

## References

- [Supabase: Backup and Restore using the CLI](https://supabase.com/docs/guides/platform/migrating-within-supabase/backup-restore)
- [Supabase CLI: `supabase db dump`](https://supabase.com/docs/reference/cli/supabase-db-dump)
- [Supabase: Database Backups](https://supabase.com/docs/guides/platform/backups)
