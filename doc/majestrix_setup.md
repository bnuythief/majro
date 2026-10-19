# Majestrix server setup

This guide takes a fresh Linux machine (Ubuntu/Debian) to a running Majestrix
login, char and map server.

## Baseline at a glance

| Setting      | Value                                      | Where it lives                    |
|--------------|--------------------------------------------|-----------------------------------|
| Game mode    | Pre-Renewal                                | `src/custom/defines_pre.hpp`      |
| Client       | 2021-11-03 (`PACKETVER 20211103`)          | upstream default, no change       |
| Base/Job EXP | 3x / 3x                                    | `conf/import/battle_conf.txt`     |
| Drops        | 2.5x (cards and MVP rewards stay 1x)       | `conf/import/battle_conf.txt`     |
| Max level    | Stock Pre-Renewal cap (99, job 50/70)      | upstream default, no change       |

Majestrix changes live in the override folders so upstream rAthena merges stay
clean:

- `conf/import/` overrides `conf/*.conf` and `conf/battle/*.conf`.
- `db/import/` overrides the YAML databases in `db/` and `db/pre-re/`.
- `src/custom/` holds compile-time switches and source hooks.
- `npc/custom/` holds custom scripts.

Only the files Majestrix customizes are committed: today that is
`conf/import/battle_conf.txt` and `db/import/README.md`. Every other file in
`conf/import/` and `db/import/` is an empty upstream template that each machine
copies locally (step 2), and git ignores it. To commit a newly customized file,
add a `!/conf/import/<file>` or `!/db/import/<file>` line to `.gitignore`.

Never commit these, because they hold passwords and IP addresses:

- `conf/import/char_conf.txt`
- `conf/import/inter_conf.txt`
- `conf/import/login_conf.txt`
- `conf/import/map_conf.txt`
- `conf/import/web_conf.txt`

## 1. Install packages

```sh
sudo apt update
sudo apt install -y git cmake make gcc g++ zlib1g-dev libpcre2-dev \
    libmariadb-dev libmariadb-dev-compat mariadb-server mariadb-client
```

## 2. Get the source

```sh
git clone https://github.com/bnuythief/majro.git
cd majro
```

Create the local config and database files from the upstream templates. `-n`
never overwrites a file that already exists, so this is safe to re-run after
pulling upstream changes that add new templates:

```sh
cp -n conf/import-tmpl/* conf/import/
cp -n db/import-tmpl/* db/import/
mkdir -p conf/msg_conf/import && cp -n conf/msg_conf/import-tmpl/* conf/msg_conf/import/
```

## 3. Build with CMake

```sh
mkdir -p build && cd build
cmake -G "Unix Makefiles" -DINSTALL_TO_SOURCE=ON -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
make -j"$(nproc)"
make install
cd ..
```

`INSTALL_TO_SOURCE=ON` puts `login-server`, `char-server`, `map-server` and
`web-server` in the repository root next to `conf/`, `db/` and `npc/`, which is
where they expect to run.

Rebuild after any change in `src/`, including switching game mode in
`src/custom/defines_pre.hpp`. Config and database files under `conf/` and
`db/` are read at startup and only need a server restart.

## 4. Set up MariaDB

Pick your own passwords. The examples below use `CHANGE_ME_DB` and
`CHANGE_ME_S1`.

```sh
sudo mariadb <<'SQL'
CREATE DATABASE ragnarok;
CREATE DATABASE ragnarok_log;
CREATE USER 'ragnarok'@'localhost' IDENTIFIED BY 'CHANGE_ME_DB';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, LOCK TABLES
    ON ragnarok.* TO 'ragnarok'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, LOCK TABLES
    ON ragnarok_log.* TO 'ragnarok'@'localhost';
FLUSH PRIVILEGES;
SQL
```

## 5. Import the SQL schema

From the repository root, import the files from `sql-files/` (see
`sql-files/README.md` for the full list):

```sh
mariadb -u ragnarok -p ragnarok     < sql-files/main.sql
mariadb -u ragnarok -p ragnarok     < sql-files/web.sql
mariadb -u ragnarok -p ragnarok     < sql-files/roulette_default_data.sql
mariadb -u ragnarok -p ragnarok_log < sql-files/logs.sql
```

A fresh install does not need `sql-files/upgrades/`; those scripts are only
for databases created by an older rAthena version. The `item_db*.sql` and
`mob_db*.sql` files are also not needed, because `use_sql_db` is off and the
servers read items and monsters from YAML.

`main.sql` creates the inter-server account `s1` / `p1`. Change it before the
server is reachable from the internet:

```sh
mariadb -u ragnarok -p ragnarok -e \
  "UPDATE login SET userid='majestrix_s1', user_pass='CHANGE_ME_S1' WHERE account_id=1;"
```

## 6. Fill in the local config files

`conf/import/inter_conf.txt`:

```
login_server_pw: CHANGE_ME_DB
ipban_db_pw: CHANGE_ME_DB
char_server_pw: CHANGE_ME_DB
map_server_pw: CHANGE_ME_DB
web_server_pw: CHANGE_ME_DB
log_db_pw: CHANGE_ME_DB
log_db_db: ragnarok_log
```

`conf/import/char_conf.txt`:

```
userid: majestrix_s1
passwd: CHANGE_ME_S1
server_name: Majestrix
// Public IP or hostname players connect to (leave out for a local test):
// char_ip: 203.0.113.10
```

`conf/import/map_conf.txt`:

```
userid: majestrix_s1
passwd: CHANGE_ME_S1
// map_ip: 203.0.113.10
```

## 7. Run

```sh
./athena-start start    # starts login, char, map and web servers
./athena-start stop
```

Players connect with a 2021-11-03 client whose `clientinfo.xml` points at the
server address and login port 6900.

## Changing the baseline

- Rates: edit `conf/import/battle_conf.txt`, then restart. Only list values
  that differ from `conf/battle/*.conf`.
- Database entries (items, monsters, EXP tables): add them to the matching file
  in `db/import/`, then restart.
- Game mode: `#define PRERE` in `src/custom/defines_pre.hpp`. Remove it for
  Renewal, then rebuild. Switching modes changes the database files the servers
  load, so treat it as a fresh start rather than a live change.
