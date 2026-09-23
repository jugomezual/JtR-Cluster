# Installing JtR Cluster

This document explains how to deploy JtR Cluster from scratch using the
packages in `SRC/`:

| File | Contents |
|---|---|
| `SRC/www.tgz` | Full web app: public part (`index.php`, `procesar.php`, `admin/`, styles and assets) + server/API (`api/node/`, `includes/`, `crontab/`, `bin/`, `config.php`) |
| `SRC/jtrcluster.sql` | Database structure (no data) |
| `SRC/jcluster-node.tgz` | Processing node source code (C daemon) |

## Requirements

**Server (web + DB + API-Rest)**
- Apache 2.4+ with `mod_rewrite`
- PHP 8.x with the `mysqli`, `curl`, `fileinfo`, `mbstring` extensions
- MySQL 8.0+ / MariaDB 10.6+
- John the Ripper (jumbo), e.g. at `/jtr-cluster/john/run/john`

**jtrcluster-node** (any machine that will contribute CPU/GPU)
- `gcc`, `make`, `libcurl4-openssl-dev`
- John the Ripper (same version as the server)
- OpenCL runtime, only if the node has a GPU

## 1. Database

Create the database and the user:

```bash
mysql -u root -p << 'SQL'
CREATE DATABASE jtrcluster CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'jtrcluster'@'localhost' IDENTIFIED BY '<choose-a-password>';
GRANT ALL PRIVILEGES ON jtrcluster.* TO 'jtrcluster'@'localhost';
FLUSH PRIVILEGES;
SQL
```

Import the structure (no data):

```bash
mysql -u jtrcluster -p jtrcluster < SRC/jtrcluster.sql
```

Insert the charsets the system uses to generate jobs (required — the
application assumes ids 0–4 exist):

```bash
mysql -u jtrcluster -p jtrcluster << 'SQL'
INSERT INTO charset (id, description) VALUES
  (0, '0123456789'),
  (1, 'a-z'),
  (2, 'A-Z'),
  (3, 'a-zA-Z0-9'),
  (4, 'Full printable ASCII');
SQL
```

## 2. Web (`www.tgz`)

```bash
mkdir -p /jtr-cluster/www
tar xzf SRC/www.tgz -C /jtr-cluster/www
```

This installs both the public-facing part of the site (`index.php`,
`procesar.php`, `estilo.css`, `admin/`, `assets/`, `images/`) and the
server/API (`api/`, `includes/`, `crontab/`, `bin/`, `config.php`) into a
single `www/` tree — the site and the API share the same deployment,
codebase, and `config.php`.

1. Edit `www/config.php` with the real database credentials created in
   step 1 (the included file ships with a blank/placeholder `DB_PASS`,
   **not the production password**):

   ```php
   define('DB_HOST', 'localhost');
   define('DB_USER', 'jtrcluster');
   define('DB_PASS', '<the-password-you-chose>');
   define('DB_NAME', 'jtrcluster');
   ```

2. Create the uploads directory and set permissions:

   ```bash
   mkdir -p /jtr-cluster/www/uploads/
   chown usuario:www-data /jtr-cluster/www/uploads/
   ```

3. Grant `www-data` group ownership to everything deployed, so Apache can
   read it (mode `640` on files, `750` on directories):

   ```bash
   chown -R usuario:www-data /jtr-cluster/www
   find /jtr-cluster/www -type f -exec chmod 640 {} \;
   find /jtr-cluster/www -type d -exec chmod 750 {} \;
   ```

4. `bin/extraer_hash` is already compiled inside the tgz. If you need to
   rebuild it (e.g. different architecture):

   ```bash
   gcc -o www/bin/extraer_hash www/bin/extraer_hash.c \
       $(mysql_config --cflags --libs) -Wall -O2
   ```

5. Publish `www/` as the document root of an Apache VirtualHost (or as an
   alias of an existing vhost), with `AllowOverride All` and `mod_rewrite`
   enabled:

   ```bash
   a2enmod rewrite
   systemctl restart apache2
   ```

6. Register the task that recovers stalled jobs (every 10 minutes):

   ```bash
   crontab -e
   # Add the line:
   */10 * * * * php /jtr-cluster/www/crontab/reset_stale_jobs.php
   ```

## 3. Processing node (`jcluster-node.tgz`)

Repeat these steps on **every machine** that will contribute CPU or GPU to
the cluster (including, if applicable, the web server itself).

```bash
mkdir -p /jtr-cluster/jtr-cluster-node
tar xzf SRC/jcluster-node.tgz -C /jtr-cluster/jtr-cluster-node
```

1. Install build dependencies:

   ```bash
   apt install gcc make libcurl4-openssl-dev
   ```

2. Edit `config.txt` (included in the package as a template) with the
   server's IP:

   ```
   API_URL=http://<server-ip>/jtrcluster/api/node
   JOHN_PATH=/jtr-cluster/john/run/john
   ```

   **Important**: if this node runs on the same machine as the web server,
   use `API_URL=http://127.0.0.1/jtrcluster/api/node` instead of the public
   IP — in deployments behind NAT, a machine calling its own external IP
   (hairpin NAT) can fail.

3. Build and install the service:

   ```bash
   cd /jtr-cluster/jtr-cluster-node
   make install     # compiles, installs the binary and the SysV init.d service
   ```

4. Start the service:

   ```bash
   service jtr-cluster-node start
   ```

5. Verify it's running and registering:

   ```bash
   service jtr-cluster-node status
   tail -f /var/log/jtr-cluster-node/node.log
   ```

## 4. Final verification

1. Open `http://<server-ip>/jtrcluster/` in a browser.
2. Submit a test MD5 hash (e.g. `abc123` →
   `e99a18c428cb38d5f260853678922e03`).
3. Go to `/jtrcluster/admin/` (*Nodes* tab) and check that the
   newly-installed node appears registered with its devices.
4. In the *Processing* tab, jobs should start being assigned to the node
   shortly after submitting the test hash.

## Notes

- Run a single `jtr-cluster-node` instance per machine — multiple
  instances on the same host cause race conditions when requesting jobs.
- Always change the default database password before deploying to
  production; the `config.php` and `config.txt` included in `SRC/` are
  templates with example values, not real credentials.
- All jobs and devices operate at a single fixed level (level 6) as of
  version 1.4 — no extra configuration per GPU/CPU type is needed for job
  assignment to work correctly.
