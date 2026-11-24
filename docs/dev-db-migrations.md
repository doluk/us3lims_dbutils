# Dev/staging database: schema upgrade and rollback

Scope: a development or staging LIMS host (e.g. astfvm: RHEL 8, MariaDB 10.3.39) whose
instance databases (`uslims3_*`) carry local schema changes not on any branch of
`us3_sql`. Production hosts keep the procedure in the top-level README (dbupgrade
stage1/stage2 + `uslims_upgrade.php`). Nothing here changes that procedure, `us3_sql`
or this repository's tools.

## Why the existing tools are not enough on a dev host

| Tool | What it does | Limitation on a dev host |
|---|---|---|
| `dbupgrade/stage1+stage2` | dump data, drop, recreate from `us3.sql`, reimport | stops when a dump has a table/column `us3.sql` lacks (i.e. your local changes); all-or-nothing; no per-change rollback |
| `uslims_db_schemas.php --compare` | builds `uslimsref` from `us3.sql`, diffs schema dumps | reports only; no generated DDL |
| `migrations/` + `migrations.php` | idempotent, state-checking host/instance steps, no ledger | release-granular, PHP, no `down` |
| `uslims_db_stash.php` | copies a db to `stash_<date>_<name>`, sets metadata `pending`, offers to drop | meant for retiring an instance; `mysqldump` without `--routines`; not a snapshot tool |

Hence two tools, each used for what it does well:

- **skeema** (declarative, state diff): one-time convergence of a drifted database onto a
  known target, and afterwards as a drift detector.
- **dbmate** (imperative, ordered `up`/`down` files with a ledger): every change after
  the baseline, including data-carrying changes skeema cannot express
  (`people.gmpReviewer int` → `people.gmpReviewerRole enum`).

## 0. Installation on RHEL 8 (once)

Both tools are statically linked Go binaries: no runtime dependencies, independent of
RHEL 8's glibc. Run as root (`sudo -i`). Versions are pinned; check the release pages
before upgrading them.

### 0.1 skeema (v1.14.2)

MariaDB 10.3 is supported by skeema v1 but deprecated since v1.14.0. Stay on v1.x
for as long as the host runs 10.3; skeema v2 may drop it.

RPM (installs to `/usr/bin`):

```bash
dnf install -y https://github.com/skeema/skeema/releases/download/v1.14.2/skeema_amd64.rpm
```

Or the tarball, if `dnf` cannot reach GitHub directly:

```bash
cd /tmp && curl -fsSLO https://github.com/skeema/skeema/releases/download/v1.14.2/skeema_1.14.2_linux_amd64.tar.gz
```

```bash
tar -xzf /tmp/skeema_1.14.2_linux_amd64.tar.gz -C /tmp skeema && install -m 0755 /tmp/skeema /usr/local/bin/skeema
```

```bash
skeema version
```

Community edition manages tables, procedures and functions. Views, triggers and events
are Premium-only; check the diff in 1.3 for any of these. `us3.sql` is not expected to
define them.

### 0.2 dbmate (v2.38.2)

No RPM is published; install the binary:

```bash
curl -fsSL -o /usr/local/bin/dbmate https://github.com/amacneil/dbmate/releases/download/v2.38.2/dbmate-linux-amd64 && chmod 0755 /usr/local/bin/dbmate && restorecon -v /usr/local/bin/dbmate
```

```bash
dbmate --version
```

### 0.3 Notes for RHEL 8

- `/usr/local/bin` is in root's login-shell `PATH` (`sudo -i`) but not in sudo's
  `secure_path`: `sudo dbmate ...` from a user shell fails. Use `sudo -i`, the full
  path, or the RPM for skeema.
- `mysqldump`/`mysql` come from the installed `mariadb` package; nothing else is needed.
- Behind an HTTP proxy, export `https_proxy` before `curl`/`dnf`.

### 0.4 Credentials and helpers

Reuse `dbupgrade/my.cnf` (root, mode 600). skeema reads `~/.my.cnf`; give it one copy:

```bash
install -m 600 ~us3/lims/database/utils/dbupgrade/my.cnf ~/.my.cnf
```

dbmate needs a URL; build it at runtime so the password never lands in a file under git:

```bash
export DBPW="$(sed -n 's/^password=//p' ~/.my.cnf | head -1)"
```

Snapshot/restore helpers used below (logical, includes routines; adequate for dev sizes):

