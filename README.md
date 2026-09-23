# JtR Cluster

Distributed password-hash cracking system built on top of
[John the Ripper](https://www.openwall.com/john/). Users submit a hash
(or an encrypted file) through a web interface; the system splits the
search space into jobs ("quanta") and distributes them across any number
of CPU/GPU nodes, notifying the user by email when the password is found
or the search space is exhausted.

## Abstract

Password cryptanalysis applies different techniques to find weaknesses in
cryptographic algorithms or implementations used for password protection.
Password cryptanalysis has implications in security assessment,
vulnerability discovery or improved password policies and faces increasing
challenges due to hash complexity and hardware heterogeneity, surpassing
the capabilities of traditional tools such as John the Ripper (JtR). This
study introduces JtR Cluster, a distributed and scalable system that
optimizes password cracking by integrating computational complexity
management through the simultaneous use of CPU and GPU resources, while
balancing the load across different types of hardware. Its centralized
architecture manages the dynamic assignment of hash jobs across
distributed nodes and allows system monitoring via a web interface. A case
study was conducted with a central server and three processing servers
with five graphics cards (GPUs) each. The results obtained demonstrate
that JtR Cluster not only improves efficiency through automatic resource
coordination but also makes brute-force attacks on longer and more robust
passwords feasible, something that would be prohibitively difficult with
individual solutions. This centralized orchestration and dynamic
distribution capability validates JtR Cluster as an efficient and
scalable solution for password cryptanalysis.

## System overview

### Architecture

The system is split into two clearly separate components that only talk to
each other over HTTP: the **server** (web app + REST API + database), which
is the single, central, stateful authority for the whole cluster, and
**`jtr-cluster-node`**, a lightweight, stateless C client that can be
deployed on any number of machines without ever touching the database
directly.

```
┌─────────────────────────────────────────┐          ┌──────────────────────────────────┐
│              SERVER  (www/)             │          │         jtr-cluster-node         │
│        Apache + PHP 8  +  MySQL         │          │             (C daemon)           │
│                                         │          │                                  │
│  [Browser] ──► index.php / procesar.php │          │   CPU worker thread(s)           │
│               (public submission form,  │   HTTP   │   GPU worker thread(s)           │
│                admin dashboard)         │ ◄──────► │   one thread per active device   │
│                                         │  REST    │                                  │
│  REST API ──► /api/node/register.php    │  API     │   loop: register → GET job →     │
│               /api/node/job.php         │          │         run john → POST result   │
│               /api/node/result.php      │          │                                  │
│                                         │          │   (invokes John the Ripper       │
│  MySQL (jtrcluster): nodes, cpus, gpus, │          │    under the hood)               │
│         hashes, jobs, charset           │          │                                  │
└─────────────────────────────────────────┘          └──────────────────────────────────┘
  Single instance — 150.214.150.33                    One instance per cracking machine
  Owns all state: scheduling, quantum                 (including that same host, acting
  generation, device/job bookkeeping,                 as a local node). No direct DB
  web UI                                               access — only talks to the REST API.
```

This separation means the server never reaches out to a node: every
interaction is a node calling the server's REST API
(`register` → `job` → `result`, repeated in a loop, one loop per device).
Scaling out is just a matter of starting `jtr-cluster-node` on another
machine pointed at the same `API_URL` — no server-side configuration
change is needed to add or remove nodes.

### Components

| Component | Description |
|---|---|
| `www/index.php`, `procesar.php` | Public hash/file submission form and its handler (inserts the hash, generates the quanta) |
| `www/admin/` | Admin panel (Dashboard, Nodes, Processing, Hashes cracked, Files cracked) |
| `www/api/node/register.php` | `POST` — node/device registration (with performance benchmark) |
| `www/api/node/job.php` | `GET` — job assignment to a device |
| `www/api/node/result.php` | `POST` — job result reporting |
| `www/crontab/reset_stale_jobs.php` | Periodic task (cron, every 10 min) that recovers stalled `processing` jobs |
| `www/includes/` | Shared PHP libraries (DB connection, formatting helpers, quantum generation, renderers) |
| `www/bin/extraer_hash` | Binary that extracts the hash from an uploaded file (zip/7z/rar/pdf/Office) |
| `jtr-cluster-node/` | C daemon running on each cracking machine: hardware detection/benchmarking, CPU and GPU worker threads, HTTP client towards the API |
| `john/` | John the Ripper binary used by the workers for the actual cracking |

### Workflow

1. The user submits a hash (or file) from `index.php`.
2. `procesar.php` validates the hash, inserts it into the `hashes` table,
   and generates the jobs (`jobs`) that cover its search space according to
   the given charset and password length (`jobs_generate_quantums()` in
   `includes/jobs.php`).
3. Each cracking node requests work (`GET /api/node/job`) for each of its
   free devices; the server assigns the next available `pending` job and
   marks it `processing`.
4. The node runs `john` with the assigned mask and, once finished, reports
   the result (`POST /api/node/result`): password found, or search space
   exhausted with no match.
5. Once the password is found (or the space is exhausted without finding
   it), the remaining pending jobs for that hash are discarded/closed, and
   the user is notified by email.

All jobs and devices operate at a single fixed level (level 6): every job
covers the same search-space size regardless of which device processes it,
giving uniform, predictable behavior across the whole cluster (see v1.4 in
the version history).

### Supported hash and file types

Hash: `MD5`, `SHA1`, `SHA256`, `SHA512`, `bcrypt`.
File: ZIP, 7z, RAR, PDF, Office documents (hash extraction via
`bin/extraer_hash`).

### Database

MySQL/MariaDB engine, database `jtrcluster`. Main tables: `nodes`, `cpus`,
`gpus`, `hashes`, `jobs`, `charset`.

---

## Installation process

### Requirements

**Server (web + DB)**
- Apache 2.4+ with `mod_rewrite`
- PHP 8.x with the `mysqli`, `curl`, `fileinfo`, `mbstring` extensions
- MySQL 8.0+ / MariaDB 10.6+
- John the Ripper (jumbo) at `/jtr-cluster/john/run/john`

**Cracking nodes**
- `gcc`, `make`, `libcurl-dev`
- John the Ripper (same version as the server)
- OpenCL runtime (GPU nodes only)

### 1. Database

```bash
mysql -u root -p << 'SQL'
CREATE DATABASE jtrcluster CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'jtrcluster'@'localhost' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON jtrcluster.* TO 'jtrcluster'@'localhost';
FLUSH PRIVILEGES;
SQL

mysql -u jtrcluster -p jtrcluster < www/schema_db/schema.sql
```

The `charset` table must contain rows 0–4 (digits, lowercase, uppercase,
alphanumeric, full ASCII) — see `schema.sql`.

### 2. Web frontend

1. Publish `www/` as the document root of an Apache VirtualHost (or link it
   from the document root), with `AllowOverride All`.
2. Create the uploads directory and set permissions:
   ```bash
   mkdir -p www/uploads/
   chown usuario:www-data www/uploads/
   ```
3. Set the DB credentials in `www/config.php`
   (`DB_HOST`, `DB_USER`, `DB_PASS`, `DB_NAME`).
4. Compile `bin/extraer_hash` if no compiled binary exists yet:
   ```bash
   gcc -o www/bin/extraer_hash www/bin/extraer_hash.c \
       $(mysql_config --cflags --libs) -Wall -O2
   ```
5. Grant `www-data` group ownership to every PHP file served by Apache
   (mode `640`, group `www-data`).
6. Register the stale-job cleanup task (every 10 minutes):
   ```
   */10 * * * * php /jtr-cluster/www/crontab/reset_stale_jobs.php
   ```

### 3. Cracking node (`jtr-cluster-node`)

1. Install build dependencies: `apt install gcc make libcurl4-openssl-dev`.
2. Configure `/etc/jtr-cluster-node/config.txt`:
   ```
   API_URL=http://<server-ip>/jtrcluster/api/node
   JOHN_PATH=/jtr-cluster/john/run/john
   ```
   **Important**: if the node runs on the same machine as the web server,
   use `API_URL=http://127.0.0.1/...` instead of the public IP — on this
   server, hairpin NAT makes calls to the machine's own external IP fail.
3. Build and install the service:
   ```bash
   cd jtr-cluster-node/
   make install     # compiles, installs the binary and the SysV init.d service
   ```
4. Start the service:
   ```bash
   service jtr-cluster-node start
   ```
5. Verify it's running:
   ```bash
   service jtr-cluster-node status
   tail -f /var/log/jtr-cluster-node/node.log
   ```

### 4. Verification

- Open `http://<server-ip>/jtrcluster/` in a browser.
- Submit a test MD5 hash (e.g. `abc123` →
  `e99a18c428cb38d5f260853678922e03`).
- Check `/jtrcluster/admin/` (*Processing* tab) that jobs are being
  assigned to the active nodes.
- The node log should show the initial registration and job requests.

### Notes

- Run a single `jtr-cluster-node` instance per machine — multiple
  instances on the same host cause race conditions when requesting jobs.
- Change the default database credentials before deploying to production.

---

## Version history

**Version 1.0 (01/07/2025)** Authors: Juan Rafael & Julio Gómez López (Final Degree Project)
- Initial release of JtR Cluster.

---

**Version 1.1 (12/04/2026)** Author: Julio Gómez López & Raúl Baños Navarro
- Entire codebase translated to English.
- Web: created the `/admin` section with extended management features.
- Processing node: added support for NVIDIA GPUs (previously only AMD via rusticl).

---

**Version 1.2 (12/04/2026)** Author: Julio Gómez López & Raúl Baños Navarro
- Web: admin panel reorganized into tabs — Dashboard, Nodes (with color-coded status: green=online, red=offline, blue=processing), Processing, Hashes cracked, Files cracked.
- Web: fixed CPU performance info display in admin panel.
- Web: fixed bug in job generation to correctly cover the full search space for any charset.
- Processing node: unified `gpu_report.c`, `cpu_report.c` and `processing.c` into a single module with shared libraries. Added configuration file (`SERVER_IP`, `JOHN_PATH`). Full activity logging implemented.

---

**Version 1.3 (13/04/2026)** Author: Julio Gómez López & Raúl Baños Navarro
- General: improved database schema for better host and device registration.
- Web: migrated to shared library structure (`/includes/`). Created common stylesheets. Added automatic refresh every 5 seconds on real-time pages (Processing, Dashboard, Nodes). Limited active job display to 50 entries for performance.
- Web: if a submitted hash has already been cracked, the result is returned immediately without reprocessing. Improved display of Hashes cracked and Files cracked sections. Created `hash_report.php` with a full job execution report per hash.
- Processing node: improved safe startup sequence — kills any running `process_node` or `john` instances, cleans up all temporary files from previous runs. Eliminated unnecessary temporary files. Improved thread-based parallelism.

---

**Version 1.4 (23/09/2026)** Author: Julio Gómez López & Raúl Baños Navarro
- General: migrated the coordination server — previously a custom C server listening on port 10000 — to a REST API.
- Web (admin): removed inline CSS styles and per-job level badges from the Nodes and Processing views in favour of the shared stylesheet. Fixed a decimal/thousands separator bug in `admin/hash_report.php`.
- Web: `crontab/reset_stale_jobs.php` reworked to use per-device-type stale timeouts.
- Processing node: improved overall system performance by reducing pauses between communications and optimizing quantum processing.
