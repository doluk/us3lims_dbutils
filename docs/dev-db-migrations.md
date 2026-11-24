# Dev/staging database: schema upgrade and rollback

Scope: a development or staging LIMS host (e.g. astfvm) whose instance databases
(`uslims3_*`) carry local schema changes not on any branch of `us3_sql`. Production
hosts keep the procedure in the top-level README (dbupgrade stage1/stage2 +
`uslims_upgrade.php`). Nothing here changes that procedure.

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

## 0. Prerequisites (once)

```bash
# both are single static binaries
curl -fsSL -o /usr/local/bin/dbmate https://github.com/amacneil/dbmate/releases/latest/download/dbmate-linux-amd64 && chmod +x /usr/local/bin/dbmate
```

skeema: install the Linux binary from its release page (`skeema version` to check).
Community edition covers tables, procedures and functions; triggers and views need
Premium (LIMS instances use neither as far as `us3.sql` goes; verify with the diff in 1.3).

Credentials: reuse `dbupgrade/my.cnf` (root, mode 600). skeema reads `~/.my.cnf`-style
files; dbmate needs a URL, built at runtime so the password never lands in a file
under git:

```bash
export DBPW="$(sed -n 's/^password=//p' ~us3/lims/database/utils/dbupgrade/my.cnf | head -1)"
```

Snapshot/restore helpers used below (logical, includes routines; adequate for dev sizes):

```bash
snap() { mysqldump --defaults-file=$HOME/.my.cnf --single-transaction --routines --triggers --events "$1" | gzip > ~/dbsnap/"$1"-$(date +%Y%m%d%H%M%S).sql.gz; }
restore() { mysql -e "DROP DATABASE IF EXISTS \`$1\`; CREATE DATABASE \`$1\`" && zcat "$2" | mysql "$1"; }
```

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
   skeema init -h localhost -d ~/skeema-target --schema uslimsref
   ```
   (`uslims_db_schemas.php --compare` builds the same `uslimsref`.)
3. **Inventory the drift** (read-only):
   ```bash
   skeema init -h localhost -d ~/skeema-dev --schema uslims3_<inst>
   git diff --no-index ~/skeema-target ~/skeema-dev   # ignore the .skeema files
   ```
   Each hunk is one of: (a) on main, missing locally → take; (b) local experiment →
   keep, move to a dbmate migration in step 2, or drop; (c) semantic change needing
   data movement → hand-written SQL.
4. **Data-carrying changes first, by hand.** Example:
   ```sql
   ALTER TABLE people ADD COLUMN gmpReviewerRole ENUM('NONE','REVIEWER','APPROVER') NOT NULL DEFAULT 'NONE';
   UPDATE people SET gmpReviewerRole = CASE gmpReviewer WHEN 1 THEN 'REVIEWER' WHEN 2 THEN 'APPROVER' ELSE 'NONE' END;
   ```
   (enum values and mapping: take from `us3.sql` and the code, not from this example.)
   skeema would emit `DROP COLUMN gmpReviewer` + `ADD COLUMN gmpReviewerRole`; it refuses
   such unsafe changes without `--allow-unsafe`, which is the safeguard here.
5. **Converge.** Point the target directory at the instance and apply:
   ```bash
   skeema diff  -d ~/skeema-target  --schema uslims3_<inst>   # review the DDL
   skeema push  -d ~/skeema-target  --schema uslims3_<inst> --allow-unsafe
   skeema diff  -d ~/skeema-target  --schema uslims3_<inst>   # exit 0 = converged
   ```
   (`--schema` override or a per-instance `.skeema`; check the option name against your
   skeema version.) Repeat per instance.
6. **Cross-check with the repo's own validators:** `php uslims_upgrade.php --check`
   (its `instance_schema` step must be `ok`) and `php uslims_db_schemas.php --compare --show-diffs`.

Local changes classified (b)-keep are now absent from the database; they come back as
dbmate migrations in 2.2, which makes them reviewable and reversible.

## 2. Incremental changes with dbmate

### 2.1 Layout and baseline

Migrations belong next to the schema they change, i.e. in `us3_sql`:

```
us3_sql/
  us3.sql ...                    # unchanged, remains the fresh-install path
  migrations/
    20261124000000_baseline.sql  # = schema after step 1 (comment-only body, see below)
    20261201093000_people_gmp_reviewer_role.sql
```

The baseline is not executed on existing instances; it only marks the starting point.
Body: `-- migrate:up` / `-- migrate:down` with a comment "fresh installs load us3.sql".
Register it in each converged instance:

```bash
for db in $(mysql -NBe "SHOW DATABASES LIKE 'uslims3\\_%'" | grep -v uslims3_global); do
  mysql "$db" -e "CREATE TABLE IF NOT EXISTS schema_migrations (version VARCHAR(128) PRIMARY KEY);
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

### 2.3 Dev loop (one instance)

```bash
export DATABASE_URL="mysql://root:${DBPW}@127.0.0.1:3306/uslims3_<inst>"   # URL-encode DBPW if it has @ : / %
export DBMATE_MIGRATIONS_DIR=./migrations                                   # dbmate default is ./db/migrations
cd ~us3/lims/database/sql
dbmate --no-dump-schema new people_gmp_reviewer_role   # creates migrations/<ts>_...sql
snap uslims3_<inst>
dbmate --no-dump-schema up          # apply pending
dbmate --no-dump-schema status
dbmate --no-dump-schema rollback    # undo last; edit; up again
```

If `down` is wrong or the database is in a half-applied state: `restore uslims3_<inst> <snapshot>`.

All instances: loop over `uslims3_%` as in 2.1 with `--url` per database.

### 2.4 Keep `us3.sql` and the migrations consistent

Each `us3_sql` change edits `us3.sql` **and** adds a migration. Check (CI or before
merge): build db A from `us3.sql`, db B from `us3.sql@baseline` + `dbmate up`, then
`skeema diff` (or `uslims_db_schemas.php`) between them must be empty. This is the
remaining role of skeema: drift detection.

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
- dbmate's `--no-dump-schema` avoids writing `db/schema.sql` (needs `mysqldump` in PATH
  otherwise and is redundant with `us3.sql`).

## Open points

- MariaDB version on the dev host vs. skeema's supported range.
- Whether `us3_sql` accepts a `migrations/` directory, or migrations live in a dev-only repo.
- Enum values/mapping for `gmpReviewerRole` (example above is illustrative).