```bash
mkdir -p ~/dbsnap
snap() { mysqldump --single-transaction --routines --triggers --events "$1" | gzip > ~/dbsnap/"$1"-$(date +%Y%m%d%H%M%S).sql.gz; }
restore() { mysql -e "DROP DATABASE IF EXISTS \`$1\`; CREATE DATABASE \`$1\`" && zcat "$2" | mysql "$1"; }
```

## Dev-only repository

Migrations and the skeema target live in a separate, dev-only git repository, not in
`us3_sql` or here. Suggested layout (cloned to e.g. `~/us3-dev-db`):

```
us3-dev-db/
  .env.example                     # DBMATE_MIGRATIONS_DIR=./migrations, DBMATE_NO_DUMP_SCHEMA=true
  skeema-target/                   # generated in 1.2, regenerated per us3_sql update
  migrations/
    20261124000000_baseline.sql
    20261201093000_people_gmp_reviewer_role.sql
  bin/each-instance                # loop helper, 2.3
```

The repo records which `us3_sql` commit the baseline corresponds to (in the baseline
file's comment).

## 1. One-time baseline with skeema

Goal: every dev instance on exactly the schema of the `us3_sql` branch you develop
against, plus a reviewed record of every local difference.

1. **Backup.** Full binary backup (`uslims_db_binary_backup.php`) and `snap` per instance.
2. **Target schema directory.** `us3.sql` is a load script (DROPs, INSERTs, `source`), not
   skeema's one-CREATE-per-file layout, so materialise it and pull from a database:
   ```bash
   cd ~us3/lims/database/sql && git checkout <target branch> && git pull --ff-only
   mysql -e 'DROP DATABASE IF EXISTS uslimsref; CREATE DATABASE uslimsref'
   mysql uslimsref < us3.sql            # loads us3_procedures.sql, us3_dispatch.sql
   skeema init -h localhost -d ~/us3-dev-db/skeema-target --schema uslimsref
   ```
   (`uslims_db_schemas.php --compare` builds the same `uslimsref`.)
3. **Inventory the drift** (read-only):
   ```bash
   skeema init -h localhost -d ~/skeema-dev --schema uslims3_<inst>
   git diff --no-index ~/us3-dev-db/skeema-target ~/skeema-dev   # ignore the .skeema files
   ```
   Each hunk is one of: (a) on main, missing locally → take; (b) local experiment →
   keep, move to a dbmate migration in step 2, or drop; (c) semantic change needing
   data movement → hand-written SQL.
4. **Data-carrying changes first, by hand.** For `gmpReviewer` → `gmpReviewerRole`:
   ```sql
   ALTER TABLE people ADD COLUMN gmpReviewerRole ENUM('NONE','REVIEWER','APPROVER') NOT NULL DEFAULT 'NONE';
   UPDATE people SET gmpReviewerRole = CASE gmpReviewer WHEN 1 THEN 'REVIEWER' WHEN 2 THEN 'APPROVER' ELSE 'NONE' END;
   ```
   Mapping: 0 = NONE, 1 = REVIEWER, 2 = APPROVER. Any other value would silently become
   NONE; check first that none occurs (must return no rows):
   `SELECT gmpReviewer, COUNT(*) FROM people WHERE gmpReviewer NOT IN (0,1,2) OR gmpReviewer IS NULL GROUP BY 1`.
   skeema would emit `DROP COLUMN gmpReviewer` + `ADD COLUMN gmpReviewerRole`; it refuses
   such unsafe changes without `--allow-unsafe`, which is the safeguard here.
5. **Converge.** Point the target directory at the instance and apply:
   ```bash
   cd ~/us3-dev-db/skeema-target
   skeema diff --schema uslims3_<inst>                  # review the DDL
   skeema push --schema uslims3_<inst> --allow-unsafe
   skeema diff --schema uslims3_<inst>                  # exit 0 = converged
   ```
   Repeat per instance.
6. **Cross-check with the repo's own validators:** `php uslims_upgrade.php --check`
   (its `instance_schema` step must be `ok`) and `php uslims_db_schemas.php --compare --show-diffs`.

Local changes classified (b)-keep are now absent from the database; they come back as
dbmate migrations in 2.2, which makes them reviewable and reversible.

## 2. Incremental changes with dbmate

### 2.1 Baseline

The baseline file is not executed on existing instances; it marks the starting point:

```sql
-- us3_sql <commit>; fresh instances load us3.sql from that commit, then dbmate up
-- migrate:up
-- migrate:down
```

Register it in each converged instance (table definition as dbmate creates it):

```bash
for db in $(mysql -NBe "SHOW DATABASES LIKE 'uslims3\\_%'" | grep -v '^uslims3_global$'); do
  mysql "$db" -e "CREATE TABLE IF NOT EXISTS schema_migrations (version VARCHAR(255) PRIMARY KEY);
                  INSERT IGNORE INTO schema_migrations VALUES ('20261124000000');"
done
```

### 2.2 Migration file format

```sql
-- migrate:up
ALTER TABLE people ADD COLUMN gmpReviewerRole ENUM('NONE','REVIEWER','APPROVER') NOT NULL DEFAULT 'NONE';
UPDATE people SET gmpReviewerRole = CASE gmpReviewer WHEN 1 THEN 'REVIEWER' WHEN 2 THEN 'APPROVER' ELSE 'NONE' END;
ALTER TABLE people DROP COLUMN gmpReviewer;

-- migrate:down
ALTER TABLE people ADD COLUMN gmpReviewer INT NOT NULL DEFAULT 0;
UPDATE people SET gmpReviewer = CASE gmpReviewerRole WHEN 'REVIEWER' THEN 1 WHEN 'APPROVER' THEN 2 ELSE 0 END;
ALTER TABLE people DROP COLUMN gmpReviewerRole;
```

Rules:
- MariaDB DDL commits implicitly; dbmate's transaction does not protect a half-applied
  file. Keep one logical change per file, and `snap` before `up`.
- Write `down` so that `up; down; up` is lossless on the data you care about.
- Stored procedures stay out of dbmate: `us3_procedures.sql` is reloaded as a whole
  (`uslims_upgrade.php` step 9 tracks it by checksum). Edit it there and reload.
- No `DELIMITER` lines (client-only syntax).
- MariaDB 10.3 has no `RENAME COLUMN` (10.5+); use `CHANGE COLUMN old new <full definition>`.

### 2.3 Dev loop

```bash
cd ~/us3-dev-db
export DBMATE_MIGRATIONS_DIR=./migrations DBMATE_NO_DUMP_SCHEMA=true
export DATABASE_URL="mysql://root:${DBPW}@127.0.0.1:3306/uslims3_<inst>"   # URL-encode DBPW if it has @ : / %
dbmate new people_gmp_reviewer_role   # creates migrations/<ts>_people_gmp_reviewer_role.sql
snap uslims3_<inst>
dbmate up          # apply pending
dbmate status
dbmate rollback    # undo last; edit; up again
```

Via the local socket instead of TCP: `mysql://root:${DBPW}@/uslims3_<inst>?socket=/var/lib/mysql/mysql.sock`.

If `down` is wrong or the database is in a half-applied state: `restore uslims3_<inst> <snapshot>`.

All instances (`bin/each-instance up`, `bin/each-instance status`):

```bash
#!/bin/bash
set -euo pipefail
for db in $(mysql -NBe "SHOW DATABASES LIKE 'uslims3\\_%'" | grep -v '^uslims3_global$'); do
  echo "== $db"; dbmate --url "mysql://root:${DBPW}@127.0.0.1:3306/$db" "$@"
done
```

### 2.4 After a `us3_sql` update

When `us3_sql` main gains schema changes you need on the dev host, either write them as
a dbmate migration in the dev repo (preferred; keeps rollback), or repeat section 1
against the new commit and record a new baseline. Check consistency: build db A from the
new `us3.sql`, db B from the baseline commit's `us3.sql` + `dbmate up`, then
`skeema diff` between them shows exactly your dev-only changes and nothing else.

## 3. Rollback summary

| Situation | Action |
|---|---|
| last migration wrong, `down` correct | `dbmate rollback` |
| `down` wrong / partial apply | `restore <db> <snap>` |
| baseline (step 1) went wrong | `restore` per instance, or the binary backup |

## 4. Known interactions

- `schema_migrations` is a table `us3.sql` does not define. `uslims_upgrade.php`'s
  `instance_schema` check only looks for missing objects (unaffected).
  `dbupgrade/stage2` reimports data-only dumps after recreating from `us3.sql`; the
  ledger is then lost or rejected. On a dbmate-managed dev host, do not use stage1/stage2;
  if needed, re-register the ledger afterwards as in 2.1.
- `DBMATE_NO_DUMP_SCHEMA=true` avoids writing `db/schema.sql` (redundant with `us3.sql`).
